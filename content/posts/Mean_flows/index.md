---
title: "浅谈NeurIPS'25 一步生成模型：Mean Flow"
date: 2026-11-02
draft: false
tags: ["图像生成", "Flow matching", "Mean Flow"]
categories: ["论文"]
summary: "图像生成：Mean Flow"
---

## 背景：Flow matching
有个直观的想法就是：我们可以考虑噪声的初始高维状态是一个二维平面上的点，同时这个平面内是一个关于时间的速度场，我们只要将这个点按照这个速度场的方向去移动就可以得到真实世界的图像分布。

基于这个动机，对于初始状态$x_0 \sim p_{data}$，不妨考虑加噪后的目标是$x_1 \sim  \mathcal{N}(0,I)$。

考虑$x_0$到$x_1$的路径上存在一个经过时间$t$采样的$x_t$:

$$
x_t = t \cdot x_1 + (1 - t) \cdot x_0
$$

经过求导可以得到，这个路径上的速度为$v(x_t,t) = x_1 - x_0$。


我们希望我们预测的速度场$v(x_t,t)$与$x_1 - x_0$的期望误差最小。

生成图像的时候，就是噪声到真实图像的逆向过程，这个过程依赖于多次的瞬时速度，才能使噪声向真实世界趋近。


基于此，Mean Flow 利用平均速度的思想，达到了一步生成图像的目的。

## MeanFlow 数学推导


### 1. 平均速度的定义

对 \(0\le r\lt t\le 1\)，定义从 \(r\) 到 \(t\) 的平均速度

\[
u(z_t,r,t)=\frac{1}{t-r}\int_r^t v^*(z_s,s)\,ds,
\]


$z_t$是当前位置。

### 2. MeanFlow 恒等式

令

$$
F(t)=\int_r^t v^*(z_s,s)\,ds.
$$

根据平均速度的定义，

$$
F(t)=(t-r)u(z_t,r,t).
$$

对 \(t\) 求导。


$$
\begin{aligned}
\frac{dF}{dt}
&= u(z_t,r,t) \\
&\quad + (t-r)\frac{d}{dt}u(z_t,r,t).
\end{aligned}
$$

因此

$$
\begin{aligned}
v^*(z_t,t)
&= u(z_t,r,t) \\
&\quad + (t-r)\frac{d}{dt}u(z_t,r,t).
\end{aligned}
$$

移项得到 MeanFlow 恒等式：

$$
\boxed{ u(z_t,r,t) = v^*(z_t,t) - (t-r)\frac{d}{dt}u(z_t,r,t) }
$$


### 3. 全导数的展开

根据链式法则，

$$
\begin{aligned}
\frac{d}{dt}u(z_t,r,t)
&= \partial_t u(z_t,r,t) \\
&\quad + \left(\frac{dz_t}{dt}\right)^\top \nabla_x u(z_t,r,t).
\end{aligned}
$$

又因为 \(\frac{dz_t}{dt}=v^*(z_t,t)\)，所以

$$
\boxed{ \frac{d}{dt}u(z_t,r,t) = \partial_t u(z_t,r,t) + v^*(z_t,t)^\top \nabla_x u(z_t,r,t) }
$$


### 4. 训练目标

实际训练时用条件速度 \(v=x_1-x_0\) 作为边缘速度的采样估计。设网络输出为 \(u_\theta(x,r,t)\)，恒等式残差为

$$
R_\theta = u_\theta(x_t,r,t) - (x_1-x_0) + (t-r)\frac{d}{dt}u_\theta(x_t,r,t).
$$

如果直接最小化 \(\|R_\theta\|^2\)，梯度会穿过 JVP 项。实际实现中通常对恒等式右侧使用 stop-gradient，把右侧作为回归目标：

$$
\boxed{
\mathcal L_{\mathrm{MF}} = \mathbb E \left[ \left\| u_\theta(x_t,r,t) - \operatorname{sg}\left( (x_1-x_0) - (t-r)\frac{d}{dt}u_\theta(x_t,r,t) \right) \right\|^2 \right]
}
$$

其中 \(\operatorname{sg}(\cdot)\) 表示停止梯度。

因为如果对最后一项求导，反向传播就会穿过这个导数项，导致要计算二阶导，训练更贵也更不稳定。可以看下面代码方便理解：
```
# 会穿过 JVP，二阶反传
loss = ((u - (v - (t - r) * dudt)) ** 2).mean()

# 不穿过 JVP，只更新 u
target = (v - (t - r) * dudt).detach()
loss = ((u - target) ** 2).mean()
```

### 5. 一步采样

由位移关系

$$
x_1-x_0=u(x_1,0,1).
$$

因此从噪声 \(x_1\) 出发，一步生成数据为

$$
\boxed{ \hat x_0=x_1-u_\theta(x_1,0,1) }
$$

这就是 MeanFlow 的 one-step generation，只需要一次网络前向。




