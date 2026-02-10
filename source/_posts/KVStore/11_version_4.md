---
title: 11. KVStore 版本四 4️⃣ Debug / Utility API
date: 2026-02-10 14:00:00
categories:
  - [KVStore, v4.0]  
tags:
  - kvstore
---


# 4️⃣ Debug / Utility API

## 1. 函数集合
```c
const char* kvstore_strerror();
uint32_t crc32();
static int kvstore_crc_check();
```

## 2. 函数复盘 + 规范注释

### 2.1 `void kvstore_debug_set_mode()`
```c

/**
 * kvstore_strerror - 将 KVstore 错误码转换为可读字符串
 *
 * 设计说明：
 *  - 纯函数（无副作用，无状态），不进行任何 I/O 或日志输出
 *  - 返回的字符串为静态只读字符串，调用方不得修改
 *
 * 使用场景：
 *  - 调试输出
 *  - 日志系统
 *  - 对外 API 错误提示
 */
const char* kvstore_strerror(int err) {
    switch (err) {
        case KVSTORE_OK:
            return "OK";
        case KVSTORE_ERR_NULL:
            return "null pointer";  // NULL 指针的拦截
        case KVSTORE_ERR_INTERNAL:
            return "internal error";  // 状态机非法转移或者逻辑错误
        case KVSTORE_ERR_IO:
            return "io error";  // 磁盘满、权限不足
        case KVSTORE_ERR_READONLY:
            return "read-only mode";  // 系统处于受限状态（如备份中）
        case KVSTORE_ERR_NOT_FOUND:
            return "key not found";
        case KVSTORE_ERR_WAL_CORRUPTED:
            return "data corrupted";
        default:
            return "unknown error";
    }
}
```

### 2.2 `uint32_t crc32()`
```c

// CRC 函数（简单版）- 循环冗余校验
/**
 * crc32 - 计算字符串的 32 位循坏冗余码（CRC32）
 */
uint32_t crc32(const char* s) {
    // 1. 初始化寄存器
    uint32_t crc = 0xFFFFFFFF;

    // 2. 逐字节处理字符串
    // 遍历输入字符串，直到遇到 '\0'
    while (*s) {
        crc ^= (unsigned char)*s++;
        for (int i = 0; i < 8; i++) {
            crc = (crc >> 1) ^ (0xEDB88320 & -(crc & 1));
        }
    }

    return ~crc;
}
```

### 2.3 `static int kvstore_crc_check()`

```c
/**
 * kvstore_crc_check - 校验单行日志的完整性
 *  - 成功返回 1， 失败返回 0
 * 注意：此函数会修改 line 字符串（就地切割）
 *      是确保存储引擎在崩溃恢复时不载入脏数据的最后屏障
 */
static int kvstore_crc_check(const char* payload, const char* crc_str) {
    // 1. 基础校验
    if (!payload || !crc_str) return 0;

    // 2. 字符串转数字
    char* end = NULL;
    uint32_t crc_stored = (uint32_t)strtoul(crc_str, &end, 10);

    /* 3. 健壮性检查：校验 CRC 字段本身是否损坏  */
    if (end == crc_str || *end != '\0')
        return 0;

    uint32_t crc_calc = crc32(payload);
    return crc_calc == crc_stored;
}
```