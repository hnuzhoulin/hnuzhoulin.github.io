---
title: ceph full或者nearfull的解决方法探究
date: 2016/04/20 14:43:58
updated: 2017/03/12 11:35:35
categories: ceph
tags: [排错,full]
description: ceph full或者nearfull的解决方法探究
---
## 1.问题
昨晚在测试环境新建了7个rbd，然后在不同节点上向这些rbd执行fio，对rbd进行初始化。今早来发现fio没有按时完成，没有输出信息，一看集群状态如下：

```bash
root@console:~/CeTune/benchmarking# ceph health detail
HEALTH_ERR 1 full osd(s); 3 near full osd(s)
osd.4 is full at 95%
osd.0 is near full at 87%
osd.2 is near full at 90%
osd.3 is near full at 90%
```

这个对于见过场面的我来说不是什么大事（PS：其实主要因为是测试环境），于是乎习惯性翻笔记，但是发现以前的操作处理记录没了，于是乎今天记录一下今天的解决步骤。

## 2.解决
### 2.0.根本之道
集群满了，说明空间不够用，因此添加OSD才是最终的出路，但是一旦集群有OSD达到full的级别，及时添加OSD，集群也不会进行数据平衡操作，因此先要能让集群动起来在根据实际情况选择添加OSD还是删除数据。

### 2.1.调整OSD的full比例
执行下面的命令调整OSD的full和nearfull的比例：

```bash
ceph tell osd.* injectargs '--mon-osd-nearfull-ratio 0.98'
ceph tell osd.* injectargs '--mon-osd-full-ratio 0.98'
```

上面两个是最容易想到的，但是实际上调整这两个后并不能很好的解决我的这个问题，这个时候及时选择删除rbd以腾出空间依然弹出错误如下：

```bash
root@console:~/cluster# rbd rm volume-b0d3175b-7e74-4f71-aae5-39a53eda11ad
2016-04-08 08:56:27.911165 7f4cee0da7c0  0 client.104114.objecter  FULL, paused modify 0x1cc3a60 tid 2
```

### 2.2.调整pg的full比例
在回看我的场景，基本是所有的OSD的数据都满了，也就是说整个集群（包括pg和OSD）都是full的状态，因此还需要设置一个pg的full比例：

```bash
ceph pg set_full_ratio 0.98
```

此时再次执行删除rbd操作就会开始工作了，删除可能会比较慢

PS：**不推荐的删除方式**，先解挂所有rbd，再删除所有rbd，最后直接删除OSD数据目录下current目录下的所有文件。即使是这样，由于我的集群整个满了，删除都消耗了十几分钟。

### 2.3.调整osd-backfill的full比例
如果你发现是数据在OSD之间分布不均匀，你可以通过降低用量比较大的OSD的reweight值，这个值得范围是 `0~1` 标明权重，参考[这篇博文](http://cephnotes.ksperis.com/blog/2013/12/09/ceph-osd-reweight)：

```bash
ceph osd reweight 30 0.96
```

还有一种weight是根据硬盘容量计算出来的，实在不行的时候也可以司马当做活马医进行微调，但是请谨慎，关于两种weight的分析见[博文](http://cephnotes.ksperis.com/blog/2014/12/23/difference-between-ceph-osd-reweight-and-ceph-osd-crush-reweight);

进行上述调整后就会将数据从`OSD.30`中迁出一部分数据，但是我碰到一个情况是这种调整，一般是当前节点的OSD之间的数据迁移，比较少会迁移到别的节点去，如果此时当前节点的所有OSD的用量都大于85%的话，此时会出现状态为 `active+remapped+wait_backfill+backfill_toofull` 的PG，而有希望能够进行数据匀一匀，就需要调整backfill的full比例了：

```bash
ceph tell osd.* injectargs '--osd-backfill-full-ratio 0.9'
```

## 3.扩展
对于只有部分OSD用量满的情况跟前面的处理方式有很大不同。当然，**让集群能够动起来是最开始要做的事** ，这里的情况跟前面第二节介绍的方法还是要所有区别，尤其是对于线上环境，因为一旦调整这个全局的比例，可能整个集群的数据都会动起来，可能会有更多的数据迁移。因此另一种解决方法是临时调整、单个调整。但是，但是，但是，最最重要也是最先要解决的就是找到为什么只有少数OSD空间用量会满。

### 3.1.数据分布不均匀的原因

#### 3.1.1.crushmap问题
出现这种问题的原因可能是你手动修改过crushmap，将某些OSD划到了单独的组；或者是数据分布规则设定有误等。这种情况就只能自己到处crushmap分析一下了。不好举例，有问题的欢迎沟通。

#### 3.1.2.pg数设置不合理
这个是目前我遇到的最主要的一种坑。ceph的数据存储结构应该都很容易查到。就是 **file->object->pg->OSD->physics disk** 。因此，一旦这里的pg数设置过小，pg到OSD的映射不均匀就会造成OSD上分配到的数据不均匀。这种额解决方法就是重新调整 **pg_num** 和 **pgp_num** 。有关调整这部分的工作和注意事项等见我的另一篇博文：[调整pg数量的步骤](http://www.itzhoulin.com/blog/post/hnuzhoulin/increase-pg-num)

有一个python程序可以看各个pool在各个OSD上的pg分布，由此可以判断一下pg是否分布均匀。进而使用上面的方法调整该pool的pg_num和pgp_num。脚本点击[这里](https://gist.github.com/hnuzhoulin/fef193f5e26533fc08c86ad30209646d)

### 3.2.调整OSD的weight
对于只是少数OSD容量满的情况下，不管是上一小节中的调整crushmap还是pg_num的方法解决，最首先也还是需要集群能够进行正常读写操作，因此可以设置单个OSD的full和nearfull比例，也可以使用调整weight的方法。

关于一个OSD的weght，一共有两种：

 - 一种就是在crush自动设置的weight，这个值是根据OSD所在硬盘空间大小算出来的，1T的硬盘可用空间，这个值为1.
 - 另一种就是人为添加的用以表示数据分布权重值的 reweight，这个值介于0-1之间，越小表示分布权重越低。

两种weoght都可以在命令 `ceph osd ree` 中看到，如下：

```bash
root@node1:~# ceph osd tree
# id    weight  type name       up/down reweight
-1      13.65   root default
-2      2.73            host node1
0       2.73                    osd.0   up      1
-3      2.73            host node2
1       2.73                    osd.1   up      1
-4      2.73            host node3
2       2.73                    osd.2   up      1
-5      2.73            host node4
3       2.73                    osd.3   up      1
-6      2.73            host node5
4       2.73                    osd.4   up      1
```

从上面的额输出来看，第二列的weight就是crush根据硬盘可用容量计算出来的，最后一列的reweight就是这个OSD的数据分布权重。这里我们应对某一个OSD满的另一种方法就是调整reweight，以降低数据在该OSD上的分布，调整的效果可以看看这个[博文](http://cephnotes.ksperis.com/blog/2013/12/09/ceph-osd-reweight/)，此文ceph中国论坛有[翻译稿](http://bbs.ceph.org.cn/article/38)

ceph提供有自动调节reweight的工具：`ceph osd reweight-by-utilization`
