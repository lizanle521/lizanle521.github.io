---
layout: post
title:  "CompleteableFuture详解"
date:   2019-03-15 12:43:01 +0800
categories: java基础
tag: 多线程
---

* content
{:toc}

## 异步并发选型
| 描述 | Future | FutureTask	| CompletionService | CompletableFuture| 
| ----------- | ----------------| -----------------| -----------------|---------------------|

|原理|Future接口|接口RunnableFuture的唯一实现类，RunnableFuture接口继承自Future+Runnable|内部通过阻塞队列+FutureTask接口|JDK8实现了Future, CompletionStage两个接口|
|多任务并发执行|支持|支持|支持|支持|
|获取任务结果的顺序|按照提交顺序获取结果|未知	|支持任务完成的先后顺序|支持任务完成的先后顺序|
|异常捕捉|自己捕捉|自己捕捉|自己捕捉|原生API支持，返回每个任务的异常|
|建议|CPU高速轮询，耗资源，可以使用，但不推荐|功能不对口，并发任务这一块多套一层，不推荐使用|推荐使用，没有JDK8CompletableFuture之前最好的方案，没有质疑|API极端丰富，配合流式编程，速度飞起，推荐使用！|


在Java 8中, 新增加了一个包含50个方法左右的类: CompletableFuture，提供了非常强大的Future的扩展功能，可以帮助我们简化异步编程的复杂性，
提供了函数式编程的能力，可以通过回调的方式处理计算结果，并且提供了转换和组合CompletableFuture的方法。
要想准确的理解CompletableFuture，我们必须了解他实现的接口的Future和CompletionStage

## Future源码简介翻译
 一个Future代表了一个异步计算的结果，他提供了一些方法用来监测计算是否完成，去等待他的完成，去获得计算的结果。
 他的计算完成的时候，结果只能get()方法获取，否则一直阻塞。可以通过 cancle()方法来进行取消计算，其他的方法被用来检查任务是否正常完成或者
 已经取消，一旦计算完成，计算就不能被取消。如果你想去使用`Future`由于他的可取消性，但你又不知道会返回一个什么样的结果，你可以使用
 Future<?> 来声明结果类型并且返回null作为潜在任务的结果。
 
## CompletionStage源码简介翻译
  CompletionStage是一个异步计算的`可能`的步骤,这个步骤可能不会执行。在其他步骤执行完成之后进行操作或者计算值。CompletionStage的计算终止
  则他就complete了，但这可能会依次引发其他依赖的步骤。这个接口定义的功能只有几种基本的形式，能够被扩展成大量的方法去覆盖广阔的使用场景。
  - 步骤执行的计算可能通过Function，Consumer,或者Runnable（使用方法名分别包括apply,accept,run）来表现，依赖他们要求的参数或者产生的结果。
  例如：
  ```text
        stage.thenApply(x->square(x)).thenAccept(x->System.out.println(x)).thenRun(()->System.out.println()).
  ```
  一个额外的形式是`compose()`,应用于步骤本身的功能，而不是他们的结果。
  
  - 一个步骤的执行可能是被单个步骤的完成触发，或者两个步骤，或者两个步骤之一。
  单个步骤的依赖被设计为使用方法前缀为`then`,那些被两个步骤完成触发的步骤 可以用对应的方法`combine`去组合他们的结果或者影响
  那些被两个步骤之一`either`触发的步骤，则无法保证哪个结果或者影响被使用于当前触发的步骤计算。
  
  - 步骤之间的依赖控制着步骤计算的触发，但是并没有保证任何特定的顺序，此外，一个新步骤的计算的执行可能被任意以下三种方式安排：
  1. 默认运行
  2. 默认异步运行（使用后缀为`async`的方法使用步骤默认异步执行的能力）
  3. 自定义（通过提供Executor）
  默认的运行属性和异步模式在本接口的实现类里边指定，不是接口指定。特定的Executor参数可能有许多的运行属性，甚至可能不支持并行执行
  
  - 两个方法用来支持处理当正在触发的步骤正常或者异常完成的情况。
  1. whenComplete方法支持注入一个动作无论结果是啥样的，否则在他完成的时候保留结果
  2. handle方法额外允许一个步骤去执行替代结果或者让其他依赖步骤去进一步处理
  在其他所有的案例中，如果一个步骤计算由于一个未检查的异常突然终止，那么所有的步骤一样的获取到它的异常结果，并且是CompletionExceptin保存了这个错误原因。
  如果一个步骤依赖`both`两个步骤，并且两个步骤都抛出了异常，CompletionExceptin就可能持有任意一个步骤的异常。
  如果一个步骤依赖`either`任一两个步骤，并且其中一个步骤抛出了异常，没有任何保证这个依赖的步骤会正常或者异常。
  在`whenComplete`的例子中，当提供的动作本身遇到了异常，如果CompletableFuture没有完成异常，那么这个异常的动作会完成异常
  
   
## CompletableFuture源码简介翻译
 这是一个能被明确完成的(设置状态和值)的`Future`，也可能被用作`CompletionStage`,支持依赖的函数和动作在完成的时候触发
 当两个或者两个以上的线程尝试去
 - complete
 - completeExceptionally
 - cancel

 的时候，作为一个CompletableFuture，只有一个线程能够完成。
 除了这些和直接操作结果和状态的相关的方法，CompletableFuture实现了`CompletionStage`的以下策略：
 - 为依赖完成的非异步方法提供的动作，可能被使当前CompletableFuture完成的线程执行，或者由其他完成方法的任意调用者执行。
 - 所有没有指明Executor参数的异步的方法被ForkJoinPool#commonPool执行，除非不支持并行运行，这种情况下每个task都会new一个线程去执行。
 为了简化监控，debug,和跟踪，所有生成的异步任务都是AsynchronousCompletionTask这个标记接口的实例
 - 所有的CompletionStage方法是相对独立的实现于其他公开的方法的，所以子类中的覆盖实现其他方法不会影响当前方法的调用

CompletableFuture也实现了`Future`的以下策略
- 自从FutuTask这个类没有直接的控制结果的计算导致他的等待被完成，取消被当做他的一种异常完成的形式，方法cancel和CompleteExceptionally(new CancellationException()).
方法 #isCompletedExceptionally能够用于判断CompletableFuture是否有任何形式的异常完成。


