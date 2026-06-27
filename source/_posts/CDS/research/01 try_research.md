---
title: 预研究: 随机容错 CDS 的起点与思考
date: 2026-06-27 16:30:00
tags:
  - 无线传感网
  - CDS
  - 预研究
categories:
  - CDS
  - 随笔
  - Roadmap
  - Open-Questions
mathjax: true
---

## 1. 预研究闭环：

> **明确问题定义 → 设计一个初步算法 → 做一个小实验 → 总结发现问题 → 再改算法。**

-----

## 2. 论文核心：

- 随机场景：节点 active / inactive，探测前未知；
- 探测模型：probe active 节点会揭示闭邻域；
- 原算法目标：构造 root 所在 active 连通分量的 CDS；
- 原算法不足：只保证 connected，不保证 fault-tolerant；
- 论文结论中明确提到：stochastic $(k,m)$-CDS 是未来方向。

切入点：

> 在 stochastic probing model 下，研究 fault-tolerant CDS 的自适应构造问题。

---

## 3. 试行版本

### 问题定义

```
输入：
G=(V,E)，每个节点 active 概率 p(v)，root r。

节点状态：
active / inactive，初始未知；probe active 节点揭示闭邻域；probe inactive 节点只揭示自身。

目标：
构造 root 所在 active 连通分量上的 2-connected CDS。

优化目标：
最小化 probing cost。
```

试用版本：
$$
(k,m)=(2,1)
$$
也即：

> 随机场景下的 2-connected CDS。

更具体地说：

> 原论文只要求构造 connected CDS，而现在考虑在相同 stochastic probing model 下进一步引入容错约束，使得构造出的虚拟骨干不仅连通，而且具备 2-connected 鲁棒性。

###  和原论文的区别

```
原论文：
只要求 connected CDS。

扩展：
要求 2-connected CDS，因此不仅要考虑支配和连通，还要考虑骨干内部割点、block、leaf block 等容错结构。

原论文选择节点：
根据 NWS(v) 和 NCS(v)。

想法：
在此基础上加入 FTs(v)，用来衡量节点或路径对 2-connected 增强的贡献。
```



----
## 4. 动态 adaptive


>在算法执行过程中，同时观察当前骨干的**支配状态、连通状态和容错状态**，
动态决定下一步 probe 哪个节点或哪条路径。

需要维护几个状态：

```
1. white 节点：未知
2. gray 节点：已知 active，但未 probe
3. black 节点：已 probe 且 active，当前骨干
4. removed 节点：已知 inactive

5. 当前骨干 C = black nodes
6. 当前 G[C] 的割点
7. 当前 G[C] 的 block / leaf block
8. 当前还有哪些 active 节点未被支配
9. 当前哪些结构不满足 2-connected
```

 6、7 为“容错状态”。

---

## 5. 设计一个动态评分函数（或者叫 动态收益函数）

原论文每一步选节点主要看：

$$
NW_S(v)+NC_S(v)
$$

其中：

- $NW_S(v)$：信息揭示收益；
- $NC_S(v)$：连接黑色分量收益。

对于容错，可以加入一个新指标，比如：

$$
FT_S(v)
$$

表示节点 $v$ 对容错结构的贡献。

那么**评分(收益)** 可以为：

$$
Gain(v)=\alpha\cdot NW_S(v)+\beta\cdot NC_S(v)+\gamma\cdot FT_S(v)
$$

考虑**概率和代价**后的选择指标（也即当前这个节点值不值得 **探测** -> **性价比**）：

$$
Score(v)=\frac{p(v)\cdot Gain(v)}{Cost(v)} = \frac{概率\cdot 收益}{探测成本}
$$

如果考虑路径增强，可以写：

$$
Score(P)=\frac{Pr(P)\cdot FT_S(P)}{Cost(P)}
$$

> 表示这条路径的**容错增强性价比**

其中：

$$
Pr(P)=\prod_{u\in P\cap white}p(u)
$$

表示路径中未知节点都 active 的概率。

第一个“研究点”：
 **在 stochastic 场景下，最短路径不一定最好，概率高、容错收益大的路径可能更合理（收益更大）。**

---

## 6. 初步算法框架

预研究阶段的算法框架（待完善）：

```
Adaptive Fault-Tolerant CDS Framework

1. 初始化 root r，probe r，r 变黑，揭示 N[r]
2. 维护当前黑色骨干 C
3. 每轮检查当前状态：
   - 是否还有未支配的已知 active 节点？
   - G[C] 是否已经 2-connected？
   - 是否存在割点或 leaf block？
4. 构造候选对象：
   - 候选节点 v：用于揭示新区域或连接骨干
   - 候选路径 P：用于连接 leaf block、减少割点、增强 2-connected
5. 对候选节点 / 路径计算评分：
   - 信息收益
   - 支配收益
   - 连通收益
   - 容错收益
   - 成功概率 ?
   - 探测代价
6. 选择评分最高的节点或路径进行 probe
7. 根据 probe 结果更新颜色、骨干、block 结构
8. 重复直到 root 所在 active 连通分量被支配，且 G[C] 满足 2-connected
```
----

## 7. 评价指标


1. probing cost：主动探测节点数 / 探测代价  (  probing cost  |S| ? )
2. backbone size：最终骨干节点数  （ $|C_{2CDS}|$ ）
3. success rate：成功构造 2-connected CDS 的比例 ( 成功次数 / 总的运行次数 ？ )
4. redundancy：相比普通 CDS 增加了多少节点 ( $|C_{2CDS}| - |C_{CDS}|  ？
\frac{|C_{2CDS}|}{|C_{CDS}|}$ ？)
5. running time：运行时间 （随机场景下的 CDS 与 2-CDS 构造时间）





















