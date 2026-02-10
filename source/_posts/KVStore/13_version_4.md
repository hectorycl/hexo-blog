---
title: 13. KVStore 版本四 6️⃣ WAL 写入
date: 2026-02-10 14:30:00
categories:
  - [KVStore, v4.0]  
tags:
  - kvstore
---


# 6️⃣ WAL 写入

## 1. 函数集合
```c
static int kvstore_log_put();
static int kvstore_log_del();
```

## 2. 函数复盘 + 规范注释

### 2.1 `static int kvstore_log_put()`
```c

/**
 * kvstore_log_put - 将写入操作持久到 WAL 日志文件
 *
 * 设计定位：
 *  - WAL 层函数
 *  - 只负责“把一次 PUT 操作顺序写入磁盘日志”
 *  - 不修改内存结构，不调用 B+ 树
 *
 * 日志格式
 *      PUT <key>|<crc32>\n
 *
 * 执行流程：
 *  1. 校验日志文件是否已打开
 *  2. 构造日志内容字符串
 *  3. 计算 CRC32 校验值
 *  4. 写入日志文件并强制刷盘
 *  5. 更新日志统计信息（大小 / 操作次数）
 *
 * snprintf(): 将变量格式化为字符串，并安全写入缓冲区
 *
 */
static int kvstore_log_put(kvstore* store, int key, long value) {
    // 1. 安全边界检查
    if (!store->log_fp) return KVSTORE_ERR_NULL;

    // 2. 序列化 - 将 操作类型，key 和 value 格式化为字符串
    char buf[128];
    snprintf(buf, sizeof(buf), "PUT %d %ld", key, value);

    // 3. 生成“数据指纹”
    uint32_t crc = crc32(buf);

    // 4. 物理写入与格式编排
    int bytes = fprintf(store->log_fp, "%s|%u\n", buf, crc);

    // 5. 强制落地
    fflush(store->log_fp);
    int fd = fileno(store->log_fp);
    if (fd >= 0)
        fsync(fd);

    // 6. 元数据更新
    store->ops_count += 1;
    store->log_size += bytes;

    return KVSTORE_OK;
}

```

### 2.2 `static int kvstore_log_del()`
```c

/**
 * kvstore_log_del - 将删除操作持久化到 WAL
 *
 * 删除在磁盘上不是物理抹除，而是追加一条 DEL 指令
 *
 * 日志格式：
 *      DEL <key>|<crc32>\n
 *
 */
static int kvstore_log_del(kvstore* store, int key) {
    // 1. 安全边界检查
    if (!store->log_fp) return KVSTORE_ERR_NULL;

    // 2. 序列化
    char buf[128];
    snprintf(buf, sizeof(buf), "DEL %d", key);

    // 3. 生成数据指纹
    uint32_t crc = crc32(buf);

    // 4. 物理写入
    int bytes = fprintf(store->log_fp, "%s|%u\n", buf, crc);

    // 5. 强制落地
    fflush(store->log_fp);  // fsync(fileno(store->log_fp)); // 强制操作系统把数据写进物理磁盘

    int fd = fileno(store->log_fp);
    if (fd >= 0)
        fsync(fd);

    // 6. 元数据统计
    store->log_size += bytes;
    store->ops_count += 1;

    return KVSTORE_OK;
}

```

