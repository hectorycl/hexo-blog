---
title: 20. KVStore 版本五 B+树并发总结&梳理（一）
date: 2026-03-3 20:00:00
categories:
  - [KVStore, v5.0]  
tags:
  - kvstore
---

# B+树并发总结&梳理（一）


## 1 为什么要给 B+ 树加锁
在单线程环境下，B+ 树的插入、删除、查找操作可以顺序执行，不会出现任何问题。但是在实际系统中，例如数据库或者存储引擎，**多个线程可能同时访问同一棵 B+树**。如果没有任何并发控制机制，就会产生严重的问题。

简单来说：
> 加锁的目的，就是保证 B+ 树在多线程环境下的结构一致性和内存安全。


如果没有锁，会发生什么？
> 1. 并发插入导致结构损坏    -  数组越界等
> 2. 并发分裂导致树结构错误  -  指针问题导致段错误
> 3. 读写冲突               -  访问已释放的阶段（段错误）

最简单的方案：**全局锁**
给整棵树加一把锁：`pthread_mutex_t tree_lock`

每次操作：
```c++
pthread_mutex_lock(&tree_lock);
bptree_insert(tree, key);
pthread_mutex_unlock(&tree_lock);
```
这样可以保证：**同一时间只有一个线程操作 B+ 树**

优点：
> 实现简单，不易出错

缺点：
> 并发性能极差，只有一个线程在工作
所有线程都在等待锁。

**更好的方案：细粒度锁**
即`node-level lock`, 这样就允许
```
 Thread A 访问 左子树
 Thread B 访问 右子树
```
两者完全可以并行，这就是数据库中长用的策略：`Latch Coupling(Crabbing)`,即锁耦合。

----
## 2 核心设计：蟹行锁策略

#### 2.1 什么是蟹行锁
蟹行锁（Crabbing / Latch Coupling）是一种**B+树并发控制算法**
> 简单理解：像螃蟹走路一样，一只脚先踩住下一块石头，再松开上一块。

也就是说：
```
锁住父节点
    ↓
锁住子节点
    ↓
释放父节点
```
可以保证：
 - 结构安全
 - 锁粒度最小
 - 高并发

这也是几乎所有数据库实现B+树索引的方式，例如：
 - MySQL
 - PostgreSQL
 - InnoDB
都采用类似策略

#### 2.2 蟹行锁的核心思想
三条核心规则：

> 1. 访问节点前必须先锁住节点。
```c++
lock(node)
```

> 2. 在释放父节点锁之前，必须先锁住子节点。
```c++
lock(child)
unlock(parent)
```

> 3. 如果子节点**安全（safe node）**,就可以释放祖先锁。
安全的定义：对于**插入操作**
```c++
node->key_count < MAX_KEYS 
```
因为不会触发`split`,所以父节点不会被修改，可以安全释放**祖先锁**

这就是 **Safe Node Rule**。


#### 2.3 Path Stack
引入了一下结构：
```c++
typedef struct {
    bptree_node* nodes[HEIGHT_MAX];
    int top;
} bptree_write_path;
```
作用：**托管所有持锁节点**
path 栈记录：`root`→`internal`→`internal`→`leaf`
例如：path = [root, n1, n2, leaf]

好处：
 - 支持 split 向上递归
 - 统一释放锁，避免锁泄漏、死锁


#### 2.3 find_leaf_write_safe 的逻辑

代码：
```c++
/**
 * 悲观写路径查找
 *
 * 核心策略：蟹行锁
 *
 * @param
 *  tree：指向 B+ 树的指针
 *  key : 带插入的键值
 *  path: 用于托管路径锁的栈（外出分配）
 *  bptree_node*: 返回加锁后的叶子节点，失败返回 NULL
 */
static bptree_node* bptree_find_leaf_write_safe(bptree* tree, int key, bptree_write_path* path) {
    if (!tree) return NULL;

    // 重置路径栈
    path->top = 0;

    // 1. 获取全局 root 锁（仅保护 tree->root 指针在缩高/分裂时的原子性）
    pthread_mutex_lock(&tree->root_lock);  

    if (tree->root == NULL) {
        pthread_mutex_unlock(&tree->root_lock);  // 必须在 return 前解锁！
        return NULL;
    }

    bptree_node* curr = tree->root;

    // 锁住根节点（只加一次锁！）
    pthread_rwlock_wrlock(&curr->latch);  // 写操作拿锁
    pthread_mutex_unlock(&tree->root_lock);

    // 将根节点入栈
    path->nodes[path->top++] = curr;

    while (!curr->is_leaf) {
        // 查找下一个子节点索引
        int i = 0;
        while (i < curr->key_count && key >= curr->keys[i]) i++;
        bptree_node* next = curr->children[i];

        // 2. 锁住孩子, 并入栈
        pthread_rwlock_wrlock(&next->latch);

        // 先入栈，此时 path 保证了包含当前所有持锁节点
        path->nodes[path->top++] = next;

        // 3. 关键：判断孩子是否安全
        if (next->key_count < MAX_KEYS) {
            // 释放除 next 之外的所有祖先锁
            // 此时栈里有:[root, ..., parent, next]
            // 只需保留刚锁中的这个 next （它是 path->nodes[path->top - 1]）
            for (int j = 0; j < path->top - 1; j++) {
                pthread_rwlock_unlock(&path->nodes[j]->latch);
                path->nodes[j] = NULL;
            }

            // [路径重整]：将 next 设为栈底元素，重置 top
            // 这样 path 栈中就只剩下这一个“安全起始点”
            path->nodes[0] = next;
            path->top = 1;
        }

        curr = next;
    }

    // 此时 curr 是加锁的叶子，path 栈中存放了所有需要保持锁定状态的节点
    return curr;
}
```
**step 1:初始化 path**: `path->top = 0;`

**step 2:锁 root 指针**：`pthread_mutex_lock(&tree->root_lock)`,保护`tree->root`, 因为在 split root 时，root 可能改变。如果不加锁，线程可能看到**半更新状态**

**step 3:锁 root node**:
```c++
bptree_node* curr = tree->root;

pthread_rwlock_wrlock(&curr->latch);
```
写操作必须使用 write lock(`pthread_rwlock_wrlock`),保证只有一个 **writer**
然后**释放`root lock`**: `pthread_mutex_unlock(&tree->root_lock);` , 因为 `root pointer`已经稳定

**step 4:root 入栈**：`path->nodes[path->top++] = curr;`, 现在 `path = [root]`。

**step 5:向下遍历树**：`while (!curr->is_leaf)`

**step 6:选择 child**:
```c++
int i = 0;
while (i < curr->key_count && key >= curr->keys[i]) 
  i++;

bptree_node* next = curr->children[i];
```
找到下一层节点

**step 7: 锁 child**:`pthread_rwlock_wrlock(&next->latch);`
关键顺序：**先锁 chlid, 再考虑释放 parent**,否则会出现 race condition（竞态条件）。

**step 8:child 入栈**：`path->nodes[path->top++] = next;`,例如：`path = [root, internal, child]`。

**step 9:判断 safe node**:`if (next->key_count < MAX_KEYS)`,即如果 child 不会 **overflow**,说明不会 **split**,因此**父节点不需要修改！**

**step 10:释放祖先锁**：
```c++
for (int j = 0; j < path->top - 1; j++)
{
    pthread_rwlock_unlock(&path->nodes[j]->latch);
}
```
释放: `[root, parent, ...]`,只保留`next`

**step 11:重整 path**:
```c++
path->nodes[0] = next;
path->top = 1;
```
此时 path 变成了 `[next]`,这一步很重要！以后 split 只能影响 `next → leaf`,之前的祖先节点已经安全。

**step 12:继续向下**：`curr = next`,继续遍历

**step 13:到达叶子**：最终 `curr = leaf`,返回 `locked leaf`,此时 **path 栈中保存所有必须保持锁的节点**，例如：`[internal, leaf]`,如果 leaf split,**可以通过 path 向上修改parent!!**


## 3 难点攻克：并发下的节点分裂

如果没有正确的锁管理，可能发生：
> 情况1:父节点被其他线程修改,直接**段错误**

如果只锁 leaf,但是 parent 没有锁，可能发生：
> 情况2：父节点已经被释放锁

#### 3.1 数据库系统的解决方案就是 **Path Stack**

因为 **split 可能需要向上递归修改父节点**。
流程可能是：
```
leaf split
↓
parent insert
↓
parent overflow
↓
parent split
↓
grandparent insert
```
如果没有 path: 必须**重新查找 parent**,但 parent 可能已经改变，所以 **path 栈保存了整条安全路径**

#### 3.2 path 栈与蟹行锁配合
如果 child 是安全节点，不会 `split`,因此**父节点不会被修改**，于是可以释放祖先锁，path 只保留**可能发生 split 的节点**。 这叫：**锁托管（Lock Ownership Transfer）**

path 栈保证了三个关键性质：
1. split 向上递归安全
2. 不需要重新搜索 parent
3. 防止死锁、锁泄漏











