---
title: 16. KVStore 版本四 9️⃣ Snapshot / Compaction
date: 2026-02-10 15:00:00
categories:
  - [KVStore, v4.0]  
tags:
  - kvstore
---




# 9️⃣ Snapshot / Compaction

## 1. 函数集合

```c
static void kvstore_maybe_compact();
int kvstore_compact();
static int kvstore_compact_internal();

static int compact_write_cb();
static int snapshot_write_cb();
```

## 2. 函数复盘 + 规范注释

### 2.1 `static void kvstore_maybe_compact()`

```c

/**
 * kvstore_maybe_compact - 监控系统负载并自动触发日志压缩
 *
 * 采取基于容量（Size）和频率（Ops）的双重触发机制策略。
 *
 * 设计说明：
 *  - compaction 属于维护行为，不影响 PUT / DEL 正确性
 *  - 即使 compaction 失败，WAL 仍然可保证数据安全
 */
static void kvstore_maybe_compact(kvstore* store) {
    // 1. 安全校验
    if (!store) return;

    // 2. 阈值判断：多维度监控
    if (store->log_size >= KVSTORE_MAX_LOG_SIZE ||
        store->ops_count >= KVSTORE_MAX_OPS) {
        // 3. 执行压缩逻辑
        int rc = kvstore_compact(store);

        // 4. 状态重置与 Epoch 切换
        if (rc == KVSTORE_OK) {
            // 旧的 WAL 已经被清理，系统进入一个新的“纪元 [Epoch] ”
            store->log_size = 0;
            store->ops_count = 0;
        } else {
            // 5. 容错处理
            // 压缩失败（磁盘写保护不足或临时空间不足），不重置计数器。下次 PUT 再触发此操作
            // 体现系统的“自愈”意图
            fprintf(stderr, "[WARN] kvstore compaction failed, will retry later\n");
        }
    }
}

```

### 2.2 `static int kvstore_compact_internal()`

```c

/**
 * kvstore_compact_internal - 执行物理层面的日志压缩与重组
 *
 * 核心思想：
 *  - 将当前内存中的完整状态（B+ 树） 重写为一个新的 WAL
 *  - 丢弃历史冗余日志，降低 replay 成本
 *
 * 崩溃安全性保证（crash-safe）: → crash consistency（崩溃一致性）
 *  - 使用临时文件 + rename 的原子替换语义
 *  - 任意时刻宕机，磁盘上要么是旧 WAL,要么是新的 WAL
 */
static int kvstore_compact_internal(kvstore* store) {
    if (!store) return KVSTORE_ERR_NULL;

    const char* tmp_path = "data.compact";
    const char* data_path = store->log_path;

    /* 1. 创建临时 WAL 文件 */
    FILE* fp = fopen("data.compact", "w");
    if (!fp) {
        perror("fopen compact");
        return KVSTORE_ERR_IO;
    }

    /* 2. 写 WAL Header */
    fprintf(fp, "%s\n", KVSTORE_LOG_VERSION);

    /* 3.全量扫描内存状态，生成新的 WAL */
    if (bptree_scan(store->tree, compact_write_cb, fp) != 0) {
        fclose(fp);
        unlink(tmp_path);
        return KVSTORE_ERR_INTERNAL;
    }

    /* 4. 强制物理落盘 */
    fflush(fp);
    fsync(fileno(fp));
    fclose(fp);

    /* 5. 原子替换旧 WAL */
    FILE* old_fp = store->log_fp;
    store->log_fp = NULL;

    if (rename(tmp_path, data_path) != 0) {
        perror("rename");
        store->log_fp = old_fp;  // 回滚
        return KVSTORE_ERR_INTERNAL;
    }
    fclose(old_fp);

    /* 6. 重新打开 WAL 文件 */
    store->log_fp = fopen(data_path, "a+");
    if (!store->log_fp) {
        perror("reopen data.log");
        return KVSTORE_ERR_INTERNAL;
    }

    /* 7. 创建 snapshot  */
    kvstore_create_snapshot(store);

    return KVSTORE_OK;
}
```

### 2.3 `static int compact_write_cb()`

```c

/**
 * compact_write_cb - B+ 树扫描回调函数，负责将村花记录写入压缩日志
 *
 * @details
 *  - 该函数实现了“幂等化归档”。不记录历史过程（A->B->C）,只记录最终结果（C）
 *
 */
static int compact_write_cb(int key, long value, void* arg) {
    // 1. 类型安全还原
    FILE* fp = (FILE*)arg;

    // 2. 序列化（Serialization）
    char payload[128];
    snprintf(payload, sizeof(payload), "PUT %d %ld", key, value);

    // 3. 完整性重塑 - 每一行新日志都必须重新计算 CRC
    uint32_t crc_val = crc32(payload);

    // 4. 标准化格式输出 - 严格遵循格式: Payload | CRC\n
    if (fprintf(fp, "%s|%u\n", payload, crc_val) < 0) {
        return KVSTORE_ERR_INTERNAL;
    }

    return KVSTORE_OK;
}
```

### 2.4 `static int snapshot_write_cb()`
```c

/**
 * snapshot_write_cb - B+ 树扫描回调函数，专门用于生成持久化快照
 *
 * @details
 *  - 采用与 WAL 相同的文本格式，降低系统复杂性，
 *    使得快照可以被是为一个“预压缩的日志文件”。
 *  - snapshot 仅用于冷启动加速，不参与数据正确性保证
 *
 * @note
 *  compact_write_cb 的目的是更新 WAL,未来可以在 WAL 中记录更复杂的元数据（时间戳）
 *  snapshot_write_cb 的目的是数据备份，未来可以把它改成二进制格式以节省空间
 *
 *
 */
static int snapshot_write_cb(int key, long value, void* arg) {
    // 1. 类型转换：从通用上下文还原文件句柄
    FILE* fp = (FILE*)arg;

    // 2. 构造操作语义（Serialization）
    char payload[128];
    snprintf(payload, sizeof(payload), "PUT %d %ld", key, value);

    // 3. 计算 CRC 校验码
    uint32_t crc_val = crc32(payload);

    // 4. 物理写入
    fprintf(fp, "%s|%u\n", payload, crc_val);

    return KVSTORE_OK;
}
```
