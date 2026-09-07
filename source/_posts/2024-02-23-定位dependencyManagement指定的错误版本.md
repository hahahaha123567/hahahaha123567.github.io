---
title: 定位dependencyManagement指定的错误版本
date: 2024-02-23 11:30:00
tags: 
- Java
- Maven
description: dependencyManagement 中的 spring-cloud-dependencies 导致 shardingsphere-jdbc 引入依赖的版本出错, 使用 mvn help:effective-pom 分析并解决
---

# 发现问题

项目引入`shardingsphere-jdbc`和`shardingsphere-cluster-mode-repository-zookeeper`后启动报错

```
Caused by: java.lang.NoClassDefFoundError: org/apache/curator/connection/StandardConnectionHandlingPolicy
	at org.apache.curator.framework.CuratorFrameworkFactory$Builder.<init>(CuratorFrameworkFactory.java:147)
	at org.apache.curator.framework.CuratorFrameworkFactory$Builder.<init>(CuratorFrameworkFactory.java:130)
	at org.apache.curator.framework.CuratorFrameworkFactory.builder(CuratorFrameworkFactory.java:78)
	at org.apache.shardingsphere.mode.repository.cluster.zookeeper.ZookeeperRepository.<init>(ZookeeperRepository.java:65)
	at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
	at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62)
	at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
	at java.lang.reflect.Constructor.newInstance(Constructor.java:423)
	at java.lang.Class.newInstance(Class.java:442)
	at java.util.ServiceLoader$LazyIterator.nextService(ServiceLoader.java:380)
	... 90 common frames omitted
Caused by: java.lang.ClassNotFoundException: org.apache.curator.connection.StandardConnectionHandlingPolicy
	at java.net.URLClassLoader.findClass(URLClassLoader.java:387)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:419)
	at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:352)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:352)
	... 100 common frames omitted
```

可以看到`CuratorFrameworkFactory`类中引用的`StandardConnectionHandlingPolicy`没有被找到

项目的`pom.xml`配置如下

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.example</groupId>
    <artifactId>mvn-test</artifactId>
    <version>1.0.0-SNAPSHOT</version>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>Hoxton.SR4</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>org.apache.shardingsphere</groupId>
                <artifactId>shardingsphere-jdbc-core</artifactId>
                <version>5.4.1</version>
            </dependency>
            <dependency>
                <groupId>org.apache.shardingsphere</groupId>
                <artifactId>shardingsphere-cluster-mode-repository-zookeeper</artifactId>
                <version>5.4.1</version>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <dependency>
            <groupId>org.apache.shardingsphere</groupId>
            <artifactId>shardingsphere-jdbc-core</artifactId>
        </dependency>
        <dependency>
            <groupId>org.apache.shardingsphere</groupId>
            <artifactId>shardingsphere-cluster-mode-repository-zookeeper</artifactId>
        </dependency>
    </dependencies>
</project>
```

其中`shardingsphere-cluster-mode-repository-zookeeper`会引入依赖`curator-framework` `curator-client` `curator-recipes`

# 定位问题

按照经验, 出现依赖项冲突后执行`mvn dependency:tree > tree`后检查依赖树发现了奇怪的现象

```
[INFO] org.example:mvn-test:jar:1.0.0-SNAPSHOT
[INFO] +- org.apache.shardingsphere:shardingsphere-jdbc-core:jar:5.4.1:compile
[INFO] \- org.apache.shardingsphere:shardingsphere-cluster-mode-repository-zookeeper:jar:5.4.1:compile
[INFO]    +- org.apache.shardingsphere:shardingsphere-cluster-mode-repository-api:jar:5.4.1:compile
[INFO]    +- org.apache.curator:curator-framework:jar:4.0.1:compile
[INFO]    +- org.apache.curator:curator-client:jar:5.5.0:compile
[INFO]    |  \- org.apache.zookeeper:zookeeper:jar:3.7.1:compile
[INFO]    \- org.apache.curator:curator-recipes:jar:4.0.1:compile
```

5.4.1版本的`shardingsphere-cluster-mode-repository-zookeeper`引入的`curator-client`是正确版本5.5.0, 但是`curator-framework`和`curator-recipes`都是错误的旧版本4.0.1

检查了`shardingsphere-cluster-mode-repository-zookeeper`的pom文件后确认curator的几个库期望结果应该均为5.5.0

控制变量法, `dependency`没有异常的话问题可能出在`dependencyManagement`中, 删除其他无关依赖项后发现问题出在`spring-cloud-dependencies`中, 定位具体原因的话需要使用`mvn help:effective-pom`

```bash
mvn help:effective-pom > effective.pom
```

输出的文件中, 配置的`dependencyManagement`会展开, 其中可以发现指定了`curator-framework`和`curator-recipes`的版本

```xml
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-framework</artifactId>
    <version>4.0.1</version>
</dependency>
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-recipes</artifactId>
    <version>4.0.1</version>
</dependency>
```

# 解决问题

`dependencyManagement`不支持exclusion, 需要自己声明所需的版本

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>Hoxton.SR4</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <dependency>
            <groupId>org.apache.shardingsphere</groupId>
            <artifactId>shardingsphere-jdbc-core</artifactId>
            <version>5.4.1</version>
        </dependency>
        <dependency>
            <groupId>org.apache.shardingsphere</groupId>
            <artifactId>shardingsphere-cluster-mode-repository-zookeeper</artifactId>
            <version>5.4.1</version>
        </dependency>
        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-framework</artifactId>
            <version>5.5.0</version>
        </dependency>
        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-recipes</artifactId>
            <version>5.5.0</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```

其他

基于springboot的项目引入sharding-jdbc

5.0-5.2版本官方建议使用`shardingsphere-cluster-mode-repository-zookeeper-curator`

[使用 Spring Boot Starter :: ShardingSphere](https://shardingsphere.apache.org/document/5.0.0/cn/user-manual/shardingsphere-jdbc/usage/governance/spring-boot-starter/)

5.3版本开始

[模式配置 :: ShardingSphere](https://shardingsphere.apache.org/document/current/cn/user-manual/shardingsphere-jdbc/java-api/mode/)

[shardingsphere-jdbc-core-spring-boot-starter,When will version 5.3.0 be released? · Issue #24258 · apache/shardingsphere](https://github.com/apache/shardingsphere/issues/24258)

参考

[Apache Maven Help Plugin – Introduction](https://maven.apache.org/plugins/maven-help-plugin/index.html)