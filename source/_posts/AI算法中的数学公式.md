---
title: AI 算法中的数学公式
date: 2019-05-24 20:22:41
categories:
 - 机器学习
tags:
 - 数学原理
 - 机器学习
mathjax: true
---

AI 算法中的一些数学公式

<!-- more -->

## [DDPG](https://arxiv.org/pdf/1509.02971.pdf)

$$
\begin{aligned}
\nabla_{\theta^\mu}J&\approx\mathbb{E}_{s_t\backsim\rho^\beta}[\nabla_{\theta^\mu}Q(s,a|\theta^Q)|_{s=s_t,a=\mu(s_t|\theta^\mu)}] \\\\
&=\mathbb{E}_{s_t\backsim\rho^\beta}[\nabla_{a}Q(s,a|\theta^Q)|_{s=s_t,a=\mu(s_t)}\nabla_{\theta_\mu}\mu(s|\theta^\mu)|_{s=s_t}]
\end{aligned}
$$

## GEN
$$g=<V,E,\phi>$$

$$\forall v\in V,v\rightarrow x_v\rightarrow \mu_v$$

$$N_v$$

$$\mu_v=F(x_v,\sum_{j\in N_v}\mu_j)$$

$$P=\lbrace P_1,P_2,\cdots,P_n\rbrace$$

$$ReLU(\cdot)=max\lbrace 0,\cdot\rbrace$$

$$\mu_v^{(t)}=tanh(W_1x_v+\sigma(\sum_{j\in N_v}\mu_j^{(t-1)}))$$

$$\sigma(x)=P_1ReLU(P_2\cdots ReLU(P_n(x)))$$

$$\mu_g=W_2\sum_{v\in V}\mu_v^T$$

$$Z=W_3\mu_g$$

$$Q=\lbrace p,1-p\rbrace,p\in[0,1]$$

$$Q=Softmax(Z)$$

$$\min_{W_1,W_2,W_3,...,P_1,P_2,...,P_n}\sum_{i=1}^m(H(Q,l))$$

在[这里](https://stackedit.io/editor)将Markdown的公式转换为MathML代码（Word）[参考](https://blog.csdn.net/bendanban/article/details/52823171)