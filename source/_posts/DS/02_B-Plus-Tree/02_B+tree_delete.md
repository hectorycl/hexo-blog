---
title: 02. B+ 树删除复盘
date: 2026-01-18 10:00:00
categories:
  - [DS, B-Plus-Tree]  # 重点：这表示 DS 是父分类，B-Plus-Tree 是子分类
tags:
  - 数据结构
  - B+ 树
---


# B+ 树删除实现：从 Bug 到稳定架构的完整复盘

> 关键词：B+ 树、删除操作、下溢（underflow）、borrow / merge、fixup、架构设计

这篇文章不是一份“标准答案式”的 B+ 树删除教程，而是一份**真实的实现复盘**：

- 为什么“只删一个 key”，树却会整体崩掉？
- 为什么 merge 看似成功，parent 却被删空？
- 为什么修复逻辑一定要拆成 `fix_leaf / fix_internal`？

如果你正在**手写 B+ 树**，尤其是 C/C++ 版本，那么这篇文章基本可以帮你避开 90% 的坑。

---

## 一、B+ 树删除：真正的难点在哪？

很多教程会告诉你：

> 删除 = 从叶子删 → 不够就借 → 借不到就合并 → 向上递归

**但真正的难点不在算法步骤，而在“职责边界”**。

也就是说：

- 谁负责修改 parent？
- merge 做到哪一步为止？
- underflow 是谁检测、谁修复？

如果这些问题不想清楚，代码一定会出现：

- parent key 被删两次
- root 被误判为空
- children 指针错位
- 树结构“看起来还能跑，但已经坏了”

---

## 二、基础定义（以 Order = 6 为例）

```c
#define ORDER 6
#define MAX_KEYS (ORDER - 1)      // 5
#define MAX_CHILDREN ORDER        // 6
#define MIN_KEYS ((ORDER + 1) / 2 - 1) // 2
```

- **叶子节点 / 内部节点**：最少 `MIN_KEYS`
- **root 允许特例**：可以少于 MIN_KEYS

---

## 三、总体设计原则（非常重要）

### 👉 核心原则：单一职责

在最终稳定版本中，我采用的是：

> **方案 A：merge 只合并，不修改 parent**

也就是说：

| 模块                 | 只做什么        | 不做什么        |
| -------------------- | --------------- | --------------- |
| delete_from_leaf     | 删除 key        | 不修结构        |
| fix_leaf             | borrow / merge  | 不删 parent key |
| merge_leaf           | 合并两个叶子    | 不动 parent     |
| delete_from_internal | 删除 parent key | 不合并节点      |
| fix_internal         | 修复 internal   | 不越权删除      |

📌 **所有 parent 的 key / child 删除，只能在一个地方发生**。

---

## 四、删除总入口：`bptree_delete`

```c
int bptree_delete(bptree* tree, int key) {
    bptree_node* leaf = bptree_find_leaf(tree, key);
    if (!leaf) return BPTREE_ERR;

    if (bptree_delete_from_leaf(leaf, key) != BPTREE_OK)
        return BPTREE_ERR;

    // root 是叶子，允许为空
    if (leaf == tree->root) return BPTREE_OK;

    if (bptree_is_underflow(tree, leaf)) {
        bptree_fix_leaf(tree, leaf);
    }
    return BPTREE_OK;
}
```

**关键点：**

- delete 只负责“触发修复”
- 不关心 borrow / merge 细节

---

## 五、下溢判断：`bptree_is_underflow`

```c
int bptree_is_underflow(bptree* tree, bptree_node* node) {
    if (node == tree->root) return BPTREE_OK;
    return node->key_count < MIN_KEYS;
}
```

📌 **root 是特例，千万别忘**。

---

## 六、叶子修复：`bptree_fix_leaf`

### 修复顺序（必须这样）：

1. borrow left
2. borrow right
3. merge

```c
void bptree_fix_leaf(bptree* tree, bptree_node* leaf) {
    if (bptree_borrow_from_left_leaf(leaf) == BPTREE_OK) return;
    if (bptree_borrow_from_right_leaf(leaf) == BPTREE_OK) return;

    int parent_key_idx = -1;

    bptree_node* left  = bptree_get_left_sibling(leaf);
    bptree_node* right = bptree_get_right_sibling(leaf);

    bptree_node* merge_node = NULL;

    if (left) {
        merge_node = left;          // left + leaf
    } else if (right) {
        merge_node = leaf;          // leaf + right
    } else {
        return; // 非 root 不可能
    }

    if (bptree_merge_leaf(tree, merge_node, &parent_key_idx) != BPTREE_OK)
        return;

    // merge 之后，只修 parent
    if (merge_node->parent && bptree_is_underflow(tree, merge_node->parent)) {
        bptree_fix_internal(tree, merge_node->parent);
    }
}
```

### ⭐ 一个关键细节

> **merge_leaf 的参数必须是“左节点”**

因此在 fix_leaf 中必须显式判断左右兄弟。

---

## 七、叶子合并：`bptree_merge_leaf`

```c
int bptree_merge_leaf(bptree* tree, bptree_node* left, int* out_parent_key_idx) {
    if (!left || !left->parent || !out_parent_key_idx) return BPTREE_ERR;

    bptree_node* parent = left->parent;
    int idx = bptree_find_child_index(parent, left);
    if (idx < 0 || idx >= parent->key_count) return BPTREE_ERR;

    bptree_node* right = parent->children[idx + 1];
    if (!right) return BPTREE_ERR;

    if (left->key_count + right->key_count > MAX_KEYS)
        return BPTREE_ERR;

    // 拷贝数据
    int old = left->key_count;
    for (int i = 0; i < right->key_count; i++) {
        left->keys[old + i]   = right->keys[i];
        left->values[old + i] = right->values[i];
    }

    left->key_count += right->key_count;
    left->next = right->next;

    *out_parent_key_idx = idx;
    return BPTREE_OK;
}
```

📌 **注意：这里不修改 parent，不 free right**。

---

## 八、Internal 修复：`bptree_fix_internal`

逻辑和 leaf 完全对称：

```c
void bptree_fix_internal(bptree* tree, bptree_node* node) {
    if (bptree_borrow_from_left_internal(node) == BPTREE_OK) return;
    if (bptree_borrow_from_right_internal(node) == BPTREE_OK) return;

    int parent_key_idx = -1;
    bptree_node* left = bptree_get_left_sibling(node);

    bptree_node* merge_node = left ? left : node;

    if (bptree_merge_internal(tree, merge_node, &parent_key_idx) != BPTREE_OK)
        return;

    if (merge_node->parent && bptree_is_underflow(tree, merge_node->parent)) {
        bptree_fix_internal(tree, merge_node->parent);
    }
}
```

---

## 九、一个血泪教训：为什么 parent 不能在 merge 里删？

我一开始在 `merge_internal` 里写了：

```c
parent->keys[...] 移动
parent->children[...] 移动
parent->key_count--
```

结果是：

- parent 被删了 **两次**
- root 被错误 shrink
- 整棵树“看起来能跑，但结构已经坏了”

💡 **最终结论：**

> parent 的 key 删除，必须集中在 `delete_from_internal` 里完成。

---

## 十、总结（如果你只记住三点）

1️⃣ **merge 只合并，不动 parent**  
2️⃣ **fix 只调度，不做具体删除**  
3️⃣ **parent 的 key 删除只能有一个入口**

如果你能做到这三点，B+ 树删除一定是稳定的。

---

## 后记

这份实现不是抄教材，而是一步一步踩坑踩出来的。

如果你也在手写 B+ 树，希望这篇文章能少让你 debug 几个晚上。

如果你愿意，我可以下一篇帮你整理：

- 🔥 B+ 树插入的“同一套设计哲学”
- 🔥 一张完整的删除流程调用图
- 🔥 或者把这套代码改成“竞赛 / 面试版精简实现”

欢迎继续交流。

