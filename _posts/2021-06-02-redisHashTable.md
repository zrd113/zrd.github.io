---
layout: post
title: redis-字典
date: 2021-06-02
Author: zrd
tags: [redis]
toc: true
---

## 一、 字典的定义

```
//哈希表结构
typedef struct dictht{
         //哈希表数组，每个元素保存一个dictEntry结构，dictEntry保存着一个键值对
         dictEntry **table;
         //哈希表大小
         unsigned long size;
         //哈希表大小掩码，用于计算索引值，等于size - 1
         unsigned long sizemask;
         //该哈希表已有节点的数量
         unsigned long used;
}dictht;

//哈希表节点结构
typedef struct dictEntry{
         //键
         void *key;
         //值，可以是一个指针、uint64_t整数或者int64_t整数
         union{
           void *val;
            uint64_tu64;
            int64_ts64;
            }v;
         // 指向下个哈希表节点，形成链表来解决冲突问题
         struct dictEntry *next;
}dictEntry;

//字典结构
typedef struct dict{
         //特定类型函数，每个dictType保存了一组用于操作特定类型键值对的函数，redis会为用途不同的字典设
         //置不同类型的特定函数
         dictType *type;
         //保存了需要传给特定类型函数的可选参数
         void *privdata;
         //哈希表数组，保存了两个哈希表，一个用来存储数据，一个用来rehash
         dictht ht[2];
         //rehash索引 当rehash不在进行时 值为-1
         int trehashidx; 
}dict;

//特定类型函数
typedef struct dictType{
         //计算哈希值的函数 
         unsigned int  (*hashFunction) (const void *key);
         //复制键的函数
         void *(*keyDup) (void *privdata,const void *key);
         //复制值的函数
         void *(*keyDup) (void *privdata,const void *obj);
          //复制值的函数
         void *(*keyCompare) (void *privdata,const void *key1, const void *key2);
         //销毁键的函数
         void (*keyDestructor) (void *privdata, void *key);
         //销毁值的函数
         void (*keyDestructor) (void *privdata, void *obj);
}dictType;
```

## 二、 键的冲突

redis计算hash值和索引值的方式如下
```
hash = dict->type->hashFunction(key); index = hash & dict->ht[0].sizemask;
```
redis的哈希表使用拉链法来解决冲突，当索引产生冲突时会将节点添加在链表表头，也就是头插法。

## 三、 rehash

当哈希表逐渐增多或者减少时，为了让负载因子维持在一个合理的范围之内，会对哈希表的大小进行相应的扩展或收缩。

### 1. rehash操作需要满足以下条件

    1.1 服务器目前没有执行BGSAVE(rdb持久化)命令或者BGREWRITEAOF(AOF文件重写)命令，并且散列表的负载因子大于等于1。
    1.2 服务器目前正在执行BGSAVE命令或者BGREWRITEAOF命令，并且负载因子大于等于5。
    1.3 当负载因子小于0.1时，程序自动开始执行收缩操作。

### 2. rehash步骤

    2.1 为ht[1]分配空间
        扩展操作：ht[1]的大小为 第一个大于等于ht[0].used*2的2的n次方幂。
        收缩操作: ht[1]的大小为 第一个大于等于ht[0].used的2的n次方幂。
    2.2 将保存在ht[0]中的键值对重新计算键的散列值和索引值，然后放到ht[1]指定的位置上。
    2.3 将ht[0]包含的所有键值对都迁移到了ht[1]之后，释放ht[0],将ht[1]设置为ht[0],并创建一个新的ht[1]哈希表为下一次rehash做准备。

### 3. 渐进式rehash

当哈希表中的数据很多时，如果一次性的将全部键值对rehash到ht[1]，庞大的计算量会使服务器在一段时间内停止服务，为了解决一次性扩容耗时过多的情况，可以将扩容操作穿插在正常操作的过程中，分批完成。以下为详细步骤：
    
    3.1 在字典中维持一个索引计数器变量rehashidx，并将它的值设置为0，表示rehash工作正式开始。
    3.2 在 rehash 进行期间，每次对字典执行添加、删除、查找或者更新操作时，程序除了执行指定的操作以外，还会顺带将ht[0]哈希表在rehashidx索引上的所有键值对rehash到ht[1]，当rehash工作完成之后，程序将rehashidx属性的值增一。
    3.3 随着字典操作的不断执行，最终在某个时间点上，ht[0]的所有键值对都会被rehash至ht[1]，这时程序将rehashidx属性的值设为-1，表示rehash操作已完成。

    注意：在进行渐进式rehash的过程中，字典会同时使用ht[0]和ht[1]两个哈希表，所以在渐进式rehash进行期间，字典的删除、查找、更新等操作会在两个哈希表上进行，新添加到字典的键值对一律会被保存到ht[1]里面，而ht[0]则不再进行任何添加操作。