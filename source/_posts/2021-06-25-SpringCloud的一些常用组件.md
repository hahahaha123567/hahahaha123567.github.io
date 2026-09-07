---
title: SpringCloud的一些常用组件
date: 2021-06-25
tags: 
- Java
- Spring
- SpringCloud
description: SpringCloud的一些常用组件
---

# 服务组成

![spring_cloud_architecture_highlights](../../../../image/spring_cloud_architecture_highlights.svg)

## service discovery 服务发现

- Eureka (netflix)
- Consul
- Zookeeper
- Kubernetes
- nacos (alibaba)

## API gateway API网关

- spring cloud gateway

## cloud configuration 云配置

- spring cloud config
- nacos (alibaba)

## circuit breaker 服务熔断

- Hystrix (netflix)
- Resilience4J
- Sentinel (alibaba)
- Spring Retry

## tracing 链路跟踪

- Spring Cloud Sleuth
- zipkin

## test 测试

- Spring Cloud Contract

## others 其他

### Spring Cloud Stream

- Kafka
- RabbitMQ
- Feign

### Distributed Transaction

- Seata

# 套件

## Spring Cloud Netflix

原组件

- Hystrix(熔断器)
- Hystrix Dashboard(监控)
- Ribbon(客户端负载均衡)
- Zuul(网关)
- Archaius(云配置)

现在项目内只有Eureka

## Netflix替代

Spring Cloud 2020.0.0版本彻底删除掉了Netflix除Eureka外的所有组件, 推荐的替代品如下

- Resilience4j
- Micrometer + Monitoring System
- Spring Cloud Loadbalancer
- Spring Cloud Gateway
- Spring Cloud Config

Feign从Netflix转交给OpenFeign, 当前可以使用Spring Cloud Loadbalancer作为http-client实现

---
__reference__

[Spring | Cloud](https://spring.io/cloud)

[微服务架构 - Alibaba 生态整合（一） | BladeCode](https://incoder.org/2020/11/11/microservices-alibaba1/#SpringCloud-VS-SpringCloud-Alibaba)