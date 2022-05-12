---
title: "理解 RNN"
date: 2022-04-21T11:38:04+08:00
archives: 
    - 2022
tags:
    - 机器学习
image: /images/rnn-banner.png
leading: false
draft: false
---

循环神经网络 (Recurrent Neural Network, RNN) 是一种用于处理时间序列数据的神经网络结构。包括文字、语音、视频等对象。这些数据有两个主要特点：

- 数据无固定长度
- 数据是有时序的，相邻数据之间存在相关性，非相互独立

考虑这样一个问题，如果要预测句子下一个单词是什么，一般需要用到当前单词以及前面的单词，因为句子中前后单词并不是独立的。比如，当前单词是“很”，前一个单词是“天空”，那么下一个单词很大概率是“蓝”。循环神经网络就像人一样拥有记忆的能力，它的输出依赖于当前的输入和记忆，刻画出一个序列当前的输出与之前信息的关系。

循环神经网络适用于许多序列问题中，例如文本处理、语音识别以及机器翻译等。

## 基本结构
{{<cimg src="/images/rnn_scratch.jpg" >}}

如果把上面有 W 的那个带箭头的圈去掉，它就变成了普通的全连接神经网络。图中每个圆圈可以看作是一个单元，而且每个单元做的事情也是一样的，因此可以折叠呈左半图的样子。用一句话解释 RNN，就是一个单元结构重复使用。 

简单理清一下各符号的定义：
- $x_t$ 表示 t 时刻的输入
- $y_t$ 表示 t 时刻的输出
- $s_t$ 表示 t 时刻的记忆，即隐藏层的输出
- U 是输入层到隐藏层之间的权重矩阵
- W 是记忆单元到隐藏层之间的权重矩阵
- V 是隐藏层到输出层之间的权重矩阵
- U, W, V 都是权重矩阵，在不同时刻 t 之间是**共享权重**的

从右半图可以看到，RNN 每个时刻隐藏层输出都会传递给下一时刻，因此每个时刻的网络都会保留一定的来自之前时刻的历史信息，并结合当前时刻的网络状态一并再传给下一时刻。

比如在文本预测中，文本序列为 "machine"，则输入序列和标签序列分别为 "machin" 和 "achine"，网络的概览图如下：
{{<cimg src="/images/rnn-train.svg" >}}

## 前向计算
{{<cimg src="/images/rnn-step-forward.png" height=300 >}}

在一个循环神经网络中，假设隐藏层只有一层。在 t 时刻神经网络接收到一个输入 $x_t$，则隐藏层的输出 $s_t$ 为：
$$
s_t = \tanh(Ux_t + Ws_{t-1} + b_s)
$$

在神经网络刚开始训练时，记忆单元中没有上一时刻的网络状态，这时候 $s_{t-1}$ 就是一个初始值。

在得到隐藏层的输出后，神经网络的输出 $y_t$ 为：
$$
y_t = \mathrm{softmax}(Vs_t + b_y)
$$

设 t 时刻的输入 $x_t$ 的维度为 $(n_x, m)$，其中 $n_x$ 为变量 $x$ 的维度（输入层节点数量），$m$ 为样本数量；隐藏层节点的数量为 $n_s$；输出层节点数量为 $n_y$。则各变量的信息如下：

| 变量  | 说明                           | 维度         |
| ----- | ------------------------------ | ------------ |
| $x_t$ | t 时刻输入                   | $(n_x, m)$   |
| $s_t$ | t 时刻隐藏层输出             | $(n_s, m)$   |
| $U$   | 输入层到隐藏层之间的权重矩阵   | $(n_s, n_x)$ |
| $W$   | 记忆单元到隐藏层之间的权重矩阵 | $(n_s, n_s)$ |
| $V$   | 隐藏层到输出层之间的权重矩阵   | $(n_y, n_s)$ |
| $b_s$ | 隐藏层偏置量                   | $(n_s, 1)$   |
| $b_y$ | 输出层偏置量                   | $(n_y, 1)$   |

某个时刻的 RNN 前向计算流程如下：

```python
def rnn_cell_forward(xt, s_prev, parameters):
    """
    Implements a single forward step of the RNN-cell

    Arguments:
    xt -- Your input data at timestep "t", numpy array of shape (n_x, m).
    s_prev -- Hidden state at timestep "t-1", numpy array of shape (n_s, m)
    parameters -- Python dictionary containing:
        U -- Weight matrix multiplying the input, numpy array of shape (n_s, n_x)
        W -- Weight matrix multiplying the hidden state, numpy array of shape (n_s, n_s)
        V -- Weight matrix relating the hidden-state to the output, numpy array of shape (n_y, n_s)
        bs -- Bias, numpy array of shape (n_s, 1)
        by -- Bias relating the hidden-state to the output, numpy array of shape (n_y, 1)
    
    Returns:
    s_next -- Next hidden state, of shape (n_s, m)
    yt_pred -- Prediction at timestep "t", numpy array of shape (n_y, m)
    cache -- Tuple of values needed for the backward pass, contains (s_next, s_prev, xt, parameters)
    """
    U = parameters["U"]
    W = parameters["W"]
    V = parameters["V"]
    bs = parameters["bs"]
    by = parameters["by"]

    # Compute next hidden state
    s_next = np.tanh(np.dot(W, a_prev) + np.dot(U, xt) + bs)
    # Compute output of the current cell
    yt_pred = softmax(np.dot(V, s_next) + by)
    # Store values needed for backward propagation in cache
    cache = (s_next, s_prev, xt, parameters)
    return s_next, yt_pred, cache
```

接下来考虑沿时间线前向计算的过程，设总的时间步长为 $T_x$，此时网络的输入 $x$ 的维度为 $(n_x, m, T_x)$，$s_0$ 为初始隐藏状态，可以使用一些策略进行初始化：
- 置 0
- 随机值
- 作为模型参数训练学习

通常采用零向量或随机向量作为初始状态，但通过学习得出的初始值可能效果更好，见 [6]。

```python
def rnn_forward(x, s0, parameters):
    """
    Implement the forward propagation of the recurrent neural network

    Arguments:
    x -- Input data for every time-step, of shape (n_x, m, T_x).
    s0 -- Initial hidden state, of shape (n_s, m)
    parameters -- See `rnn_cell_forward`

    Returns:
    s -- Hidden states for every time-step, numpy array of shape (n_s, m, T_x)
    y_pred -- Predictions for every time-step, numpy array of shape (n_y, m, T_x)
    caches -- Tuple of values needed for the backward pass, contains (list of caches, x)
    """
    
    # Initialize "caches" which will contain the list of all caches
    caches = []
    
    n_x, m, T_x = x.shape
    n_y, n_s = parameters["V"].shape

    s = np.zeros((n_s, m, T_x))
    y_pred = np.zeros((n_y, m, T_x))
    
    s_next = s0
    # loop over all time-steps
    for t in range(T_x):
        # Update next hidden state, compute the prediction
        s_next, yt_pred, cache = rnn_cell_forward(x[:,:,t], s_next, parameters)
        s[:,:,t] = s_next
        y_pred[:,:,t] = yt_pred
        caches.append(cache)
            
    caches = (caches, x)
    return s, y_pred, caches
```

## 反向传播

$$
\frac{\partial E}{\partial a^{t-3}} = \frac{\partial L}{\partial a^{t}}
\frac{\partial a^t}{\partial a^{t-1}}\frac{\partial a^{t-1}}{\partial a^{t-2}}
\frac{\partial a^{t-2}}{\partial a^{t-3}} + \frac{\partial L}{\partial a^{t-1}}
\frac{\partial a^{t-1}}{\partial a^{t-2}}
\frac{\partial a^{t-2}}{\partial a^{t-3}} +
\frac{\partial L}{\partial a^{t-2}}
\frac{\partial a^{t-2}}{\partial a^{t-3}} +
\frac{\partial L}{\partial a^{t-3}}
$$

$$
\frac{\partial E}{\partial a^{t-3}} = \frac{\partial a^{t-2}}{\partial a^{t-3}} \left(\frac{\partial L}{\partial a^{t}}
\frac{\partial a^t}{\partial a^{t-1}}\frac{\partial a^{t-1}}{\partial a^{t-2}} +
\frac{\partial L}{\partial a^{t-1}}
\frac{\partial a^{t-1}}{\partial a^{t-2}} +
\frac{\partial L}{\partial a^{t-2}}
\right) + \frac{\partial L}{\partial a^{t-3}}
$$

$$
\frac{\partial E}{\partial a^{t-3}} = \frac{\partial a^{t-2}}{\partial a^{t-3}} \frac{\partial E}{\partial a^{t-2}} + \frac{\partial L}{\partial a^{t-3}}
$$


## 示例

## 结论

## 参考

[1] [开小灶｜循环神经网络RNN讲解(一)](https://www.bilibili.com/read/cv15812073)

[2] [RNN神经网络模型综述](https://mp.weixin.qq.com/s?__biz=MzAxMjMwODMyMQ==&mid=2456338385&idx=1&sn=8e9194c87d3ac6f9134c112b28724e0c&chksm=8c2fc5dfbb584cc9dddbcaea7da777157437f59637f189f2d40f76606c4f52be4b886973bf91&scene=21#wechat_redirect)

[3] [The Unreasonable Effectiveness of Recurrent Neural Networks](http://karpathy.github.io/2015/05/21/rnn-effectiveness/)

[4] [循环神经网络 - 动手学深度学习](https://zh.d2l.ai/chapter_recurrent-neural-networks/rnn.html)

[5] [Building your Recurrent Neural Network - Step by Step](https://datascience-enthusiast.com/DL/Building_a_Recurrent_Neural_Network-Step_by_Step_v1.html)

[6]: https://r2rt.com/non-zero-initial-states-for-recurrent-neural-networks.html

\[6\] [Non-Zero Initial States for Recurrent Neural Networks][6]

[7] [Tips for Training Recurrent Neural Networks](https://danijar.com/tips-for-training-recurrent-neural-networks/)
