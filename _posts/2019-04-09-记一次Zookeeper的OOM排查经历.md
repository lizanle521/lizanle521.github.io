---
layout: post
title:  "记一次Zookeeper的OOM排查经历"
date:   2019-04-09 16:12:01 +0800
categories: Zookeeper
tag: 问题排查
---

* content
{:toc}

### 事情的起因
事情的发生梗概
1. 某一天，线上Zookeeper 连不上了，我们代码中用的zk的分布式锁，用不了了就下不了单，各种报警

2. 我去线上看了一下，cpu100%。 先把日志备份了一下（重启会覆盖日志），然后看了一下日志里边的错误。
没有其他错误，就是有些Socket异常。我抓了zk日志里边的一个 sessionid,查找了一下sessionid的建立和关闭时刻，发现竟然长达60秒。
于是我把sessionid拿到服务器的日志里去查找，发现是一个定时任务在调用一个dubbo服务，dubbo用了分布式锁，在锁期间，这个dubbo服务
里边去请求了一个地址，而这个地址的域名是被封了（没有备案），导致这个dubbo一直阻塞直到超时（没错，我们的dubbo超时设置为了60s）。
zk的超时时间默认是40s. 就是说链接时间内，这个会话一直存在（分布式锁是临时顺序节点，dubbo服务没结束，锁没释放，会话没退出）。这就触发（不稳定的触发）了隐藏的一个著名的
bug`nio epoll` . 因为Zookeeper 3.4.8版本默认用的是NIOServerCnxn,原生的NIO就有这个bug.此处不展开。

### 事情的承接
1. 怎么解决这个bug呢。很显然，我们知道netty解决了这个问题，通过对空转计数重建selector. 并且Zookeeper3.4.x版本也是引入了对netty的支持。
我们可以在zoo.cfg里配置
```text
serverCnxnFactory=org.apache.zookeeper.server.NettyServerCnxnFactory
```
就可以达到支持netty了。

为了保险起见，我特意设置了一个zkServer.sh 里边的jvm参数
```text
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/path/to/zkjvmerror
``` 
看起来一切都十分完美。
### 事情的转折
突然有一天，我又发现Zookeeper连不上了，心里一凉。马上上线查看，zk进程还在，但是已经不响应了。看了一下端口开启情况，如图：
![Alt zkclosewait](/styles/images/zkclosewait.png) 
看到这里大家会说，哎哟，closewait,这个我知道，肯定是zk代码写的不对，导致的链接没有释放。
很显然，zk如果有这么严重的问题的话，就不会有这么广阔的使用场景了。
没办法，我又备份了日志，并且先重启了zk. 然后将zkjvmerror（对，没错，就是这个heapdump日志）下载到了本地。
下载了eclipseMemoryAnalyzer,将这个文件导入了mat.
这么一打开，不看不知道，一看吓一跳。各位，请看下图：
![Alt overview](/styles/images/matoverview.png)
DataTree这个对象占用了1.3G. 明显的内存泄露没跑了。为什么DataTree会占用这么大的内存呢。我们知道DataTree是zk的内存的抽象
就是zk的各种节点都在这里有保存。那么他既然是zk的内存抽象，所以肯定要继续分析它(他是根对象)，我们要看看这个DataTree里边的
对象，于是我们点击这个里边的Leak Suspects-->Details. 我们发现下图：
![Alt mattree](/styles/images/mattree.png)
这说明什么，DataTree里边的WatchManager占了99%。那我们继续看这个对象的内存占用，怎么看呢，我们点击一个WatchManager-->List Objects
--> with outgoing references . 看WatchManager的出引用，就是看他包含了哪些东西。我们看到如下图：
![Alt watchmanager](/styles/images/matwatchmanager.png)
这说明里边的watch2paht内存泄露了。
本来我怀疑的是：
watch2path保存了会话 和 会话下的路经集合。NIOServerCnxn会有一个移除会话的操作。但是NettyServerCnxn中是用的用户线程池，所以不需要移除。
Netty用户线程池用的是 Executors.newCachedThreadPool()建立的worker线程池，这个线程池的缺点就是如果线程运行时间长，则其余的线程则会无限new
在60秒内建立了大量的会话（11万个），
导致了内存泄露并最终溢出。
结果呢，我对这个heapdump进行了一个oql的查询：
```text
SELECT established.fastTime FROM org.apache.zookeeper.server.NettyServerCnxn  
SELECT (lastResponseTime - established.fastTime) FROM org.apache.zookeeper.server.NettyServerCnxn  
```
然后我将里边的 fastTime 这个毫秒时间戳通过 new Date(fastTime) 并格式化之后，发现这些链接建立并不是在一分钟之内。
我就去查找了一下zookeeper的jira.发现了一个如下的issue:
```text
https://issues.apache.org/jira/browse/ZOOKEEPER-3131?jql=project%20%3D%20ZOOKEEPER%20AND%20text%20~%20OOM
```
这个issue里边的一段话是这么说的：
```text
In some cases, the variable watch2Paths in Class WatchManager does not remove the entry, 
ven if the associated value "HashSet" is empty already. 
The type of key in Map watch2Paths is Watcher, instance of NettyServerCnxn. 
If it is not removed when the associated set of paths is empty, 
it will cause the memory increases little by little, and OutOfMemoryError triggered finally. 
```
显然，我们换成了NettyServerCnxn之后导致的 。这个issue已经被fix了。

### 本地验证
我为了验证是NettyServerCnxn导致的问题，进行了一个简单的验证。
用的是[apache Dubbo 样例源码](https://github.com/apache/incubator-dubbo-samples)里边的dubbo-sample-basic模块。
思路很简单，就是dubbo-consumer启动后会建立连接，断掉后会断开连接
只需要在启动的时候设置 
```text
 System.setProperty(ServerCnxnFactory.ZOOKEEPER_SERVER_CNXN_FACTORY,"org.apache.zookeeper.server.NettyServerCnxnFactory");
```
就可以启用NettyServerCnxnFactory。然后我们启用provider,对consumer不停的断开重启断开重启，在jconsole里边我们手动强制heapdump,
再次导入mat,用
```text
SELECT established.fastTime FROM org.apache.zookeeper.server.NettyServerCnxn  
```
一看，明明只有2个会话，却存在这么多的 会话watch对象。就很显然了。目前还没去看3.6.0版本是怎么解决这个问题的。
### 最终解决方案
1. dubbo超时时间设为10s
2. zookeeper版本升级到3.6.0











