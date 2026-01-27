---
title: 03. KVStore 版本三
date: 2026-01-27 22:16:00
categories:
  - [KVStore, v3.0]  # 父分类，子分类
tags:
  - kvstore
---


# 版本三 日志压缩（Compaction）与可恢复存储引擎

## feat; 实现 v3 核心架构 -- WAL、CRC、快照恢复与日志压缩

## V3 的目标： 引入完整的存储引擎架构

### V3 引入的核心机制

1. **WAL（Write-Ahead Logging）**
2. **CRC 校验，防止脏日志**
3. **Snapshot（快照）**
4. **日志压缩（Compaction）**

### V3 引入的核心机制

1. **WAL（Write-Ahead Logging）**
2. **CRC 校验，防止脏日志**
3. **Snapshot（快照）**
4. **日志压缩（Compaction）**

```
内存：
    B+ Tree（完整最新状态）

磁盘：
    snapshot 文件     -> 某一时刻的全量状态
    WAL 文件          -> snapshot 之后的增量操作
```

启动流程变为：
```
1. 读取 snapshot
2. 构建 B+ 树
3. replay snapshot 之后的 WAL
```

## WAL（Write-Ahead Log）

### 为什么还需要 WAL？

即使有 snapshot，也必须保证：

> **任何一次修改，必须先落盘，再修改内存**

否则一旦进程崩溃：

- 内存已变
- 磁盘未写
- 数据不一致

### V3 中 WAL 的角色

- 所有 `PUT / DEL`：
  1. 先写 WAL
  2. 再修改 B+ 树
- WAL 是**崩溃恢复的唯一可信来源**

## CRC：保证日志是“可信的”

### 问题背景

日志写到一半程序崩溃，可能导致：

- 行被截断
- 内容被破坏
- replay 时产生脏数据

### V3 的做法

每一条 WAL 记录都包含：

```
[CRC] [TYPE] [KEY] [VALUE]
```

启动 replay 时：

- 逐条校验 CRC
- 一旦发现 CRC 错误：
  - 停止 replay
  - 丢弃后续日志

> **宁可丢数据，也不写错数据**


## Snapshot（快照）

### 什么是 Snapshot？

Snapshot 是：

> **某一时刻 B+ 树完整状态的磁盘映像**

它不是 WAL 的替代，而是 WAL 的“截断点”。

------

### Snapshot 的作用

- 缩短启动时间
- 为 Compaction 提供安全边界
- 让系统拥有“干净状态”

### Snapshot 文件内容

逻辑上等价于：

```
对 B+ 树做一次中序遍历
顺序写出所有 key-value
```
## 日志压缩（Compaction）

### 为什么必须 Compaction？

假设：

```
PUT 1 100
PUT 1 200
PUT 1 300
DEL 1
```

最终状态：**key 1 不存在**

但 WAL 却写了 4 条。

------

### Compaction 的本质

> **用当前内存中的最终状态，生成新的 snapshot**
> 并丢弃旧 WAL

------

### Compaction 过程（简化版）

```
1. 冻结写入（或切换 WAL）
2. 遍历 B+ 树
3. 生成新的 snapshot
4. 删除旧 WAL
5. 切换到新 WAL
```

------

### Compaction 之后

磁盘状态变为：

```
snapshot_v2.dat
wal_v2.log
```

系统继续运行，日志重新变短。

------

## 启动恢复流程（V3）

完整启动逻辑：

```
1. 如果 snapshot 存在：
       load snapshot -> 构建 B+ 树
2. 打开 WAL
3. replay WAL（CRC 校验）
4. 服务就绪
```

这已经是**真实数据库的标准恢复流程**。

------

## 代码结构

### kvstore.h

```c
// /home/ubuntu/c++2512/KVstore/include/kvstore.h

/**
 * kvstore 结构体负责整个存储引擎，包含一个指向 bptree 的指针
 * B+ 树则是用来存储键值对的数据结构
 */

#ifndef KVSTORE_H
#define KVSTORE_H

#include <stdio.h>

#include "index/bptree.h"

#define KVSTORE_MAX_OPS 1000
#define KVSTORE_MAX_LOG_SIZE 4 * 1024 * 1024  // 4MB

// kvstore error codes
#define KVSTORE_OK 0

// 通用错误（-1 ~ -9）
#define KVSTORE_ERR_NULL   -1    // 空指针 / 非法参数
#define KVSTORE_ERR_INTERNAL -2  // 内部错误（不应该发生）
#define KVSTORE_ERR_IO       -3  // 文件 / IO 错误

// 状态相关错误（-10 ~ -19）
#define KVSTORE_ERR_READONLY  -10 // 只读模式，禁止写
#define KVSTORE_ERR_RECOVERY  -11 // 恢复中（预留）

// 数据相关错误（-20 ~ -29）
#define KVSTORE_ERR_NOT_FOUND  -20 // key 不存在
#define KVSTORE_ERR_CORRUPTED  -21 // 数据损坏（crc / 格式）

// 空间 / 资源错误（-30 ~ -39）
// #define KVSTORE_ERR_NO_MENORY  -30
// #define KVSTORE_ERR_NO_SPACE   -31




#ifdef __cplusplus
extern "C" {
#endif

// 定义 KVSTORE 的结构体
typedef struct _kvstore {
    bptree* tree;  // B+ 树指针

    FILE* log_fp;  // 日志文件指针
    char log_path[256];

    size_t log_size;   // 当前日志大小（字节）
    size_t ops_count;  // 自上次 compaction 以来的操作次数

    /*  0：正常模式
        1：只读模式（replay 中）
    */
    int readonly;  // 是否处于只读模式（replay / recovery）
} kvstore;

// =============  kvstore 基本操作函数  =============
// 创建 kvstore
kvstore* kvstore_create(const char* log_path);
void kvstore_destroy(kvstore* store);

// KV 操作
int kvstore_insert(kvstore* store, int key, long value);
int kvstore_search(kvstore* store, int key, long* out_value);
int kvstore_delete(kvstore* store, int key);
int kvstore_compact(kvstore* store);

#ifdef __cplusplus
}
#endif

#endif

```

------

### kvstore.c

```c
#include "kvstore.h"

#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>

static int kvstore_open_log(kvstore* store, const char* path);
static int kvstore_replay_log(kvstore* store);
static int kvstore_log_put(kvstore* store, int key, long value);
static int kvstore_log_del(kvstore* store, int key);
static int kvstore_load_snapshot(kvstore* store);
static int kvstore_create_snapshot(kvstore* store);
static int kvstore_snapshot_exists(const char* path);
static uint32_t crc32(const char* s);
static int kvstore_check_log_header(FILE* fp);
static int kvstore_replay_log(kvstore* store);
static int kvstore_apply_put(kvstore* store, int key, long value);
static int kvstore_apply_del(kvstore* store, int key);
const char* kvstore_strerror(int err);
// ========  public API (对外)  ==========
int kvstore_put(kvstore* store, int key, long value);
int kvstore_del(kvstore* store, int key);

// 创建 KVstore
kvstore* kvstore_create(const char* log_path) {
    kvstore* store = (kvstore*)malloc(sizeof(kvstore));  // 分配内存
    if (!store) {
        perror("malloc kvstore");
        exit(EXIT_FAILURE);
    }

    store->tree = bptree_create();
    store->log_fp = NULL;
    store->log_size = 0;
    store->ops_count = 0;

    /* 1. 先加载 snapshot*/
    kvstore_load_snapshot(store);

    /* 2. 再打开日志 */
    if (kvstore_open_log(store, log_path) != 0) {
        free(store);
        return NULL;
    }

    // 数据回复发生在 B+ 树创建之后，对用户透明
    /* 3. replay snapshot 之后的 WAL */
    kvstore_replay_log(store);

    return store;
}

/**
 * 销毁 kvstore (清理函数)
 *
 *  *  - 如果只 free(store), store 指针没有了，B+ 树所有节点没有被释放，变成了“无法访问的垃圾” -> 内存泄漏 ！
 *  - bptree_destroy 递归 free B+ 树的每个节点，最后再把 store 结构体占用的内存释放掉
 *
 * 代码层面的理解：
 *  - 创建一个对象时是由外向内：
 *      sotre = malloc(...)
 *      sotre->tree = bptree_create()
 *  - 销毁时必须严格反序（由外向内）：
 *      bptree_destroy(...)
 *      free(store)
 *
 *
 * fclose 作用：
 *      - 刷盘（Flush）:写日志时，系统为了效率，通常会将数据放在缓冲区。调用 fclose
 *                      会强迫系统把缓冲区里剩下的数据全部写进硬盘，否则丢失几条记录
 *      - 释放句柄：操作系统对一个程序能同时打开的文件数量是有上限的。
 *                  如果不关闭，文件就会被一直锁定，其他程序（甚至是下次启动的程序）可能无法再次访问它。
 *
 *
 */
void kvstore_destroy(kvstore* store) {
    if (!store) return;

    if (store->log_fp) {
        fclose(store->log_fp);
    }

    bptree_destroy(store->tree);
    free(store);

    return;
}

/**
 * 内部使用：只修改内存结构，不写 WAL
 *  - 无论是重放旧日志，还是正常写入，最终都要调用它
 *  - Replay 期间，数据库正是处于 readonly 模式，此处不能检查 readonly !!!
 *      否则，启动恢复时所有的旧数据都会被拦截，导致恢复失败
 */
static int kvstore_apply_put(kvstore* store, int key, long value) {
    if (!store || !store->tree) return KVSTORE_ERR_NULL;

    int ret = bptree_insert(store->tree, key, value);
    if (ret != BPTREE_OK) {
        return KVSTORE_ERR_INTERNAL;
    }

    return KVSTORE_OK;
}

/**
 * 内部使用：del
 */
static int kvstore_apply_del(kvstore* store, int key) {
    if (!store || !store->tree) return KVSTORE_ERR_NULL;

    int ret = bptree_delete(store->tree, key);
    if (ret != BPTREE_OK) {
        return KVSTORE_ERR_NOT_FOUND;
    }

    return KVSTORE_OK;
}

/**
 * API: 对外 PUT
 * 先 WAL, 再 apply
 *  - WAL 已落盘 -> replay 可恢复
 *  - apply 只在内存 -> 可重做
 */
int kvstore_put(kvstore* store, int key, long value) {
    if (!store) return KVSTORE_ERR_NULL;

    /*启动
        → replay 日志（readonly = 1）
        → replay 结束（readonly = 0）
        → 正常 PUT / DEL
    */
    // 只读模式不能修改 put ? todo
    if (store->readonly) {
        return KVSTORE_ERR_READONLY;
    }

    // 1. 先写 WAL (Write-Ahead Log)
    if (kvstore_log_put(store, key, value) != 0) {
        return KVSTORE_ERR_IO;
    }

    // 2. 再写修改内容
    int ret = kvstore_apply_put(store, key, value);
    if (ret != 0) {
        return KVSTORE_ERR_CORRUPTED;
    }

    // 3. 统计信息
    store->ops_count++;

    return KVSTORE_OK;
}

/**
 * API: del
 */
int kvstore_del(kvstore* store, int key) {
    if (!store) return KVSTORE_ERR_NULL;

    // 检查模式
    if (store->readonly) {
        return KVSTORE_ERR_READONLY;
    }

    if (kvstore_log_del(store, key) != 0) {
        return KVSTORE_ERR_IO;
    }

    int ret = kvstore_apply_del(store, key);
    if (ret != 0) return KVSTORE_ERR_INTERNAL;

    store->ops_count++;
    return KVSTORE_OK;
}

// 插入数据到 KVstore
int kvstore_insert(kvstore* store, int key, long value) {
    // 1. 基础检查
    if (!store) return KVSTORE_ERR_NULL;

    // 2. 权限检查：只读模式下拒接插入操作
    if (store->readonly) return KVSTORE_ERR_READONLY;

    // 3. 先写日志（WAL）
    if (kvstore_log_put(store, key, value) != 0) {
        return KVSTORE_ERR_IO;
    }

    // 4. 再应用到内存（Apply）- 落实
    if (kvstore_apply_put(store, key, value) != KVSTORE_OK) {
        return KVSTORE_ERR_INTERNAL;
    }

    // 5. 维护工作
    store->ops_count++;
    kvstore_maybe_compact(store);  // 检查是否需要压缩

    return KVSTORE_OK;
}

// 查找数据
int kvstore_search(kvstore* store, int key, long* value) {
    return bptree_search(store->tree, key, value);
}

// 删除数据
int kvstore_delete(kvstore* store, int key) {
    // 1. 基础检查
    if (!store) return KVSTORE_ERR_NULL;

    // 2. 权限检查：只读模式下拒接删除操作
    if (store->readonly) return KVSTORE_ERR_READONLY;

    // 3. 先写日志（WAL）
    if (kvstore_log_del(store, key) != 0) return KVSTORE_ERR_IO;

    // 4. 应用到内存（Apply）
    int ret = kvstore_apply_del(store->tree, key);
    if (ret != 0) {
        return KVSTORE_ERR_INTERNAL;
    }

    // 5. 维护工作
    store->ops_count++;
    kvstore_maybe_compact(store);

    return ret;
}

/**
 * 打开 / 创建日志文件
 * static - 函数仅当前文件（.c）可见，是一个私有方法，test_kvstore.c 无法直接调用
 * const - 保证函数内部不会直接修改这个路径字符串
 * snprintf - 把传入的路径字符串 path 复制到 store 结构体的 log_path 字符数组中
 *          - 会参考 store->log_path ，确保最多只拷贝数组能容纳的长度！
 * fopen - 系统调用，去硬盘上找这个文件
 * a+   - 追加/读写模式，持久化的关键
 *      - Append(a) -> 文件已存在，光标直接跳到文件末尾   ---- 追加
 *      - Update(+) -> 允许既能写也能读
 *      - Create    -> 如果文件不存在，自动创建一个新文件
 *
 * perror - 自动打印失败的具体原因
 *  */
static int kvstore_open_log(kvstore* store, const char* path) {
    snprintf(store->log_path, sizeof(store->log_path), "%s", path);

    // 日志初始化（第一次创建）
    if (ftell(store->log_fp) == 0) {
        fprintf(store->log_fp, "KVSTORE_LOG_V1\n");
        fflush(store->log_fp);
    }

    store->log_fp = fopen(path, "a+");
    if (!store->log_fp) {
        perror("fopen log file");
        return -1;
    }

    return KVSTORE_OK;
}

/**
 * 日志重放（replay）
 *
 * rewind - 把文件的读取+指针（光标） 重置到文件开头（a+ 模式打开， 默认在文件末尾）
 * fgets  - 从文件中读取一行，直到换行符 \n 或者读满了255个字符
 * sscanf - 从字符串里提取数据吗，返回成功汽配并复制的变量个数
 *          scanf(...): 从键盘输入（标准输入）读取。
 *          fscanf(fp, ...): 直接从文件读取。
 *          sscanf(str, ...): 从内存中的字符串读取
 *
 *
 */
static int kvstore_replay_log(kvstore* store) {
    if (!store || !store->log_fp) return KVSTORE_ERR_NULL;

    char line[256];

    // a+ 模式下，文件指针默认在末尾
    rewind(store->log_fp);

    while (fgets(line, sizeof(line), store->log_fp)) {
        int key;
        long value;

        if (sscanf(line, "PUT %d %ld", &key, &value) == 2) {
            kvstore_apply_put(store->tree, key, value);
        } else if (sscanf(line, "DEL %d", &key) == 1) {
            kvstore_apply_del(store->tree, key);
        } else {
            // 忽略格式错误的行
            continue;
        }
    }

    return KVSTORE_OK;
}

/**
 * 写日志函数
 * PUT
 */
static int kvstore_log_put(kvstore* store, int key, long value) {
    if (!store->log_fp) return KVSTORE_ERR_NULL;

    char buf[128];
    int len = snprintf(buf, sizeof(buf), "PUT %d %ld", key, value);
    uint32_t crc = crc32(buf);

    int bytes = fprintf(store->log_fp, "%s | %u\n", buf, crc);
    fflush(store->log_fp);

    store->ops_count += 1;
    store->log_size += bytes;

    return KVSTORE_OK;
}

/**
 * 删除日志函数
 * DEL
 */
static int kvstore_log_del(kvstore* store, int key) {
    if (!store->log_fp) return KVSTORE_ERR_NULL;

    char buf[128];
    int len = snprintf(buf, sizeof(buf), "DEL %d", key);
    uint32_t crc = ctc32(buf);

    int bytes = fprintf(store->log_fp, "%s | %u\n", buf, crc);
    fflush(store->log_fp);

    store->log_size += bytes;
    store->ops_count += 1;

    return KVSTORE_OK;
}

/**
 * 数据的“翻译官” - dump :转储
 * 拿到一条数据，按格式写进文件
 */
static int compact_write_cb(int key, long value, void* arg) {
    FILE* fp = (FILE*)arg;  // 强转，把万能指针还原为文件指针
    fprintf(fp, "PUT %d %ld\n", key, value);
    return KVSTORE_OK;
}

// kvstore 完全不知道 B+ 树内部结构
/**
 * 日志压缩 - 数据库的“返老还童术”
 */
int kvstore_compact(kvstore* store) {
    if (!store) return KVSTORE_ERR_NULL;

    const char* tmp_path = "data.compact";
    const char* data_path = store->log_path;

    /* 1. 打开临时文件 */
    FILE* fp = fopen("data.compact", "w");
    if (!fp) {
        perror("fopen compact");
        return -1;
    }

    /* 2. 顺序扫描 B+ 树 */
    if (bptree_scan(store->tree, compact_write_cb, fp) != 0) {
        fclose(fp);
        return -1;
    }

    /* 3. 刷盘，保证落盘 */
    fflush(fp);         // 把 C 语言层面的缓冲区数据堆推给操作系统内核
    fsync(fileno(fp));  // 让磁盘磁头动起来，把物理电信号写进硬盘扇区
    fclose(fp);

    /* 4. 原子替换旧数据文件 */
    FILE* old_fp = store->log_fp;
    store->log_fp = NULL;

    if (rename(tmp_path, data_path) != 0) {  // 在 Linux / Unix 上：rename 是原子操作
        perror("rename");
        store->log_fp = old_fp;  // 回滚
        return -1;
    }
    fclose(old_fp);

    /* 5. 重新打开 log 文件 */
    // 就的连接断开了，重新打开已经“瘦身”的新文件
    store->log_fp = fopen(data_path, "a+");
    if (!store->log_fp) {
        perror("reopen data.log");
        return -1;
    }

    // 创建 snapshot
    kvstore_create_snapshot(store);
    return KVSTORE_OK;
}

/**
 * 自动触发器
 */
static void kvstore_maybe_compact(kvstore* store) {
    if (!store) return KVSTORE_ERR_NULL;

    // store->log_size - 空间维度，日志文件太大，需要压缩
    // store->ops_count - 时间/效率维度，防止“重放”太慢
    if (store->log_size >= KVSTORE_MAX_LOG_SIZE ||
        store->ops_count >= KVSTORE_MAX_OPS) {
        kvstore_compact(store);

        // 重置计数器
        store->log_size = 0;
        store->ops_count = 0;
    }
}

/**
 * 写回调（复用compaction 思路）
 */
static int snapshot_write_cb(int key, long value, void* arg) {
    FILE* fp = (FILE*)arg;
    fprintf(fp, "PUT %d %ld", key, value);
    return KVSTORE_OK;
}

/**
 * 创建 snapshot(核心)
 *  - 快照本质上就是内存数据的“全量备份”
 *  - 在日志压缩后创建！
 */
static int kvstore_create_snapshot(kvstore* store) {
    if (!store) return KVSTORE_ERR_NULL;

    const char* tmp = "data.snapshot.tmp";
    const char* snap = "data.snapshot";  // 冷启动快照

    FILE* fp = fopen(tmp, "w");
    if (!fp) {
        perror("fopen snapshot");
        return -1;
    }

    if (bptree_scan(store->tree, snapshot_write_cb, fp) != 0) {
        fclose(fp);
        return -1;
    }

    fflush(fp);
    fsync(fileno(fp));
    fclose(fp);

    if (rename(tmp, snap) != 0) {
        perror("rename snapshot");
        return KVSTORE_ERR_NULL;
    }
    return KVSTORE_OK;
}

/**
 * 加载（读） -- 冷启动加速的关键
 */
static int kvstore_load_snapshot(kvstore* store) {
    if (!store) return KVSTORE_ERR_NULL;

    FILE* fp = fopen("data.snapshot", "r");
    if (!fp) return KVSTORE_OK;  // snapshot 不存在，正常情况

    char line[256];
    while (fgets(line, sizeof(line), fp)) {
        int key;
        long value;

        if (sscanf(line, "PUT %d %ld", &key, &value) == 2) {
            kvstore_apply_put(store->tree, key, value);
        }
    }

    fclose(fp);
    return KVSTORE_OK;
}

// CRC 函数（简单版）- 循环冗余校验
static uint32_t crc32(const char* s) {
    uint32_t crc = 0xFFFFFFFF;
    while (*s) {
        crc ^= (unsigned char)*s++;
        for (int i = 0; i < 8; i++) {
            crc = (crc >> 1) ^ (0xEDB88320 & -(crc & 1));
        }
    }

    return ~crc;
}

// 先校验 Header + Version
static int kvstore_log_header(FILE* fp) {
    char line[64];
    if (!fgets(line, sizeof(line), fp)) return -1;

    if (strncmp(line, "KVSTORE_LOG_V1", 14) != 0) {
        fprintf(stderr, "Invalid log version\n");
        return -1;
    }

    return KVSTORE_OK;
}


/**
 * 校验单行日志的完整性
 *  - 成功返回 1， 失败返回 0
 * 注意：此函数会修改 line 字符串（就地切割）
 */
static int kvstore_crc_check(char* line) {
    // 1. 去掉末尾换行符
    line[strcspn(line, "\r\n")] = '\0';  // 去掉换行 \n

    // 2. 寻找分隔符
    char* sep = strchr(line, '|');
    if (!sep) return 0;  

    // 3. 切割字符串
    *sep = '\0';  // 把 '|' 改成了 '\0'（字符串结束符），把这一行一刀切成两断
        // NOTE: split line in-place: "CMD|CRC" -> "CMD\0CRC"

    // 4. 读取存储的 CRC
    uint32_t crc_stored = (uint32_t)strtoul(sep + 1, NULL, 10);
    
    // 5. 计算并对比
    return (crc32(line) == crc_stored);
}

/**
 * 校验每一条日志
 *  - replay 不调用 public API, 所以不会触发 readonly 检查
 *  - apply 是 “特权通道”
 */
static int kvstore_replay_log(kvstore* store) {
    if (!store || !store->tree) return KVSTORE_ERR_NULL;

    store->readonly = 1;  // 进入只读恢复模式 1

    char line[256];
    rewind(store->log_fp);

    if (kvstore_check_log_header(store->log_fp) != 0) {
        return KVSTORE_ERR_NULL;
    }

    while (fgets(line, sizeof(line), store->log_fp)) {
        
        // 使用封装好的校验函数
        if (!kvstore_crc_check(line)) {
            // 发现损坏，将数据库置为只读以防万一，并报错
            store->readonly = 1; 
            return KVSTORE_ERR_CORRUPTED;
        }
        
        // 校验通过后，直接对 line 进行解析
        int key;
        long val;
        if (sscanf(line, "PUT %d %ld", &key, &val) == 2) {
            kvstore_apply_put(store->tree, key, val);
        } else if (sscanf(line, "DEL %d", &key) == 1) {
            kvstore_apply_del(store->tree, key);
        }
    }

    // replay 完成，恢复正常模式 0
    store->readonly = 0;
    return KVSTORE_OK;
}

/**
 * 错误码 -> 人类可读字符串
 *
 * 外部调用者使用     if rc != KVSTORE_OK ...
 */
const char* kvstore_strerror(int err) {
    switch (err) {
        case KVSTORE_OK:
            return "OK";
        case KVSTORE_ERR_NULL:
            return "null pointer";
        case KVSTORE_ERR_INTERNAL:
            return "internal error";
        case KVSTORE_ERR_IO:
            return "io error";
        case KVSTORE_ERR_READONLY:
            return "read-only mode";
        case KVSTORE_ERR_NOT_FOUND:
            return "key not found";
        case KVSTORE_ERR_CORRUPTED:
            return "data corrupted";
        default:
            return "unknown error";
    }
}
```






## V3 总结
KVstore V3 是一个**质变版本**：

- 从“日志重放程序”
- 进化为
- **具备完整恢复语义的存储引擎**

它已经包含了：

- WAL
- CRC
- Snapshot
- Compaction

这套结构，和真实数据库（LevelDB / RocksDB / InnoDB）的思想是一致的。

## 下一个版本目标

### KVstore V4：并发、锁与写放大优化

- 读写并发控制
- Compaction 与前台请求解耦
- WAL 分段
- 更真实的工程挑战