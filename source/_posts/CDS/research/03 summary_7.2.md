---
title: 随机容错 CDS 的相关定义
date: 2026-07-2 18:00:00
tags:
  - 无线传感网
  - CDS
categories:
  - CDS
  - paper_5
  - summary
mathjax: true
---

# 一、工作内容

概括：

> 搭建了路径版本的基础工程，并完成了从“随机图生成 → 初始 CDS 构造 → 骨干割点分析 → leaf block 识别 → 候选增强路径生成”的完整流程。

也就是说，现在代码已经可以做到：

```
1. 生成随机无线传感网图
2. 随机确定节点 active / inactive 状态
3. 筛选 root 所在活跃连通分量是 2-connected 的测试场景
4. 构造一个普通 CDS 作为初始骨干 C
5. 检查 C 是否是 CDS、是否 2-connected
6. 找出 G[C] 中的割点
7. 找出 G[C] 中的 leaf block
8. 对每个 leaf block 生成候选增强路径
```

目前已经跑出了合理结果：

```
root 活跃分量是 2-connected
初始 C 是 CDS
初始 C 不是 2-connected
候选增强路径成功生成
```

这说明路径版本的前半部分已经打通了。

------

# 二、改动文件？

主要涉及这几个文件：

```
adaptive_FT_path/
├─ config.py
├─ graph_model.py
├─ connectivity_check.py
├─ adaptive_probe.py
├─ path_augmentation.py
└─ main.py
```
----

# 三、效果

今天最终达到效果：

## 1. 找到了可行随机场景

输出里：

```
G[target_nodes] 是否 2-connected: True
```

说明 root 所在活跃分量本身具备构造 2-CDS 的可行性。

------

## 2. 构造出了普通 CDS

输出里：

```
C 是否是 CDS: True
```

说明初始骨干 `C` 已经满足普通随机 CDS 的要求：

```
连通 + 支配 root 活跃分量
```

------

## 3. 初始 C 还不是 2-connected

输出里：

```
G[C] 割点数量: 5
G[C] 是否 2-connected: False
```

说明这个初始 CDS 还没有容错能力，需要路径增强。

------

## 4. 找到了 leaf block

输出里：

```
G[C] leaf blocks: [[28, 58], [6, 59], [0, 43]]
```

说明能识别当前骨干中的薄弱边缘结构。

------

## 5. 成功生成候选增强路径

最终生成了 15 条候选路径，例如：

```
候选路径 2:
leaf block = [28, 58]
u = 28, v = 6
path = [28, 2, 6]
inner_nodes = [2]
新增节点数量 = 1
```

这说明：

> 路径版本已经能为当前 CDS 找到可能消除割点的绕行路径。




> 今天完成了路径版本的前半部分：从随机活跃图中**构造初始 CDS**，并成功**识别**当前 CDS 的 leaf block 和候选增强路径；

> 下一步要把这些候选路径**接入 adaptive probing**，让路径上的未知节点通过**探测**确认后再加入骨干。











