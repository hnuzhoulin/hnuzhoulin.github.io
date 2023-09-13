---
title: pre-split-filestore-diir-when-create-pool
tags: [filestore]
top: false
date: 2018-10-21 15:19:18
updated: 2018-10-21 15:19:18
categories: ceph
description: OSD后端使用filestore时，目录分裂会给运行中的集群带来严重的性能下降，在10.2.11版本正式支持的目录预分裂功能可以提前规避这种运行中可能的性能下降。
---

## 背景
使用filestore的ceph是将最终的object存在带文件系统的磁盘上的，这样就有一个单目录下文件数的问题需要处理：在文件系统场景下，目录下的文件数过大会影响在操作该目录下文件的性能。

因此ceph针对这个提出了split和merge的概念来应对，即当一个pg下的object数超过设置的阈值的时候，执行split操作：

- 在pg的根目录下新建一定数量的子目录
- 将根目录下的所有object根据一定规则移动到上述新建的子目录中
- pg执行分裂期间是会block对该pg的操作请求的

从上述过程很容易知道在分裂期间，集群的性能是会大打折扣的，我在实际场景下就碰到过很多次这种情况，一旦分裂期间client端也有比较大负载的时候会有大量的slow request出现，这样业务会有明显的性能降低感知，甚至直接导致业务超时。对此我们还专门做了告警，一旦检测到有pg开始分裂，我们会格外关注这个集群，同时跟业务做好沟通，能迁移的先迁移部分负载，不能迁移的调大超时时间。

另外一个merge其实就是split的逆过程，也就是当object低于阈值时，回收子目录，object转移到上一级数据目录。

## 对策
其实早在4年前就有人提到这个问题，当时Sage也有提到响应的解决方法，详情见：[Disk saturation during PG folder splitting](http://tracker.ceph.com/issues/7593)

主要的解决办法其实也很明确：提前分裂，一旦分裂即使object数下降也不执行merge。

至于这个解决方案，社区有两种方法：(两种方法都需要的前提条件：设置`filestore merge threshold为一个负数`，即不合并)
1. 手动为某一个osd执行分裂，使用工具`ceph-objectstore-tool`，找到一个issue见：https://tracker.ceph.com/issues/21366
2. 在新建pool的时候添加一个参数`expected_num_objects`，该pool所有相关的osd按照这个object总数去预先执行split

但是兜兜转转的，一直没能真正实现，我是直到jewel的10.2.11和Luminous的12.2.7才实现上面的第二种方法：建pool时提前分裂。（中间有些小版本没测试，看官方的release，在Luminous12.2.5已经修复这个问题：[pool create cmd's expected_num_objects is not correctly interpreted](http://tracker.ceph.com/issues/22530)）

## 实际测试
- 在global段设置如下参数：

```
filestore merge threshold = -10
```

- 重启所有服务：重启mon和osd服务
- 执行新建pool的命令，添加预期的object总数，我设置的是3亿，请根据实际修改

```
ceph osd pool create .rgw.buckets.data 16384 16384 replicated site1_sata_replicated_ruleset 300000000
```

- 查看效果

```
root@cld-osd1-48:/home/ceph/var/lib/osd/ceph-0/current/49.111b_head# ls DIR_B/DIR_1/DIR_1/
DIR_1  DIR_5  DIR_9  DIR_D
root@cld-osd1-48:/home/ceph/var/lib/osd/ceph-0/current/49.111b_head# ls DIR_B/DIR_1/DIR_1/DIR_1/
DIR_0  DIR_1  DIR_2  DIR_3  DIR_4  DIR_5  DIR_6  DIR_7	DIR_8  DIR_9  DIR_A  DIR_B  DIR_C  DIR_D  DIR_E  DIR_F
root@cld-osd1-48:/home/ceph/var/lib/osd/ceph-0/current/49.111b_head# ls DIR_B/DIR_1/DIR_1/DIR_1/DIR_1/
root@cld-osd1-48:/home/ceph/var/lib/osd/ceph-0/current/49.111b_head#
```

## 其他问题
1. 如果pg设置过小的话，很容易出现split，因为这意味着单pg下需要承载的object会更多，但是也不能为了防止分裂，将pg数设置过高。
2. `ceph-objectstore-tool`工具看起来是只能修改一个osd的分裂，没有继续测试是否生效
3. 碰到过偶发性的，部分osd分裂不完全，即一部分已经移到pg子目录了，一部分还是pg根目录，这时性能大大降低，该osd上会有大量的slow request。
4. 从L版开始这个问题看起来就不存在了，因为是使用裸设备直接存取。


------
> 个人浅见，不准确的地方欢迎指正。