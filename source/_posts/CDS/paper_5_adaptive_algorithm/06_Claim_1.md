---
title: paper_5 第3节 Claim 1
date: 2026-06-26 11:30:00
tags:
  - 无线传感网
  - CDS
  - 论文阅读
categories:
  - CDS
  - paper_5
---

# Claim 1

它想证明：

$$
E[|S|]=\sum_{u\in V}E[cw(u)]
$$

也就是：

> **算法的期望探测次数 = 所有节点收到的期望 charge 总和。**

这个结论一旦成立，后面可以不直接分析 $|S|$，而是转去分析所有节点收到的 charge。

>所以 Claim 1 的作用就是：**把 probing cost 转化成 charge 分析。**

---

## 1. 先看左边：$E[|S|]$ 

Algorithm 1 中：

$$
S
$$

表示被算法主动 probe 的**节点集合**。

所以：

$$
|S|
$$

就是算法主动探测的**节点数量**。

而每 probe 一个节点，probing cost 是 1。

因此：

$$
|S|
$$

就是算法的总 **probing cost(探测代价)**。

由于节点状态随机，所以 $|S|$ 也是**随机变量！**。
 因此定理要分析的是：

$$
E[|S|]
$$

也就是算法的**期望探测次数**。

------

## 2. 再看右边：$\sum_{u\in V}E[cw(u)]$ ？

这里：

$$
cw(u)
$$

表示节点 $u$ 收到的 **charge 数量**。

注意，charge 是证明中的虚拟费用，不是算法真实操作。

$$
\sum_{u\in V}cw(u)
$$

表示所有节点收到的 **charge 总和**。

因为**过程随机，所以取期望**：

$$
\sum_{u\in V}E[cw(u)]
$$

表示所有节点收到的**期望 charge 总和**。

Claim 1 说：

$$
E[|S|]=\sum_{u\in V}E[cw(u)]
$$

翻译后就是：

> 算法平均 **probe** 了多少次，等于**所有**节点平均收到多少 **charge**。

-----

## 3. 为什么等号可能成立？

因为 charging method 是**故意**这样设计的。

Algorithm 1 每一轮 probe 一个节点 $v_i$，真实代价是：

$$
1
$$

charging method 规定：

> 这一轮让若干个节点收到 charge，每个节点收到的 charge 大小是

> $$
> w_{S_{i-1}}(v_i)
> $$

如果这一轮有：

$$
ch(v_i)
$$

**个**节点收到 charge，那么这一轮 charge 总额就是：

$$
ch(v_i)\cdot w_{S_{i-1}}(v_i)
$$

作者前面通过权重定义证明：

$$
E[ch(v_i)\mid \Gamma]=\frac{1}{w_{S_{i-1}}(v_i)}
\tag{9}
$$

于是这一轮的期望 charge 总额是：

$$
E[ch(v_i)\mid \Gamma]\cdot w_{S_{i-1}}(v_i)
$$

代入公式 (9)：

$$
=
\frac{1}{w_{S_{i-1}}(v_i)}
\cdot
w_{S_{i-1}}(v_i)
=1
$$

这刚好等于这一轮 probe 的真实代价。

所以每一轮都有：

> **期望 charge 总额 = 1 = 本轮 probing cost**

所有轮加起来，就得到：

$$
\text{期望总 charge}=\text{期望总 probe 次数}
$$

这就是 Claim 1 的核心直觉。

-----

## 4. 白色节点为什么要用“期望”？

灰色节点已经确定 active，所以本轮 charge 数是确定的（NCS(v) - 1）。

但白色节点状态未知，有随机性。

假设算法 probe 白色节点 $v_i$。

它有两种可能。

## 情况一：$v_i$ inactive

概率：

$$
1-p(v_i)
$$

只给自己 charge，所以：

$$
ch(v_i)=1
$$

## 情况二：$v_i$ active

概率：

$$
p(v_i)
$$

它变黑，自己不 charge，给其他白色邻居 charge，所以：

$$
ch(v_i)=NWS(v_i)-1
$$

因此：

$$
E[ch(v_i)]
=
(1-p(v_i))\cdot 1
+
p(v_i)(NWS(v_i)-1)
$$

也就是：

$$
E[ch(v_i)]
=
1-p(v_i)+p(v_i)(NWS(v_i)-1)
$$

而白色节点权重定义就是：

$$
w(v_i)=
\frac{1}
{1-p(v_i)+p(v_i)(NWS(v_i)-1)}
$$

所以：

$$
E[ch(v_i)]=\frac{1}{w(v_i)}
$$

于是：

$$
E[ch(v_i)]\cdot w(v_i)=1
$$

也就是说，白色节点虽然实际情况有随机性，但从**期望上**看，本轮 charge 总额也等于 1。

---
## 5. Claim 1 的核心等式怎么来的？

原文中最关键的一步是：

$$
\sum_{u\in V}
E[1_{v\to u}cw(u)\mid \Gamma]
=
w_{S_{i-1}}(v_i)
E[ch(v_i)\mid \Gamma]
=1
$$

拆开一下。

------

### 第一部分

$$
\sum_{u\in V}
E[1_{v\to u}cw(u)\mid \Gamma]
$$

表示：

> 在固定当前状态 $\Gamma$ 下，节点 $v$ 这一轮向所有节点发出的期望 charge 总额。

如果 $v$ charge 了 $ch(v)$ 个节点，每个节点收到：

$$
w(v)
$$

所以总额是：

$$
w(v)\cdot ch(v)
$$

取期望：

$$
w(v)\cdot E[ch(v)\mid \Gamma]
$$

------

### 第二部分

根据公式 (9)：

$$
E[ch(v_i)\mid \Gamma]=\frac{1}{w(v_i)}
$$

所以：

$$
w(v_i)\cdot E[ch(v_i)\mid \Gamma]
=
w(v_i)\cdot \frac{1}{w(v_i)}
=1
$$

所以每个被 probe 节点在它那一轮贡献的期望 charge 总额就是 1。

------

## 为什么等于 $E[|S|]$？

经过前面的化简，得到：

$$
\sum_{u\in V}E[cw(u)]
=
\sum_{v\in V}Pr[v\in S]
$$

右边是什么？

每个节点 $v$ 有一定概率被算法 probe。

所以：

$$
\sum_{v\in V}Pr[v\in S]
$$

就是被 **probe 节点数量的期望**。

因为：

$$
|S|=\sum_{v\in V}1_{v\in S}
$$

取期望：

$$
E[|S|]
=
E\left[\sum_{v\in V}1_{v\in S}\right]
=
\sum_{v\in V}Pr[v\in S]
$$

所以：

$$
\sum_{u\in V}E[cw(u)]
=
E[|S|]
$$

Claim 1 得证。
























