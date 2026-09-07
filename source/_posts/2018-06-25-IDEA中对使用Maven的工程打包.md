---
title: IDEA中对使用Maven的工程打包
date: 2018-06-25 06:50:43
tags:
- Java
description: 只是记录一下遇到的坑；以后还是要按照 Maven 的规范进行打包
---

如果 **工程的 META-INF 信息没有在工程的根目录下生成** ，则直接使用 IDEA 打包 jar 文件时，会从 Maven 中获取配置信息；

如果 **此时** `pom.xml` **中也没有完整的打包配置**，则会读取默认的信息。

此时，打包出的 jar 文件的 META-INF 不包括启动类的名称，无法直接运行。

```bash
$java -jar Nico.jar
Nico.jar中没有主清单属性
```

# Spring Boot应用打包逻辑

[SpringBoot（一） 初识 | BladeCode](https://incoder.org/2019/06/23/springboot1/#Spring-%E6%89%93%E5%8C%85)
[SpringBoot（二） 启动分析JarLauncher | BladeCode](https://incoder.org/2019/07/05/springboot2/)
[SpringBoot（五）多环境配置 | BladeCode](https://incoder.org/2020/02/02/springboot5/)