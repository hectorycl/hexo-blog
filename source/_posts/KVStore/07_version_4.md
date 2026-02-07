---
title: 07. KVStore 版本四 架构深度解析
date: 2026-02-07 17:00:00
categories:
  - [KVStore, v4.0]  
tags:
  - kvstore
---


# kvstore 架构深度解析

## 1. Recovery 层（恢复）

> 核心逻辑: 它是系统启动时的第一道防线

 - **流程：** `open` -> `load_snapshot`(加载基准数据) -> `replay_log`(重放增量日志)
 - **设计原因**：内存数据库必须在重启后恢复到宕机前的状态。先读快照是为了速度，后重放日志是为了数据完整性
 - **关键函数：**
  - `kvstore_recover()`: 总入口。
  - `kvstore_load_snapshot()`: 从磁盘加载静态数据；
  - `kvstore_replay_log()`: 逐条读取 WAL 并调用 `replay_put/del` 




## 2. Logging 层 （持久化/WAL）

> 核心逻辑：Write-Ahead Logging (WAL)。 在修改内存前，先写入磁盘。

 - **设计原因：**磁盘顺序写远快于随机写。只要日志在，数据就不会丢。
 - **关键函数：**
  - `kvstore_log_put() / kvstore_log_del()`:将操作序列化为二进制流并写入文件。
  - `kvstore_log_header():` 写入校验信息，防止文件损坏导致无法识别。

## 3. Execution 层（执行/原子性控制）

> 核心逻辑：它是 Public API 和内部实现之间的协调者

- **设计原因：**所有的写操作（PUT/DEL）都必须经过统一的“漏斗”。 它负责：1. 检查状态机; 2. 调用 Logging 持久化; 3. 调用 Apply 修改内存。

- **关键函数：**
 - `kvstore_exec_write():` **整个项目的灵魂**。它决定了一个操作是“先写日志再改内存”的原子过程。
 - `kvstore_state_allow():` 权限拦截，比如“正在恢复中”或“已损坏”时的拒绝写入。


## 4. Apply 层（内存映射）

> 核心逻辑：真正修改内存数据结构（B+ 树）的地方。

 - **设计原因：**区分 `internal` 和普通`apply`。 `internal` 通常不带锁或不触发状态检查，用于 Recovery 期间的高效回放。
 - **关键函数：**
  - `kvstore_apply_put_internal()`:纯粹的内存指针操作。
  - `kvstore_replay_put()`:被 Recovery 调用，跳过日志写入阶段，直接调用 Apply。


## 5. Maintenance 层（维护/压缩）

> 核心逻辑：解决日志无限增长的问题。

- **设计原因：**日志文件不能一直增加，否则恢复太慢，`Compaction` 通过将当前内存状态存为新快照并删除旧日志来释放空间。
- **关键函数：**
 - `kvstore_maybe_compact()`: 自动检查（基于阈值）。
 - `kvstore_compaction_internal()`: 执行合并，期间可能需要切换状态机防止并发冲突。
 - `snapshot_write_cb():` 回调函数，用于在遍历内存数据时写入磁盘。



## 6. State Machine 层 (状态机)

> 核心逻辑：管理系统的生存周期（Ready, Recovering, Compaction, Failed）。

- **关键函数：**
 - `kvstore_transit_state()`: 状态转移触发器。
 - `kvstore_enter_failed()`: 一旦发生不可逆的 IO 错误，立即进入失败态，保护数据。


## 7. Error 层（诊断）

> 核心逻辑：提供人类可读的上下文。
 
 - **关键函数：**
  - `kvstore_strerror()`: 将内部错误码转为字符串。
  - `crc32()`: 用于检测日志条目是否损坏。


------

## 核心写入流程

> 当调用`kvstore_put(key, val)`时，内部发生了什么？？

1. **API 入口：**`kvstore_put()` 调用 `kvstore_exec_write()`。

2. **状态检查：**`kvstore_state_allow()` 确认当前是 `READY` 状态

3. **持久化（Logging）：**`kvstore_log_put()` 将数据写入磁盘。

4. **修改内存（Apply）：**`kvstore_apply_put()` 更新内存索引。

5. **维护检查（Maintenance）：**`kvstore_maybe_compact()`判断是否需要压缩。




