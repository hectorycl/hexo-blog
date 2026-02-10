---
title: 09. KVStore 版本四 2️⃣Public API — 基本操作
date: 2026-02-10 12:00:00
categories:
  - [KVStore, v4.0]  
tags:
  - kvstore
---

# 2️⃣Public API — 基本操作

## 1. 函数集合
```c
int kvstore_put();
int kvstore_del();
int kvstore_create_snapshot();
```


## 2. 函数复盘 + 规范注释
### 2.1 `int kvstore_put()`
```c
/**
 * kvstore_put - 向存储引擎写入或更新一个键值对
 *
 * @details
 * 该函数是“逻辑入口”。不做对数据进行实际的 B+ 树插入，而是将操作封装后，
 * 转发至 Execution 的原子写入路径
 *
 *
 * [关键不变量]
 *  - 所有用户写操作，必须先成功写 WAL，才能修改内存
 *  - Apply Layer（内部 put_internal） 只负责“状态变更”，不做权限和持久化判断
 *
 * exec 只给在线写用，replay 只做 apply
 */
int kvstore_put(kvstore* store, int key, long value) {
    // 1. 基础安全校验
    if (!store || !key || !value)
        return KVSTORE_ERR_NULL;

    // 2. 进入原子执行路径
    return kvstore_exec_write(store, KVSTORE_OP_PUT, key, value);
}
```

### 2.2 `int kvstore_del()`
```c

/**
 * kvstore_del - 对外 DEL 接口（删除一个 key）
 *
 * 设计原则：
 *  1. 对外 API 永不直接修改内存数据结构
 *  2. 所有写操作必须遵循：WAL → apply 的顺序
 *
 * 注意：
 *  - 若 WAL 写入成功但是 apply 尚未执行时系统崩溃，
 *    启动时可通过 replay WAL 恢复该操作
 *  - apply 层不做权限与模式判断，只负责“该内存”
 *
 * 设计理解：
 *  - kvstore_del 本身并不删除数据，真正删除发生在 apply 层。
 */
int kvstore_del(kvstore* store, int key) {
    // 1. 参数合法性校验
    if (!store || !key)
        return KVSTORE_ERR_NULL;

    // 2. 逻辑删除的封装
    return kvstore_exec_write(store, KVSTORE_OP_DEL, key, 0);
}
```

### 2.3 `int kvstore_create_snapshot()`
```c

/**
 * kvstore_create_snapshot - 创建内存状态的快照
 *
 * @details
 *  - 采用 tmp 文件 + rename 的原子替换策略
 *  - 使用 B+ Tree scan + callback 解耦策略 （还要重点学习）
 *  - 显示 flush + fsync 保证落盘
 */
static int kvstore_create_snapshot(kvstore* store) {
    if (!store) return KVSTORE_ERR_NULL;

    // 先写 tmp, 成功后再原子替换
    const char* tmp = "data.snapshot.tmp";
    const char* snap = "data.snapshot";

    // 1. 打开临时快照文件
    FILE* fp = fopen(tmp, "w");
    if (!fp) {
        perror("fopen snapshot");
        return KVSTORE_ERR_INTERNAL;
    }

    // 2. 遍历 B+ Tree, 写出所有 key-value
    if (bptree_scan(store->tree, snapshot_write_cb, fp) != 0) {
        fclose(fp);
        return KVSTORE_ERR_INTERNAL;
    }

    // 强制刷新到磁盘，保证 crash-safe
    fflush(fp);
    fsync(fileno(fp));
    fclose(fp);

    // 4. 原子替换正式快照文件
    if (rename(tmp, snap) != 0) {
        perror("rename snapshot");
        return KVSTORE_ERR_NULL;
    }
    return KVSTORE_OK;
}
```
