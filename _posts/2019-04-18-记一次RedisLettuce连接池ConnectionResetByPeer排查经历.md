---
layout: post
title:  "记一次RedisLettuce连接池ConnectionResetByPeer排查经历"
date:   2019-04-18 11:42:01 +0800
categories: Netty
tag: Netty
---

* content
{:toc}

### 事情的起因
我们之前的项目采用的是zookeeper的分布式锁，由于zookeeper是自己装的，作用很多，做了分布式锁的同时做了很多个项目的注册中心，且是单机的，
导致一旦zookeeper挂掉影响多个项目的可用性。耦合性太强。于是，我在一个新的项目中，重新采用springboot2架构了一套新的开发框架。同时借这个机会
引入了redis的分布式锁。虽然redis的分布式在高并发下有惊群效应，但是对我们这个项目，这个redis分布式锁更加合理。
其实引入分布式锁本身这个只是引入的版本有问题，就是redisson版本3.5.0在获取分布式锁的情况下，经常报错：
```text
org.redisson.client.RedisTimeoutException: Redis server response timeout (3000 ms) occured for command: (EVAL) with params: [if (redis.call('exists', KEYS[1]) == 0) then redis.call('publish', KEYS[2], ARG
V[1]); return 1; end;..., 2, lock:anchor:tag:0:platform:null, redisson_lock__channel:{lock:anchor:tag:0:platform:null}, 0, 30000, 7895d555-0613-49c5-ad99-526e6a7d59eb:53] channel: [id: 0x2c2da1f1, L:/192.
168.10.162:49911 - R:test.redis.uuuwin.com/121.40.155.5:7480]
        at org.redisson.command.CommandAsyncService$10.run(CommandAsyncService.java:646)
        at io.netty.util.HashedWheelTimer$HashedWheelTimeout.expire(HashedWheelTimer.java:682)
        at io.netty.util.HashedWheelTimer$HashedWheelBucket.expireTimeouts(HashedWheelTimer.java:757)
        at io.netty.util.HashedWheelTimer$Worker.run(HashedWheelTimer.java:485)
        at java.lang.Thread.run(Thread.java:745)
```
这个情况大家都遇到过，可以看这个[链接地址](https://www.cnblogs.com/junge8618/p/9241927.html),很简单，升级一下版本就好了。
说到这里，问题才开始。正因为我们只是升级了一下版本，所以我们担心问题还出现，所以就对日志格外关心，并且经常放一段时间突然去访问一下（是测试环境，还没上线）
我们发现，经常是在早晨的第一次访问出现如下错误：
```text
org.springframework.data.redis.RedisSystemException: Redis exception; nested exception is io.lettuce.core.RedisException: java.io.IOException: Connection reset by peer
        at org.springframework.data.redis.connection.lettuce.LettuceExceptionConverter.convert(LettuceExceptionConverter.java:74)
        at org.springframework.data.redis.connection.lettuce.LettuceExceptionConverter.convert(LettuceExceptionConverter.java:41)
        at org.springframework.data.redis.PassThroughExceptionTranslationStrategy.translate(PassThroughExceptionTranslationStrategy.java:44)
        at org.springframework.data.redis.FallbackExceptionTranslationStrategy.translate(FallbackExceptionTranslationStrategy.java:42)
        at org.springframework.data.redis.connection.lettuce.LettuceConnection.convertLettuceAccessException(LettuceConnection.java:268)
        at org.springframework.data.redis.connection.lettuce.LettuceConnection.multi(LettuceConnection.java:672)
        at sun.reflect.GeneratedMethodAccessor173.invoke(Unknown Source)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.lang.reflect.Method.invoke(Method.java:498)
        at org.springframework.data.redis.core.RedisConnectionUtils$ConnectionSplittingInterceptor.invoke(RedisConnectionUtils.java:351)
        at org.springframework.data.redis.core.RedisConnectionUtils$ConnectionSplittingInterceptor.intercept(RedisConnectionUtils.java:329)
        at org.springframework.data.redis.core.RedisConnectionUtils$ConnectionSplittingInterceptor.invoke(RedisConnectionUtils.java:359)
        at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:186)
        at org.springframework.aop.framework.JdkDynamicAopProxy.invoke(JdkDynamicAopProxy.java:212)
        at com.sun.proxy.$Proxy124.multi(Unknown Source)
        at org.springframework.data.redis.core.RedisConnectionUtils.potentiallyRegisterTransactionSynchronisation(RedisConnectionUtils.java:163)
        at org.springframework.data.redis.core.RedisConnectionUtils.doGetConnection(RedisConnectionUtils.java:145)
        at org.springframework.data.redis.core.RedisConnectionUtils.bindConnection(RedisConnectionUtils.java:70)
        at org.springframework.data.redis.core.RedisTemplate.execute(RedisTemplate.java:209)
        at org.springframework.data.redis.core.RedisTemplate.execute(RedisTemplate.java:184)
        at org.springframework.data.redis.core.AbstractOperations.execute(AbstractOperations.java:95)
        at org.springframework.data.redis.core.DefaultValueOperations.get(DefaultValueOperations.java:53)
        at com.xx.xx.service.redis.RedisService.getAndSet(RedisService.java:116)
        at com.xx.xx.service.redis.RedisService$$FastClassBySpringCGLIB$$25a3d0d6.invoke(<generated>)
```
这个错误导致第一次访问页面全空白。问题说大也不大，说小也不小。做为较真的技术人，我们只想知道为什么会出现这种情况
### 事情的承接
我们就开始对问题进行了分析：connection reset by peer .表示当前服务器接受到了通信对端发送的TCP RST信号，即通信对端已经关闭了连接，
通过RST信号希望接收方关闭连接。这个问题很常见，在tcp的通信过程中
- 当尝试和未开放的服务器端口建立tcp连接时，服务器tcp将会直接向客户端发送reset报文；
- 双方之前已经正常建立了通信通道，也可能进行过了交互，当某一方在交互的过程中发生了异常，如崩溃等，异常的一方会向对端发送reset报文，通知对方将连接关闭；
- 当收到TCP报文，但是发现该报文不是已建立的TCP连接列表可处理的，则其直接向对端发送reset报文；
- ack报文丢失，并且超出一定的重传次数或时间后，会主动向对端发送reset报文释放该TCP连接；

简单的总结：通信过程中，对方链接关闭了，我方去链接或者通信就会报这个错误。
于是，这简单啊，我们设置客户端的连接超时时间不就完了？对吧，超过一定的空闲时间，自己关闭连接呀。完美，就这么干。
于是我们设置了连接参数
```text
# 超时时间
spring.redis.timeout=10000
spring.redis.database=0
# 连接池最大连接数（使用负值表示没有限制）
spring.redis.lettuce.pool.max-active=8
# 连接池最大阻塞等待时间（使用负值表示没有限制）
spring.redis.lettuce.pool.max-wait=10000
# 连接池中的最大空闲连接
spring.redis.lettuce.pool.max-idle=8
# 连接池中的最小空闲连接
spring.redis.lettuce.pool.min-idle=0
# 关闭超时时间
spring.redis.lettuce.shutdown-timeout=1000
```
这样，最小的空闲链接数是0，空闲一阵子后，客户端链接也关闭了，不会有问题了吧。
可是，结果非常的令人沮丧，第二天早晨一测，又出现了！！再拖下去，

### 事情的转折
又一天上午，我在想，既然这个问题这么邪门，我先去了解一下这个链接池的介绍吧
```text
Connection factory creating <a href="http://github.com/mp911de/lettuce">Lettuce</a>-based connections.
 * <p>
 * This factory creates a new {@link LettuceConnection} on each call to {@link #getConnection()}. Multiple
 * {@link LettuceConnection}s share a single thread-safe native connection by default.
 * <p>
 * The shared native connection is never closed by {@link LettuceConnection}, therefore it is not validated by default
 * on {@link #getConnection()}. Use {@link #setValidateConnection(boolean)} to change this behavior if necessary. Inject
 * a {@link Pool} to pool dedicated connections. If shareNativeConnection is true, the pool will be used to select a
 * connection for blocking and tx operations only, which should not share a connection. If native connection sharing is
 * disabled, the selected connection will be used for all operations.
```
一看不要紧，看了就恍然大悟，原来这个连接池是reactor模式的，真正的redis链接只有一个，并且不会去校验这个链接。
马上改下代码：
```text
 @Bean
    public RedisConnectionFactory getConnectionFactory() {
        //单机模式
        RedisStandaloneConfiguration configuration = new RedisStandaloneConfiguration();
        configuration.setHostName(host);
        configuration.setPort(port);
        configuration.setDatabase(database);
        configuration.setPassword(RedisPassword.of(password));
        //哨兵模式
        //RedisSentinelConfiguration configuration1 = new RedisSentinelConfiguration();
        //集群模式
        //RedisClusterConfiguration configuration2 = new RedisClusterConfiguration();
        LettuceConnectionFactory factory = new LettuceConnectionFactory(configuration, getPoolConfig());
        // 使用前先校验连接
        factory.setValidateConnection(true);
        //factory.setShareNativeConnection(false);//是否允许多个线程操作共用同一个缓存连接，默认true，false时每个操作都将开辟新的连接
        return factory;
    }
```
添加了一行`factory.setValidateConnection(true);`. 增加这个之后，每次getConnection都会去校验 connection的 active属性。
```text
StatefulConnection<E, E> getConnection() {

			synchronized (this.connectionMonitor) {

				if (this.connection == null) {
					this.connection = getNativeConnection();
				}
                // 如果设置为校验链接，那么就调用validateConnection
				if (getValidateConnection()) {
					validateConnection();
				}

				return this.connection;
			}
		}
```
我们来看看 validateConnection源码：
```text
void validateConnection() {

			synchronized (this.connectionMonitor) {

				boolean valid = false;
                // 校验链接状态
				if (connection != null && connection.isOpen()) {
					try {

						if (connection instanceof StatefulRedisConnection) {
							((StatefulRedisConnection) connection).sync().ping();
						}

						if (connection instanceof StatefulRedisClusterConnection) {
							((StatefulRedisConnection) connection).sync().ping();
						}
						valid = true;
					} catch (Exception e) {
						log.debug("Validation failed", e);
					}
				}
                // 连接已断开
				if (!valid) {

					if (connection != null) {
						connectionProvider.release(connection);
					}

					log.warn("Validation of shared connection failed. Creating a new connection.");

					resetConnection();
					// 获取新连接
					this.connection = getNativeConnection();
				}
			}
		}
```
那这个链接的 isOpen状态是怎么改变的呢 。先看isOpen源码
```text
public boolean isOpen() {
        return active;
    }
```
只是一个属性的校验，那么这个校验的属性在什么时候改变呢。
```text
public void deactivated() {
        active = false;
    }
```
deactivated方法什么时候调用的呢
```text

    @Override
    public void notifyChannelInactive(Channel channel) {

        if (isClosed()) {
            RedisException closed = new RedisException("Connection closed");
            cancelCommands("Connection closed", drainCommands(), it -> it.completeExceptionally(closed));
        }

        sharedLock.doExclusive(() -> {

            if (debugEnabled) {
                logger.debug("{} deactivating endpoint handler", logPrefix());
            }

            connectionFacade.deactivated();
        });

        if (this.channel == channel) {
            this.channel = null;
        }
    }
    
   
```
最终我们找到调用者 是 CommandHandler里边的 channelInactive方法
```text
 public void channelInactive(ChannelHandlerContext ctx) throws Exception {

        if (debugEnabled) {
            logger.debug("{} channelInactive()", logPrefix());
        }

        if (channel != null && ctx.channel() != channel) {
            logger.debug("{} My channel and ctx.channel mismatch. Propagating event to other listeners.", logPrefix());
            super.channelInactive(ctx);
            return;
        }

        tracedEndpoint = null;
        setState(LifecycleState.DISCONNECTED);
        setState(LifecycleState.DEACTIVATING);

        endpoint.notifyChannelInactive(ctx.channel());
        endpoint.notifyDrainQueuedCommands(this);

        setState(LifecycleState.DEACTIVATED);

        PristineFallbackCommand command = this.fallbackCommand;
        if (isProtectedMode(command)) {
            onProtectedMode(command.getOutput().getError());
        }

        rsm.reset();

        if (debugEnabled) {
            logger.debug("{} channelInactive() done", logPrefix());
        }

        super.channelInactive(ctx);
    }
```
我们知道，tcp的通信是全双工的，断开连接 对方能知道。CommandHandler 继承了 ChannelDuplexHandler 
而这个handler会接受到 连接的 建立 断开等IO消息。所以，下次如果服务器连接断开了，那么客户端就能收到通知，将本地连接active置为false
这样子我们的RedisLettuceConnectionFactory.getConnection的时候就能够判断连接是否可用，如果不可用，就重新连接连接。
而不是像这次一样直接报错 connect reset by peer . 我们本来有其他措施可以做，譬如业务层重试。但是方案不是优雅。

### 最终解决方案
添加了一行`factory.setValidateConnection(true);`













