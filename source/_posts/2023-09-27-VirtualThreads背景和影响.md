---
title: Virtual Threads背景和影响
date: 2023-09-27 12:00:00
tags: 
- Java
- JDK21
- 并发
description: Virtual Threads背景和影响
---

# 背景

## JEP(JDK Enhancement Proposals)推进

将虚拟线程引入Java 平台。虚拟线程是轻量级线程，可以显著减少编写、维护和观察高吞吐量并发应用程序的工作量

- 使以简单的每个请求线程风格编写的服务器应用程序能够以接近最佳的硬件利用率进行扩展

- 使使用 java.lang.Thread API 的现有代码能够以最小的更改采用虚拟线程

- 使用现有 JDK 工具轻松进行虚拟线程故障排除、调试和分析

2022-09-20 Java19发布, virtual threads作为preview功能发布 https://openjdk.org/jeps/425

2023-03-21 Java20发布, virtual threads作为second preview预览功能发布 https://openjdk.org/jeps/436

2023-09-19 Java21(LTS)发布, virtual threads正式发布 https://openjdk.org/jeps/444

## Loom

Loom 是一个OpenJDK的项目，旨在探索、孵化和交付构建在其之上的 Java VM 功能和 API，以支持 Java 平台上易于使用、高吞吐量的轻量级并发和新的编程模型

主要包含2部分

1. Virtual threads

2. Structured concurrency

https://wiki.openjdk.org/display/loom

## Structured Concurrency 结构化并发

通过引入结构化并发 API 来简化并发编程。结构化并发将在不同线程中运行的相关任务组视为单个工作单元，从而简化错误处理和取消、提高可靠性并增强可观察性

2022-09-20 Java19发布, structured concurrency作为Incubator开始孵化 https://openjdk.org/jeps/428

2023-03-21 Java20发布, structured concurrency作为Second Incubator https://openjdk.org/jeps/437

2023-09-19 Java21(LTS)发布, structured concurrency作为preview功能发布 https://openjdk.org/jeps/453

# 对开发的影响

## 框架开发or应用开发

对于使用线程或并行的库和框架来说，将是一件大事。库作者能够实现巨大的性能和可扩展性提升，同时简化代码库，使其更易维护。大多数使用线程池和平台线程的 Java 项目都能够从切换至虚拟线程的过程中受益，候选项目包括 Tomcat、Undertow 和 Netty 这样的 Java 服务器软件，以及 Spring 和 Micronaut 这样的 Web 框架

不会对普通的 Java 开发人员产生太大的影响，因为这些开发人员可能正在使用某些库来处理并发的场景。但是，在一些比较罕见的场景中，比如你可能进行了大量的多线程操作但是没有使用库，那么这些特性就是很有价值的了。虚拟线程可以毫不费力地替代你现在使用的线程池。根据现有的基准测试，在大多数情况下它们都能提高性能和可扩展性

https://www.infoq.cn/article/wg5qybla1ps222larj3y

## 库依赖版本

https://github.com/spring-projects/spring-framework/wiki/Spring-Framework-Versions

https://stackoverflow.com/questions/42659920/is-there-a-compatibility-matrix-of-spring-boot-and-spring-cloud

## VS 反应式编程

对于应用开发者来说，Virtual Threads 和 Reactive 的目的一样，都是对于IO密集型服务，通过提高CPU的使用率来实现提高服务的吞吐量

在Virtual Threads没有作为JEPS正式提出前，知乎上就有一些讨论，大部分人认为 Virtual Threads 这样将负担交给JVM的方式更好，而不是自己组织Reactive代码写回调

- Oracle的Reactive驱动ADBA停止开发，现在使用较多的MySQL Reactive驱动是Spring主导开发的R2DBC

- loom的tech lead认为正确的方向是"Code like sync, works like async"

[如何看待Project Loom? - 知乎](https://www.zhihu.com/question/67579790)

[对于后端开发，响应式编程真的是大势所趋吗？ - 知乎](https://www.zhihu.com/question/375996978)

# 其他语言的coroutine

## go的goroutine

[深入Go语言 - 8 - colobu.com](https://colobu.com/2016/06/27/dive-into-go-8/)

[Go 调度器跟踪 - colobu.com](https://colobu.com/2016/04/19/Scheduler-Tracing-In-Go/)

Go所有线程都是"虚拟线程"

Java兼容GUI、Android，Virtual Thread是Thread的子类，支持手动指定Virtual Thread pin到 Platform Thread的逻辑

## kotlin的coroutine

[Coroutines | Kotlin Documentation - kotlinlang.org](https://kotlinlang.org/docs/coroutines-overview.html)

--- 

参考文章

[Java21手册（一）：虚拟线程 Virtual Threads - 掘金 - juejin.cn](https://juejin.cn/post/7280746515526058038)