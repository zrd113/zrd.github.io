---
layout: post
title: 模板模式及在Netty中的应用
date: 2021-08-17
Author: zrd
tags: [Java, Netty]
toc: true
---

## 介绍

模板模式就是在抽象类中定义一个操作的骨架，将一些关键步骤放到子类中去实现，是行为模式之一。当实现一个算法（业务策略）时，有不变的逻辑，也有变化的逻辑，那么就可以把不变的逻辑抽象出来，放到父类，做为模板，将可变的逻辑下放到子类去动态实现。不变的部分，可以封装为一个final的public方法，目的是防止外部改变覆写它的同时也能使用。将动态的部分抽象为一个接口或者抽象方法，让子类去实现。

## Netty中的应用

在 Netty 中很多地方都用到了模板模式，比如说 SimpleChannelInboundHandler，它为用户提供的更便捷的入站处理器骨架实现，先来看下它的 channelRead 源码：

```
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    boolean release = true;
    try {
        if (acceptInboundMessage(msg)) {
            I imsg = (I) msg;
            channelRead0(ctx, imsg);
        } else {
            release = false;
            ctx.fireChannelRead(msg);
        }
    } finally {
        if (autoRelease && release) {
            ReferenceCountUtil.release(msg);
        }
    }
}
```

该方法就是封装的不变的部分，是一个骨架，里面的 channelRead0 是变化的逻辑，是一个抽象方法，让子类实现。

## 与策略模式的比较

模板方法优点是会增强复用，减少冗余代码，并且封装了不变的部分，符合开闭原则，后续代码也容易扩展。缺点是在这个模式里，所有子类强依赖父类，如果父类有方法变动，子类都需要修改。

两个模式都能改变类的行为，都是类的行为模式。但是策略模式改变类的行为，依靠的是委托机制，而模板方法模型改变类的行为，依靠的是继承，按照设计原则，应该是策略模式比较先进，因为它不会受制于父类的变化而被拖累。
