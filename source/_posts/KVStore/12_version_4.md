---
title: 12. KVStore 版本四 5️⃣ Recovery / Startup 内部实现
date: 2026-02-10 14:00:00
categories:
  - [KVStore, v4.0]  
tags:
  - kvstore
---


5️⃣ Recovery / Startup 内部实现

## 1. 函数集合
```c
static int kvstore_replay_log();
static int kvstore_load_snapshot();
static int kvstore_log_header();
```

## 2. 函数复盘 + 规范注释

### 2.1 `static int kvstore_replay_log()`
```c
/**
 * kvstore_replay_log - 重放 WAL 日志以重建内存索引
 *
 * 功能：
 *  - 从 WAL 文件中顺序读取历史操作（PUT / DEL）
 *  - 校验每一条日志记录完整性（CRC）
 *  - 将合法操作重新 apply 到内存中的 B+ Tree
 *
 * 设计约束：
 *  - 不写 WAL (否则会造成日志膨胀)
 *  - 不走 Public API (绕过 readonly / 权限检查)
 *  - 只调用 apply_internal 系列函数
 */
static int kvstore_replay_log(kvstore* store) {
    // 1. 环境检查
    if (!store || !store->tree || !store->log_fp) RETURN_ERR(KVSTORE_ERR_NULL);

    // 2. 状态机防御
    if (store->state != KVSTORE_STATE_RECOVERING) {
        RETURN_ERR(KVSTORE_ERR_INTERNAL);
    }

    char line[256];
    int rc = KVSTORE_OK;
    replay_stats stats = {0};

    // 3. 重置指针：从日志开头开始读
    rewind(store->log_fp);

    // 4. 校验日志头(Header Check)
    if (!fgets(line, sizeof(line), store->log_fp)) {
        goto out;  // 空文件是合法的（刚刚初始化）
    }

    if (kvstore_log_header(line) != KVSTORE_OK) {
        stats.corrupted++;
        rc = KVSTORE_ERR_WAL_CORRUPTED;
        goto out;
    }

    // 5. 循环重放日志体
    while (fgets(line, sizeof(line), store->log_fp)) {
        /* A. 清洗数据 */
        if (line[0] == '\n' || line[0] == '\r' || line[0] == ' ') continue;
        line[strcspn(line, "\r\n")] = '\0';  // 统一去掉换行符

        /* B. 拆分 payload | crc */
        char* sep = strchr(line, '|');
        if (!sep) {
            // 容忍尾行被破坏，不报错，直接当作读完
            stats.skipped++;
            rc = KVSTORE_OK;
            break;
        }

        *sep = '\0';
        char* crc_str = sep + 1;

        /* C. CRC 数据完整性校验 */
        if (!kvstore_crc_check(line, crc_str)) {
            stats.corrupted++;

            if (feof(store->log_fp)) {
                // 如果是最后一行损坏，可能是断电，容忍它
                rc = KVSTORE_OK;
            } else {
                // 如果是中间行损坏，说明文件逻辑断裂，必须停止
                rc = KVSTORE_ERR_WAL_CORRUPTED;
                fprintf(stderr, "[REPLAY] 中间数据损坏! Payload: [%s]\n", line);
            }
            break;
        }

        /* D. 解析动作并 Apply 到内存 */
        int key;
        long val;
        int ret;

        if (sscanf(line, "PUT %d %ld", &key, &val) == 2) {
            ret = kvstore_replay_put(store, key, val);
        } else if (sscanf(line, "DEL %d", &key) == 1) {
            ret = kvstore_replay_del(store, key);
        } else {
            // 解析失败（比如格式写错了），计入 skipped
            stats.skipped++;
            continue;

            // rc = KVSTORE_ERR_INTERNAL;
            // break;
        }

        if (ret == KVSTORE_OK) {
            stats.applied++;
        } else {
            rc = ret;
            break;
        }
    }
    // 在退出前打印统计信息
    int total = stats.applied + stats.skipped + stats.corrupted;

    // 6. 统计收尾
out:

    printf(
        "[RECOVERY] applied=%d skipped=%d corrupted=%d success_rate=%.2f%%\n",
        stats.applied,
        stats.skipped,
        stats.corrupted,
        total ? (100.0 * stats.applied / total) : 100.0);

    if (rc != KVSTORE_OK) {
        return kvstore_fatal(store, rc);
    }

    return KVSTORE_OK;
}
```

### 2.2 `static int kvstore_load_snapshot()`
```c
/**
 * kvstore_load_snap - 加载磁盘快照，以快速恢复基础数据
 *
 * @details
 * 该函数是系统启动的第一步。将磁盘上的全量固化数据载入内存，
 * 使得后续的 WAL 重放只需处理自“自上次快照以来的增量”，从而大幅缩短启动时间。
 *
 * 设计说明：
 *  - snapshot 是内存 KV 状态的“全量物化”
 *  - 只包含 PUT, 不包含 DEL
 *  - snapshot 不存在是合法情况（首次启动）
 *  - 加载过程不写 WAL, 不做权限检查
 *
 *
 * 启动恢复流程：
 *  1. load_snapshot()  -> 快速恢复到最近状态
 *  2. replay_log()     -> 重放 snapshot 之后的 WAL
 */
static int kvstore_load_snapshot(kvstore* store) {
    // 1. 基础校验
    if (!store) return KVSTORE_ERR_NULL;

    // 2. 尝试开启快照文件
    FILE* fp = fopen("data.snapshot", "r");
    if (!fp) {
        // snapshot 不存在，允许慢启动（会跳过快照）
        return KVSTORE_OK;
    }

    char line[256];
    int key;
    long value;

    // 3. 全量数据注入循环
    while (fgets(line, sizeof(line), fp)) {
        // 跳过空行
        if (line[0] == '\n' || line[0] == '\r')
            continue;

        // snapshot 行格式固定为：PUT key value
        if (sscanf(line, "PUT %d %ld", &key, &value) == 2) {
            kvstore_apply_put_internal(store->tree, key, value);
        }

        // 非法行： 忽略，snapshot 是可信文件
    }

    // 5.资源回收
    fclose(fp);
    return KVSTORE_OK;
}
```

### 2.3 `static int kvstore_log_header()`
```c

/**
 * kvstore_log_header - 验证 WAL 日志文件的元数据合法性
 *
 * 这是启动恢复的第一步，如果版本号对不上，吧必须立即停止
 */
static int kvstore_log_header(const char* line) {
    // 1. 字符串前缀比对
    if (strncmp(line, KVSTORE_LOG_VERSION, strlen(KVSTORE_LOG_VERSION)) != 0) {
        // 2.致命错误输入
        fprintf(stderr, "Invalid log version. Expected: %s\n", KVSTORE_LOG_VERSION);
        return KVSTORE_ERR_WAL_CORRUPTED;
    }

    // 3. 通过验证
    return KVSTORE_OK;
}
```