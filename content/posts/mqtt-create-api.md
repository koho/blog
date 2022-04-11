---
title: "使用 MQTT 实现 API 接口"
date: 2022-03-15T09:28:08+08:00
archives: 
    - 2022
tags:
    - Golang
image: /images/mqtt.png
leading: false
draft: false
---

MQTT (Message Queuing Telemetry Transport Protocol) 消息队列遥测传输协议，是一种轻量级的发布/订阅模式的消息传输协议，运行在 TCP 协议栈之上，为其提供有序、可靠、双向连接的网络连接保证。
> MQTT最大优点在于，可以以极少的代码和有限的带宽，为连接远程设备提供实时可靠的消息服务。做为一种低开销、低带宽占用的即时通讯协议，使其在物联网、小型设备、移动应用等方面有较广泛的应用。

最近项目上需要用到 MQTT 采集设备数据，然后项目中的某个应用又需要向外提供接口，项目中的其他应用都需要用到 MQTT，因此想到利用 MQTT 完成多个应用之间的通讯。

API 接口一般是请求/响应模型，MQTT v5 也开始支持请求响应的模式了，具体的流程如下：

{{<cimg "/images/mqtt-req-resp.png" >}}

1. 请求方在发布消息时包含 **响应主题(Response Topic)** 属性，比如使用客户端标识符 (Client ID)。
2. 请求方在发布消息时包含 **对比数据(Correlation Data)** 属性，比如使用当前时间戳。请求可以是异步的，如果同时发送多个请求，那请求方需要一个字段来区分响应消息属于哪个请求。
3. 响应方收到消息后，进行业务处理，处理完成后向该消息的 **响应主题** 属性指定的主题发布响应消息，同时该响应消息的 **对比数据** 属性与请求消息的一致。
4. 请求方收到响应消息后，根据 **对比数据** 属性把消息转发给对应的请求。

## 实现
本文使用 [github.com/eclipse/paho.golang](https://github.com/eclipse/paho.golang) 这个包举例。

### 服务端
服务端的工作和 Web 框架类似，监听多个 Topic，为每个 Topic 指定处理函数。当一个请求到来时，创建新的协程来单独处理业务。
```go
type Engine struct {
	Subscriptions map[string]paho.SubscribeOptions
	router        *paho.StandardRouter
	cm            *autopaho.ConnectionManager
}

func New() *Engine {
	return &Engine{
		router:        paho.NewStandardRouter(),
		Subscriptions: make(map[string]paho.SubscribeOptions),
	}
}

func (engine *Engine) Route(topic string, handler func(string, []byte) []byte) {
	engine.Subscriptions[topic] = paho.SubscribeOptions{QoS: 0}
	engine.router.RegisterHandler(topic, func(publish *paho.Publish) {
	    // 每个请求单独一个协程处理
		go func() {
			defer func() {
				if err := recover(); err != nil {
					apiLog.Error(err)
				}
			}()
			ret := handler(publish.Topic, publish.Payload)
			if publish.Properties.ResponseTopic != "" && len(ret) > 0 {
				engine.cm.Publish(context.Background(), &paho.Publish{
					QoS:        0,
					Retain:     false,
					Topic:      publish.Properties.ResponseTopic,
					Payload:    h.ret,
					Properties: &paho.PublishProperties{CorrelationData: publish.Properties.CorrelationData},
				})
			}
		}()
	})
}

func (engine *Engine) Run(mqttUrl, mqttUser, mqttPasswd string) {
	var err error
	u, err := url.Parse(mqttUrl)
	if err != nil {
		log.Fatal(err)
	}
	cc := autopaho.ClientConfig{
		BrokerUrls: []*url.URL{u},
		KeepAlive:  30,
		OnConnectionUp: func(manager *autopaho.ConnectionManager, connack *paho.Connack) {
			if _, err := manager.Subscribe(context.Background(), &paho.Subscribe{Subscriptions: engine.Subscriptions}); err != nil {
				engine.cm.Disconnect(context.Background())
			}
		},
		OnConnectError: func(err error) {
			log.Println(err)
		},
		ClientConfig: paho.ClientConfig{
			Router: engine.router,
		},
	}
	cc.SetUsernamePassword(mqttUser, []byte(mqttPasswd))
	engine.cm, err = autopaho.NewConnection(context.Background(), cc)
	if err != nil {
		log.Fatal(err)
	}
}
```
使用上就像常见的 Web 框架一样，指定 Topic 和处理函数：
```go
api := New()
api.Route("devices/+", func(topic string, payload []byte) []byte {
    log.Println(string(payload))
    return []byte("OK")
})
api.Run("mqtt://127.0.0.1:1883", "user", "pass")
```

### 客户端
需要支持异步请求，也就是说多个请求可以同时发出，且它们都共用一条客户端连接。

关于如何区分响应消息，有两种方法：
- 使用 **对比数据** 属性
- 使用不同的 **响应主题** 属性

这里更推荐使用第一种方法。

```go
type Handler struct {
	sync.Mutex
	c          *paho.Client
	respTopic  string
	correlData map[string]chan *paho.Publish
}

func NewHandler(c *paho.Client) (*Handler, error) {
	h := &Handler{
		c:          c,
		respTopic:  fmt.Sprintf("%s/responses", c.ClientID),
		correlData: make(map[string]chan *paho.Publish),
	}
	c.Router.RegisterHandler(h.respTopic, h.responseHandler)
	_, err := c.Subscribe(context.Background(), &paho.Subscribe{
		Subscriptions: map[string]paho.SubscribeOptions{
			h.respTopic: {QoS: 1},
		},
	})
	if err != nil {
		return nil, err
	}
	return h, nil
}

func (h *Handler) addCorrelID(cID string, r chan *paho.Publish) {
	h.Lock()
	defer h.Unlock()
	h.correlData[cID] = r
}

func (h *Handler) getCorrelIDChan(cID string) chan *paho.Publish {
	h.Lock()
	defer h.Unlock()
	rChan, ok := h.correlData[cID]
	if ok {
		delete(h.correlData, cID)
	}
	return rChan
}

func (h *Handler) Request(pb *paho.Publish) (*paho.Publish, error) {
	cID := fmt.Sprintf("%d", time.Now().UnixNano())
	rChan := make(chan *paho.Publish, 1)

	h.addCorrelID(cID, rChan)

	if pb.Properties == nil {
		pb.Properties = &paho.PublishProperties{}
	}

	pb.Properties.CorrelationData = []byte(cID)
	pb.Properties.ResponseTopic = h.respTopic
	pb.Retain = false

	_, err := h.c.Publish(context.Background(), pb)
	if err != nil {
		return nil, err
	}
	for {
		select {
		case resp := <-rChan:
			return resp, nil
		case <-time.After(10 * time.Second):
			h.getCorrelIDChan(cID)
			return nil, errors.New("timeout error")
		}
	}
}

func (h *Handler) responseHandler(pb *paho.Publish) {
	if pb.Properties == nil || pb.Properties.CorrelationData == nil {
		return
	}
	rChan := h.getCorrelIDChan(string(pb.Properties.CorrelationData))
	if rChan == nil {
		return
	}
	rChan <- pb
}
```
这样就如 HTTP 接口一样，实现请求/响应模式：
```go
h, err := NewHandler(client)
if err != nil {
    panic(err)
}
response, err := h.Request(&paho.Publish{
	Topic:   "api/getData",
	Payload: []byte("dev1"),
})
if err != nil {
    panic(err)
}
log.Println(string(response.Payload))
```

## 参考

[1] [请求响应 - MQTT 5.0 新特性](https://www.emqx.com/zh/blog/mqtt5-request-response)

[2] [Eclipse Paho for golang](https://github.com/eclipse/paho.golang)
