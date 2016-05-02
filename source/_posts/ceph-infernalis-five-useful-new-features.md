---
title: ceph的infernalis版本的五个很实用的新功能
date: 2015/11/24 12:42:46
updated: 2016/02/01 00:05:54
categories: ceph
tags: [infenalis,tool]
description: ceph的infernalis版本的五个很实用的新功能：rbd指定容量单位，查看镜像实际占用空间，osd性能分析器，动态启用object-map，默认镜像格式变为2
---
*本文由 Ceph中国社区-半天河翻译，Thomas校稿。*

英文出处：[Sébastien Han](http://www.sebastien-han.fr/blog/2015/11/18/five-useful-new-features-from-ceph-infernalis/)  欢迎加入 [翻译组](http://7xj5dz.com1.z0.glb.clouddn.com/qun.png)

![](http://www.sebastien-han.fr/images/ceph-infernalis-5-new-features.jpg)

Infernalis已于几周前发布，我对所做的工作印象深刻。因此我将向你展示新版本中5个非常实用的新功能。

## 1.使用单位来创建镜像和基准测试
在此之前，默认的单位是 **MB** ，因此我们不得不填写经过换算后的镜像大小

    $ rbd create a -s 1G

## 2.镜像实际使用空间
由于镜像是稀疏的并且很多的虚拟存储管理器支持discard，因此具备跟踪镜像实际使用空间的能力是很不错的。在这里显然我们强烈推荐使用object-map，因为它将加快计算速度。

```
$ rbd du a
warning: fast-diff map is not enabled for a. operation may be slow.
NAME PROVISIONED USED
a          1024M    0

$ rbd -p rbd bench-write a --io-size 4096 --io-threads 1 --io-total 4096 --io-pattern rand
bench-write  io_size 4096 io_threads 1 bytes 4096 pattern rand
  SEC       OPS   OPS/SEC   BYTES/SEC
elapsed:     0  ops:        1  ops/sec:    83.42  bytes/sec: 341671.93

$ rbd du a
warning: fast-diff map is not enabled for a. operation may be slow.
NAME PROVISIONED  USED
a          1024M 4096k
```

## 3.OSD性能分析器
一个新的命令，它连接Daemon进程的socket并显示很多统计数据。

```
$ ceph daemonperf osd.1
---objecter--- -----------osd-----------
writ read actv|recop rd   wr   lat  ops |
  0    0    0 |   0    0   53k 339   13
  0    0    0 |   0    0  753k 516  180
  0    0    0 |   0    0  843k 149  206
  0    0    0 |   0    0  507k  34  123
  0    0    0 |   0    0  630k  45  150
  0    0    0 |   0    0  626k  33  149
  0    0    0 |   0    0  573k  57  138
  0    0    0 |   0    0  172k  49   42
  0    0    0 |   0    0    0    0    0
  0    0    0 |   0    0    0    0    0
```

## 4.动态的启用和禁用镜像特性
下面的例子展示了如何在镜像创建之后启动object map。

```
$ rbd create a -s 1G --image-feature exclusive-lock
$ rbd info a
rbd image 'a':
        size 1024 MB in 256 objects
        order 22 (4096 kB objects)
        block_name_prefix: rbd_data.856d51f8ceac
        format: 2
        features: exclusive-lock
        flags:

$ rbd feature enable a object-map
```

## 5.默认镜像格式变为format 2
归功于该功能，我们无需在创建镜像时指定格式，也无需在 *ceph.conf* 中添加一行配置。



转载自：[ceph中国社区](http://bbs.ceph.org.cn/article/22)