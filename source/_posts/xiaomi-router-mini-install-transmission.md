---
title: xiaomi-router-mini-install-transmission
tags: [xiaomi,transmission]
top: false
date: 2017-01-12 18:21:36
updated: 2017-01-12 18:28:30
categories: tool
description: 小米路由器Mini版安装transmission挂PT
---

主要参考两篇论坛高手的教程：
- http://www.miui.com/thread-2093928-1-1.html
- http://bbs.xiaomi.cn/t-12985345

其实我只是小小合并了一下下，方法也是基于openwrt的源来安装。

## 1.安装
### 1.1.开启ssh
对于小米路由器mini来说，打开 http://d.miwifi.com/rom/ssh  按照提示操作即可。

### 1.2.修改openwrt源地址到基础软件包源
由于小米路由器mini的空间不足，所以所有相关软件都安装到外接的移动硬盘内。

默认带的源地址已经不存在，需要修改到最新的。同时安装transmission需要的依赖和transmission软件本身不在同一个源内，所以需要分批添加分批更新。

修改文件和内容如下：

```
root@XiaoQiang:~# vi /etc/opkg.conf
src/gz attitude_adjustment http://downloads.openwrt.org/barrier_breaker/14.07/ramips/mt7620a/packages/base
dest root /data
dest ram /tmp
lists_dir ext /data/var/opkg-lists
option overlay_root /data
dest usb /extdisks/sda1/opkg

arch all 100
arch ramips 200
arch ramips_24kec 300
```

**说明：**
1. 第一行是base源地址，是一些依赖项的
2. `dest usb /extdisks/sda1/opkg` 是告知opkg插件安装命令增加一个插件安装目录，也就是USB存储设备中的opkg目录下（opkg目录是我在USB存储设备中新建的目录）。以后使用opkg命令安装插件时使用“opkg -d usb install ...”来安装到USB存储设备中。
3. 最后三行arch开头的很重要，不添加这三行安装的时候会出现 “no valid architecture” 的错误

### 1.3.更新源
正常更新源命令如下：

```
root@XiaoQiang:~# opkg update
```

但是实际上这里会提示你压根没有opkg，所以需要手动上传一个，附件是我用的opkg，我把它放到了目录 `/extdisks/sda1/opkg` ，因此实际执行：


```
root@XiaoQiang:~#extdisks/sda1/opkg/opkg update
```

补充说明：
如果你先用packages源地址(http:// downloads.openwrt.org/barrier_breaker/14.07/ramips/mt7620a/packages/packages)试图直接安装Transmission，会提示缺少依赖的插件包：

```
root@XiaoQiang:~# opkg -d usb install transmission-daemon
Installing transmission-daemon (2.84-1) to usb...
Downloading http:// downloads.openwrt.org/barrier_breaker/14.07/ramips/mt7620a/packages/packages/transmission-daemon_2.84-1_ramips_24kec.ipk.
Collected errors:
* satisfy_dependencies_for: Cannot satisfy the following dependencies for transmission-daemon:
* libc * libcurl * libopenssl * libpthread * libevent2 * librt *
* opkg_install_cmd: Cannot install package transmission-daemon.
```

由此可以，需要先用b

### 1.4.安装Transmission所需的基础插件包
上面提到的依赖中libc无法通过“opkg -d usb install libc”来直接安装，只能手动下载后安装它。

首先进入到一个能够下载文件的目录，可以是/tmp临时目录，但我用/extdisks/sda1/opkg目录，下载后可以以后留用：

```
root@XiaoQiang:~# cd /tmp
或者
root@XiaoQiang:~# mkdir /extdisks/sda1/opkg
root@XiaoQiang:~# cd /extdisks/sda1/opkg/
```

下载libc基础插件包（它的地址可以通过在浏览器中打开base源地址，搜索“libc”找到）：

```
root@XiaoQiang:/extdisks/sda1/opkg# wget http://downloads.openwrt.org/barrier_breaker/14.07/ramips/mt7620a/packages/base/libc_0.9.33.2-1_ramips_24kec.ipk
```

安装它（它会自动安装依赖的libgcc包）：

```
root@XiaoQiang:/extdisks/sda1/opkg# ./opkg -d usb install libc_0.9.33.2-1_ramips_24kec.ipk
```

安装Transmission所依赖的其他插件包，可以一起安装：

```
root@XiaoQiang:/extdisks/sda1/opkg# ./opkg -d usb install libcurl libevent2 libopenssl libpthread librt
```

可以通过命令“opkg list-installed”列出当前安装的插件包：
```
root@XiaoQiang:/extdisks/sda1/opkg# ./opkg list-installed
libc - 0.9.33.2-1
libcurl - 7.38.0-1
libevent2 - 2.0.21-1
libgcc - 4.8-linaro-1
libopenssl - 1.0.1j-1
libpolarssl - 1.3.8-2
libpthread - 0.9.33.2-1
librt - 0.9.33.2-1
zlib - 1.2.8-1
```

### 1.5.安装Transmission
首先要改成packages源地址，并更新源：
```
root@XiaoQiang:/extdisks/sda1/opkg# vi /etc/opkg.conf
src/gz attitude_adjustment http://downloads.openwrt.org/barrier_breaker/14.07/ramips/mt7620a/packages/packages
---
root@XiaoQiang:/extdisks/sda1/opkg# ./opkg update
```

安装Transmission的两个组件：transmission-daemon（核心程序），transmission-web（网页控制中心）

```
root@XiaoQiang:/extdisks/sda1/opkg# opkg -d usb install transmission-daemon
root@XiaoQiang:/extdisks/sda1/opkg# opkg -d usb install transmission-web
```


至此安装结束，下面来配置和启动Transmission。

## 2.配置Transmission
- 因为我们不是默认安装到/data，而是按照到USB存储设备，所以运行下面这个命令添加“TRANSMISSION_WEB_HOME”环境变量，来告知Transmission网页控制台的所在目录：

```
root@XiaoQiang:/extdisks/sda1/opkg# export TRANSMISSION_WEB_HOME=/extdisks/sda1/opkg/usr/share/transmission/web/
```

- 启动并生成默认的配置目录（我将配置目录同样制定到USB存储设备中）：

```
# /extdisks/sda1/opkg/usr/bin/transmission-daemon -g /extdisks/sda1/opkg/transmission-daemon
```
- 编辑Transmission的配置文件，其中"download-dir"是默认下载到的目录，而"rpc-port"是网页控制台所用的端口，默认是9091：

```
root@XiaoQiang:/extdisks/sda1/opkg# vi /extdisks/sda1/opkg/transmission-daemon/settings.json
修改以下：
    "download-dir": "/extdisks/sda1/Downloads", 
    "rpc-port": 9876,
    "rpc-whitelist-enabled": false, 
如果要设置用户名和密码登陆（注意保留引号，感谢segafans分享方法）：
    "rpc-authentication-required": true,
    "rpc-password": "密码",
    "rpc-username": "用户名",
```

**补充说明：**
修改端口的原因在于默认端口9091已经被系统占用，名为“plugincenter”（插件中心？）的程序：
```
root@XiaoQiang:/extdisks/sda1# netstat -lpa | grep 9091
tcp 0 0 localhost:9091 0.0.0.0:* LISTEN 8099/plugincenter
```
- 重启Transmission使修改后的配置生效：

```
root@XiaoQiang:/extdisks/sda1/opkg# killall -HUP transmission-daemon
```

## 3.配置防火墙
- 编辑防火墙配置文件，在文件最后添加以下内容：

```
root@XiaoQiang:/extdisks/sda1/opkg# vi /etc/config/firewall
添加以下：
config rule 'transmission_web'
        option src 'wan'
        option dest_port '9876'
        option proto 'tcp'
        option target 'ACCEPT'
        option name 'transmission mgmt from wan'
config rule 'transmission_peer_tcp'
        option src 'wan'
        option dest_port '51413'
        option proto 'tcp'
        option target 'ACCEPT'
        option name 'transmission incoming tcp'
config rule 'transmission_peer_udp'
        option src 'wan'
        option dest_port '51413'
        option proto 'udp'
        option target 'ACCEPT'
        option name 'transmission incoming udp'
```

- 重启防火墙使配置生效：

```
root@XiaoQiang:/extdisks/sda1/opkg# /etc/init.d/firewall restart
```

至此，你可以在浏览器中输入地址“192.168.31.1:9876”来访问Transmission的网页控制台（出于习惯，我把路由器地址改成了192.168.1.1）。

- 设置开机自启动
将下面两行命令添加到/etc/rc.local中的“exit 0”之前
```
# export TRANSMISSION_WEB_HOME=/extdisks/sda1/opkg/usr/share/transmission/web/
# /extdisks/sda1/opkg/usr/bin/transmission-daemon -g /extdisks/sda1/opkg/transmission-daemon
```

**注意：**
部分情况会不生效，此时重启路由器后需要SSH到路由器下，运行下面两个命令，可能是因为在启动脚本运行时，USB存储设备还没准备好。如果有谁成功实现自启动就好了。

## 4.问题
1. 我添加了4个种子下载，然后路由器就死掉了，可以ping通，但是app无法管理，web管理平台打不开，transmission的web打不开，
2. 设置了只允许同时下载两个种子，删掉了一切可以删掉的插件，看着它跑了一个小时，睡一觉起来在外网能ping通回去，检测transmission的端口也开着，但是各种web和app无法打开。