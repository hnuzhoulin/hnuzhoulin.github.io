---
title: php-sdk访问ceph对象存储时405错误
date: 2015/11/22 22:09:04
updated: 2015/11/22 22:10:06
categories: ceph
tags: [rgw,dns]
description: php-sdk访问ceph对象存储时405错误
---
## 现象
在使用ceph对象存储的php sdk备份数据到ceph时，在对象存储创建bucket失败，错误如下：

    [Code] => MethodNotAllowed

查看rgw日志错误代码是 **405**

但是使用cloudberry访问S3是没问题的，使用IP

## 环境
尝试过F版和H版，rgw使用civetweb创建，在ceph.conf中使用下面的配置修改端口为80：

    [client.rgw.gcibackup]
    host = gcibackup
    log file = /var/log/ceph/client.rgw.gcibackup.log
    rgw_frontends =civetweb port=80
    rgw print continue = false

php这里使用的是域名访问云存储，域名解析是自己用dnsmasq搭的，设置为s3.storage.com

## 解决
再次尝试备份到生产环境云存储中，这个以前是测过没问题的，同样的错误。

总结一下发现唯一的问题就是域名和rgw的配置问题，同时搜索邮件列表从 [radosgw: "MethodNotAllowed" response](http://www.spinics.net/lists/ceph-devel/msg12135.html) 中获得启发。

首先是将rgw的配置加上 **rgw dns name =backup**

然后将域名修改为主机名一致即： **backup.storage.com**

再次尝试一切OK。