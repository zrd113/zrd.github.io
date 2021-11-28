---
layout: post
title: Netty编解码器实现原理
date: 2021-09-10
Author: zrd
tags: [Netty]
toc: true
---

## 编码解码概述

Decoder(解码器) 继承自 Inbound 事件处理器，而Encoder(编码器)继承自Outbound事件处理器。当服务端通过网络IO接收到字节序列时，从底层网络套接字中将字节流读取到接受缓存区(ByteBuf)，服务端的职责首先需要从二进制流中解码出一个完整的请求，然后“读懂请求”的含义进行对应的业务逻辑处理，处理完毕后首先需要将响应结果编码成二进制流，通过网络进行传递，客户端收到二进制流后同样进行解码。

Netty 中共包含4个编解码核心类：
(1)ByteToMessageDecoder 解码器，将字节流解码成消息(message)。
(2)MessageToByteEncoder 编码器，将消息(message)编码成字节流。
(3)MessageToMessageEncoder 编码器,将消息编码成”另一种消息“，更通用，”另一种消息“由泛型指定。
(4)MessageToMessageDecoder 解码器，将消息解码成“另一种消息”，更通用，“另一种消息”由泛型指定。

## 解码器实现原理

引入解码器的目的是为了解决网络编程中的粘包问题。ByteToMessageDecoder 是 Netty 解码器实现的基类，是一种模板设计模式。Netty 的扩展是基于事件链机制，即解码器实现的是 InBound 事件处理器。

### channelRead

```
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    // 如果msg是ByteBuf则进行解码，否则沿着事件链向后传播
    if (msg instanceof ByteBuf) {
        // 用来存储解码器解码出来的消息
        CodecOutputList out = CodecOutputList.newInstance();
        try {
            // first为true，表示第一次解码消息
            first = cumulation == null;
            cumulation = cumulator.cumulate(ctx.alloc(),
                    first ? Unpooled.EMPTY_BUFFER : cumulation, (ByteBuf) msg);
            
            // 对缓存区中的数据进行解码，结果存放在out中
            callDecode(ctx, cumulation, out);
        } catch (DecoderException e) {
            throw e;
        } catch (Exception e) {
            throw new DecoderException(e);
        } finally {
            try {
                // 缓存区不为空并且缓存区中的数据已全部处理，重置numReads与cumulation。
                if (cumulation != null && !cumulation.isReadable()) {
                    numReads = 0;
                    cumulation.release();
                    cumulation = null;
                //  numReads 超过 discardAfterReads，需要将缓存区中已读取的数据抛弃
                } else if (++numReads >= discardAfterReads) {
                    numReads = 0;
                    discardSomeReadBytes();
                }
                // 解码后的数据向后传播
                int size = out.size();
                firedChannelRead |= out.insertSinceRecycled();
                fireChannelRead(ctx, out, size);
            } finally {
                // 对out进行回收
                out.recycle();
            }
        }
    } else {
        ctx.fireChannelRead(msg);
    }
}
```

接下来看下 callDecode

```
protected void callDecode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
    try {
        while (in.isReadable()) {
            int outSize = out.size();
            // 有解码好的内容直接发送出去
            if (outSize > 0) {
                fireChannelRead(ctx, out, outSize);
                out.clear();
                // 继续解码前检查下handler是否移除
                if (ctx.isRemoved()) {
                    break;
                }
                outSize = 0;
            }
            // cumulation可读字节缓存起来
            int oldInputLength = in.readableBytes();
            // 调用抽象方法docode进行解码
            decodeRemovalReentryProtection(ctx, in, out);
            if (ctx.isRemoved()) {
                break;
            }
            
            // outSize = 0说明没有解析出东西
            if (outSize == out.size()) {
                // 没有读取，字节数据不够
                if (oldInputLength == in.readableBytes()) {
                    break;
                } else {
                // 读了但是没有解析出数据
                    continue;
                }
            }
            // 解析出数据了，但是readIndex没动，说明出错了
            if (oldInputLength == in.readableBytes()) {
                throw new DecoderException(
                        StringUtil.simpleClassName(getClass()) +
                                ".decode() did not read anything but decoded a message.");
            }
            if (isSingleDecode()) {
                break;
            }
        }
    } catch (DecoderException e) {
        throw e;
    } catch (Exception cause) {
        throw new DecoderException(cause);
    }
}
```

## 编码器实现原理

MessageToByteEncoder是outbound处理器，只需 wrtie 事件做处理。

### write

```
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
    ByteBuf buf = null;
    try {
        // 判断对象类型是否符合该编码器期待的类型，符合则对数据进行编码，否则直接调用ctx.write传播write事件。
        if (acceptOutboundMessage(msg)) {
            I cast = (I) msg;
            buf = allocateBuffer(ctx, cast, preferDirect);
            try {
                // 调用抽象方法进行编码
                encode(ctx, cast, buf);
            } finally {
                ReferenceCountUtil.release(cast);
            }
            // buf可读则将数据传递下去
            if (buf.isReadable()) {
                ctx.write(buf, promise);
            } else {
                buf.release();
                ctx.write(Unpooled.EMPTY_BUFFER, promise);
            }
            buf = null;
        } else {
            ctx.write(msg, promise);
        }
    } catch (EncoderException e) {
        throw e;
    } catch (Throwable e) {
        throw new EncoderException(e);
    } finally {
        if (buf != null) {
            buf.release();
        }
    }
}
```