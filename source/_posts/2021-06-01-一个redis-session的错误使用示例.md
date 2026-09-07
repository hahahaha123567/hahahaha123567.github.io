---
title: 一个redis-session的错误使用示例
date: 2021-06-01
tags: 
- Java
- Spring
description: 服务里定时出现的神秘异常
---

# 现象

某个模块的几个子服务上线后会定时出现如下的异常日志

```log
14:56:00.009 [ing-scheduled-1] [FJNJFVNG] ERROR o.s.s.s.TaskUtils$LoggingErrorHandler    - Unexpected error occurred in scheduled task 
org.springframework.data.redis.serializer.SerializationException: Cannot deserialize; nested exception is org.springframework.core.serializer.support.SerializationFailedException: Failed to deserialize payload. Is the byte array a result of corresponding serialization for DefaultDeserializer?; nested exception is java.io.StreamCorruptedException: invalid stream header: 22657870
	at org.springframework.data.redis.serializer.JdkSerializationRedisSerializer.deserialize(JdkSerializationRedisSerializer.java:84)
	at org.springframework.data.redis.serializer.SerializationUtils.deserializeValues(SerializationUtils.java:54)
	at org.springframework.data.redis.serializer.SerializationUtils.deserialize(SerializationUtils.java:62)
	at org.springframework.data.redis.core.AbstractOperations.deserializeValues(AbstractOperations.java:223)
	at org.springframework.data.redis.core.DefaultSetOperations.members(DefaultSetOperations.java:216)
	at org.springframework.data.redis.core.DefaultBoundSetOperations.members(DefaultBoundSetOperations.java:152)
	at org.springframework.session.data.redis.RedisSessionExpirationPolicy.cleanExpiredSessions(RedisSessionExpirationPolicy.java:129)
	at org.springframework.session.data.redis.RedisIndexedSessionRepository.cleanupExpiredSessions(RedisIndexedSessionRepository.java:407)
	at org.springframework.scheduling.support.DelegatingErrorHandlingRunnable.run(DelegatingErrorHandlingRunnable.java:54)
	at org.springframework.scheduling.concurrent.ReschedulingRunnable.run(ReschedulingRunnable.java:93)
	at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511)
	at java.util.concurrent.FutureTask.run(FutureTask.java:266)
	at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.access$201(ScheduledThreadPoolExecutor.java:180)
	at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:293)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)
Caused by: org.springframework.core.serializer.support.SerializationFailedException: Failed to deserialize payload. Is the byte array a result of corresponding serialization for DefaultDeserializer?; nested exception is java.io.StreamCorruptedException: invalid stream header: 22657870
	at org.springframework.core.serializer.support.DeserializingConverter.convert(DeserializingConverter.java:78)
	at org.springframework.core.serializer.support.DeserializingConverter.convert(DeserializingConverter.java:36)
	at org.springframework.data.redis.serializer.JdkSerializationRedisSerializer.deserialize(JdkSerializationRedisSerializer.java:82)
	... 16 common frames omitted
Caused by: java.io.StreamCorruptedException: invalid stream header: 22657870
	at java.io.ObjectInputStream.readStreamHeader(ObjectInputStream.java:937)
	at java.io.ObjectInputStream.<init>(ObjectInputStream.java:395)
	at org.springframework.core.ConfigurableObjectInputStream.<init>(ConfigurableObjectInputStream.java:65)
	at org.springframework.core.ConfigurableObjectInputStream.<init>(ConfigurableObjectInputStream.java:51)
	at org.springframework.core.serializer.DefaultDeserializer.deserialize(DefaultDeserializer.java:70)
	at org.springframework.core.serializer.support.DeserializingConverter.convert(DeserializingConverter.java:73)
	... 18 common frames omitted
```

# 排查思路

根据异常栈底的类名`DeserializingConverter`大概可以知道是反序列化的问题，但是什么反序列化会定时执行并抛出异常呢？

这个时候我们要看第一个异常栈，从栈底开始，忽略`ThreadPoolExecutor`等没有逻辑特征的类，往下看

首先是`ReschedulingRunnable`，是spring定时任务的默认接口，服务中用到定时任务的地方不算多，但是我们不能以此为依据，因为springboot本身也包含很多定时任务

接下来我们看`RedisIndexedSessionRepository`，找到异常栈中的代码
```java
public void cleanupExpiredSessions() {
	this.expirationPolicy.cleanExpiredSessions();
}
```
是服务在清理过期的session，继续往下debug可以看到服务使用代码里默认的`JdkSerializationRedisSerializer`执行redis的member操作检查过期key时产生的反序列化异常

再从异常栈顶往下跟踪，看看哪一步开始可以修改默认的serializer。要注意idea不支持外部库的变量引用查询，在跟踪字段的时候需要文件内搜索

发现在`RedisHttpSessionConfiguration`中调用了`RedisIndexedSessionRepository`的`setDefaultSerializer()`，Configuration在配置类中提供了serializer的注入方法
```java
@Autowired(required = false)
@Qualifier("springSessionDefaultRedisSerializer")
public void setDefaultRedisSerializer(RedisSerializer<Object> defaultRedisSerializer) {
    this.defaultRedisSerializer = defaultRedisSerializer;
}
```
我们在spring配置中注入名为`springSessionDefaultRedisSerializer`的自定义serializer即可

# 相关知识

spring-session-data-redis和spring-data-redis相对独立，redis客户端的serializer等需要独立配置

RedisHttpSessionConfiguration 本身是一个 Spring 配置类, 会向 Spring 容器注册 sessionRepository, redisMessageListenerContainer 等实例

RedisHttpSessionConfiguration 会注册 Redis 消息监听器容器 RedisMessageListenerContainer, 并将 RedisIndexedSessionRepository 作为 Redis 消息订阅的监听器, 因为它实现了 MessageListener 接口。当 Redis 中 key 过期或销毁时, 会通知将 RedisIndexedSessionRepository 调用其onMessage() 方法来处理消息

[Spring Session Data Redis 源码解析](https://juejin.cn/post/6844904159091621901)

排查问题的时候要灵活阅读异常栈的信息