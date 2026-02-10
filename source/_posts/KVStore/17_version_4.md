---
title: 17. KVStore 版本四 🔟 状态机
date: 2026-02-10 15:30:00
categories:
  - [KVStore, v4.0]  
tags:
  - kvstore
---




# 🔟 状态机

## 1. 函数集合

```c
static int kvstore_transit_state();
static int kvstore_enter_recovering();
static int kvstore_enter_ready();
static int kvstore_enter_failed();
static int kvstore_enter_compaction();
static void kvstore_exit_compaction();
static void kvstore_apply_state();
static int kvstore_fatal();
```

## 2. 函数复盘 + 规范注释

### 2.1 `static int kvstore_transit_state()`

```c
/**
 * kvstore_transit_state - 执行安全的状态切换
 *
 * @details
 *  - 确保系统生命周期符合预设路径，防止逻辑越权
 */
static int kvstore_transit_state(kvstore* store, kvstore_state_t next) {
    if (!store)
        return KVSTORE_ERR_NULL;

    kvstore_state_t prev = store->state;

    /* 1. 状态转移合法性检查 */
    switch (prev) {
        case KVSTORE_STATE_INIT:
            // INIT 只能去干活先恢复（RECOVERING）
            if (next != KVSTORE_STATE_RECOVERING)
                return KVSTORE_ERR_INTERNAL_STATE;
            break;

        case KVSTORE_STATE_RECOVERING:
            // 恢复成功去 READY,恢复失败（CRC 错误）去 CORRUPTED
            if (next != KVSTORE_STATE_READY && next != KVSTORE_STATE_CORRUPTED)
                return KVSTORE_ERR_INTERNAL_STATE;
            break;

        case KVSTORE_STATE_READY:
            // 运行中允许自我保持，或者遇到致命错误转为 CORRUPTED
            if (next != KVSTORE_STATE_READY && next != KVSTORE_STATE_CORRUPTED)
                return KVSTORE_ERR_INTERNAL_STATE;
            break;

        case KVSTORE_STATE_CLOSING:
            // 已经要关门了，不能再去任何其他状态（不允许有 next）
            return KVSTORE_ERR_INTERNAL_STATE;

        case KVSTORE_STATE_CORRUPTED:
            // 坏掉的系统是“死胡同”，除非销毁重启，否则不能逃逸
            return KVSTORE_ERR_INTERNAL_STATE;

        default:
            return KVSTORE_ERR_INTERNAL_STATE;
    }

    /* 2. 真正的修改状态 */
    // 只有通过了上面的审查，才允许修改内存中的状态值
    store->state = next;

    /* 3. 派生状态同步 */
    kvstore_apply_state(store);

    return KVSTORE_OK;
}

```
### 2.2 `static int kvstore_enter_recovering()`

```c

/**
 * 状态转换的语义化 API
 *  - 进入恢复期
 */
static int kvstore_enter_recovering(kvstore* store) {
    return kvstore_transit_state(store, KVSTORE_STATE_RECOVERING);
}

```

### 2.3 `static int kvstore_enter_ready()`

```c

/**
 * 系统完成恢复，可以对外提供完整服务
 */
static int kvstore_enter_ready(kvstore* store) {
    return kvstore_transit_state(store, KVSTORE_STATE_READY);
}

```

### 2.4 `static int kvstore_enter_failed()`
```c

/**
 * 系统进入不可恢复错误态，只允许 destroy
 */
static int kvstore_enter_failed(kvstore* store) {
    return kvstore_transit_state(store, KVSTORE_STATE_CORRUPTED);
}

```
### 2.5 `static int kvstore_enter_compaction()`
```c

/**
 * kvstore_enter_compaction - 进入压缩前置保护状态（只读）
 *
 * 设计意图：
 *  - compaction 是一个耗时的“重量级操作”。在执行期间，必须通过将 state 切换为
 *    KVSTORE_STATE_READONLY(只读) 来拦截所有并发的写请求（PUT / DEL）
 *
 *  - 只有处于 READY(正常服务)的系统才能发起压缩。此函数确保压缩操作不会在
 *    恢复中（RECOVERY）或已损坏（CURRUPTED）的情况下被非法触发
 */
static int kvstore_enter_compaction(kvstore* store) {
    // 1. 基础安全校验
    if (!store) return KVSTORE_ERR_NULL;

    // 2. 状态准入判定
    if (store->state != KVSTORE_STATE_READY)
        return KVSTORE_ERR_READONLY;

    // 3. 状态降级 - 降级为 READONLY
    // 所有的 PUT 和 DEL 请求都会被 kvstore_state_allow 拦截
    // 保证在扫描 B+ 树生成快照时，内存中的树结果时静止的，不会被并发修改
    store->state = KVSTORE_STATE_READONLY;

    // 4. 应用生效
    kvstore_apply_state(store);

    return KVSTORE_OK;
}

```
### 2.6 `static void kvstore_exit_compaction()`
```c

/**
 * kvstore_exit_compaction - 退出压缩阶段：恢复系统压缩前的运行状态
 *
 * 设计说明：
 *  - 压缩完成后，通过传入 prev 参数，可以将系统恢复到他所前的状态
 *  - 无论压缩成功还是失败，都必须调用此函数来解锁系统权限，否则系统将永远停留在只读模式
 *
 *  这里通常是写锁的地方 ！
 */
static void kvstore_exit_compaction(kvstore* store, kvstore_state_t prev) {
    // 1. 状态回溯
    store->state = prev;

    // 2. 派生状态生效 - 会根据恢复否的 state 重新打开写权限
    kvstore_apply_state(store);
}

```
### 2.7 `static void kvstore_apply_state()`
```c

/**
 * kvstore_apply_state
 *  - 派生状态同步器。根据主状态配置系统的物理运行参数
 *
 * @details
 *  - 将逻辑状态（READY/FAILED）与物理行为（READYONLY/MODE）解耦
 *  - 除了特定的 READY 状态外，其余所有路径默认开启 readonly=1，实现最小权限原则
 *
 * -该函数是状态机的最终落地。每当 state 发生跳变，必须调用此函数以确保系统北村标志与预期行为同步。
 *
 */
static void kvstore_apply_state(kvstore* store) {
    switch (store->state) {
        case KVSTORE_STATE_INIT:
            // 初始态：正常模式但禁止写入
            store->mode = KVSTORE_MODE_NORMAL;
            store->readonly = 1;
            break;

        case KVSTORE_STATE_RECOVERING:
            // 恢复态必须强制设置为 REPLAY 模式
            // 确保系统处于“只回放、不产生新日志”
            store->mode = KVSTORE_MODE_REPLAY;
            store->readonly = 1;
            break;

        case KVSTORE_STATE_READY:
            // 唯一的“通车”时刻，允许用户写入数据
            store->mode = KVSTORE_MODE_NORMAL;
            store->readonly = 0;
            break;

        case KVSTORE_STATE_READONLY:
            // 维护/只读态：维持 NORMAL 逻辑但切断写入路径
            store->mode = KVSTORE_MODE_NORMAL;
            store->readonly = 1;
            break;

        case KVSTORE_STATE_CLOSING:
        case KVSTORE_STATE_CORRUPTED:
            // 安全降级：无论是正常关闭还是故障
            // 第一步永远是撤销写入权限（readonly = 1）,防止脏数据写入磁盘
            store->readonly = 1;
            break;

        default:
            // 兜底：未知即风险。进入最严格的只读模式保护现场
            store->readonly = 1;
            break;
    }
}

```
### 2.8 `static int kvstore_fatal()`
```c
/**
 * kvstore_fatalb
 *  - 这个错误认定为不可恢复
 */
static int kvstore_fatal(kvstore* store, int err) {
    // 1. 鲁棒性检查
    if (!store)
        return err;

    // 2. 状态熔断（Failsafe）
    // 一旦调用此函数，状态机会立即切换到 CORRUPTED (损坏)状态
    kvstore_enter_failed(store);

    // 3. 错误透传 - 将引起致命故障的原始错误码原样返回
    return err;
}

```
