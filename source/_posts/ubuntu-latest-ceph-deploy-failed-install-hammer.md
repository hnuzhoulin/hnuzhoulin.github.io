---
title: Ubuntu下最新版ceph-deploy安装Hammer失败
tags: [部署]
top: false
date: 2017-03-14 14:46:02
updated: 2017-03-14 14:46:02
categories: ceph
description: Ubuntu下最新版ceph-deploy安装Hammer失败的解决方法
---


## 一.问题

近来需要部署一套Ceph集群，在纠结了一阵子版本之后还是果断的选择了才发布过更新的 `Hammer`，不求新功能新特性的情况下，这个应该是最稳定的，按照以往的经验，参照自己的博文[本地源ceph-deploy安装](../../../../2016/06/04/ceph-deploy) 就开始安装了，但是出现了几个错误。

## 二.ubuntu官方源的ceph-deploy

第一次安装使用的是ubuntu官方源的ceph-deploy，虽然安装上述方法指定了安装H版本，但是不管怎么变换姿势，始终都是安装的F版本，而且包的来源也都是官方源而不是我指定的本地H版本的源。

## 三.直接使用本地ceph源安装ceph-deploy

这次试了一下先把ubuntu官方源干掉，直接将本地H版的源写进`sources.list`，然后安装ceph-deploy，再次按照原来的博文修改后进行安装，这次的错如下图：

![找不到ceoh-osd包](https://ww1.sinaimg.cn/large/006tNbRwgy1fdmef00fesj313a07adhu.jpg)

我去，这是什么鬼，H版什么时候有ceph-osd这个包了，于是google了一下，找到了邮件列表里面的一个回答，[ubuntu下最新的ceph-deploy安装Hammer失败](http://lists.ceph.com/pipermail/ceph-users-ceph.com/2016-September/012911.html)

看来是有段时间呢没有关注ceph-deploy源码了，J版之后才会有ceph-osd，ceph-mon等独立的包，而最新版的ceph-deploy有这个bug。我查看了一下自己的ceph-deploy版本，果然是`1.5.37`，按照邮件列表的推荐彻底卸载所有相关包之后使用pip安装1.5.25版本：

```bash
pip install ceph-deploy==1.5.25
```

再次安装一切正常。