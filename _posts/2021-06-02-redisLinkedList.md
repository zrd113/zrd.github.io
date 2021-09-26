---
layout: post
title: redis-链表
date: 2021-06-02
Author: zrd
tags: [redis]
toc: true
---

## 一、 链表的定义

在redis中，列表的底层实现之一就是链表，它是一种双向无环链表，发布订阅、慢查询、监视器等功能都用到了链表。
```
//链表结构
typedef struct list{
    //表头节点
    listNode *head;
    //表尾节点
    listNode *tail;
    //链表所包含的节点数量
    unsigned long len;
    //节点值复制函数
    void *(*dup)(void *ptr);
    //节点值释放函数
    void *(*free)(void *ptr);
    //节点值对比函数
    int (*match)(void *ptr,void *key);
}list;

//链表节点结构
typedef struct listNode{ 
	// 前置节点 
	struct listNode *prev; 
	// 后置节点 
	struct listNode *next; 
	// 节点的值 
	void *value; 
} listNode;
```

## 二、 链表特点

### 1. 双向

链表节点带有prev和next指针，获取前驱后继的时间复杂度都为O(1)。

### 2. 无环

表头结点的prev和表尾节点的next都指向null。

### 3. list结构带head和tail指针

通过head和tail指针可以以O(1)的时间复杂度获得表头和表尾节点。

### 4. list结构包括len属性

可以以O(1)的时间复杂度获得list的节点数量。

### 5. 多态

链表节点通过void指针保存值，使其可以保存不同类型的值。

## 三、 使用场景

### 1. 带有阻塞功能的消息队列

在项目中，经常会遇到很多耗时的操作，比如将一条微博同步给上百万个用户，如果直接在响应用户的请求过程中执行会等待很长时间，所以我们可以先将某个任务放在队列中，然后由后台线程负责执行，这样只需要一个入队操作就可以向用户返回结果了。在Redis中可以通过RPUSH将消息推入队列，然后通过BLPOP来阻塞获取消息。

```
public class BlockMessageQueue {
    private String key;
    private Jedis jedis;
    public BlockMessageQueue(String key) {
        this.key = key;
        jedis = new Jedis("**.**.**.**", 6379);
        jedis.auth("**");
    }
    public Long addMessage(String ...message) {
        return jedis.rpush(key, message);
    }
    public String getMessage(int timeout) {
        //如果队列没有数据，会阻塞timeout秒，如果timeout为0则会一直阻塞
        List<String> result = jedis.blpop(timeout, key);
        //result包括两个值，第一个是弹出队列，第二个是弹出值
        if (result != null) {
            return result.get(1);
        }
        return null;
    }  
    public Long len() {
        return jedis.llen(key);
    }
}
```