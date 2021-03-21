---
title: keras预定义激活函数
mathjax: true
date: 2020-01-04 19:02:01
categories:
- 机器学习
tags:
- 机器学习
---

[Available activations](https://keras.io/activations/)

[Activation function](https://en.wikipedia.org/wiki/Activation_function)

`keras`常见预定义激活函数公式定义

<!-- more -->

# softmax

$(0, 1)$

$$S_i=\frac{e^i}{\sum_je^j}$$

# elu

$(-1, +\infin)$

$$f(x)=\begin{cases}
x & x \geq 0 \\
\alpha(e^x-1) & x < 0
\end{cases}$$

# selu

$(-\lambda\alpha, +\infin)$

[李宏毅课程：SELU 激活函数](https://www.jianshu.com/p/3a43a6a860ef)

$$f(x)=\lambda\begin{cases}
x & x \geq 0 \\
\alpha(e^x-1) & x < 0
\end{cases}$$

$\alpha = 1.6732632423543772848170429916717$

$\lambda = 1.0507009873554804934193349852946$

# softplus

$(0, +\infin)$

$$f(x)=ln(1+e^x)$$

# softsign

$(-1, 1)$

$$f(x)=\frac{x}{1+|x|}$$

# relu

$[0, +\infin]$

$$f(x)=max(0,x)$$

# tanh

$[-1, 1]$

$$tanhx=\frac{sinhx}{coshx}=\frac{e^x-e^{-x}}{e^x+e^{-x}}$$

# sigmoid

$(0, 1)$

$$f(x)=\frac{1}{1+e^{-x}}$$

# hard_sigmoid

$[0 ,1]$

$$f(x)=\begin{cases}
0 & x < -2.5 \\
0.2x+0.5 & -2.5 \leq x \leq 2.5\\
1 & x > 2.5
\end{cases}$$

# linear

Input tensor, unchanged.