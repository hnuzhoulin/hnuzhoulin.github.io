---
title: Ceph里手动修复Object
date: 2015/11/24 12:02:18
updated: 2016/02/01 00:05:50
categories: ceph
description: 翻译Sébastien的博客，手动修复scrub errors
---
*本文由 Ceph中国社区-半天河翻译，高华敏 校稿。*

英文出处：[Sébastien Han](http://www.sebastien-han.fr/blog/2015/04/27/ceph-manually-repair-object/)  欢迎加入 [ceph中国社区翻译组](http://7xj5dz.com1.z0.glb.clouddn.com/qun.png)

![ceph-manually-repair-objects](http://sebastien-han.fr/images/ceph-manually-repair-objects.jpg)

调试scrub错误有窍门，你甚至无需了解调试究竟是怎样进行的。

假设你的集群状态信息类似下面的：

    health HEALTH_ERR 1 pgs inconsistent; 2 scrub errors

现在让我们开始对故障进行排错吧。

## 定位PG
用下面这条简单的命令找到出问题的PG：

```
$ sudo ceph health detail
HEALTH_ERR 1 pgs inconsistent; 2 scrub errors
pg 17.1c1 is active+clean+inconsistent, acting [21,25,30]
2 scrub errors
```

由上可知，出问题的PG是 `17.1c1` ，该PG的acting OSD是 21、25和30 。

你可以尝试运行 `ceph pg repair 17.1c1` ，后检查问题是否得以修复。这条命令有时可行，但如果不能修复故障的话，你就得深挖

## 定位故障
为了找到故障的根本起因，我们需要到OSD的日志文件里仔细看看。使用这个简单命令：grep -Hn 'ERR' /var/log/ceph/ceph-osd.21.log，注意，如果日志文件是循环写入的，你需要使用zgrep替代grep。 

日志中的这段话显示了故障的根本起因：

```
log [ERR] : 17.1c1 shard 21: soid 58bcc1c1/rb.0.90213.238e1f29.00000001232d/head//17 digest 0 != known digest 3062795895
log [ERR] : 17.1c1 shard 25: soid 58bcc1c1/rb.0.90213.238e1f29.00000001232d/head//17 digest 0 != known digest 3062795895
```

这些日志表明object digest应该是3062795895而现在实际情况是0 。

## 定位object
现在需要深入OSD.21的目录，幸运的是前面我们抓到的日志已经足够清晰简洁明了的表明对应object信息：

- 问题PG：17.1c1
- OSD编号：21、25，30
- Object名字：rb.0.90213.238e1f29.00000001232d

接下来，我们搜索这个object：

```
$ sudo find /var/lib/ceph/osd/ceph-21/current/17.1c1_head/ -name 'rb.0.90213.238e1f29.00000001232d*' -ls
671193536 4096 -rw-r--r--   1 root     root      4194304 Feb 14 01:05 /var/lib/ceph/osd/ceph-21/current/17.1c1_head/DIR_1/DIR_C/DIR_1/DIR_C/rb.0.90213.238e1f29.00000001232d__head_58BCC1C1__11
```

还有一些其他事项需要检查：
- 查看PG每个acting OSD上的该object的大小
- 查看PG每个acting OSD上的该object的MD5

检查之后，对比所有这些object，找到出故障的那个object。

## 修复故障
按照下面的方法移除故障object即可：
- 停止错误的object所在OSD
- flush日志（`ceph-osd -i <id> --flush-journal` ）
- 将故障object移到别的地方
- 重新启动OSD服务
- 执行命令 `ceph pg repair 17.1c1`

> 直接删除object看起来有些粗暴，但最终是由Ceph job做了一些工作。在有三个副本的情况下，以上方法显然能够正常使用，因为将其中两个版本和另一个版本进行对比是相当容易的。但如果只有2个副本，情况则有些不同。Ceph可能因无法解决版本冲突，导致故障依然存在。所以，一个简单的窍门是，选择一个最新版本的object，在集群上将其标记为 noout，停掉有错误版本object的OSD，稍等片刻后，重启OSD，并取消之前设置在最新版object上的noout标记。这时，集群就会将正确版本的object同步到有错误版本object的OSD。

转载自：[ceph中国社区](http://bbs.ceph.org.cn/article/17)