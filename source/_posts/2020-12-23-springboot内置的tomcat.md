---
title: springboot内置的tomcat
date: 2020-12-23
tags: 
- Java
- Spring
- 并发
description: .
---

# 服务器接收到网络包

1. 网卡接收到网络包,使用DMA不经过CPU直接将数据包放到Circular buffer(圆形缓冲区,适合先进先出,旧数据读出后新数据不需要移位)
2. 网卡发送硬中断信号,操作系统使用软中断处理多个数据包,没有新数据再恢复终端
3. 操作系统将数据从Circular buffer拷贝到sk_buff缓冲区,网络层,传输层,通过socket转交给用户进程

简单解析  
[你不好奇 Linux 网络发包过程吗？ - InfoQ 写作平台](https://xie.infoq.cn/article/6ba14b756c3019cc737ed48a6)  
[Linux网络 - 数据包的接收过程_Linux程序员 - SegmentFault 思否](https://segmentfault.com/a/1190000008836467)  
源码解析  
[图解Linux网络包接收过程 - 知乎](https://zhuanlan.zhihu.com/p/256428917)
[就是要你懂网络--网络包的流转 | plantegg](https://plantegg.github.io/2019/05/08/%E5%B0%B1%E6%98%AF%E8%A6%81%E4%BD%A0%E6%87%82%E7%BD%91%E7%BB%9C--%E7%BD%91%E7%BB%9C%E5%8C%85%E7%9A%84%E6%B5%81%E8%BD%AC/)

# springboot支持不同的Java servlet服务器

服务器接收到http请求, 操作系统建立了TCP连接之后

见 [78. Embedded Web Servers](https://docs.spring.io/spring-boot/docs/2.1.10.RELEASE/reference/html/howto-embedded-web-servers.html) , springboot默认内置tomcat服务器, 可以通过修改pom配置切换为jetty或undertow

# tomcat内置线程池

springboot内置的tomcat [org.apache.tomcat.embed:tomcat-embed-core](https://mvnrepository.com/artifact/org.apache.tomcat.embed/tomcat-embed-core) 中,负责处理请求的线程池由AbstractEndpoint维护

```java
public void createExecutor() {
    internalExecutor = true;
    TaskQueue taskqueue = new TaskQueue();
    TaskThreadFactory tf = new TaskThreadFactory(getName() + "-exec-", daemon, getThreadPriority());
    executor = new ThreadPoolExecutor(getMinSpareThreads(), getMaxThreads(), 60, TimeUnit.SECONDS,taskqueue, tf);
    taskqueue.setParent( (ThreadPoolExecutor) executor);
}

如果使用springboot默认的参数,corePoolSize=10,maxPoolSize=200,keepAliveTime=60s,
```

https://blog.csdn.net/l4642247/article/details/81631770