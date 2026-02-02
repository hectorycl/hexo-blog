---
title: 06. KVStore 版本三 语义统一
date: 2026-02-02 15:30:00
categories:
  - [KVStore, v3.0]  
tags:
  - kvstore
---

# KVStore 运行时状态与访问控制设计文档


## 1. 文档目的与背景
本文档用于说明 KVstore 在**运行时阶段** 的整体设计，重点描述：
- State (状态)：系统当前所处的生命周期阶段
- Mode (模式)：系统对外暴露的访问语义
- Readonly 协调机制：如何在不破坏一致性的前提下，安全地执行 compaction 等重量级操作

设计的核心目标是：
> 保证在任意时刻，KVstore 的外部行为与其内部状态严格一直，避免非法写入与状态泄漏

## 2.设计总览
KVstore 在运行时通过 State + Mode 的组合来约束行为：
- State 决定“系统当前能不能做事” （生命周期 / 健康状况）
- Mode 决定“系统对外怎么做事” （访问语义）

二者共同作用于
- public API 的合法性检查
- readonly / replay /compaction 等特殊阶段的访问控制

## State 设计

### 3.1 State 枚举定义
```c
typedef enum {
    KVSTORE_STATE_INIT,           // 尚未开始恢复
    KVSTORE_STATE_RECOVERING,     // snapshot load / WAL replay
    KVSTORE_STATE_READY,          // 正常对外服务
    KVSTORE_STATE_READONLY,       // compaction / 维护阶段

    KVSTORE_STATE_CLOSING,        // 关闭中
    KVSTORE_STATE_CORRUPTED,      // 不可恢复错误
} kvstore_state_t;
```

### 3.2 各 State 的语义说明

- 只有 `READY` 状态允许写操作
- 任何非 READY 状态都必须对写进行拦截

## Mode 设计
### 4.1 Mode 枚举类型
```c
typedef enum {
    KVSTORE_MODE_NORMAL,   // 正常运行模式
    KVSTORE_MODE_REPLAY,   // WAL 回放模式
    KVSTORE_MODE_READONLY, // 只读保护模式
} kvstore_mode_t;
```

### 4.2 Mode 的设计动机
Mode 的存在用于描述“访问语义”，而非生命周期，主要解决以下问题：
- replay 阶段需要执行 PUT / DEL, 但不能触发 WAL
- compaction 阶段精致任何外部写，但允许内部扫描
- debug / 测试中需要强制切换行为

**核心原则**
> Mode 不直接决定系统状态是否合法，合法性由 State 决定；Mode 只影响“怎么执行”。】



## 5. State / Mode 对照表

| State \ Mode | NORMAL | REPLAY        | READONLY  |
| ------------ | ------ | ------------- | --------- |
| INIT         | ❌      | ⚠️（内部使用） | ❌         |
| RECOVERING   | ❌      | ✅             | ❌         |
| READY        | ✅      | ❌             | ⚠️（维护） |
| READONLY     | ❌      | ❌             | ✅         |
| CLOSING      | ❌      | ❌             | ❌         |
| CORRUPTED    | ❌      | ❌             | ❌         |

说明：
-  ×：非法使用
- ❗：受限使用
-  √：合法

## 6. API 访问矩阵
### 6.1 Public API

| API             | READY | READONLY | RECOVERING |
| --------------- | ----- | -------- | ---------- |
| kvstore_put     | ✅     | ❌        | ❌          |
| kvstore_del     | ✅     | ❌        | ❌          |
| kvstore_get     | ✅     | ✅        | ⚠️          |
| kvstore_compact | ✅     | ❌        | ❌          |

说明：
- public API 必须首先检查 store->state

- 不允许绕过 readonly / recovering 保护




## Compaction 设计说明

### 7.1 设计说明
- 在不阻塞的情况下执行日志压缩
- 防止 compaction 期间发生写入，避免状态撕裂（状态泄漏）

### 7.2 设计流程
1. 记录当前 state 
2. 切换至 KVSTORE_STATE_READONLY
3. 执行 kvstore_compaction_internal
4. 恢复之前的 `state`
```c
rev = store->state;
kvstore_enter_compaction(store);
kvstore_compact_internal(store);
kvstore_exit_compaction(store, prev);

```

## 8. 为什么 internal 函数必须是 static
### 8.1 封装边界
- internal 函数 **不做合法性校验**
- 默认调用者已经满足所有前置条件

若暴露为非 static:
- 极易被绕过 state / mode 检查
- 造成非法写、状态泄漏

### 8.2 设计层清晰
| 层级             | 职责                       |
| ---------------- | --------------------------|
| Public API       | 参数校验 + state/mode 检查 |
| Apply / Internal | **纯逻辑***执行            |
| Data Structure   | 数据结构操作               |

`static` 明确了：

> **“这个函数只能在本编译单元内，被受控地使用”**

## 9. 设计阶段总结
当前 KVstore 已完成：
- 运行时状态机设计
- 访问语义抽象
- readonly / compaction 协调机制
- 清晰的 public / internal 分层
已具备**工程运行时一致性基础**





