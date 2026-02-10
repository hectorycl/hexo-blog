---
title: 10. KVStore 版本四 3️⃣ Public API — 高级接口
date: 2026-02-10 14:00:00
categories:
  - [KVStore, v4.0]  
tags:
  - kvstore
---


# 3️⃣ Public API — 高级接口

## 1. 函数集合
```c
int kvstore_search();
int kvstore_compact();
```

## 2. 函数复盘 + 规范注释

### 2.1 `int kvstore_search()`
```c

/**
 * kvstore_search - 对外查询接口（存储引擎中检索指定键的值）
 *
 * 设计原则：
 *  1. 这是 kvstore 的“读路径（read path）”
 *  2. 读操作允许在只读模式（readonly / replay 模式）下执行
 *  3. 实际的数据查找完全交给底层索引结构（B+ 树）
 *
 * 注意：
 *  - search 本身不关心系统模式（NORMAL / READONLY / REPLAY）
 *  - 只要系统中存在索引结构，就可以安全查询
 */
int kvstore_search(kvstore* store, int key, long* value) {  // kvstore_get
    // 1. 防御性
    if (!store || !store->tree) return KVSTORE_ERR_NULL;

    // 2.状态机校验
    if (!kvstore_state_allow(store->state, KVSTORE_OP_GET)) {
        return KVSTORE_ERR_INTERNAL_STATE;
    }

    // 3. 委派查询（将任务交给数据结构层[Apply 层]）
    return bptree_search(store->tree, key, value);
}
```

### 2.2 `int kvstore_compact()`
```c

/**
 * kvstore_compact - 统筹日志压缩流程，负责运行态的平滑切换
 *
 * @details
 *  - 通过 enter/exit 函数实现逻辑锁，确保快照期间把内存索引不被修改
 *
 * 设计模式：包裹模式 （保存-修改-还原）
 *  1. 现场保护：在执行前通过 prev 变量备份当前的系统状态
 *  2. 状态锁定：调用 enter_compaction 切换为只读，防止压缩期间数据被篡改
 *  3. 核心执行：将压缩工作委派给内部函数 compact_internal 处理
 *  4. 现场恢复：无论压缩成功与否，最终都通过 exit_compaction 恢复之前的系统状态
 *
 *  不会导致数据库永久锁死在只读模式，系统几倍自我恢复能力
 *
 * [分层好处]
 *  - kvstore_compact 关注的是流程控制；kvstore_compact_internal 关注的是文件 IO
 *
 * 体现了面向切面编程（AOP）的思想
 */
int kvstore_compact(kvstore* store) {
    int ret;
    kvstore_state_t prev;

    // 1. 安全性检查
    if (!store) return KVSTORE_ERR_NULL;

    // 2. 状态保存
    prev = store->state;

    // 3. 前置处理：锁定状态（此时禁止写入）
    ret = kvstore_enter_compaction(store);
    if (ret != KVSTORE_OK)
        return ret;

    // 4. 核心执行 - 执行具体的快照生成和 WAL 阶段逻辑
    ret = kvstore_compact_internal(store);

    // 5. 后置处理：解锁恢复
    // 无论压缩成功与否，都必须调用此函数将状态还原（prev）
    kvstore_exit_compaction(store, prev);

    return ret;
}
```
