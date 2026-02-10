---
title: 14. KVStore 版本四 7️⃣ 原子执行路径
date: 2026-02-10 15:00:00
categories:
  - [KVStore, v4.0]  
tags:
  - kvstore
---


# 7️⃣ 原子执行路径

## 1. 函数集合
```c
static int kvstore_exec_write();
static int kvstore_replay_put();
static int kvstore_replay_del();
static int kvstore_state_allow();
```

## 2. 函数复盘 + 规范注释

### 2.1 `static int kvstore_exec_write()`
```c

/**
 * kvstore_exec_write
 *  - 原子写入路径：统筹协调日志持久化与内存索引更新
 *
 * 通过状态机 kvstore_state_allow 进行准入控制
 *
设计意图：
    kvstore_put ─┐
                ├─ wal
                ├─ apply
                └─ runtime
    kvstore_del ─┘

    kvstore_replay_* ──→ apply only
 */
static int kvstore_exec_write(kvstore* store, kvstore_op_t op, int key, int value) {
    int ret;

    /* 1. 基础合法性检查  */
    if (!store)
        return KVSTORE_ERR_NULL;

    /* 2.状态检查：防火墙机制 */
    if (!kvstore_state_allow(store->state, op)) {
        return KVSTORE_ERR_INTERNAL_STATE;
    }

    /* 3. 第一阶段：先写 WAL(Durability First) */
    switch (op) {
        case KVSTORE_OP_PUT:
            ret = kvstore_log_put(store, key, value);
            break;
        case KVSTORE_OP_DEL:
            ret = kvstore_log_del(store, key);
            break;

        default:
            return KVSTORE_ERR_INTERNAL;
    }

    if (ret != KVSTORE_OK)
        return kvstore_fatal(store, ret);

    /* 4. 第二阶段：再 apply 到内存(Update Index) */
    switch (op) {
        case KVSTORE_OP_PUT:
            ret = kvstore_apply_put(store, key, value);
            break;
        case KVSTORE_OP_DEL:
            ret = kvstore_apply_del(store, key);
            break;

        default:
            return KVSTORE_ERR_INTERNAL;
    }

    // 如果内存更新失败，属于致命逻辑错误
    if (ret != KVSTORE_OK) {
        return kvstore_fatal(store, ret);
    }

    /* 5. 运行态维护 */
    store->ops_count++;
    kvstore_maybe_compact(store);

    return KVSTORE_OK;
}

```

### 2.2 `static int kvstore_replay_put()`
```c

/**
 * kvstore_replay_put - 将 WAL 日志中的历史 PUT 记录重应用到内存索引
 *
 * @details
 *  - 该函数是“影子写入”。它绕过了日志持久化逻辑，仅对 B+ 树执行 apply 操作
 *  - 通过状态机确保该函数只能在恢复阶段使用、
 *  - 绝不调用写日志逻辑，防止产生死循环（重放日志产生日志）
 */
int kvstore_replay_put(kvstore* store, int key, long value) {
    // 1. 基础合法性检查
    if (!store) return KVSTORE_ERR_NULL;

    // 2. 状态机“重放权限”检查
    if (!kvstore_state_allow(store->state, KVSTORE_OP_REPLAY))
        return KVSTORE_ERR_INTERNAL_STATE;

    // 3. 重放阶段的核心逻辑：直接写入内存
    int ret = kvstore_apply_put(store, key, value);

    // 4. 恢复期间的错误诊断
    if (ret != KVSTORE_OK) {
        printf("Replay PUT failed: key=%d, err=%d", key, ret);
        return ret;
    }

    return KVSTORE_OK;
}

```

### 2.3 `static int kvstore_replay_del()`
```c

/**
 * kvstore_replay_del - 将 WAL 日志中的历史 DEL 记录到重应用到内存索引
 *
 * 该函数确保了“删除”这一事实在系统重启后依然生效。
 */
int kvstore_replay_del(kvstore* store, int key) {
    // 1. 基础合法性检查
    if (!store) return KVSTORE_ERR_NULL;

    // 2. 状态机校验
    if (!kvstore_state_allow(store->state, KVSTORE_OP_REPLAY))
        return KVSTORE_ERR_INTERNAL_STATE;

    // 3. 逻辑执行：内存抹除
    return kvstore_apply_del(store, key);
}
```

### 2.4 `static int kvstore_state_allow()`
```c

/**
 * 系统的中央决策矩阵，判定当前状态与从左的兼容性
 *
 * - 将全系统的逻辑收拢于此，避免逻辑碎片化。
 * - 严格隔离“恢复态”与“就绪态”，确保 WAL 重放过程不受业务干扰
 *
 * 状态机与业务代码解耦
 *  - kvstore_put 中不再写 if(store->recovering)
 *  - 业务函数只需调用一次 state_allow, 以后想再增加一个状态，
 *    只需要修改这个矩阵函数即可
 *
| state ↓ / op → | REPLAY | PUT | DEL | GET | CLOSE | DESTROY |
| -------------- | ------ | --- | --- | --- | ----- | ------- |
| INIT           | ❌      | ❌   | ❌   | ❌   | ❌     | ❌       |
| RECOVERING     | ✅      | ❌   | ❌   | ❌*  | ❌     | ❌       |
| READY          | ❌      | ✅   | ✅   | ✅   | ✅     | ❌       |
| READONLY       | ❌      | ❌   | ❌   | ✅   | ✅     | ❌       |
| CLOSING        | ❌      | ❌   | ❌   | ❌   | ❌     | ❌       |
| CORRUPTED      | ❌      | ❌   | ❌   | ❌   | ❌     | ✅       |

 */
static int kvstore_state_allow(kvstore_state_t state, kvstore_op_t op) {
    // 设计哲学：这是一个“白名单”机制。默认不准做任何事，除非明确允许
    switch (op) {
        /* 1. 写操作：最严格的区域 */
        // PUT 和 DEL 只有在 READY 状态下进行
        case KVSTORE_OP_PUT:
        case KVSTORE_OP_DEL:
            return state == KVSTORE_STATE_READY;

        /* 2. 读操作：相对宽松 */
        // 允许系统在不接受新写入的情况下（比如维护期），依然可以提供查询服务
        case KVSTORE_OP_GET:
            return state == KVSTORE_STATE_READY || state == KVSTORE_STATE_READONLY;

        /* 3.系统内部操作：特权指令 */
        // REPLAY 只能在 RECOVERING 状态执行
        case KVSTORE_OP_REPLAY:
            return state == KVSTORE_STATE_RECOVERING;

        /* 4. 生命周期操作：收尾保障 */
        // 只要不是正在关闭（CLOSING），都可以申请关闭或销毁
        case KVSTORE_OP_CLOSE:
        case KVSTORE_OP_DESTROY:
            return state != KVSTORE_STATE_CLOSING;

        default:
            return 0;  // 兜底：未知操作一律拦截
    }
}
```


