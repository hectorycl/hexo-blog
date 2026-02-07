---
title: 08. KVStore 版本四 1️⃣Public API — 生命周期（最顶层入口）
date: 2026-02-07 22:20:00
categories:
  - [KVStore, v4.0]  
tags:
  - kvstore
---


# 1️⃣Public API — 生命周期（最顶层入口）


## 1. 函数集合
```c
kvstore* kvstore_create();
void kvstore_destroy();

kvstore* kvstore_open();
int kvstore_recover();
int kvstore_log_open();
int kvstore_log_close();
int kvstore_close();
```

## 2. 函数复盘 + 规范注释

### 2.1 `kvstore* kvstore_create()`
```c

/**
 * 初始化 kvstore 实例对象（内存分配阶段）
 *
 * @note 该函数仅在内存中创建结构。此时系统处于 KVSTORE_STATE_INIT 状态，
 * 在调用 kvstore_open() 或 kvstore_recover() 之前，无法进行数据操作。
 */
kvstore* kvstore_create() {
    // 1. 结构体内存分配
    kvstore* store = malloc(sizeof(*store));
    if (!store) return NULL;

    // 2. 核心索引初始化（apply 层的物质基础）
    store->tree = bptree_create();
    if (!store->tree) {
        free(store);  // 异常处理：防止内存泄漏，索引创建失败必须释放外层容器
        return NULL;
    }

    // 3. 状态机初始化
    store->state = KVSTORE_STATE_INIT;

    // 4. 持久化元数据清理（Logging / Maintenance 层）
    store->log_fp = NULL;
    store->log_size = 0;
    store->ops_count = 0;

    // 5. 运行模式设置（Error/Debug 层）
    store->mode = KVSTORE_MODE_NORMAL;
    store->readonly = 0;  // 默认可写

    return store;
}
```


### 2.1 `void kvstore_destroy()`

```c

/**
 * kvstore_destroy - 销毁并彻底回收 kvstore 实例的所有资源
 *
 * 功能：
 *  1. 检查并强制关闭尚未停机的日志与状态
 *  2. 调用索引引擎的销毁接口，释放复杂的内存树结构
 *  3. 释放主控制块内存
 *
 *  - 支持 NULL 传入，具有幂等性
 *  - 一旦调用此函数，原指针 store 将失效，不能再次解引用
 */
void kvstore_destroy(kvstore* store) {
    if (!store) return;

    /**
     *  1. 状态机与持久化补齐：
     * 确保在销毁内存前，所有文件 IO (Logging 层已安全关闭) */
    if (store->state != KVSTORE_STATE_CLOSED) {
        kvstore_close(store);
    }

    /**
     * 2. 索引层清理：
     * 先销毁子结构（Tree）,在销毁父结构（Store）*/
    if (store->tree) {
        bptree_destroy(store->tree);
        store->tree = NULL;
    }

    /* 3. 最终释放 store 本体，将控制块占用的堆空间还给 OS */
    free(store);
}

```

### 2.3 `kvstore* kvstore_open()`
```c

/**
 * kvstore_open - 系统唯一合法构造函数
 *
 * 负责编排整个 KVstore 的启动流程，遵循“全或无”原则，确保返回的指针处于
 *  READY 状态，或者在失败时彻底清理所有资源
 *
 * @layers:
 * 1. Lifecycle: 分配内存对象。
 * 2. Logging: 挂载磁盘文件。
 * 3. Recovery: 恢复快照(Snapshot)并重放(Replay)增量日志
 * 4. State Machine: 管理从 INIT -> RECOVERY -> READY 的状态演变
 *
 * wal_path - 日志文件的路径（若文件不存在则创建）
 *
 */
kvstore* kvstore_open(const char* wal_path) {
    kvstore* store = NULL;
    int ret;

    /* 1. [Lifecycle] 初始化内存结构  */
    store = kvstore_create();
    if (!store) return NULL;

    /* 2. [Logging] 绑定磁盘持久化层 */
    ret = kvstore_log_open(store, wal_path);
    if (ret != KVSTORE_OK) goto fail;

    /* 3. [Recovery] 加载最近的一次数据快照（允许不存在） */
    ret = kvstore_load_snapshot(store);
    if (ret != KVSTORE_OK) goto fail;

    /* 4. [State Machine] 锁定状态为恢复中，拦截并发访问量 */
    ret = kvstore_enter_recovering(store);
    if (ret != KVSTORE_OK) goto fail;

    /* 5. [Recovery] 重放 WAL 增量操作，对其内存与磁盘 */
    ret = kvstore_replay_log(store);
    if (ret != KVSTORE_OK) goto fail;

    /* 6. [State Machine] 完成恢复，正式开放读写权限 */
    ret = kvstore_enter_ready(store);
    if (ret != KVSTORE_OK) goto fail;

    return store;

fail:
    /* [Error/Cleanup] 统一资源回收 */
    kvstore_close(store);
    return NULL;
}
```


### 2.4 `int kvstore_recover()`
```c

/**
 * kvstore_recover - 触发数据恢复流程，对齐内存与磁盘状态
 *  - 拦截异常：kvstore_state_allow
 *  - 异常降级：一旦恢复逻辑（replay_log）失败，立即调用 enter_failed 锁定系统。
 */
int kvstore_recover(kvstore* store) {
    if (!store) return KVSTORE_ERR_NULL;

    /* 1. 状态校验 */
    if (!kvstore_state_allow(store->state, KVSTORE_OP_REPLAY))
        return KVSTORE_ERR_INTERNAL_STATE;

    /* 2. replay WAL - 执行核心重放*/
    // 遍历 WAL 文件，调用 apply 层 修改内存
    int ret = kvstore_replay_log(store);
    if (ret != KVSTORE_OK) {
        kvstore_enter_failed(store);
        return ret;
    }

    /* 3. 恢复完成，进入 READY */
    return kvstore_enter_ready(store);
}
```


### 2.5 `kvstore_log_open`
```c

/**
 * kvstore_log_open - 开启或初始化 WAL 日志系统
 * 
 * @details
 * 本函数负责将内存实例与磁盘文件正式对接。不仅负责文件句柄的分配，
 * 还承担了“格式化”新日志文件的任务。
 * 
 * 功能：
 *  - 打开或者创建 WAL file
 *  - 确保 log header 存在
 *  - 为仅可追加写入准备文件
 *
 *  NOTE:
 *  - does NOT replay log
 *  - does NOT change kvstore state
 */
int kvstore_log_open(kvstore* store, const char* wal_path) {
    // 1. 基础参数合法性校验
    if (!store || !wal_path)
        return KVSTORE_ERR_NULL;

    // 2. 以读写追加模式开启物理文件
    FILE* fp = fopen(wal_path, "a+");
    if (!fp) return KVSTORE_ERR_IO;

    // 3. 绑定文件状态至 store 控制块
    store->log_fp = fp;
    snprintf(store->log_path, sizeof(store->log_path), "%s", wal_path);

    // 4. 探测文件物理大小，用于触发 Compaction 逻辑
    fseek(fp, 0, SEEK_END);
    long size = ftell(fp);
    store->log_size = size;

    // 5. 关键：初始化空日志的 Header
    if (size == 0) {
        fprintf(fp, "%s\n", KVSTORE_LOG_VERSION);
        fflush(fp);
        int fd = fileno(fp);

        // 持久化核心，强制 OS 执行物理刷盘，确保 Header 落地
        if (fd >= 0) fsync(fd);

        store->log_size = ftell(fp);
    }

    // 6. 重置偏移量，为接下来的 Recovery/Replay 流程做准备
    fseek(fp, 0, SEEK_SET);

    return KVSTORE_OK;
}
```

### 2.6 `kvstore_log_close`
```c

/**
 * kvstore_close - 安全关闭 WAL 日志系统
 * 
 * @details
 * 该函数负责断开内存实例与物理磁盘日志的连接。不仅仅是释放句柄，更是
 * 确保所有尚未写入磁盘的数据得到最后的妥善处理
 * 
 * 
 */
int kvstore_log_close(kvstore* store) {
    // 1. 基础校验：防止对空对象进行操作
    if (!store) return KVSTORE_ERR_NULL;

    // 2. 幂等性支持：若已关闭则直接跳过
    if (!store->log_fp)
        return KVSTORE_OK;

    /**
     * 3. 数据完整性保护：
     * 将 C 库缓冲区中的最后字节同步至 OS 内核 */
    fflush(store->log_fp);

    /**
     * 4. 句柄释放：
     * 归还操作系统资源，并立即将指针归零以防逻辑层重复利用 */
    fclose(store->log_fp);
    store->log_fp = NULL;  // 防止野指针

    /**
     * 5. 状态同步：
     * 清除关于磁盘文件的元数据记录，确保内存状态与 IO 状态严格一致*/
    store->log_size = 0;

    return KVSTORE_OK;
}
```

### 2.7 `kvstore_close()`

```c

/**
 * kvstore_close - 逻辑停机：安全停止 KVStore 的运行
 * 
 * @details
 * 该函数实现了从“在线”到“离线”的平滑过渡。它不仅关闭物理文件，更重要的是通过
 * 状态机变更，在关闭期间建立起逻辑防火墙
 * 
 * [设计逻辑]
 * 1. 采用中间态（CLOSING）,确保在执行物理 I/O 操作时，
 *   外部新的业务请求会被拦截，避免数据在关闭瞬间产生的不一致
 * 2. 委托 Logging 层完成文件句柄的释放和缓冲区刷新
 * 3. 此函数仅执行逻辑关闭，并不释放 store 指针内存，若要彻底销毁，
 *   后续要调用 kvstore_destroy() 
 * 
 */
int kvstore_close(kvstore* store) {
    /* 1. 安全边界检查 */
    if (!store) return KVSTORE_ERR_NULL;

    /* 2. 幂等性支持：避免重复关闭导致的系统逻辑混乱 */
    if (store->state == KVSTORE_STATE_CLOSED)
        return KVSTORE_OK;

    /* 3. [状态机] 第一阶段：设置“正在关闭屏障”，拒绝新的 Put/Del 操作 */
    kvstore_transit_state(store, KVSTORE_STATE_CLOSING);

    /* 4. [状态机] 物理操作：将缓冲区数据推向磁盘并关闭文件描述符 fd */
    kvstore_log_close(store);  // 内部 fclose + fsync

    /* 3. [状态机] 第二阶段：标记实例已进入静默态 */
    kvstore_transit_state(store, KVSTORE_STATE_CLOSED);

    return KVSTORE_OK;
}
```


