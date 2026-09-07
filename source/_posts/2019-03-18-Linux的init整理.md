---
title: Linux的init整理
date: 2019-03-18 19:50:37
tags:
- Linux
description: initd, service, systemd, systemctl
---

# 概述

Linux 操作系统的启动首先从 BIOS 开始, 接下来进入 boot loader, 由 bootloader 载入内核, 进行内核初始化

内核初始化的最后一步就是启动 pid 为 1 的 [init](https://zh.wikipedia.org/wiki/Init) 进程. 这个进程是系统的第一个进程,  负责产生其他所有用户进程。

# Sysvinit 

[浅析 Linux 初始化 init 系统，第 1 部分: sysvinit](https://www.ibm.com/developerworks/cn/linux/1407_liuming_init1/index.html)

# UpStart

[浅析 Linux 初始化 init 系统，第 2 部分: UpStart](https://www.ibm.com/developerworks/cn/linux/1407_liuming_init2/index.html)

# Systemd

[浅析 Linux 初始化 init 系统，第 3 部分: Systemd](https://www.ibm.com/developerworks/cn/linux/1407_liuming_init3/index.html)

