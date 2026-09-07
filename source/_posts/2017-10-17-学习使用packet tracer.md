---
title: 学习使用packet tracer
date: 2017-10-17 00:50:21
tags: 
- 计算机网络 
description: 计网lab2开始使用packet tracer，实验原理很简单，但因为第一次用所以走了很多弯路，做个记录，也希望能帮到其他萌新
---
# 配置设备

## 界面说明

每个设备都包括Physical, Config, Cli(PC可以在Desktop的Command Prompt)

在进行实验时，要先在Physical设置好要用的网卡，然后在Config(GUI界面)或者Cli(命令行界面)进行配置

以PC为例

![pc_physical](../../../../image/pc_physical.png)

红色框为当前可选的网卡列表

蓝色框为当前设备，需要用到其中的网卡插槽与电源开关**（更换网卡时需要关闭电源，更换结束后打开电源）**

黄色框内为当前选中的网卡信息，更换网卡时需要将当前设备上的网卡拖回网卡列表，然后将黄色框内的网卡拖到当前设备的网卡槽内

## PC

![pc_config_settings](../../../../image/pc_config_settings.png)

在config-settings界面设置PC的IPV4/IPV6的Gateway和DNS Server，可以设置DHCP获取或自己静态设定

![pc_config_interface](../../../../image/pc_config_interface.png)

在config-interface界面对当前所接的网卡（图中为FastEthernet0）进行配置

可以设置MAC Address, IPV4/IPV6的IP Address和Subnet Mask, 获取方式为DHCP或手动静态设置

## Switch

同PC，注意Switch可以插多个网卡

Switch连接后需要等待几秒才能联通

## Router

![router_config_interface](../../../../image/router_config_interface.png)

**Router的端口默认为OFF，需要手动打开**

在config-interface下对每个网卡端口进行配置

# 组建网络

若提示`The cable cannot connect to the port`可能为设备网卡端口已经用完，可以给设备增加网卡

鼠标停留在设备上可以显示该设备的IP等信息

鼠标停留在端口上可以显示两个端口的信息

## Wirelsee Network

PC要先更换为无线网卡（默认为有线）

PC上的SSID与密码要与提供网络的无线AP/无线Router一致，然后就可以自动连接

## VLAN

Switch的端口可以对VLAN进行划分，在逻辑上分成多个子网

Router可以将端口划分成多个逻辑子端口，与包括多个VLAN的Switch通过将端口设置成Trunk进行连接，基本，命令如下

```
R1#conf t                                           # 开始配置终端
R1(config)#interface FastEthernet0/0.1              # 创建新的逻辑子端口
R1(config-subif)#encapsulation dot1Q 1              # 用802.1q协议对刚创建的逻辑子端口进行封装，并且VLAN number设置为1
R1(config-subif)#ip address 10.1.0.25 255.255.0.0   # 设置这个逻辑子端口的IP Address与Subnet Mask
R1(config-subif)#exit                               # 退出第一个逻辑子端口的配置
R1(config)#interface FastEthernet0/0.2
R1(config-subif)#encapsulation dot1Q 2
R1(config-subif)#ip address 10.2.0.25 255.255.0.0
R1(config-subif)#exit
```

## Static Route

这部分主要介绍一下静态路由的概念

路由器，根据中文译名可以知道是网络中用来指路的设备。一个数据包带着Source Address和Destination Address来到路由器，让路由器给它指路，然后才能传向下一个路由器。因此，在配置静态路由时，需要提供Subnet Address, Mask, Next Hop。

如果一个路由器收到的包的Destination Address与路由表中的Mask取交集后的地址就是这个Mask对应的子网，就说明路由器认识去那个子网的路，让数据包前往Next Hop，继续让下一个路由器给它指路。

在配置Static Route时要注意路径是对称的，只配置了子网A到子网B的路径而没有配置B回A的路径同样会使A内PC ping不通B内PC。

---
参考资料

[Packet Tracer 5——11路由器实现Vlan间通信](https://wenku.baidu.com/view/b9ff6431b90d6c85ec3ac6eb.html)
