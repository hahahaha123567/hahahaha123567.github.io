---
title: 排查Netty应用的direct memory泄漏
date: 2024-06-03 19:00:00
tags: 
- Java

---

首先要明确, 程序抛出`OutOfMemoryError`的地方并不一定是导致OOM的原因, 如果其他地方导致了程序OOM之后去申请内存也会抛出`OutOfMemoryError`

[Reactor Netty Reference Guide](https://projectreactor.io/docs/netty/release/reference/index.html#faq.memory-leaks)

> A.4. 如何调试内存泄漏?
> 
> 默认情况下, Reactor Netty 使用直接内存,因为在有许多本地 I/O 操作(使用套接字)时,这样可以提高性能,因为可以消除拷贝操作。由于分配和释放操作是昂贵的,所以 Reactor Netty 也默认使用池化缓冲区。更多信息请参见[Reference counted objects · netty/netty Wiki](https://github.com/netty/netty/wiki/Reference-counted-objects)。
> 
> 为了能够调试直接内存和池化缓冲区的内存问题,Netty 提供了一种特殊的内存泄漏检测机制。请按照排查缓冲区泄漏的说明启用此机制。除了 Netty 提供的说明,Reactor Netty 还提供了一个特殊的日志记录器(__reactor.netty.channel.LeakDetection__),它有助于识别内存泄漏可能位于 Reactor Netty 内部还是 Reactor Netty 已经将缓冲区所有权转交给应用程序/框架。默认情况下,此日志记录器处于禁用状态。要启用它,请提高日志级别到 __DEBUG__
> 
> 另一种检测内存泄漏的方法是监控 __reactor.netty.bytebuf.allocator.active.heap.memory__ 和 __reactor.netty.bytebuf.allocator.active.direct.memory__ 指标:
> 
> __reactor.netty.bytebuf.allocator.active.heap.memory__ 提供了从堆缓冲区池分配的正在使用的缓冲区所消耗的实际字节数。
> 
> __reactor.netty.bytebuf.allocator.active.direct.memory__ 提供了从直接缓冲区池分配的正在使用的缓冲区所消耗的实际字节数。
> 
> 如果上述指标持续增长,则可能存在缓冲区内存泄漏。
> 
> 在调试内存泄漏问题时,考虑减少使用的内存(例如 __-XX:MaxDirectMemorySize、-Xms、-Xmx__)。应用程序拥有的内存越少,内存泄漏就会发生得越早。

[长连接Netty服务内存泄漏，看我如何一步步捉“虫”解决 - 京东云技术团队 - 博客园](https://www.cnblogs.com/jingdongkeji/p/17345525.html)

> 一、Netty的内存泄漏排查其实并不难，Netty提供了比较完整的排查内存泄漏工具
> 
> JVM 选项 __-Dio.netty.leakDetection.level__
> 
> 本次内存泄漏问题，我们通过本地设置泄漏检测级别为高级，即： __-Dio.netty.leakDetectionLevel=advanced__ 定位到了具体内存泄漏的代码。
> 
> 同时Netty也给出了避免泄漏的最佳实践：
> 
> - 在 __PARANOID__ 泄漏检测级别以及 __SIMPLE__ 级别运行单元测试和集成测试。
> - 在 __SIMPLE__ 级别向整个集群推出应用程序之前，请先在相当长的时间内查看是否存在泄漏。
> - 如果有泄漏，灰度发布中使用 __ADVANCED__ 级别，以获得有关泄漏来源的一些提示。
> - 不要将泄漏的应用程序部署到整个群集。
> 
> 二、解决Netty内存泄漏，Netty也提供了指导方案，主要有三种方式
> 
> 1. 手动释放，哪里使用了，使用完就手动释放，这个对使用方要求比较高了。
> 2. 如果处理过程中不确定ByteBuf是否应该被释放，那交给Netty的__ReferenceCountUtil.release（msg）__来释放，这个方法会判断上下文中是否可以释放，简单方便。
> 3. 升级ChannelHandler为SimpleChannelHandler， 在SimpleChannelHandler中，Netty对收到的所有消息都调用了ReferenceCountUtil.release（msg），升级接口，可能对现有API改动会比较大。
