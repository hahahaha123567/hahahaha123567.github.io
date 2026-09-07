---
title: 基于freeswitch的unimrcp的demo
date: 2019-03-23 23:50:37
tags:
- freeswitch
description: 其实就是按照别人的教程实现了一个demo
---

# unimrcp

## 实现

[unimrcp](http://www.unimrcp.org/) 是一个开源的mrcp实现库

将业务代码以 [plugin](http://www.unimrcp.org/manuals/html/PluginImplementationManual.html) 的形式插入到unimrcp中执行, 完成后编译, 运行/usr/bin/unimrcp/下的脚本即可启动mrcp服务器

## 配置

配置文件 `/usr/bin/unimrcp/conf/unimrcpserver.xml`

实现mrcpv2需要的基本配置:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<unimrcpserver xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
               xsi:noNamespaceSchemaLocation="unimrcpserver.xsd"
               version="1.0">
  <properties>
    <ip>172.19.**.171</ip>
  </properties>

  <components>
    <resource-factory>
      <resource id="speechsynth" enable="true"/>
      <resource id="speechrecog" enable="true"/>
      <resource id="recorder" enable="true"/>
      <resource id="speakverify" enable="true"/>
    </resource-factory>

    <sip-uas id="SIP-Agent-1" type="SofiaSIP">
      <sip-port>8060</sip-port>
      <sip-transport>udp,tcp</sip-transport>
      <ua-name>UniMRCP SofiaSIP</ua-name>
      <sdp-origin>UniMRCPServer</sdp-origin>
      <sip-session-expires>600</sip-session-expires>
      <sip-min-session-expires>120</sip-min-session-expires>
    </sip-uas>

    <mrcpv2-uas id="MRCPv2-Agent-1">
      <mrcp-port>1544</mrcp-port>
      <max-connection-count>100</max-connection-count>
      <max-shared-use-count>100</max-shared-use-count>
      <force-new-connection>false</force-new-connection>
      <rx-buffer-size>1024</rx-buffer-size>
      <tx-buffer-size>1024</tx-buffer-size>
      <inactivity-timeout>600</inactivity-timeout>
      <termination-timeout>3</termination-timeout>
    </mrcpv2-uas>

    <media-engine id="Media-Engine-1">
      <realtime-rate>1</realtime-rate>
    </media-engine>

    <rtp-factory id="RTP-Factory-1">
      <rtp-port-min>5000</rtp-port-min>
      <rtp-port-max>6000</rtp-port-max>
    </rtp-factory>

    <plugin-factory>
      <engine id="Demo-Synth-1" name="demosynth" enable="false"/>
	    <engine id="my-Synth-1" name="my-synth" enable="true"/>
      <engine id="Demo-Recog-1" name="demorecog" enable="false"/>
	    <engine id="my-Recog-1" name="my-recog" enable="true"/>
      <engine id="Demo-Verifier-1" name="demoverifier" enable="true"/>
      <engine id="Recorder-1" name="mrcprecorder" enable="true"/>
    </plugin-factory>
  </components>

  <settings>
    <rtp-settings id="RTP-Settings-1">
      <jitter-buffer>
        <adaptive>1</adaptive>
        <playout-delay>50</playout-delay>
        <max-playout-delay>600</max-playout-delay>
        <time-skew-detection>1</time-skew-detection>
      </jitter-buffer>
      <ptime>20</ptime>
      <codecs own-preference="false">PCMU PCMA L16/96/8000 telephone-event/101/8000</codecs>
      <rtcp enable="false">
        <rtcp-bye>1</rtcp-bye>
        <tx-interval>5000</tx-interval>
        <rx-resolution>1000</rx-resolution>
      </rtcp>
    </rtp-settings>
  </settings>

  <profiles>
    <mrcpv2-profile id="uni2">
      <sip-uas>SIP-Agent-1</sip-uas>
      <mrcpv2-uas>MRCPv2-Agent-1</mrcpv2-uas>
      <media-engine>Media-Engine-1</media-engine>
      <rtp-factory>RTP-Factory-1</rtp-factory>
      <rtp-settings>RTP-Settings-1</rtp-settings>
    </mrcpv2-profile>
  </profiles>
</unimrcpserver>
```

可以看到, 最后的profile分别指定了各参数使用的子配置; 基本上, 默认的配置修改了ip即可正常运行

注意: 

1. `plugin-factory` 中添加自己的plugin, 禁用默认的相同功能的plugin
2. 区分各端口的作用, 默认的8060是sip通信的端口, 1544是mrcp通信的端口, 5000-6000是rtp通信的端口; 另外默认1554为rtsp的端口, 是mrcpv1使用的

# freeswitch

## 安装unimrcp模块

源码路径/modules.xml取消mod_unimrcp的注释

make mod_unimrcp-install

设置fs启动时自动加载该模块: 取消/usr/local/freeswitch/conf/autoload_configs/modules.conf.xml中mod_unimrcp的注释

## 配置一份mrcp

运行路径/conf/mrcp_profiles/下新增unimrcpserver-mrcp-v2.xml

```xml
<include>
    <profile name="unimrcpserver-mrcp2" version="2">
        <!-- MRCP 服务器地址 -->
        <param name="server-ip" value="8"/>
        <!-- MRCP SIP 端口号 -->
        <param name="server-port" value="8060"/>
        <param name="resource-location" value=""/>
        
        <!-- FreeSWITCH IP, 端口以及 SIP 传输方式 -->
        <param name="client-ip" value="172.19.**.171" />
        <param name="client-port" value="5069"/>
        <param name="sip-transport" value="udp"/>
        
        <param name="speechsynth" value="speechsynthesizer"/>
        <param name="speechrecog" value="speechrecognizer"/>
        <param name="rtp-ip" value="172.19.46.171"/>
        <param name="rtp-port-min" value="4000"/>
        <param name="rtp-port-max" value="5000"/>
        <param name="codecs" value="PCMU PCMA L16/96/8000"/>
        
        <!-- Add any default MRCP params for SPEAK requests here -->
        <synthparams>
        </synthparams>
        
        <!-- Add any default MRCP params for RECOGNIZE requests here -->
        <recogparams>
            <!--param name="start-input-timers" value="false"/-->
        </recogparams>
    </profile>
</include>
```

注意: 
1. mrcp服务器的端口是提供sip通信的端口
2. 指定的fs的端口是专门用来和mrcp服务器交换的, 和fs自己的internal/external端口都不能相同

## 指定默认使用该mrcp配置

运行路径/conf/autoload_configs/unimrcp.conf.xml

```xml
<configuration name="unimrcp.conf" description="UniMRCP Client">
    <settings>
        <param name="default-tts-profile" value="unimrcpserver-mrcp2"/>
        <param name="default-asr-profile" value="unimrcpserver-mrcp2"/>
        <param name="log-level" value="DEBUG"/>
        <param name="enable-profile-events" value="false"/>
        
        <param name="max-connection-count" value="100"/>
        <param name="offer-new-connection" value="1"/>
        <param name="request-timeout" value="3000"/>
    </settings>
    
    <profiles>
        <X-PRE-PROCESS cmd="include" data="../mrcp_profiles/*.xml"/>
    </profiles>
</configuration>
```

## 写语法文件和脚本

略

总之, 要分清fs服务器和mrcp服务器, 清楚每个端口的区别

---

参考资料

1. [构建简单的智能客服系统（一）——FreeSWITCH 搭建与配置 | Cotin's Homepage](https://cotin.tech/AI/FreeswitchSetting/)
2. [构建简单的智能客服系统（二）——基于 UniMRCP 实现讯飞 ASR MRCP Server | Cotin's Homepage](https://cotin.tech/AI/UniMRCPASR/)
3. [构建简单的智能客服系统（三）——基于 UniMRCP 实现讯飞 TTS MRCP Server | Cotin's Homepage](https://cotin.tech/AI/UniMRCPTTS/)