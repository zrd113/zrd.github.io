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
//链表节点结构
typedef struct listNode{ 
	// 前置节点 
	struct listNode *prev; 
	// 后置节点 
	struct listNode *next; 
	// 节点的值 
	void *value; 
} listNode;

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