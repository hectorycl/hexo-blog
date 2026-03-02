---
title: 19. KVStore 版本五 B+ 树并发删除：函数调度全流程
date: 2026-03-2 17:00:00
categories:
  - [KVStore, v5.0]  
tags:
  - kvstore
---

## B+ 树并发删除：函数调度全流程

#### 1. 入口与锁初始化：`bptree_delete`
这是整个操作的“指挥官”，它负责初始化环境。

 - **动作：**创建空栈`bptree_write_path path`。
 - **状态决策：**调用`bptree_find_leaf_delete_safe`。


#### 2.渗透加锁阶段：`bptree_find_leaf_delete_safe`
这是整个并发处理最核心的过滤环节。

 - **流程：**
    1. 锁住`root_lock` → 锁住`root->latch(WRLock)` → 释放`root_lock`。
    2. **循环向下：**锁住子节点`next->latch`。
    3. **安全判断：**
     - **若子节点“丰满”：**调用`bptree_unlock_all_in_path(path)`。这会释放当前节点以上的所有锁，并清空栈。大大提升了并发度，因为后续操作只锁定了这棵子树。
     - **若子节点“危险”：**不解锁，继续持有父锁，向下走（悲观策略）。
 - **结束状态：**返回叶子节点指针，此时`path`栈中记录了从“最后一个安全节点”到“叶子”的所有 WRLock。



#### 3.局部修改阶段：`bptree_delete_from_leaf`

- **动作：**在叶子写锁保护下，执行数组删除。
- **调度转向：**
    - **失败（Key 不存在）：**回到`bptree_delete`,执行其他`unlock_all`然后返回
    - **成功但无下溢：**同上。
    - **成功且有下溢（key_count < MIN_KEYS）:**调用`bptree_delete_fixup`。


#### 4. 修复调度中枢：`bptree_delete_fixup`
> 这个函数负责“弹栈”并发任务。

 - **关键动作：**`path->top--`(弹出当前节点，暴露父节点)
 - **任务分发：**
    - 若是叶子节点：调用`bptree_fix_leaf`
    - 若是内部节点：调用`bptree_fix_internal`

#### 5. 结构重组接力：`fix_leaf`/`fix_internal`
这是逻辑最复杂的部分，涉及兄弟节点的交互

 - **借键分支（Borrow）:**
    - 调用 `bptree_borrow_from_left/right_leaf/internal`。
    - **并发点：**借键只涉及局部3个节点（父，己，兄），修改完后直接`return`,不需要向上递归

 - **合并分支（Merge）：**
    - 调用`bptree_merge_leaf/internal`。
    - 调用`bptree_delete_from_internal(parent, idx)`。
    - **关键点：**合并会导致父节点少一个 Key
    - **递归决策：**调用`bptree_is_underflow(parent)`,如果父节点也撑不住了，**再次递归调用**`bptree_delete_fixup(tree, path, parent)`。

#### 6. 物理收尾与全局缩高：`bptree_delete` 尾部
> 当所有修复函数执行完毕，控制权回到`bptree_delete`。

 - 根节点缩高：
  1. 加`root_lock`
  2. 若`root->key_count == 0`,提升孩子。
  3. `path.nodes[0] = NULL`防止双重解锁。
  4. 释放`old_root`

 - **全线撤退：**调用`bptree_unlock_all_in_path(&path)`,释放所有剩余的写锁。


#### 调度流程与状态对照表
| **当前状态**      | **调用函数**                     | **并发影响范围**                                   |
| ----------------- | -------------------------------- | -------------------------------------------------- |
| **找到目标叶子**  | `find_leaf_delete_safe`          | 逐步缩小锁定的分支，释放祖先锁。                   |
| **叶子 Key 够多** | `delete_from_leaf`               | 仅锁定叶子及其最近的不安全祖先。                   |
| **左兄够借**      | `borrow_from_left`               | **局部锁**：锁定父、己、左兄，操作完即结束。       |
| **必须合并**      | `merge_leaf` + `delete_internal` | **扩散锁**：合并完成后，写锁压力向上传递至父节点。 |
| **根节点变空**    | `root_shrink`                    | **全局锁**：短暂持有 `root_lock` 切换根指针。      |