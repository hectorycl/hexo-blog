---
title: 15. KVStore 版本四 8️⃣ 内存变更 Apply 层
date: 2026-02-10 15:10:00
categories:
  - [KVStore, v4.0]  
tags:
  - kvstore
---


# 8️⃣ 内存变更 Apply 层

## 1. 函数集合
```c
static int kvstore_apply_put();
static int kvstore_apply_del();
static int kvstore_apply_put_internal();
static int kvstore_apply_del_internal();
```

## 2. 函数复盘 + 规范注释

### 2.1 `static int kvstore_apply_put()`
```c
/**
 * kvstore_apply_put - 将数据变更最终应用到内存索引（B+ 树）
 *
 *  - 该函数是纯净的。它假设所有的准入控制（状态，日志，权限）已经在上层完成
 *  - 它同时用于普通写路径和 WAL 重放
 */
static int kvstore_apply_put(kvstore* store, int key, long value) {
    // 1. 安全基石
    if (!store) return KVSTORE_ERR_NULL;

    // 2. 索引就绪检查
    if (!store->tree) return KVSTORE_ERR_INTERNAL_STATE;

    // 3. 跨层调用
    // 不关心文件，不关心 CRC，不关心状态机，只把数据塞进内存索引
    // 这种“瘦接口”设计可以非常容易把 B+ 树 换成跳表（SkipList）或红黑树，而不需要改动上层逻辑
    return kvstore_apply_put_internal(store->tree, key, value);
}

```

### 2.2 `static int kvstore_apply_del()`
```c

/**
 * kvstore_apply_del - 执行内存索引的删除操作
 *
 * @details
 *  - 带 apply_ 前缀的函数通常意味着：“这只是内存操作”。不包含写日志、不包含状体检查
 *  - 调用这个函数是不会产生磁盘 IO 的
 */
static int kvstore_apply_del(kvstore* store, int key) {
    // 1. 安全性检查
    if (!store || !store->tree) return KVSTORE_ERR_NULL;

    // 2. 委派执行 - 原子性与封装
    return kvstore_apply_del_internal(store->tree, key);
}
```

### 2.3 `static int kvstore_apply_put_internal()`
```c
/**
 * kvstore_apply_put_internal
 *  - 真正的 B+ 树写入操作与语义转换
 *
 * @details
 * - 该函数是 KV 引擎与 B+ 树算法库的“粘合剂”，负责返回底层算法返回细粒度状态，
 *   并将其归一化为存储引擎的标准返回码（即将 KV 错误码体系与底层 B+ 树的错误码体系
 *   进行解耦与转换， 适配器模式 [Adapter Pattern] 的微观实现）
 * - 保护了底层树的结构完整性，所有对树的修改必须通过此翻译层
 */
static int kvstore_apply_put_internal(bptree* tree, int key, long value) {
    // 1. 最后的屏障
    if (!tree) return KVSTORE_ERR_NULL;

    // 2. 调用底层算法引擎 - 纯粹是数据结构操作
    int ret = bptree_insert(tree, key, value);

    // 3. 作物吗映射（Error Mapping）
    switch (ret) {
        case BPTREE_OK:
        case BPTREE_UPDATED:
            // 无论是新插入，还是更新旧值，对 KV 业务来说都是“写入成功”
            return KVSTORE_OK;

        case BPTREE_ERR:
            return KVSTORE_ERR_INTERNAL;

        default:
            // 兜底：底层处理器可能出现的未预期状态
            return KVSTORE_ERR_INTERNAL;
    }
}

```

### 2.4 `static int kvstore_apply_del_internal()`
```c
/**
 * kvstore_apply_del_internal
 *  - 执行 B+ 树物理删除操作
 */
static int kvstore_apply_del_internal(bptree* tree, int key) {
    // 1. 最后的防线
    if (!tree) return KVSTORE_ERR_NULL;

    // 2. 直接委派
    return bptree_delete(tree, key);
}
```


