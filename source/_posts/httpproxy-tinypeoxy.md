---
title: httpproxy-tinypeoxy
date: 2016-06-04 14:40:48
categories: tool
tag: [proxy,linux]
description: 使用tinyproxy在ubuntu下搭建http proxy服务器
---
## 一、前言
关于为什么要玩玩HTTP代理就不用我多说了。

## 二、搭建环境

- Linux Ubuntu
-  tinyproxy

## 三、安装方法

```bash
$sudo apt-get install tinyproxy
```

安装后自动以root权限开启了tinyproxy服务，且默认监听端口是8888

## 四、启动帮助

```bash
$tinyproxy --help
Usage: tinyproxy [options]
Options are:
-d Do not daemonize (run in foreground).
-c FILE Use an alternate configuration file.
-h Display this usage information.
-l Display the license.
-v Display version information.
```

## 五、根用户的启动方法

```bash
* 默认启动
$sudo service tinyproxy start
* 重启
$sudo service tinyproxy restart
* 停止
$sudo service tinyproxy stop
```

## 六、DIY配置
### 6.1 默认配置文件位置

/etc/tinyproxy.conf
(可以从/etc/init.d/tinyproxy包装器脚本中查到)

### 6.2 默认配置说明
* 以根用户启动时，在初始化完成后切换uid/gid为nobody/nogroup
* Port 默认监听端口为8888(该端口无需用root权限绑定)
* 默认在所在网卡上监听
* Logfile (必须)日志文件, 默认/usr/var/log/tinyproxy/tinyproxy.log，在LogFile文件不存在时会警告，不会运行失败。
* Pidfile (必须)pid文件, 默认/usr/var/run/tinyproxy/tinyproxy.pid，在PidFile文件不存在时会运行失败。
* StartServers 初始启动的代理服务器子进程(默认是10个)
* Allow 允许使用tinyproxy进行HTTP代理的IP地址。默认是127.0.0.1，如果想要公开tinyproxy代理服务器，则把Allow一行注释掉。

### 6.3 Diy配置说明
tinyproxy可以以普通用户权限运行，只要监听端口是公开的就可以了。具体Diy配置方法如下:

#### 6.3.1 打包可执行程序与默认配置文件 

```bash
$which tinyproxy
/usr/sbin/tinyproxy
$cp /usr/sbin/tinyproxy ~/bin
$cp /etc/tinyproxy.conf ~/etc
```

#### 6.3.2修改配置

```bash
将Port默认的8888改成你想要的端口(如ljysrv上面的8990 TCP端口)
将Allow 127.0.0.1注释掉
将Logfile改为/tmp/tinyproxy.log
将PidFile改为/tmp/tinyproxy.pid
```

#### 6.3.3 启动

```bash
$cd ~/bin
$./tinyproxy -c ~/etc/tinyproxy.conf
```

#### 6.3.4 关闭

```bash
$killall tinyproxy
```

## 七、设置代理

```bash
export http_proxy=http://yourproxyaddress:proxyport
```