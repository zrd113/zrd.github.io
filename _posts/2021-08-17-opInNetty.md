---
layout: post
title: 观察者模式在Netty中的应用
date: 2021-08-17
Author: zrd
tags: [Java, Netty]
toc: true
---

我们通过 writeAndFlush 方法了解一下观察者模式在Netty中的应用

## writeAndFlush

这个方法是Netty里很常用的异步API，即往外写消息并且刷新缓存。

```
channel.writeAndFlush(rpcMessage).addListener((ChannelFutureListener) future -> {
    if (future.isSuccess()) {
        
    } else {

    }
})
```

当我调用 writeAndFlush 后，会立即返回一个 ChannelFuture 对象，即主题对象，然后为主题对象注册一个监听器即观察者对象，接下来底层会调用 Channel 的 pipeline 的尾节点 TailContext 的 writeAndFlush 方法，如下：

```
public final ChannelFuture writeAndFlush(Object msg) {
    return tail.writeAndFlush(msg);
}
```

接着调用到TailContext节点内部，发现生成了一个 promise 对象(主题对象)，即每次调用异步API都会默认生成一个新的 promise 对象，继续跟进这个newPromise()方法，它生成了一个DefaultChannelPromise的对象，并且将当前 Channel 对象，以及当前的线程对象传入，在executor 方法中，默认策略是如果用户没有配置 executor，那么就使用当前 Channel 绑定的NIO线程来驱动异步回调方法，

```
public ChannelFuture writeAndFlush(Object msg) {
    return writeAndFlush(msg, newPromise());
}

public ChannelPromise newPromise() {
    return new DefaultChannelPromise(channel(), executor());
}

public EventExecutor executor() {
    if (executor == null) {
        return channel().eventLoop();
    } else {
        return executor;
    }
}
```

继续跟进 writeAndFlush

```
public ChannelFuture writeAndFlush(Object msg, ChannelPromise promise) {
    write(msg, true, promise);
    return promise;
}

private void write(Object msg, boolean flush, ChannelPromise promise) {
    
    ...省略部分代码

    final AbstractChannelHandlerContext next = findContextOutbound(flush ? (MASK_WRITE | MASK_FLUSH) : MASK_WRITE);
    final Object m = pipeline.touch(msg, next);
    EventExecutor executor = next.executor();
    // Netty会自动判断当前驱动线程是不是当前Channel绑定的NIO线程
    // 如果是，那么后续该异步API的执行包括其回调方法的执行都由这个NIO线程驱动
    if (executor.inEventLoop()) {
        if (flush) {
            next.invokeWriteAndFlush(m, promise);
        } else {
            next.invokeWrite(m, promise);
        }
    // 否则会将异步API的底层逻辑封装为一个task扔到当前Channel绑定的NIO线程的MPSCQ里，排队等待被NIO线程执行
    } else {
        final WriteTask task = WriteTask.newInstance(next, m, promise, flush);
        if (!safeExecute(executor, task, promise, m, !flush)) {
            task.cancel();
        }
    }
}
```

接下来看看添加监听器的过程

```
public ChannelPromise addListener(GenericFutureListener<? extends Future<? super Void>> listener) {
    super.addListener(listener);
    return this;
}

// 父类的addListener
public Promise<V> addListener(GenericFutureListener<? extends Future<? super V>> listener) {
    checkNotNull(listener, "listener");

    synchronized (this) {
        addListener0(listener);
    }

    if (isDone()) {
        notifyListeners();
    }

    return this;
}

private void addListener0(GenericFutureListener<? extends Future<? super V>> listener) {
    if (listeners == null) {
        listeners = listener;
    } else if (listeners instanceof DefaultFutureListeners) {
        ((DefaultFutureListeners) listeners).add(listener);
    } else {
        listeners = new DefaultFutureListeners((GenericFutureListener<?>) listeners, listener);
    }
}
```

初始情况，即第一次为该API添加监听器，这个 listeners 属性一定是null，故此时它会被赋值为用户设置的 listener 对象；否则，如果是连续添加，那么 listeners 属性是为非 null 的，此时会进入 else 分支，将用户传入的监听器封装为一个默认的DefaultFutureListeners 对象。这样当后续连续添加时，会走 else if 分支，调用 listeners 的 add 方法连续收集。

回到父类的 addListener，假如在执行完核心的 addListener0 方法添加监听器后，这个异步任务已经完成了，此时需要能立即感知，故Netty在调用 addListener0 后立即调用了一次isDone方法，来判断这个异步任务是否已经执行完毕，如果确实异步API已经执行完毕，那么调用通知方法 notifyListeners 回调监听器。

```
if (isDone()) {
    notifyListeners();
}
```

假设走到这里，异步API仍然没有完成，那么最后是谁回调的监听器呢？还得继续跟进 writeAndFlush 方法。通常情况下 writeAndFlush 不会再nio线程中驱动，故Netty会将异步API的底层逻辑封装为task，本质就是一个Runnable，扔到MPSCQ排队等待NIO线程执行，最终NIO线程会执行到这个task的run方法。

```
// 省略一些代码
void invokeWriteAndFlush(Object msg, ChannelPromise promise) {
    if (invokeHandler()) {
        invokeWrite0(msg, promise);
        invokeFlush0();
    } else {
        writeAndFlush(msg, promise);
    }
}
```

新创建的 promise 对象被传入了 invokeWrite0 方法，继续跟进 invokeWrite0，它是在 pipeline 上传播这个 write 事件，最终经过一系列的出站 handler，调用到了 pipeline 的头结点 HeadContext 的 write 方法。

```
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) {
    unsafe.write(msg, promise);
}

// Netty将发送消息的过程分层了，第一层只是将待发送消息缓存到发送队列 outboundBuffer，第二层是择机真正的通过网络将消息发送出去（flush方法）
public final void write(Object msg, ChannelPromise promise) {
    assertEventLoop();

    ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    if (outboundBuffer == null) {
        try {
            ReferenceCountUtil.release(msg);
        } finally {
            // 异步API的流程失败时回调监听方法的逻辑
            safeSetFailure(promise,newClosedChannelException(initialCloseCause, "write(Object, ChannelPromise)"));
        }
        return;
    }

    int size;
    try {
        msg = filterOutboundMessage(msg);
        size = pipeline.estimatorHandle().size(msg);
        if (size < 0) {
            size = 0;
        }
    } catch (Throwable t) {
        try {
            ReferenceCountUtil.release(msg);
        } finally {
            // 异步API的流程失败时回调监听方法的逻辑
            safeSetFailure(promise, t);
        }
        return;
    }
    // 正常情况的回调是发生在flush的流程里，该方法将待发送消息msg，连同promise封装为了一个Netty封装的Entry对象，作为发送缓冲区里的链表节点
    outboundBuffer.addMessage(msg, size, promise);
}
```

接下来看 flush 方法

```
void invokeWriteAndFlush(Object msg, ChannelPromise promise) {
    if (invokeHandler()) {
        invokeWrite0(msg, promise);
        invokeFlush0();
    } else {
        writeAndFlush(msg, promise);
    }
}

// 最终会调用头节点的 flush，具体实现逻辑先不管，只需要知道最终的最终在成功刷新消息后，会删除发送缓冲区中已经刷新出去的这个消息所在的entry节点。
public boolean remove() {
    Entry e = flushedEntry;
    if (e == null) {
        clearNioBuffers();
        return false;
    }
    Object msg = e.msg;

    ChannelPromise promise = e.promise;
    int size = e.pendingSize;

    removeEntry(e);

    if (!e.cancelled) {
        ReferenceCountUtil.safeRelease(msg);
        // 异步任务流程成功后的回调监听方法
        // 调用 promise 的 trySuccess 方法，然后通知观察者
        safeSuccess(promise);
        decrementPendingOutboundBytes(size, false, true);
    }

    e.recycle();

    return true;
}

private boolean setValue0(Object objResult) {
    if (RESULT_UPDATER.compareAndSet(this, null, objResult) ||
        RESULT_UPDATER.compareAndSet(this, UNCANCELLABLE, objResult)) {
        if (checkNotifyWaiters()) {
            notifyListeners();
        }
        return true;
    }
    return false;
}
```

## 总结

主题通知观察者的流程：在异步任务完成后，会调用 safeSuccess（相应的异常情况下会调用safeSetFailur），然后是 trySuccess —> notifyListener0，最终回调了 operationComplete 方法。

一个细节，Netty的异步API的回调流程都是NIO线程驱动的，即如果外部调用该API的线程仍然是当前Channel绑定的NIO线程，那么底层其实是同步串行执行的。