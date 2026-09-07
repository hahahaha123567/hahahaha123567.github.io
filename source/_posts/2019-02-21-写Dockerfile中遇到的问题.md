---
title: 写Dockerfile中遇到的问题
date: 2019-02-21 02:32:37
tags:
- Docker
description: 大部分都是一些之前不知道的常识
---

# 文件映射

[Use volumes | Docker Documentation](https://docs.docker.com/storage/volumes/)

[深入理解Docker Volume（一） - DockOne.io](http://dockone.io/article/128)

[深入理解Docker Volume（二） - DockOne.io](http://dockone.io/article/129)

## mount

```dockerfile
# 1. 运行时指定
docker run -v /container/data hello

# 2. Dockerfile中指定
VOLUME /container/data
```

docker 会在 host 上创建一个目录并挂载到 container 的 `/container/data`

创建的目录默认路径在 `/var/lib/docker`

## bind-mount

`docker run -v /host/data:/container/data hello` 

将 `host` 的 `/host/data` 目录挂载到了 `container` 的 `/container/data` 目录下

## 覆盖

因为是“挂载”, host 的目录内容会覆盖 container 的目录:
- 如果是普通 mount 则 container 内的目录一定会被覆盖为空
- 如果是 bind-mount, 但指定的 host 目录原本不存在, 则 container 目录也会被覆盖为空

todo: volume 的顺序是否会影响文件是否被覆盖

[Docker volume 挂载时文件或文件夹不存在 - Keep Coding - SegmentFault 思否](https://segmentfault.com/a/1190000015684472)

# shell脚本

## 终端打印彩色字体

```sh
echo -e "\033[31m 我是红色 \033[0m"
```

## shell的比较选项

[Linux 技巧: Bash 测试和比较函数](https://www.ibm.com/developerworks/cn/linux/l-bash-test.html)

## shell读参数

[位置参数 - TLCL](http://billie66.github.io/TLCL/book/chap33.html)

## 星际选手

`sed` 写成了 `set`, 两个单词高亮一样, 查了几个小时, 一直以为是脚本没执行, 最后才发现是命令写错了

# 持续运行

docker 执行完脚本后就会自动停止

为了使其正常工作, 一般用 `CMD ["..."]` 显式使其执行命令, 或者使用 tailf 查看日志等

# 查看日志

## 容器运行时

```sh
# 进入容器内部查看程序的日志
docker exec -it hello bash
```

## 容器停止后

```sh
# 在 host 上查看 hello 输出的日志, 即 hello 内部打印到 bash 的信息
docker logs hello
```

# 最佳实践

## document

[Best practices for writing Dockerfiles | Docker Documentation](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)

[附录四：Dockerfile 最佳实践 · Docker —— 从入门到实践](https://yeasy.gitbooks.io/docker_practice/appendix/best_practices.html)

## example

[redis - Docker Hub](https://hub.docker.com/_/redis)

[nginx - Docker Hub](https://hub.docker.com/_/nginx)