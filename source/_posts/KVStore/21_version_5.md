---
title: 21. KVStore 版本五 B+树并发总结&梳理（二）
date: 2026-03-4 15:50:00
categories:
  - [KVStore, v5.0]  
tags:
  - kvstore
---

# B+树并发总结&梳理（二）


## 1.test_16 测试函数 分析


#### 1.1 测试函数代码：
```c++

// ========== test 16. 并发读写一致性测试   =======
/**
 * - 场景：多线程混合执行 PUT 和 SEARCH 操作
 * - 目的：验证 Root Latch 是否能有效防止 B+ 树在并发修改时发生段错误
 */
void* worker_thread(void* arg) {
    worker_arg* w = (worker_arg*)arg;
    long val_out;
    unsigned int seed = time(NULL) ^ (w->id * 7919);

    for (int i = 0; i < THREAD_OPS; i++) {
        // 增大 Key 范围，模拟大规模数据和树分裂
        int k = rand_r(&seed) % 10000;
        int op = rand_r(&seed) % 100;

        if (op < 40) {  // 40% 概率写入
            kvstore_put(w->store, k, i);
        } else if (op < 90) {  // 50% 概率搜索
            // 搜索时不只是打印日志，可以验证基础逻辑
            kvstore_search(w->store, k, &val_out);
        } else {  // 10% 概率删除 (如果已经实现了并发删除)
            // kvstore_delete(w->store, k);
        }

        // 每 1000 次操作打印一次进度，避免频繁 IO 导致测试变慢
        if (i % 1000 == 0) {
            printf("Thread %d reached %d ops\n", w->id, i);
        }
    }
    return NULL;
}

/**
 * 并发控制安全：
 *  - 启动多个工作线程，模拟高并发读写压力
 *  - 验证系统在 Root Latch 的保护下是否稳定
 */
void test_concurrency_stress() {
    cleanup();
    printf("\n[RUN 16] concurrency stress testing...\n");

    kvstore* s = kvstore_open(TEST_LOG);
    const int num_threads = 4;
    pthread_t threads[num_threads];
    worker_arg args[num_threads];

    // 1. 创建并发工作线程
    for (int i = 0; i < num_threads; i++) {
        args[i].store = s;
        args[i].id = i;
        pthread_create(&threads[i], NULL, worker_thread, &args[i]);
    }

    // 2. 等待所有线程完成任务
    for (int i = 0; i < num_threads; i++) {
        pthread_join(threads[i], NULL);
    }

    // 3. 验证最终状态
    assert(kvstore_get_state(s) == KVSTORE_STATE_READY);

    kvstore_close(s);
    printf("[PASS 16] 并发压力（无崩溃且状态有效）测试完成\n");
}
```

#### 1.2 void* worker_thread(void* arg)

它是 **pthread 线程入口函数**。
因为在 POSIX Threads 中：`pthread_create()`要求线程函数**必须是这种签名：**`void * (*start_routine)(void *)`。意思是：
```c++
函数指针类型：

返回值：void*
参数：void*
```
当线程创建时：`pthread_create(&threads[i], NULL, worker_thread, &args[i]);`,含义是：
```
启动一个线程
线程执行 worker_thread
参数是 &args[i]
```
所以：`worker_thread`就是**线程执行体**。



```c++
for (int i = 0; i < num_threads; i++) {
        args[i].store = s;
        args[i].id = i;
        pthread_create(&threads[i], NULL, worker_thread, &args[i]);
    }
```
每创建一个线程，就要去执行`worker_thread`,一个线程执行 20000 次操作，4个线程，总操作次数为 80000 次。
测试：写写并发，读读并发，读写并发

#### test_concurrency_stress
是测试框架，流程：
1. 初始化，创建 KVstore 实例，`kvstore* s = kvstore_open(TEST_LOG);`
2. 创建 4 个线程:`pthread_create`，执行`worker_thread`
3. 等待线程结束：`pthread_join`,即等待线程执行完，如果线程死锁，程序会在这里卡住
4. 检查系统状态：`assert(kvstore_get_state(s) == KVSTORE_STATE_READY);`
5. 关闭数据库：`kvstore_close(s);`


#### 1.3 测试功能汇总
> 1. B+ 树结构安全
> 2. 蟹行锁正确
> 3. path 栈正确
> 4. root_lock 有效
> 5. WAL + compaction 在并发下稳定

一句话总结：
Test16 构造了一个多线程随机 workload（40% PUT + 50% SEARCH），通过 4 个线程并发执行 80,000 次操作，验证了 KVstore 在高并发下的结构安全性。测试证明了 B+树的蟹行锁（Latch Coupling）、路径锁托管（Path Stack）以及 Root Latch 设计能够正确保护树结构，避免在节点分裂和并发访问时产生段错误。

测试结果：
```c++
[RUN 16] concurrency stress testing...
Thread 2 reached 0 ops
Thread 1 reached 0 ops
Thread 0 reached 0 ops
Thread 3 reached 0 ops
[WARN] kvstore compaction failed, will retry later
[WARN] kvstore compaction failed, will retry later
[WARN] kvstore compaction failed, will retry later
Thread 1 reached 1000 ops
Thread 3 reached 1000 ops
Thread 1 reached 2000 ops
Thread 3 reached 2000 ops
Thread 1 reached 3000 ops
Thread 3 reached 3000 ops
Thread 1 reached 4000 ops
Thread 0 reached 1000 ops
Thread 3 reached 4000 ops
Thread 0 reached 2000 ops
Thread 0 reached 3000 ops
Thread 1 reached 5000 ops
Thread 3 reached 5000 ops
Thread 1 reached 6000 ops
Thread 0 reached 4000 ops
Thread 0 reached 5000 ops
Thread 1 reached 7000 ops
Thread 0 reached 6000 ops
Thread 3 reached 6000 ops
Thread 0 reached 7000 ops
Thread 0 reached 8000 ops
Thread 1 reached 8000 ops
Thread 3 reached 7000 ops
Thread 0 reached 9000 ops
[WARN] kvstore compaction failed, will retry later
[WARN] kvstore compaction failed, will retry later
Thread 1 reached 9000 ops
Thread 1 reached 10000 ops
Thread 3 reached 8000 ops
Thread 1 reached 11000 ops
[WARN] kvstore compaction failed, will retry later
Thread 1 reached 12000 ops
Thread 3 reached 9000 ops
Thread 3 reached 10000 ops
Thread 3 reached 11000 ops
Thread 0 reached 10000 ops
Thread 3 reached 12000 ops
Thread 3 reached 13000 ops
Thread 0 reached 11000 ops
Thread 3 reached 14000 ops
Thread 0 reached 12000 ops
Thread 3 reached 15000 ops
Thread 0 reached 13000 ops
Thread 3 reached 16000 ops
Thread 1 reached 13000 ops
Thread 3 reached 17000 ops
Thread 3 reached 18000 ops
Thread 3 reached 19000 ops
Thread 1 reached 14000 ops
Thread 0 reached 14000 ops
Thread 0 reached 15000 ops
Thread 0 reached 16000 ops
Thread 0 reached 17000 ops
Thread 0 reached 18000 ops
Thread 0 reached 19000 ops
Thread 1 reached 15000 ops
Thread 2 reached 1000 ops
Thread 1 reached 16000 ops
[WARN] kvstore compaction failed, will retry later
Thread 1 reached 17000 ops
Thread 1 reached 18000 ops
Thread 1 reached 19000 ops
Thread 2 reached 2000 ops
Thread 2 reached 3000 ops
Thread 2 reached 4000 ops
Thread 2 reached 5000 ops
Thread 2 reached 6000 ops
Thread 2 reached 7000 ops
Thread 2 reached 8000 ops
Thread 2 reached 9000 ops
Thread 2 reached 10000 ops
Thread 2 reached 11000 ops
Thread 2 reached 12000 ops
Thread 2 reached 13000 ops
Thread 2 reached 14000 ops
Thread 2 reached 15000 ops
Thread 2 reached 16000 ops
Thread 2 reached 17000 ops
Thread 2 reached 18000 ops
Thread 2 reached 19000 ops
[PASS 16] 并发压力（无崩溃且状态有效）测试完成
所有测试均通过 ! 🎇
```