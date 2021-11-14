---
layout: post
title: Netty内存释放
date: 2021-09-05
Author: zrd
tags: [Netty]
toc: true
---

在 Netty 中内存的申请和释放可以分为4中场景

## 基于内存池的请求 ByteBuf

它是由 Netty 的 NioEventLoop 线程在处理 Channel 的读操作时分配，它的释放有两种策略：

1. 业务 ChannelInboundHandler 继承自 SimpleChannelInboundHandler，实现它的抽象方法 channelRead0，在这种情况下，ByteBuf 由 SimpleChannelInboundHandler 负责释放，相关代码如下：

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

2. 在业务 ChannelInboundHandler 中调用 ctx.fireChannelRead 方法，让请求消息继续向后执行，直到调用 DefaultChannelPipeline 的内部类 TailContext，由它来负责释放请求消息，相关代码如下：

```
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    onUnhandledInboundMessage(ctx, msg);
}

protected void onUnhandledInboundMessage(Object msg) {
    try {
        logger.debug(
                "Discarded inbound message {} that reached at the tail of the pipeline. " +
                        "Please check your pipeline configuration.", msg);
    } finally {
        ReferenceCountUtil.release(msg);
    }
}
```

## 基于非内存池的请求 ByteBuf

业务可以使用非内存池模式覆盖 Netty 默认的内存池模式创建请求 ByteBuf，这种模式也需要按照内存池的方式释放内存。可以通过如下代码修改内存申请策略：

```
bootstrap.handler(new ChannelInitializer<SocketChannel>() {
    protected void initChannel(SocketChannel ch) throws Exception {
        ch.config().setAllocator(UnpooledByteBufAllocator.DEFAULT)
    }
});
```

## 基于内存池的响应 ByteBuf

当我们调用 writeAndFlush 时，消息发送完之后，Netty 会自动帮我们释放内存，如果是堆内存(PooledHeapByteBuf)，则将HeapByteBuffer 转换成 DirectByteBuffer，并释放 PooledHeapByteBuf, 代码如下：

```
protected final ByteBuf newDirectBuffer(ByteBuf buf) {
    final int readableBytes = buf.readableBytes();
    if (readableBytes == 0) {
        ReferenceCountUtil.safeRelease(buf);
        return Unpooled.EMPTY_BUFFER;
    }
    final ByteBufAllocator alloc = alloc();
    if (alloc.isDirectBufferPooled()) {
        ByteBuf directBuf = alloc.directBuffer(readableBytes);
        directBuf.writeBytes(buf, buf.readerIndex(), readableBytes);
        ReferenceCountUtil.safeRelease(buf);
        return directBuf;
    }
    final ByteBuf directBuf = ByteBufUtil.threadLocalDirectBuffer();
    if (directBuf != null) {
        directBuf.writeBytes(buf, buf.readerIndex(), readableBytes);
        ReferenceCountUtil.safeRelease(buf);
        return directBuf;
    }
    return buf;
}
```

消息完整地被写到 SocketChannel 中，则释放 DirectByteBuffer，代码如下：

```
public boolean remove() {
    ChannelOutboundBuffer.Entry e = this.flushedEntry;
    if (e == null) {
        this.clearNioBuffers();
        return false;
    } else {
        Object msg = e.msg;
        ChannelPromise promise = e.promise;
        int size = e.pendingSize;
        this.removeEntry(e);
        if (!e.cancelled) {
            ReferenceCountUtil.safeRelease(msg);
            safeSuccess(promise);
            this.decrementPendingOutboundBytes((long)size, false, true);
        }
        e.recycle();
        return true;
    }
}
```
## 基于非内存池的响应 ByteBuf

与内存池类似。