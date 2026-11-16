---
title: "数论"
date: 2026-11-16
draft: false
tags: ["数论", "数学"]
categories: ["日志"]
summary: "数论知识"
---

## 欧拉定理
如果整数 $a$ 和 $n$ 互质，也就是：

$$
\gcd(a,n)=1
$$

那么：

$$
\boxed{a^{\varphi(n)}\equiv 1\pmod n}
$$

$\varphi(n)$ 叫作**欧拉函数**，表示在 $1$ 到 $n$ 之间，与 $n$ 互质的整数有多少个。


**欧拉函数**的计算方式如下所示，若：

$$
n=p_1^{a_1}p_2^{a_2}\cdots p_r^{a_r}
$$

其中 $p_1,p_2,\ldots,p_r$ 是**不同的质因数**，那么：

$$
\boxed{\varphi(n)=n\prod_{p\mid n}\left(1-\frac{1}{p}\right)}
$$


## 中国剩余定理
$$

\begin{cases}
x\equiv a\pmod m\\
x\equiv b\pmod n
\end{cases}
\quad\text{且}\quad \gcd(m,n)=1

$$

那么这个方程组在模 $mn$ 的意义下$x$有唯一解。

例如快速求解$2^{100}\mod1000$，我们可以拆分成$2^{100}\mod 8$，和$2^{100} \mod 125$，前者为0，后者2和125互质，可以得到$2^{\varphi(125)} = 2^{100} \equiv 1 \mod 125$。故$2^{100}$可设为1 + 125k，仅当k等于3时，满足8 | 376。 故答案为376。




