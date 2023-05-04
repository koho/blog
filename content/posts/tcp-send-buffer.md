---
title: "获取 TCP 发送缓冲区信息"
date: 2023-05-04T16:37:47+08:00
archives: 
    - 2023
tags:
    - 网络
    - TCP
image: /images/tcp-send-window.png
leading: false
draft: false
---

在大多数的网络编程情况下，是不需要特意关心 TCP 的缓冲区信息的。但在一些工业应用场景，如 RS485 转网口的数据采集，半双工的 485 不能同时进行读和写，这就要求 TCP 确认发送完一个指令后，才能发第二条。因为是总线网络，同时发送多条指令，对端回复数据将出现错乱。

当应用程序往套接字写入数据时，实际上只是写入了内核的发送缓冲区，接收方什么时候能收到报文是个未知数。

因此在某些需要同步状态的地方，发送方最好能确认对方收到报文后再做下一步动作。

## Linux

Linux 提供了 `ioctl(fd, SIOCOUTQ, &count)` 方法来查询一个套接字是否有未发送完成的数据。

> SIOCOUTQ
Returns the amount of unsent data in the socket send queue. The socket must not be in LISTEN state, otherwise an error (EINVAL) is returned. SIOCOUTQ is defined in <linux/sockios.h>. Alternatively, you can use the synonymous TIOCOUTQ, defined in <sys/ioctl.h>.

发送方可以使用这个方法来判断对端是否收到报文。以 Go 为例：

```go
import (
    "net"
	"golang.org/x/sys/unix"
	"unsafe"
)

func getSendQueueLength(conn *net.TCPConn) (qLen int) {
	if r, err := conn.SyscallConn(); err == nil {
		r.Control(func(fd uintptr) {
			unix.Syscall(unix.SYS_IOCTL, fd, unix.SIOCOUTQ, uintptr(unsafe.Pointer(&qLen)))
		})
	}
	return
}
```

## Windows

需要用到以下两个 API 函数，这两个函数都有对应的 IPv4 和 IPv6 版本，可按需调用，本文以 IPv4 为例。

1. **启用 TCP 连接的扩展统计信息**

> The [`SetPerTcpConnectionEStats`](https://learn.microsoft.com/zh-cn/windows/win32/api/iphlpapi/nf-iphlpapi-setpertcpconnectionestats) function sets a value in the read/write information for an IPv4 TCP connection. This function is used to enable or disable extended statistics for an IPv4 TCP connection.

```c
IPHLPAPI_DLL_LINKAGE ULONG SetPerTcpConnectionEStats(
  PMIB_TCPROW     Row,
  TCP_ESTATS_TYPE EstatsType,
  PUCHAR          Rw,
  ULONG           RwVersion,
  ULONG           RwSize,
  ULONG           Offset
);
```

其中 `Row` 是描述套接字的四元组信息，`EstatsType` 是统计信息类型，像本文需要的是发送缓冲区的信息，类型就是 `TcpConnectionEstatsSendBuff`。

注意此函数需要管理员权限。

2. **获取 TCP 连接的扩展统计信息**

> [`GetPerTcpConnectionEStats`](https://learn.microsoft.com/zh-cn/windows/win32/api/iphlpapi/nf-iphlpapi-getpertcpconnectionestats) 函数检索 IPv4 TCP 连接的扩展统计信息。

```c
IPHLPAPI_DLL_LINKAGE ULONG GetPerTcpConnectionEStats(
        PMIB_TCPROW     Row,
        TCP_ESTATS_TYPE EstatsType,
  [out] PUCHAR          Rw,
        ULONG           RwVersion,
        ULONG           RwSize,
  [out] PUCHAR          Ros,
        ULONG           RosVersion,
        ULONG           RosSize,
  [out] PUCHAR          Rod,
        ULONG           RodVersion,
        ULONG           RodSize
);
```

使用 `Rod` 参数接收发送缓冲区信息。

```c
typedef struct _TCP_ESTATS_SEND_BUFF_ROD_v0 {
  SIZE_T CurRetxQueue; // 占用重新传输队列的数据的当前字节数
  SIZE_T MaxRetxQueue; // 占用重新传输队列的数据的最大字节数
  SIZE_T CurAppWQueue; // TCP 缓冲的应用程序数据的当前字节数
  SIZE_T MaxAppWQueue; // TCP 缓冲的应用程序数据的最大字节数
} TCP_ESTATS_SEND_BUFF_ROD_v0, *PTCP_ESTATS_SEND_BUFF_ROD_v0;
```

Windows 把缓冲区的信息分成两部分数据，一个是重传，另一个是待发送。因此实际发送队列的长度是 `CurRetxQueue + CurAppWQueue`。

## 实例

Linux 上的例子上面已有介绍。下面主要说说 Windows 下的实现（IPv4 + IPv6）：

1. 导入函数

```go
var (
	libIpHlpApi                = syscall.NewLazyDLL("iphlpapi.dll")
	getPerTcpConnectionEStats  = libIpHlpApi.NewProc("GetPerTcpConnectionEStats")
	getPerTcp6ConnectionEStats = libIpHlpApi.NewProc("GetPerTcp6ConnectionEStats")
	setPerTcpConnectionEStats  = libIpHlpApi.NewProc("SetPerTcpConnectionEStats")
	setPerTcp6ConnectionEStats = libIpHlpApi.NewProc("SetPerTcp6ConnectionEStats")
)
```

2. 数据结构定义

```go
// 客户端
type Client struct {
    sync.Once
    // 记录连接的四元组信息
	tuple [1 << 6]byte
    conn *net.TCPConn
}

// TCPv4 连接
type mibTcpRow struct {
	state      uint32
	localAddr  uint32
	localPort  uint32
	remoteAddr uint32
	remotePort uint32
}

// TCPv6 连接
type mibTcp6Row struct {
	state         uint32
	localAddr     [16]byte
	localScopeId  uint32
	localPort     uint32
	remoteAddr    [16]byte
	remoteScopeId uint32
	remotePort    uint32
}

// TCP 发送队列信息
type tcpESTATSSendBuffROD struct {
	curRetxQueue uint // 占用重新传输队列的数据的当前字节数
	maxRetxQueue uint // 占用重新传输队列的数据的最大字节数
	curAppWQueue uint // TCP 缓冲的应用程序数据的当前字节数
	maxAppWQueue uint // TCP 缓冲的应用程序数据的最大字节数
}
```

3. 启用 TCP 连接的扩展统计信息

此函数一个连接只需要执行一次。

```go
func (c *Client) setupTcp() {
	var lp, rp uint32
	la := netip.MustParseAddrPort(c.conn.LocalAddr().String())
	ra := netip.MustParseAddrPort(c.conn.RemoteAddr().String())
	lp = uint32(la.Port()>>8 | la.Port()<<8)
	rp = uint32(ra.Port()>>8 | ra.Port()<<8)

	var rw = true
	var proc *syscall.LazyProc
	if la.Addr().Is4() {
		*(**syscall.LazyProc)(unsafe.Pointer(&c.tuple)) = getPerTcpConnectionEStats
		proc = setPerTcpConnectionEStats
		copy(c.tuple[unsafe.Sizeof(uintptr(0)):], unsafe.Slice((*byte)(unsafe.Pointer(&mibTcpRow{
			localAddr:  binary.LittleEndian.Uint32(la.Addr().AsSlice()),
			localPort:  lp,
			remoteAddr: binary.LittleEndian.Uint32(ra.Addr().AsSlice()),
			remotePort: rp,
		})), unsafe.Sizeof(mibTcpRow{})))
	} else {
		row := &mibTcp6Row{
			localAddr:  la.Addr().As16(),
			localPort:  lp,
			remoteAddr: ra.Addr().As16(),
			remotePort: rp,
		}
		if lif, err := net.InterfaceByName(la.Addr().Zone()); err == nil {
			row.localScopeId = uint32(lif.Index)
		}
		if rif, err := net.InterfaceByName(ra.Addr().Zone()); err == nil {
			row.remoteScopeId = uint32(rif.Index)
		}
		*(**syscall.LazyProc)(unsafe.Pointer(&c.tuple)) = getPerTcp6ConnectionEStats
		proc = setPerTcp6ConnectionEStats
		copy(c.tuple[unsafe.Sizeof(uintptr(0)):], unsafe.Slice((*byte)(unsafe.Pointer(row)), unsafe.Sizeof(mibTcp6Row{})))
	}
	// 启用 TCP 连接的扩展统计信息
	r, _, _ := proc.Call(uintptr(unsafe.Pointer(&c.tuple[unsafe.Sizeof(uintptr(0))])), 4, uintptr(unsafe.Pointer(&rw)), 0, unsafe.Sizeof(rw), 0)
	if r != windows.NO_ERROR {
		fmt.Println("setPerTcpConnectionEStats error:", windows.Errno(r))
	}
}
```

4. 获取发送队列长度

```go
func (c *Client) getSendQueueLength() (qLen int) {
	c.Do(c.setupTcp)
	var buf tcpESTATSSendBuffROD
	var rw = true
	r, _, _ := (*(**syscall.LazyProc)(unsafe.Pointer(&c.tuple))).Call(uintptr(unsafe.Pointer(&c.tuple[unsafe.Sizeof(uintptr(0))])), 4,
		uintptr(unsafe.Pointer(&rw)), 0, unsafe.Sizeof(rw),
		0, 0, 0,
		uintptr(unsafe.Pointer(&buf)), 0, unsafe.Sizeof(buf))
	if r != windows.NO_ERROR || !rw {
		return 0
	}
	return int(buf.curRetxQueue + buf.curAppWQueue)
}
```
