---
title: 删除osd的正确姿势
date: 2016/01/12 13:46:35
updated: 2016/02/01 00:05:51
categories: ceph
tags: [osd]
description: 删除osd的正确姿势
---
最近邮件列表又有关于删除osd正确姿势的讨论，按照官网的步骤走的话，在 `标记osd为out` 和 `从crushmap删除osd` 这两步都会触发数据再平衡。

根据邮件列表的指示，总结删除osd的正确姿势如下：

1. ceph osd crush reweight osd.X 0.0
... wait for rebalance to finish....
2. ceph osd out X
3. service ceph-osd stop id=X
4. ceph osd crush remove osd.X
5. ceph auth del osd.X
6. ceph osd rm X

参考链接[http://permalink.gmane.org/gmane.comp.file-systems.ceph.user/25629](http://permalink.gmane.org/gmane.comp.file-systems.ceph.user/25629)