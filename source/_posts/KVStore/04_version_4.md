---
title: 04. KVStore 版本三 总体架构 复盘 & 梳理
date: 2026-01-30 17:40:00
categories:
  - [KVStore, v3.0]  # 父分类，子分类
tags:
  - kvstore
---


# 一.总览级架构分层图
```
┌────────────────────────────────────┐
│ Public API Layer（对外接口层）      │
│ 参数校验 / 模式判断 / WAL / 调度   │
└───────────────▲────────────────────┘
                │
┌───────────────┴────────────────────┐
│ Apply Layer（应用层 / 状态变更层） │
│ 只负责：把操作应用到内存状态      │
└───────────────▲────────────────────┘
                │
┌───────────────┴────────────────────┐
│ Storage Engine（存储引擎层）       │
│ B+Tree 的插入 / 删除 / 遍历        │
└───────────────▲────────────────────┘
                │
┌───────────────┴────────────────────┐
│ WAL / Snapshot（持久化层）         │
│ 日志 / 快照 / replay / 校验        │
└────────────────────────────────────┘
```
>  越往上越“像业务”，越往下越“像工具”。

>  上面的模块**调用**下面的模块

# 二.函数归类
## ① Public API Layer（对外接口层）
**对用户 / main / 测试程序（test_kvstore）可见、做参数校验、判断是否只读（read-only）、写 WAL 、再调用 Apply Layer**

#### 函数列表

```
kvstore_create(const char *)
kvstore_destroy(kvstore *)

kvstore_insert(kvstore *, int, long)
kvstore_put(kvstore *, int, long)
kvstore_del(kvstore *, int)

kvstore_search(kvstore *, int, long *)

kvstore_debug_set_mode(kvstore *, kvstore_mode_t)
```
**kvstore_put / del 是对外写接口**


## ② Apply Layer（应用层 / 状态变更层）

> Apply Layer = 只做状态改变，不校验，不写日志

#### 函数列表
```
kvstore_apply_put(kvstore *, int, long)
kvstore_apply_del(kvstore *, int)
```
**由public API、replay、snapshot load 调用，内部会操作 B+ Tree、更新统计信息（op_counts等）**


## ③ Apply Internal（应用层内部 / 适配层）
> 给 Apply Layer 用，把 kvstore 逻辑，bptree 细节**解耦开**

> Apply Layer（应用层）

>  ↓(调用)

>  Apply Internal（贴近存储引擎）


## ④ Storage Engine（存储引擎层）
也就是 B+ Tree (bptree.c / h)
> 不在 kvstore 文件里，但逻辑上属于最底层
> 不知道 WAL, kvstore, snapshot


## ⑤ WAL / Log Layer（日志层）

**负责持久化“操作”而不是“状态”**

#### 函数列表
```
kvstore_open_log(kvstore *, const char *)
kvstore_log_put(kvstore *, int, long)
kvstore_log_del(kvstore *, int)

kvstore_log_header(const char *)
crc32(const char *)
kvstore_crc_check(const char *, const char *)
```
#### 架构理解
- kvstore_log_put / del 
    - Public API 调用
- crc32 / crc_check()
    - 数据完整性保障
- log_header
    - 文件级别元信息


## ⑥ Replay Layer（重放层 / 恢复层）

**崩溃恢复的核心**

#### 函数列表
```
kvstore_replay_log(kvstore *)
```
> replay = 只读 WAL + 调用 API 
> 物理层面：使用 fgets 把磁盘上已经存在的旧记录一行行读进内存，操作模式是 `w(Read)`, 而不是 `a+(Append)`
> 逻辑层面：只读状态下的程序，只能 `看`日志内容，不能在重放过程中又往里面写新东西。


### 正常写入流程（用户调用 / mian 调用）
**用户想存数据，必须走全套流程**

```
1. 通过 public API，调用应用层函数
int kvstore_put(kvstore* store, int key, long val) {
    ...
    // 1. 先写日志 (WAL)
    kvstore_write_log(store, "PUT", key, val); 
    // 2. 再调用内部函数修改内存
    return kvstore_apply_put(store, key, value);
}


2. 应用层函数调用应用层内部 / 适配层
static int kvstore_apply_put(kvstore* store, int key, long value) {
    ...
    return kvstore_apply_put_internal(store->tree, key, value, store->mode);
}


3.应用层内部实现对数据的操作
static int kvstore_apply_put_internal(bptree* tree, int key, long value, kvstore_mode_t mode) {
    ...
    return bptree_insert(tree, key, value);
}
```

### 重放流程（启动时调用） 
**重放时，绕过写日志的步骤，直接在内存里修改**
```
static int kvstore_replay_log(kvstore* store) {
    // ... 循环读取 line ...
    if (sscanf(line, "PUT %d %ld", &key, &val) == 2) {
        // 直接调用应用层内部函数
        // 这样就只改了内存，而不会触发额外的磁盘写入
        if 
            kvstore_apply_put_internal(store->tree, key, val);
        
        else 
            kvstore_apply_del_internal(store->tree, key);
    }
    // ...
}
```

## ⑦ Snapshot / Compaction（快照与压缩）
#### 函数列表
```
kvstore_compact(kvstore *)
kvstore_maybe_compact(kvstore *)

compact_write_cb(int, long, void *)

kvstore_create_snapshot(kvstore *)
kvstore_load_snapshot(kvstore *)

snapshot_write_cb(int, long, void *)
```
#### 架构理解
- callback:（难点、难理解 ！）
    - 给 B+ Tree 遍历使用
- snapshot:
    - 保存“状态”
- campact:
    - WAL 太大 → 压缩

## 辅助 / 工具层
```
kvsotre_strerror(int)
```
- 错误码 → 字符串， 不属于任何业务层
