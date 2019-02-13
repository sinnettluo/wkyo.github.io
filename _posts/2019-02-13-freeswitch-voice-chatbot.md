---
layout: post
title: FreeSWITCH 1.8 语音机器人集成
date: 2019-01-29 18:32:34 +0800
update: 2019-02-13 15:51:36 +0800
categories:
    - application
tags:
    - FreeSWITCH
    - chatbot
---

为了将已有的文本聊天机器人以语音交互的形式（拨打电话）进行展示，需要将现有的聊天机器人集成到FreeSWITCH中，通过软电话的形式进行拨打测试。虽然当前版本的FreeSWITCH并没有直接的机器人接口可供使用（新版的genesys貌似已经具有了这个功能），但幸运的是，我们只需要对FreeSWITCH现有的功能进行简单的整合就可以实现聊天机器人的集成。

对FreeSWITCH而言，语音机器人的工作流程步骤大致可以分为以下几个步骤：
1. 用户通过软电话拨打客服电话（测试号码，如10086）
2. FreeSWITCH响应用户呼叫，并将实时语音流传递给MRCP服务器进行语音识别
3. FreeSWITCH将识别结果推送给聊天机器人，并将回答提供给MRCP服务器进行语音合成
4. FreeSWITCH将合成语音播放给用户

在实际使用过程中，简单起见，直接使用百度修改过的UniMRCP服务器提供MRCP的ASR服务。由于百度的MRCP服务器不支持语音合成，这里使用其Web接口实现。

![FreeSWITCH Chatbot](/assets/pic/freeswitch-chatbot.jpg)

# FreeSWITCH配置

## 添加拨号计划10086

编辑`/etc/freeswitch/dialplan/default.xml`，添加自定义的拨号计划，这里是10086

```xml
<extension name="integratedbot">
    <condition field="destination_number" expression="^10086$">
        <action application="answer"/>
        <action application="echo"/>
    </condition>
</extension>
```

或者在`/etc/freeswitch/dialplan/default`目录下新建一个名为`01-integratedbot-dialplan.xml`的配置文件

```xml
<include>
	<extension name="integratedbot">
		<condition field="destination_number" expression="^10086$">
			<action application="answer"/>
			<action application="echo"/>
		</condition>
	</extension>
</include>
```

添加完毕后，就可以拨打10086进行回声测试。当其他配置完毕后，这里的`<action application="echo"/>`可以改为`<action application="python" data="integratedbot.integrated"/>`对Python脚本进行测试。

## 启用python与unimrcp模块

编辑`/etc/freeswitch/autoload_configs/modules.conf.xml`，添加`mod_python`、`mod_unimrcp`启用Python及UniMRCP模块。

- mod_python模块对外提供FreeSWITCH的Python2接口。
- mod_unimrcp模块集成了UniMRCP客户端接口，简单配置后就可以直接通过UniMRCP服务器进行语音识别与语音合成。

```xml
<configuration name="modules.conf" description="Modules">
  <modules>
    <!-- ... others ... -->
    <load module="mod_python" />
    <load module="mod_unimrcp" />
  </modules>
</configuration>
```

## MRCP配置

FreeSWITCH默认提供了UniMRCP的MRCP v1配置文件，只需要修改UniMRCP服务器的IP和端口即可。这里我们的百度UniMRCP服务器架设在本地，所以IP为`127.0.0.1`，端口为8060。

```xml
<include>
  <!-- UniMRCP Server MRCPv1 -->
  <profile name="unimrcpserver-mrcp1" version="1">
    <param name="server-ip" value="127.0.0.1"/>
    <param name="server-port" value="8060"/>
    <param name="resource-location" value=""/>
    <param name="speechsynth" value="speechsynthesizer"/>
    <param name="speechrecog" value="speechrecognizer"/>
    <!--param name="rtp-ext-ip" value="auto"/-->
    <param name="rtp-ip" value="auto"/>
    <param name="rtp-port-min" value="4000"/>
    <param name="rtp-port-max" value="5000"/>
    <!--param name="playout-delay" value="50"/-->
    <!--param name="max-playout-delay" value="200"/-->
    <!--param name="ptime" value="20"/-->
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

最后修改`/etc/freeswitch/autoload_configs/unimrcp.conf.xml`中`default-tts-profile`、`default-asr-profile`的值为UniMRCP的MRCPv1配置文件的名称`unimrcpserver-mrcp1`。

# 百度MRCP服务器

> see https://ai.baidu.com/docs#/BICC-ASR-MrcpServer/top

从[百度智能呼叫中心](https://ai.baidu.com/sdk#itma)下载[MRCP Server](http://tianzhi-public.bj.bcebos.com/MrcpServerV1.2.tar.gz)，解压至`/opt/third/unimrcp`，并执行`bootstrap.sh`脚本初始化MRCP服务器。

```
tar -xf ~/MrcpServerV1.2.tar.gz
```

百度的MrcpServerV1.2自带的初始化脚本存在问题，需要手动进行初始化

```
/opt/third/unimrcp# tar -xf libs/gcc482.tar.gz && mkdir /opt/compiler && ln -s /opt/third/unimrcp/gcc482 /opt/compiler/gcc-4.8.2
```

安装依赖
```
apt install daemontools curl
```

编辑`/opt/third/unimrcp/conf/recogplugin.json`，设置appKey、appSecret以及maxLogFileSize，默认的日志文件太大，这里需要给一个小一点的值。

```
{
    "app.appKey": "API Key",
    "app.appSecret": "Secret Key",
    ...
    "log.maxLogFileSize": 10240
    ...
}
```

启动MRCP服务器

```
cd /opt/third/unimrcp/bin && ./control start
```

# Python脚本

> see https://freeswitch.org/confluence/display/FREESWITCH/mod_python

需要注意的是FreeSWITCH当前仅支持Python2，对于Python3尚未支持。同时对于Python脚本的调用通常是以Python包为基本单位。（需要将前面的拨号计划里的`<action application="echo"/>`改为`<action application="python" data="integratedbot.integrated"/>`）

这里，我们在`/usr/share/freeswitch/scripts`下新建了一个名为`integratedbot`的Python包，其下的子模块`integrated`用于语音识别、语音合成、机器人与FreeSWITCH的交互连接。

整个目录结构
```
/usr/share/freeswitch/scripts/integratedbot/
|- freeswitch.conf/
|  |- 01-integratedbot-dialplan.xml
|  |- ...
|
|- integrated.py
```

FreeSWITCH 运行调用模块中的`handler`函数进行交互

```python
def handler(session, args):
    # 会话响应
    session.answer()

    while session.ready():
        # 播放提示音，并识别语音
        pass

        # 机器人交互
        pass

        # 语音合成
        pass

    # 结束会话
    session.hangup()
```

整个脚本中最核心的函数便是用于实时语音识别的`play_and_detect_speech`函数。该函数主要有三个参数：

1. 将要播放的音频的文件名 `local_stream://moh`
2. MRCP引擎设置 `detect:unimrcp {start-input-timers=false, 'no-input-timeout=5000,recognition-timeout=5000}`
3. MRCP语法设置（意义不大，写不写无所谓）`builtin:grammar/boolean?language=en-US;y=1;n=2`

实际使用过程中发现`play_and_detect_speech`函数，在播放音频的时候也会进行识别操作，导致开启外放时，会将播放的语音也进行识别，机器人陷入自问自答的循环，愉快地打出GG。为了避免这种情况，应当在语音播报过程中禁止语音识别。最简单的方法是先将回答语音使用`playback`单独进行播报，然后`play_and_detect_speech`中播报的一个简短的提示音（声音很小），这种情况下，`play_and_detect_speech`会进行等待，直到上一个音频播放结束才会开始识别。（不是太确定，但效果暂时是这样）

更多的信息，参见仓库[@wkyo/freeswitch-chatbot][freeswitch-chatbot]

[freeswitch-chatbot]: https://github.com/wkyo/freeswitch-chatbot


# FAQ

## `mod_python` 无法调用Python脚本

安装Python后，环境变量没有设置好，请重启服务器或是手动更新环境变量

## Python脚本修改后没有效果

请重启FreeSWITCH

## `dialplan/default`目录下的自定义拨号计划无法被匹配

疑难杂症，迷，请写入`default.xml`
