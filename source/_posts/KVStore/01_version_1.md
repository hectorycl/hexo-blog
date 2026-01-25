---
title: 01. KVStore 版本一
date: 2026-01-25 11:30:00
categories:
  - [KVStore, v1.0]  # 重点：这表示 KVStore 是父分类，v1.0 是子分类
tags:
  - kvstore
---



## 版本一

### 功能
1. 创建并销毁 KVStore 实例
2. 使用 B+ 树储存数据，支持插入、查找、删除操作

### 结构
1. kvstore 通过 bptree 管理数据

### 稳定性
1. 代码可以正常运行，执行基本功能，并且没有报错


### 文件目录
![目录结构](../img/img01.png)

### 代码

#### kvstore.h
```

// /home/ubuntu/c++2512/KVstore/include/kvstore.h

/**
 * kvstore 结构体负责整个存储引擎，包含一个指向 bptree 的指针
 * B+ 树则是用来存储键值对的数据结构
 */

#ifndef KVSTORE_H
#define KVSTORE_H
#include "index/bptree.h"


#ifdef __cplusplus
extern "C" {
#endif



// 定义 KVSTORE 的结构体
typedef struct _kvstore {
    bptree* tree;  // B+ 树指针
} kvstore;


// 创建 kvstore
kvstore* kvstore_create();
void kvstore_destroy(kvstore* store);

// 插入键值对
int kvstore_insert(kvstore* store, int key, long value);


// 查找键值对
int kvstore_search(kvstore* store, int key, long* out_value);

// 删除键值对
int kvstore_delete(kvstore* store, int key);

#ifdef __cplusplus
}
#endif

#endif

```

#### kvstore.h

```

#include "kvstore.h"

#include <stdio.h>
#include <stdlib.h>

// 创建 KVstore
kvstore* kvstore_create() {
    kvstore* store = (kvstore*)malloc(sizeof(kvstore)); // 分配内存
    if(!store) {
        perror("malloc kvstore");
        exit(EXIT_FAILURE);
    }

    store->tree = bptree_create();

    return store;
}


/**
 * 销毁 kvstore 
 *  - 如果只 free(store), store 指针没有了，B+ 树所有节点没有被释放，变成了“无法访问的垃圾” -> 内存泄漏 ！
 *  - bptree_destroy 递归 free B+ 树的每个节点，最后再把 store 结构体占用的内存释放掉
 * 
 * 代码层面的理解：
 *  - 创建一个对象时是由外向内：
 *      sotre = malloc(...)
 *      sotre->tree = bptree_create()
 *  - 销毁时必须严格反序（由外向内）：
 *      bptree_destroy(...)
 *      free(store)
 */
void kvstore_destroy(kvstore* store) {
    if(store) {
        bptree_destroy(store->tree);
        free(store);
    }
}


// 插入数据到 KVstore
int kvstore_insert(kvstore* store, int key, long value) {
    return bptree_insert(store->tree, key, value);
}

// 查找数据
int kvstore_search(kvstore* store, int key, long* value) {
    return bptree_search(store->tree, key, value);
}


// 删除数据
int kvstore_delete(kvstore* store, int key) {
    return bptree_delete(store->tree, key);
}




/**
 * - make : 编译并生成可执行文件
 * - ./test_kvstore : 运行可执行文件
 * - make clean : 清理生成的目标文件和可执行文件
 */


```


#### test_kvsotre.c
```


///home/ubuntu/c++2512/KVstore/test/test_kvstore.c
#include <stdio.h>
#include "kvstore.h"

int main() {
    // 创建一个 kvstore 实例
    kvstore* store = kvstore_create();

    // 插入数据
    kvstore_insert(store, 1, 100);
    kvstore_insert(store, 2, 200);
    kvstore_insert(store, 3, 300);

    // 查找数据
    long value;
    if(kvstore_search(store, 2, &value) == 0) {
        printf("Found key 2, value: %ld\n", value);  
    } else {
        printf("Key 2 not found!\n");
    }

    // 删除数据
    kvstore_delete(store, 2);

    // 再次查找数据
    if(kvstore_search(store, 2, &value) != 0) {
        printf("Key 2 not found after deletion.\n");
    } else {
        printf("Delete key 2 failed.");
    }

    // 销毁 KVstore
    kvstore_destroy(store);

    return 0;

}

/**
 * 手动编译：
 * 
 * - 编译源文件：
 *      gcc -c src/index/bptree.c -o src/index/bptree.o -Wall -g -I./include -Iinclude/index
 *      gcc -c src/kvstore.c -o src/kvstore.o -Wall -g -I./include
 * - 编译测试文件：
 *      g++ -c test/test_kvstore.c -o test/test_kvstore.o -Wall -g -I./include -Iinclude/index
 * 
 * - 链接所有目标文件：
 *      g++ -o test_kvstore src/index/bptree.o src/kvstore.o test/test_kvstore.o -Wall -g -I./include
 * 
 * 
 * 运行：
 *      ./test_kvstore
 */


```