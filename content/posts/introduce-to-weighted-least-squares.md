---
title: "加权最小二乘"
date: 2020-11-30T13:17:00+08:00
archives: 
    - 2020
tags:
    - 机器学习
    - 回归
image: /images/0_g0BN7JUBfBoDkriE.png
draft: false
---

{{< figure src="/images/0_g0BN7JUBfBoDkriE.png" >}}

## 介绍
多元线性回归模型的关系式为：

{{<tex>}}
$$
y_i= \beta_0+\beta_1 x_{i1}+\beta_2 x_{i2} + \dots + \beta_p x_{ip} + \epsilon_i
$$

{{</tex>}}

假设随机误差项 $\epsilon_i(i=1,\dots,n)$满足

{{<tex>}}
$$
E(\epsilon_i) = 0,\quad \text{Cov}(\epsilon_i,\epsilon_j)=\begin{cases}
    \sigma^2,   &  i=j, \\
    0,  & i \ne j.
 \end{cases}
$$
{{</tex>}}

上式通常称为**高斯-马尔可夫条件**。随机误差$\epsilon_i$的协方差为零表明随机误差项在不同的样本点之间是不相关的(在正态条件下即为独立)。随机误差项$\epsilon_i$在不同的样本点有相同的方差表明各次观测之间有相同的精度。

多元线性回归模型改写成矩阵形式如下：

{{<tex>}}
$$
Y=X\mathbf{\beta}+\epsilon,\quad \epsilon\sim\mathbf{N}(0,\sigma^2I_n)
$$
{{</tex>}}

## 最小二乘估计

最小二乘估计(LSE)就是找{{<tex>}}$\beta_0,\beta_1,\dots,\beta_p${{</tex>}}，使离差平方和

{{<tex>}}
$$
Q(\beta_0,\beta_1,\dots,\beta_p)=\sum_{i=1}^n(y_i-\beta_0-\beta_1x_{i1}-\dots-\beta_px_{ip})^2
$$
{{</tex>}}

达到最小，写成矩阵形式如下：

{{<tex>}}
$$
Q(\beta)= (Y-X\beta)^T(Y-X\beta)
$$
{{</tex>}}

上式中每个平方项的权数相同，是普通最小二乘回归参数估计方法。在误差项$\epsilon_i$等方差不相关的条件下，普通最小二乘估计是回归参数的最小方差线性无偏估计。

## 加权最小二乘

然而在异方差的条件下，平和和中的每一项的地位是不相同的。误差项$\epsilon_i$的方差$\sigma_i^2$大的项，在上式平方和中的取值就偏大，在平方和中的作用就大，因而普通最小二乘估计的回归线就被拉向方差大的项，方差大的项的拟合程度就好，而方差小的项拟合程度就差。由此求出的{{<tex>}}$\hat{\beta_0},\hat{\beta_1},\dots,\hat{\beta_p}${{</tex>}}仍然是{{<tex>}}$\beta_0,\beta_1,\dots,\beta_p${{</tex>}}的无偏估计，但不再是最小方差线性无偏估计。



加权最小二乘估计的方法是在平方和中加入一个适当的权数$w_i$，以调整各项在平方和中的作用，加权最小二乘的离差平方和为：

{{<tex>}}
$$
Q_w(\beta_0,\beta_1,\dots,\beta_p)=\sum_{i=1}^nw_i(y_i-\beta_0-\beta_1x_{i1}-\dots-\beta_px_{ip})^2
$$
{{</tex>}}

类似地，目标也是寻找$\hat{\beta}$使上式的离差平方和$Q_w$达到最小。上式可以改写为：

{{<tex>}}
$$
Q_w(\beta_0,\beta_1,\dots,\beta_p)=\sum_{i=1}^n(\sqrt{w_i}y_i-\sqrt{w_i}\beta_0-\sqrt{w_i}\beta_1x_{i1}-\dots-\sqrt{w_i}\beta_px_{ip})^2
$$
{{</tex>}}

展开为原始的模型式为：

{{<tex>}}
$$
\sqrt{w_i}y_i= \sqrt{w_i}\beta_0+\sqrt{w_i}\beta_1 x_{i1}+\sqrt{w_i}\beta_2 x_{i2} + \dots + \sqrt{w_i}\beta_p x_{ip} + \sqrt{w_i}\epsilon_i
$$
{{</tex>}}

令{{<tex>}}$\epsilon_i^*=\sqrt{w_i}\epsilon_i${{</tex>}}, 此时模型中随机误差项的方差为

{{<tex>}}
$$
Var(\epsilon_i^*)=Var(\sqrt{w_i}\epsilon_i)=w_iVar(\epsilon_i)
$$
{{</tex>}}

若使{{<tex>}}$Var(\epsilon_i^*)=w_iVar(\epsilon_i)=1${{</tex>}}，则理论上最优的权数$w_i$为误差项方差$\sigma^2$的倒数，即

{{<tex>}}
$$
w_i = \frac{1}{\sigma_i^2}
$$
{{</tex>}}

误差项方差大的项接受小的权数，以降低其在平方和中的作用；误差项方差小的项接受大的权数，以提高其在平方和中的作用。这样求出的加权最小二乘估计$\hat{\beta}$是最小方差线性无偏估计。

{{<tex>}}
$$
\hat{\beta_w}=(X^TWX)^{-1}X^TWY
$$
{{</tex>}}

但是误差项的方差$\sigma^2$是未知的，因此无法真正按照上式选取权数。在实际问题中误差项方差$\sigma^2$通常与某个自变量的水平有关，可以利用这个关系确定权数。例如$\sigma^2$与第$j$个自变量取值的平方成比例时，即{{<tex>}}$\sigma_i^2=kx_{ij}^2${{</tex>}}时，这时取权数为

{{<tex>}}
$$
w_i=\frac{1}{x_{ij}^2}
$$
{{</tex>}}

更一般的情况是误差项方差$\sigma_i^2$与某个自变量$x_j$取值的幂函数$x_{ij}^m$成比例，此时权数为

{{<tex>}}
$$
w_i=\frac{1}{x_{ij}^m}
$$
{{</tex>}}

关于自变量$x_j$的选择，选取等级相关系数最大的一个自变量；而$m$值，则选择使对数似然函数最大的一个数。

## M估计

M估计(M代表最大似然类型)定义为解决以下最小化优化问题：

{{<tex>}}
$$
\hat{\theta} = \arg\min_{\displaystyle\theta}\sum_{i=1}^n\rho(x_i, \theta)
$$
{{</tex>}}

其中$\rho$为任意实值函数，这个问题的解$\hat\theta$称为M估计。最小二乘和最大似然估计都是M估计的特例。

例如最小二乘估计：

{{<tex>}}
$$
\hat\beta=\arg\min_{\displaystyle\theta} \sum_{i=1}^n (y_i-f(x_i;\beta))^2
$$

{{</tex>}}

## 迭代加权最小二乘

IRLS (Iterative Reweighted Least Squares)用来求解$p$范数的问题，将$p$范数问题转化为2范数求解。

目标函数为

{{<tex>}}
$$
||X\beta-Y||_p = \left(\sum_{i=1}^n |\vec{x_i}\cdot\vec{\beta}-y_i|^p\right)^{1/p}
$$
{{</tex>}}

因此可以转化为最优化问题：

{{<tex>}}
$$
\begin{aligned}
f(\theta) &= \arg\min_{\displaystyle\theta}\sum_{i=1}^n |\vec{x_i}\cdot\vec{\beta}-y_i|^p\\
&=\arg\min_{\displaystyle\theta}\sum_{i=1}^n |\vec{x_i}\cdot\vec{\beta}-y_i|^{p-2}(\vec{x_i}\cdot\vec{\beta}-y_i)^2 \\
&= \arg\min_{\displaystyle\theta}\sum_{i=1}^n w_i^2(\vec{x_i}\cdot\vec{\beta}-y_i)^2
\end{aligned}
$$
{{</tex>}}

所以有{{<tex>}}$w_i^2=|\vec{x_i}\cdot\vec{\beta}-y_i|^{p-2}${{</tex>}}，则 

{{<tex>}}
$$
w_i=|\vec{x_i}\cdot\vec{\beta}-y_i|^{(p-2)/2}
$$
{{</tex>}}

写成矩阵形式为：

{{<tex>}}
$$
f(\theta)=\arg\min_{\displaystyle\theta} (W^TW(X\beta-Y))^TW^TW(X\beta-Y)
$$
{{</tex>}}

最小二乘解为

{{<tex>}}
$$
\hat\beta=(X^TW^TWX)^{-1}X^TW^TWY
$$
{{</tex>}}

其中$W$为误差权值的对角矩阵。权值的更新公式为：

{{<tex>}}
$$
W=|X\beta-Y|^{(p-2)/2}
$$
{{</tex>}}

算法设定初始权值$W=I$，然后根据上式更新权值$W$，重复迭代直到$\hat\theta$收敛。

```python
def IRLS(x, y, p=1):
    X = torch.stack([torch.ones(x.shape[0], ), torch.tensor(x)], axis=1)
    Y = torch.tensor(y).view(-1, 1).float()
    W = torch.eye(x.shape[0])
    while True:
        beta = X.T.matmul(W.T).matmul(W).matmul(X).inverse().matmul(X.T).matmul(W.T).matmul(W).matmul(Y)
        e = X.matmul(beta) - Y
        W_new = e.abs() ** ((p - 2) / 2)
        W_new = W_new / W_new.sum()
        if torch.norm(W.diag() - W_new.flatten(), 2) < 0.005:
            return beta
        print(W_new.flatten())
        W = W_new.flatten().diag()
```

 上述算法在$1.5 < p < 3$会比较容易收敛，也有一些算法变体在每次迭代过程中会局部更新$\hat\theta$，能够更快的收敛：
$$
\beta(k) = q\hat{\beta}(k) + (1 - q) \beta(k - 1)
$$

$$
q=\frac{1}{p - 1}
$$

这样$p$越大，新解占的权重越小，旧解的权重越大。还有一些变体使用了动态的$p$值，设定一个初始值$p=2$，当给定的$p>2$时，在每次迭代会增加$p$值，直到达到给定的$p$值，同样当$p<2$时则会逐渐减少到给定的值。

```python
def IRLS2(x, y, K, p):
    X = torch.stack([torch.ones(x.shape[0], ), torch.tensor(x)], axis=1)
    Y = torch.tensor(y).view(-1, 1).float()
    W = torch.eye(x.shape[0])
    beta = X.T.matmul(W.T).matmul(W).matmul(X).inverse().matmul(X.T).matmul(W.T).matmul(W).matmul(Y)
    pk = 2
    while True:
        if p >= 2:
            pk = min(p, K * pk)
        else:
            pk = max(p, K * pk)

        e = X.matmul(beta) - Y
        W_new = e.abs() ** ((pk - 2) / 2)
        W_new = W_new / W_new.sum()
        W = W_new.flatten().diag()

        # New beta
        beta1 = X.T.matmul(W.T).matmul(W).matmul(X).inverse().matmul(X.T).matmul(W.T).matmul(W).matmul(Y)

        q = 1 / (pk - 1)
        if p > 2:
            new_beta = q * beta1 + (1 - q) * beta
        else:
            new_beta = beta1

        if torch.norm(beta - new_beta, 2) < 0.0005:
            return beta
        beta = new_beta
```

从上面的定义可以看出$w_i$误差$e_i$的函数：

{{<tex>}}
$$
w_i=|e_i|^{(p-2)/2}
$$
{{</tex>}}

所以也可能把上面的$w_i$替换成其他一般函数：

{{<tex>}}
$$
w=w\left(\frac{r}{tune*s*\sqrt{1-h}}\right)
$$
{{</tex>}}

其中$r$是误差向量，$tune$是一个常量，$s$是误差的标准差估计，$h$是杠杆值向量，权重函数$w$一般是关于$r$的递减函数，比如

双平方函数：
$$
w = (|r|<1) * (1 - r^2)^2
$$
柯西函数：
$$
w=1/(1+r^2)
$$
$s$标准差的计算公式为: $s = MAD / 0.6745$，$MAD$计算了误差绝对偏差的中位数，注意在计算中位数前先删除掉最小的$p$(特征数)的数。

类似地，这种形式的权函数也可以使用IRLS进行求解。

```python
def robust_fit(x, y, wfun, tune):
    X = torch.stack([torch.ones(x.shape[0], ), torch.tensor(x)], axis=1)
    Y = torch.tensor(y).view(-1, 1).float()
    W = torch.eye(x.shape[0])
    if torch.cuda.is_available():
        X = X.cuda()
        Y = Y.cuda()
        W = W.cuda()
    while True:
        beta = X.T.matmul(W).matmul(X).inverse().matmul(X.T).matmul(W).matmul(Y)
        e = (Y - X.matmul(beta)).flatten()
        H = X.matmul(X.T.matmul(X).inverse()).matmul(X.T)
        adj_factor = 1 / (1 - H.diag()).sqrt()
        radj = e * adj_factor
        rs = radj.abs().sort(0)[0][(X.shape[1] - 1):]
        i = rs.size(0) // 2
        if rs.size(0) % 2 == 0:
            med = (rs[i - 1] + rs[i]) / 2
        else:
            med = rs[i]
        s = med / 0.6745

        W_new = wfun(radj / (s * tune))
        print(W_new, beta)
        if torch.norm(W.diag() - W_new.flatten(), 2) < 1e-5:
            return beta

        W = W_new.flatten().diag()
```

![irls1](/images/irls1.svg)


**参考文献**：

[1] [Iterative Reweighted Least Squares](https://cnx.org/exports/92b90377-2b34-49e4-b26f-7fe572db78a1@12.pdf/iterative-reweighted-least-squares-12.pdf)

[2] [Robust regression using iteratively reweighted
least-squares](https://www.tandfonline.com/doi/pdf/10.1080/03610927708827533?needAccess=true)