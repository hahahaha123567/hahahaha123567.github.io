---
title: logback的AsyncAppender产生的线程问题
date: 2019-03-18 11:50:37
tags:
- Java
- 并发
description: 第一次发现并独立解决并发问题
---

# 问题

项目里使用 logback+log4j 作为日志框架, 之前写了个mybatis插件处理日志, [mybatis日志问号替换](https://hahahaha123567.github.io/2019/03/18/2019-03-18-mybatis日志问号替换/)

输出自定义日志后, 需要把框架默认的日志过滤掉, 原 `logback.xml` 如下

```xml
<configuration>
    <appender name="rollingFileAppender" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <filter class="com.yiwise.core.log.MybatisFilter"/>
    </appender>
    <appender name="asyncRollingFileAppender" class="ch.qos.logback.classic.AsyncAppender">
        <discardingThreshold>0</discardingThreshold>
        <queueSize>512</queueSize>
        <appender-ref ref="rollingFileAppender"/>
    </appender>
    <appender name="console" class="ch.qos.logback.core.ConsoleAppender">
        <filter class="com.yiwise.core.log.MybatisFilter"/>
    </appender>
</configuration>
```

有console, rollingFileAppender, asyncRollingFileAppender三个appender, 最后生效的分别是console和日志文件asyncRollingFileAppender三个appender

在mybatis插件中, 因为担心字段映射出错抛异常的时候, 不显示自定义sql, 此时框架默认sql也被我过滤导致看不到sql情况, 所以在异常的try-catch中使用MDC控制MybatisFilter这个过滤器的开关(即出现异常时关闭这个filter显示框架默认日志)

但在测试时, ide里运行, console中日志被正常过滤, 但生成的日志文件里过滤却失败, 明明filter配置相同结果却不同, 花了很久才找到原因

# 问题产生原因

AsyncAppender有个线程类Worker, 它是一个简单的线程类, 是AsyncAppender的后台线程, 所要做的工作是: 从buffer中取出event交给对应的appender进行后面的日志推送

AsyncAppender并不处理日志, 只是将日志缓冲到一个BlockingQueue里面去, 并在内部创建一个工作线程从队列头部获取日志, 之后将获取的日志循环记录到附加的其他appender上去, 从而达到不阻塞主线程的效果。因此AsynAppender仅仅充当事件转发器, 必须引用另一个appender来工作

在业务流程中使用了logback MDC(Mapped Diagnostic Context), 即将一些运行时的上下文数据通过logback打印出来

MDC内部[使用InheritableThreadLocal](https://blog.csdn.net/a837199685/article/details/52712547)实现, 默认实现从父线程到子线程的内容浅拷贝, 但filter是运行在asyncRollingFileAppender的线程中, 读不到业务在主线程中放到MDC的内容

# 解决

filter中不再直接使用MDC, 而是用传进来的event去获取日志所记录的线程的MDC `getMDCPropertyMap`

```java
@Override
public FilterReply decide(ILoggingEvent event) {
    Map<String, String> mdcPropertyMap = event.getMDCPropertyMap();
    // ...
}
```