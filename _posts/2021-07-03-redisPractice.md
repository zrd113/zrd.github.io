---
layout: post
title: redis数据类型的使用场景
date: 2021-07-03
Author: zrd
tags: [redis]
toc: true
---

redis常见的5种数据类型是 string、list、set、hash、zset。

## 1. 最新评论列表

可以利用 List 插入的顺序排序实现评论列表，每当一个用户评论，则利用 LPUSH key value 插入到 List 队头。接着再用 LRANGE key star stop 获取列表指定区间内的元素。另外，需要通过时间范围查找的最新列表，List 类型也实现不了，需要通过有序集合 Sorted Set 类型实现，如以成交时间范围作为条件来查询的订单列表。

## 2. 排行榜

比如要一周音乐榜单，我们需要实时更新播放量，并且需要分页展示。 除此以外，排序是根据播放量来决定的，这个时候 List 就无法满足了。 我们可以将音乐 ID 保存到 Sorted Set 集合中，score 设置成每首歌的播放量，该音乐每播放一次则设置 score = score +1。

## 3. 网站的UV

set 常用来实现数据的基数统计，当一个元素从未出现过时，便在集合中增加一个元素；如果出现过，那么集合仍保持不变。比如统计网站的UV。还可以通过hash来实现，将用户 ID 作为 Hash 集合的 key，访问页面则执行 HSET 命令将 value 设置成1。即使用户重复访问，重复执行命令，也只会把这个 userId 的值设置成1。最后利用 HLEN 命令统计 Hash 集合中的元素个数就是 UV。

set虽然使用很方便，但是如果某个网页非常火爆就会消耗太多的内存，所以通过 HyperLogLog 来实现更合适，即使数据量很大需要的空间也是固定的。

## 4. 聚合统计(交集、差集、并集)

### 4.1 共同好友

QQ中的共同好友正是聚合统计中的交集。我们将账号作为 Key，该账号的好友作为 Set 集合的 value。统计两个用户的共同好友只需要两个 Set 集合的交集，SINTERSTORE user:共同好友 user:张三 user:李四。

### 4.2 每日新增好友数

统计某个App每日新增注册用户量，只需要对近两天的总注册用户量集合取差集即可。SDIFFSTORE  user:new  user:20210602 user:20210601，执行完毕，此时的 user:new 集合将是 2021/06/02 日新增用户量。

### 4.3 总共新增好友

统计2021/06/01和2021/06/02两天总共新增的用户量，只需要对两个集合执行并集。SUNIONSTORE  userid:new user:20210602 user:20210601。

## 5. 二值统计

### 5.1 判断用户登录态

Bitmap提供了 GETBIT、SETBIT 操作，通过一个偏移值 offset 对bit数组的 offset 位置的bit位进行读写操作，需要注意的是 offset 从0开始。执行 SETBIT login_status 10086 1，表示用户10086已登录，执行 GETBIT login_status 10086 检查该用户是否登陆，返回值1表示已登录。

### 5.2 用户每个月的签到情况

在签到统计中，每个用户每天的签到用1个bit位表示，一年的签到只需要365个bit位。一个月最多只有31天，只需要31个bit位即可。
执行 SETBIT uid:sign:89757:202105 15 1 表示记录用户在2021年5月16号打卡。统计该用户在 5 月份的打卡次数，使用 BITCOUNT uid:sign:89757:202105 指令。该指令用于统计给定的 bit 数组中，值 = 1 的 bit 位的数量。Redis还提供了 BITPOS key bitValue [start] [end]指令，返回数据表示 Bitmap 中第一个值为 bitValue 的 offset 位置，可以用于统计这个月首次打卡的时间。

### 5.3 连续签到用户总数

我们把每天的日期作为 Bitmap 的 key，userId 作为 offset，若是打卡则将 offset 位置的 bit 设置成 1。 key 对应的集合的每个 bit 位的数据则是一个用户在该日期的打卡记录。 一共有 7 个这样的 Bitmap，如果我们能对这 7 个 Bitmap 的对应的 bit 位做与运算。Redis 提供了 BITOP operation destkey key [key ...]这个指令用于对一个或者多个key的 Bitmap 进行位元操作。

