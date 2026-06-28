---
title: 随机容错 CDS 的相关定义
date: 2026-06-27 18:00:00
tags:
  - 无线传感网
  - CDS
categories:
  - CDS
  - 随笔
  - Roadmap
mathjax: true
---

## 1. 节点版本

> 每一轮算法只考虑“选哪个节点 $v$ 去 probe”。

也就是说，候选对象是**单个节点**：
$$
v
$$
然后给每个候选节点计算：
$$
NW_S(v),\quad NC_S(v),\quad FT_S(v)
$$
最后根据评分选择一个节点 probe。

节点版本的算法形式更接近原论文 Algorithm 1。

-----
## 2. 定义 $LB_S(v)$ 

令
$$
C
$$

为当前**黑色骨干节点**集合，即当前已经被 **probe 且 active** 的节点集合。

考虑：

$$
G[C]
$$

的 leaf blocks。定义：

$$
LB_S(v)=\{B: B\text{ 是 }G[C]\text{ 的 leaf block，且 }v\text{ 邻接 }B\text{ 中某个节点}\}
$$

也就是说，$LB_S(v)$ 表示：

> 节点 $v$ 能接触到多少个**不同 leaf block**。

-----

## 3. $FT_S(v)$ 相关定义

$$
FT_S(v)=\max\{|LB_S(v)|-1,0\}
$$

$LB_S(v)$ ≥ 2 时，才考虑**有收益**。

其中：

$$
LB_S(v)=\{B: B\text{ 是 }G[C]\text{ 的 leaf block，且 }v\text{ 与 }B\text{ 相邻}\}
$$

然后**综合收益**：

$$
Gain(v)=\alpha NW_S(v)+\beta NC_S(v)+\gamma FT_S(v)
$$

再考虑**随机性**：

$$
Score(v)=p(v)\cdot Gain(v)
$$

假设每次 probe 代价都一样是 1，那么：

$$
Cost(v)=1
$$

故：

$$
Score(v)=p(v)\cdot Gain(v)
$$

## 4. 总结

> 本阶段先研究节点版本，即每轮动态选择一个候选节点 $v$ 进行 probe。**容错收益 **$FT_S(v)$ 由该节点能够**关联的不同 leaf block 数量决定**，具体定义为$FT_S(v)=\max\{|LB_S(v)|-1,0\}$，并优先选择在**信息揭示、连通扩展和容错增强**方面综合收益较高的节点。


以后的以后升级*路径*版本时：

> 路径版本是节点版本的进一步扩展，用于处理单个节点无法有效连接 leaf block 的情况。

















