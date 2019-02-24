---
layout: post
title:  "Netty权威指南MessagePack编解码示例"
date:   2019-02-24 10:31:01 +0800
categories: Netty
tag: Netty
---

* content
{:toc}



我相信大家在学习Netty权威指南（第2版）的时候，会在第7章的MessagePack编解码代码运行中出现问题。而且是解决一个问题还有一个问题。并且书中所
带源码并不能帮你解决问题。是哪两个问题呢？
- 问题一，照着源码敲完之后，启动server和client，发现连接上服务器就没反应了。
- 问题二，当你解决了第一步阻塞的问题才会暴露出来。报错`MsgPackDecoder.decode di not read anything but decode a message`
接下来，我们就对这些问题一一重现并分析原因。

### 阻塞
详细代码请看 [这里](https://github.com/lizanle521/springaop/tree/master/src/main/java/com/lzl/netty/chapter7_msgpack/block).

先启动EchoServer.java,然后启动EchoClient.java 
client输出内容
```text
client channel active
[nioEventLoopGroup-2-1][2019-02-24 13:29:19][DEBUG]Slf4JLogger.java:81 - -Dio.netty.leakDetectionLevel: simple
encode ~~~
```
然后server再输出内容
```text
server channel active
```
这说明我们的客户端 连上了服务器，握手成功。但是却没有后续了！我的MessagePackEncoder 应该要有两个打印消息的。
```java
package com.lzl.netty.chapter7_msgpack.block;

import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandlerContext;
import io.netty.handler.codec.MessageToByteEncoder;
import org.msgpack.MessagePack;

/**
 * @author lizanle
 * @data 2019/2/24 8:05 PM
 */
public class MsgpackEncoder extends MessageToByteEncoder<Object> {
    @Override
    protected void encode(ChannelHandlerContext ctx, Object msg, ByteBuf out) throws Exception {
        System.out.println("encode ~~~");
        MessagePack messagePack = new MessagePack();
        // 将对象序列化成流
        byte[] bytes = messagePack.write(msg);
        // 如果断点进去就会发现里边报错了 org.msgpack.MessageTypeException: Cannot find template for class com.lzl.netty.chapter7_msgpack.block.UserInfo class.
        // Try to add @Message annotation to the class or call MessagePack.register(Type).
        System.out.println("after encode bytes:"+bytes.length);
        // 将流写入buffer
        out.writeBytes(bytes);
    }
}
```

然后我们断点messagePack.write()方法进去，发现源码里边是这样的
```
    public <T> byte[] write(T v) throws IOException {
        BufferPacker pk = createBufferPacker();
        if (v == null) {
            pk.writeNil();
        } else {
            @SuppressWarnings("unchecked")
            Template<T> tmpl = registry.lookup(v.getClass());
            tmpl.write(pk, v);
        }
        return pk.toByteArray();
    }
    
    public synchronized Template lookup(Type targetType) {
            Template tmpl;
    
            if (targetType instanceof ParameterizedType) {
                // ParameterizedType is not a Class<?>
                ParameterizedType paramedType = (ParameterizedType) targetType;
                tmpl = lookupGenericType(paramedType);
                if (tmpl != null) {
                    return tmpl;
                }
                targetType = paramedType.getRawType();
            }
    
            tmpl = lookupGenericArrayType(targetType);
            if (tmpl != null) {
                return tmpl;
            }
    
            tmpl = lookupCache(targetType);
            if (tmpl != null) {
                return tmpl;
            }
    
            if (targetType instanceof WildcardType) {
                // WildcardType is not a Class<?>
                tmpl = new AnyTemplate<Object>(this);
                register(targetType, tmpl);
                return tmpl;
            }
    
            Class<?> targetClass = (Class<?>) targetType;
    
            // MessagePackable interface is implemented
            if (MessagePackable.class.isAssignableFrom(targetClass)) {
                // FIXME #MN
                // following processing should be merged into lookAfterBuilding
                // or lookupInterfaceTypes method in next version
                tmpl = new MessagePackableTemplate(targetClass);
                register(targetClass, tmpl);
                return tmpl;
            }
    
            if (targetClass.isInterface()) {
                // writing interfaces will succeed
                // reading into interfaces will fail
                tmpl = new AnyTemplate<Object>(this);
                register(targetType, tmpl);
                return tmpl;
            }
    
            // find matched template builder and build template
            tmpl = lookupAfterBuilding(targetClass);
            if (tmpl != null) {
                return tmpl;
            }
    
            // lookup template of interface type
            tmpl = lookupInterfaceTypes(targetClass);
            if (tmpl != null) {
                return tmpl;
            }
    
            // lookup template of superclass type
            tmpl = lookupSuperclasses(targetClass);
            if (tmpl != null) {
                return tmpl;
            }
    
            // lookup template of interface type of superclasss
            tmpl = lookupSuperclassInterfaceTypes(targetClass);
            if (tmpl != null) {
                return tmpl;
            }
    
            throw new MessageTypeException(
                    "Cannot find template for " + targetClass + " class.  " +
                    "Try to add @Message annotation to the class or call MessagePack.register(Type).");
        }

```
看registry.lookup方法，得知messagepack会为每个要序列化的对象创建一个序列化模版。
显然，我们的UserInfo对象并不能让messagepack给我们这个对象创建模板，并且他提示我们说，要我们给对象加一个 @Message注解。
于是，我们就加了一个@Message注解，就这么一个小小的改动，就解决了第一个问题。 但是为了知道这个小小的改动，我们在这其中做了很多工作才找到问题。

### 出现异常报错
报错内容`MsgPackDecoder.decode di not read anything but decode a message`,那这又是什么原因呢？
ok,我们不是解决了上边的问题嘛，那么我们来跑一跑加了注解后的原程序。
结果server端会出现如下日志
```text
server channel active
[nioEventLoopGroup-3-1][2019-02-24 14:39:48][DEBUG]Slf4JLogger.java:81 - -Dio.netty.leakDetectionLevel: simple
decode ~~~
server received msg:["userid0","user0","kkk0"]
encode ~~~
after encode bytes:20
io.netty.handler.codec.DecoderException: MsgpackDecoder.decode() did not read anything but decoded a message.
	at io.netty.handler.codec.ByteToMessageDecoder.callDecode(ByteToMessageDecoder.java:246)
	......
	at java.lang.Thread.run(Thread.java:748)
server channel read complete
decode ~~~
server received msg:["userid0","user0","kkk0"]
encode ~~~
after encode bytes:20
server channel in active
io.netty.handler.codec.DecoderException: MsgpackDecoder.decode() did not read anything but decoded a message.
	at io.netty.handler.codec.ByteToMessageDecoder.callDecode(ByteToMessageDecoder.java:246)
	......
	at java.lang.Thread.run(Thread.java:748)
```
诶，是不是觉得哪里不对，我明明在客户端只向服务端写了一次对象，现在我服务端却收到了打印了两次收到信息的数据，而且收到的对象都是user0.
那么我们就看一下报错的源码`at io.netty.handler.codec.ByteToMessageDecoder.callDecode(ByteToMessageDecoder.java:246)`
```text
 /**
     * Called once data should be decoded from the given {@link ByteBuf}. This method will call
     * {@link #decode(ChannelHandlerContext, ByteBuf, List)} as long as decoding should take place.
     *
     * @param ctx           the {@link ChannelHandlerContext} which this {@link ByteToMessageDecoder} belongs to
     * @param in            the {@link ByteBuf} from which to read data
     * @param out           the {@link List} to which decoded messages should be added
     */
    protected void callDecode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
        try {
            while (in.isReadable()) {
                int outSize = out.size();
                int oldInputLength = in.readableBytes();
                decode(ctx, in, out);

                // Check if this handler was removed before continuing the loop.
                // If it was removed, it is not safe to continue to operate on the buffer.
                //
                // See https://github.com/netty/netty/issues/1664
                if (ctx.isRemoved()) {
                    break;
                }

                if (outSize == out.size()) {
                    if (oldInputLength == in.readableBytes()) {
                        break;
                    } else {
                        continue;
                    }
                }

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
        } catch (Throwable cause) {
            throw new DecoderException(cause);
        }
    }
```
以上的源码的的意思就是，在decode之前，记录一下 byteBuf的可读字节 和 结果list的大小。
在decode之后，再拿可读字节 和 list结果大小比较，如果list大小变化（读取了一些信息），但是byteBuf的readerIndex却没有变化。那就报错。
为啥报错呢，因为你从空气中读了一些信息。开玩笑，因为`这会导致无限循环，bytebuf永远都又可读字节，list永远都在变大`.
那我们怎么解决这个问题呢，有一个办法，安全的读完那些字节`in.skipBytes(in.readableBytes());`
这样我们就解决了这个问题。

如果有不理解的地方，欢迎发邮件 或者在 我的github项目里提issue问我哦。

### 总结
好的，到了总结的时候了，通过解决这个问题我们学到了什么呢？
- 第一，MessagePack序列化对象之前要先给对象创建模版。可以通过给对象加 `@Message`注解来解决
- 第二，我们的decode方法，如果list增加了结果，你要确保从byteBuf里读了字节。
- 第三，有时候，阻塞问题你只看线程dump是看不出来为什么阻塞的。

### 思考问题
不知道大家发现没有，有一个比较关键的问题是，为啥MessagePack的错误日志没有抛出来呢？他在哪个环节被拦截或者被吃了？怎么才能让他打印出来呢？
