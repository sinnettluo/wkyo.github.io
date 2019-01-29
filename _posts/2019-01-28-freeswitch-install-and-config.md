---
layout: post
title: FreeSWITCH 1.8 安装与配置
date: 2019-01-22 12:09:44 +0800
update: 2019-01-29 17:11:01 +0800
categories:
    - application
tags:
    - FreeSWITCH
---

FreeSWITCH是一个开源的电话软交换平台，其可以运行于Windows，Linux 以及 macOS。

# 安装

## 从Debian软件包安装

- Debian 9 "Stretch"
- FreeSWITCH 1.8 Package

```sh
# 添加 keyring
wget -O - https://files.freeswitch.org/repo/deb/freeswitch-1.8/fsstretch-archive-keyring.asc | apt-key add -
 
# 添加 FreeSWITCH 官方软件源
echo "deb http://files.freeswitch.org/repo/deb/freeswitch-1.8/ stretch main" > /etc/apt/sources.list.d/freeswitch.list
echo "deb-src http://files.freeswitch.org/repo/deb/freeswitch-1.8/ stretch main" >> /etc/apt/sources.list.d/freeswitch.list

# 更新软件源，并安装 FreeSWITCH（安装完毕后 FreeSWITCH 会自行启动）
apt-get update && apt-get install -y freeswitch-meta-all

# 访问 FreeSWITCH
# 默认情况下`event_socket`会监听IPv6地址，如果当前环境不支持IPv6，`event_socket`服务将无法启动，`fs_cli`将无法访问 FreeSWITCH。
fs_cli -rRS
```
# 客户端

比较好用的软电话客户端主要有[Linphone][] (全平台)、[X-Lite][] (Windows, macOS)。

# 配置

## 防火墙配置

对于部署在云服务器上面的 FreeSWITCH，需要暴露适当的端口，以允许sip客户端进行访问。这里我们只启用默认的internal配置文件使用的标准sip端口 `TCP/UDP:5060` 以及RTP端口 `UDP:16384-32768`。默认情况下的internal配置文件仅支持注册用户登录，使用这个配置可以避免部分未注册的恶意SIP扫描给服务器带来的负担（external配置文件支持未登录的用户自动注册）。启用5060端口可以使用预定义的账号进行登录，以及拨打5000进行IVR测试，但只能单方面的接受FreeSWITCH服务器传过来的音频（类似于提示音），而无法上传音频。启用`16384-32768`端口允许RTP媒体流，这时可以拨打9196进行回声测试。

| 来源 | 端口协议 | 策略 | 备注 |
| --- | --- | ---| --- | --- |
| 0.0.0.0/0 | TCP:5060 | 允许 | 默认internal配置文件的标准SIP端口，用于SIP信令 |
| 0.0.0.0/0 | UDP:5060 | 允许 | 默认internal配置文件的标准SIP端口，用于SIP信令 |
| 0.0.0.0/0 | UDP:16384-32768 | 允许 | RTP/RTCP 多媒体流 |

> Note：
> - 如果FreeSWITCH服务器才启动，可能需要等待几分钟才能够连接上服务器进行正常的拨号操作。
> - 使用上面的配置，客户端请使用UDP。
> - 关于NAT穿透的问题，FreeSWITCH在腾讯云上部署完毕后，就可以直接接通，尚未发现任何问题。（FreeSWITCH的systemd service文件默认添加了`-nonat`参数，禁用NAT，但依旧正常工作，比较迷）

FreeSWITCH 所使用的端口，参考 https://freeswitch.org/confluence/display/FREESWITCH/Firewall#Firewall-TypicalPorts

| 端口 | 协议 | 应用协议 | 描述 |
| --- | --- | --- | --- | --- |
| 1719 | UDP | H.323 Gatekeeper RAS port | |
| 1720 | TCP | H.323 Call Signaling | |
| 3478 | UDP | STUN service | Used for NAT traversal |
| 3479 | UDP | STUN service | Used for NAT traversal |
| 5002 | TCP | MLP protocol server |
| 5003 | UDP | Neighborhood service |
| 5060 | UDP & TCP | SIP UAS | Used for SIP signaling (Standard SIP Port, for default Internal Profile) |
| 5070 | UDP & TCP | SIP UAS | Used for SIP signaling (For default "NAT" Profile) |
| 5080 | UDP & TCP | SIP UAS | Used for SIP signaling (For default "External" Profile) |
| 8021 | TCP | ESL | Used for mod_event_socket * |
| 16384-32768 | UDP | RTP/ RTCP multimedia streaming | Used for audio/video data in SIP, Verto, and other protocols |
| 5066 | TCP | Websocket | Used for WebRTC |
| 7443 | TCP | Websocket | Used for WebRTC |


## 默认密码变更

FreeSWITCH 的默认密码为1234，当使用默认密码进行注册登陆时，会被强制等待10s，为了避免长时间等待，同时避免恶意访问，可以在配置文件`/etc/freeswitch/vars.xml`的第一行有效代码中找到`<X-PRE-PROCESS cmd="set" data="default_password=1234"/>`，将1234改为其他即可。

# FAQ

## `fs_cli`无法访问 FreeSWITCH 服务

```
root@VM-0-11-debian:~# fs_cli -rRS
[ERROR] fs_cli.c:1679 main() Error Connecting []
[INFO] fs_cli.c:1685 main() Retrying
```

有两种情况会导致这个错误：

1. 首次安装后，FreeSWITCH 服务启动失败。FreeSWITCH 安装完毕后，会尝试自启，但这里很有可能会失败，需要重新手动启动 `systemctl start freeswitch`。
2. `fs_cli` 通过 FreeSWITCH 的 `event_socket` 接口进行交互，但默认情况下 FreeSWITCH 监听 IPv6 环路，即 `::`。如果服务环境不支持IPv6，`event_socket` 接口将会启动失败（FreeSWITCH 服务仍然正常启动）。解决办法在文件`/etc/freeswitch/autoload_configs/event_socket.conf.xml`中将监听地址`listen-ip`的值改为 IPv4（本地IP `127.0.0.1` 或者环回地址 `0.0.0.0`）

## FreeSWITCH 呼叫慢

有两种情况可能会导致 FreeSWITCH 呼叫等待时间长：

使用了默认密码1234。FreeSWITCH在默认的拨号计划`/etc/freeswitch/dialplan/default.xml`中，定义了使用默认密码将会等待10s。
```xml
<condition field="${default_password}" expression="^1234$" break="never">
    <action application="log" data="CRIT WARNING WARNING WARNING WARNING WARNING WARNING WARNING WARNING WARNING "/>
    <action application="log" data="CRIT Open $${conf_dir}/vars.xml and change the default_password."/>
    <action application="log" data="CRIT Once changed type 'reloadxml' at the console."/>
    <action application="log" data="CRIT WARNING WARNING WARNING WARNING WARNING WARNING WARNING WARNING WARNING "/>
    <action application="sleep" data="10000"/>
</condition>
```

恶意SIP端口扫描。在公网环境中，由于大量恶意SIP访问，FS资源被占用，新的呼叫将会进行长时等待。最简单的办法是使用ipset与iptables工具直接屏蔽掉一部分恶意IP（可以从仓库firehol/blocklist-ipsets获取SIP恶意IP列表）。实际使用过程中发现大部分的恶意ip来自国外（特别是荷兰与德国），或许可以把整个国外的IP都ban掉。
```sh
# 创建集合blacksip并添加ip
ipset create blacksip hash:net family inet hashsize 1024 maxelem 1000000
ipset add blacksip 36.66.69.33
ipset add blacksip 37.49.231.87
ipset add blacksip ...other bad ip...
# 这里直接拒绝来自blacksip集合内ip的所有访问途径
iptables -I INPUT -m set --match-set blacksip src -j DROP
```

## FreeSWITCH 默认拨号计划

参考 https://freeswitch.org/confluence/display/FREESWITCH/Default+Dialplan+QRF

| 号码 | 描述 |
| --- | -- |
| 5000 | 语音导航演示 |
| 9195 | 回声测试（延迟5秒） |
| 9196 | 回声测试 |
| 9664 | 播放音乐 |

## 测试号码呼叫成功，拨打用户失败，提示`USER_NOT_REGISTERED`

```
6d94bfbf 2019-01-29 15:42:31.330388 [NOTICE] switch_channel.c:1104 New Channel sofia/internal/1003@117.136.54.125:48324 [6d94bfbf-b186-4e19-94ec-cb2876e5acdf]
6d94bfbf 2019-01-29 15:42:31.330388 [DEBUG] mod_sofia.c:5019 (sofia/internal/1003@117.136.54.125:48324) State Change CS_NEW -> CS_INIT
1f545a32 2019-01-29 15:42:31.330388 [NOTICE] switch_ivr_originate.c:2944 Cannot create outgoing channel of type [error] cause: [USER_NOT_REGISTERED]
```
很有可能是网络问题，公司网络下通过命令行originate和手机客户端都无法拨通PC上的客户端，但可以拨通手机客户端；然后切换手机网络后PC与手机两者都表现正常，迷（F**k）。


[Linphone]: http://www.linphone.org/
[X-Lite]: https://www.counterpath.com/x-lite-download/