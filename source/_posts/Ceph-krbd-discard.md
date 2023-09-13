---
title: Ceph-krbd-discard
tags: [翻译,krbd]
top: false
date: 2016-06-11 10:26:42
updated: 2016-06-16 09:11:42
categories: ceph
description: Ceph的rbd块是精简制备的，只有真正有数据才会占用空间，但是存在上层文件系统删除文件，但是rbd占用空间不变的问题。Ceph的krbd模块的discard支持可以解决这个问题，实现前后端空间一致。上层文件系统删除文件后，ceph会清理底层对应的对象。
---

本文英文出处：[Sébastien Han/ceph-and-krbd-discard](http://www.sebastien-han.fr/blog/2015/01/26/ceph-and-krbd-discard/) 

内核RBD模块的空间回收机制。对于运营商而言，有这样的支持是至关重要的，因为这能减轻你的容量规划。RBD镜像是稀疏的，因此创建后大小等于0 MB。稀疏镜像的主要问题在于增长，最终达到整个镜像的大小。问题是这是发生在块层面之上尤其是其上有文件系统的时候Ceph并不知道发生了什么。你可以很容易地向整个文件系统写入数据，然后删除一切，但是Ceph仍然相信这些块仍然被使用着，并持续度量。然而多亏在块设备层面支持了 discard，文件系统可以发送 discard flush 命令给块。最后，存储系统将释放这些上层已经删除的块。

这个特性自3.18版本的内核开始引入。

我们先新建一个 RBD 镜像

```bash
$ rbd create -s 10240 leseb
$ rbd info leseb
rbd image 'leseb':
  size 10240 MB in 2560 objects
  order 22 (4096 kB objects)
  block_name_prefix: rb.0.1066.74b0dc51
  format: 1

$ rbd diff rbd/leseb | awk '{ SUM += $2 } END { print SUM/1024/1024 " MB" }'
0 MB
```

将它映射到一个节点并在其上构建一个文件系统:

```bash
$ sudo rbd -p rbd map leseb
/dev/rbd0

$ sudo rbd showmapped
id pool image snap device
0  rbd  leseb -    /dev/rbd0

$ sudo mkfs.xfs /dev/rbd0
log stripe unit (4194304 bytes) is too large (maximum is 256KiB)
log stripe unit adjusted to 32KiB
meta-data=/dev/rbd0              isize=256    agcount=17, agsize=162816 blks
         =                       sectsz=512   attr=2, projid32bit=0
data     =                       bsize=4096   blocks=2621440, imaxpct=25
         =                       sunit=1024   swidth=1024 blks
naming   =version 2              bsize=4096   ascii-ci=0
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=8 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0

$ sudo mount /dev/rbd0 /mnt
```

到这里所有的设置都已经完成，我们开始写入一些数据:

```bash
$ dd if=/dev/zero of=/mnt/leseb bs=1M count=128
128+0 records in
128+0 records out
134217728 bytes (134 MB) copied, 2.88215 s, 46.6 MB/s

$ df -h /mnt/
Filesystem      Size  Used Avail Use% Mounted on
/dev/rbd0        10G  161M  9.9G   2% /mnt
```

然后我们再次检测这个镜像占用的空间大小:

```bash
$ rbd diff rbd/leseb | awk '{ SUM += $2 } END { print SUM/1024/1024 " MB" }'
142.406 MB
```

我们知道这里我们实际的数据是 128MB，文件系统的数据/元数据占用了约 ~14,406MB。检查这个份设备的 discard 属性已经启用:

```bash
root@ceph-mon0:~# cat /sys/block/rbd0/queue/discard_*
4194304
4194304
1
```

现在让我们来检查当 discard 不支持时的默认行为，我们删除前面新建的所有 128MB 的文件以释放文件系统的一些空间。不幸的是Ceph什么也没检测到，仍然认为这 128MB 的数据仍然存在。

```bash
$ rm /mnt/leseb
$ rbd diff rbd/leseb | awk '{ SUM += $2 } END { print SUM/1024/1024 " MB" }'
142.406 MB
```

现在让我们在挂载的文件系统上运行 fstrim 命令以指示块设备释放未使用的空间:

```bash
$ fstrim /mnt/
$ rbd diff rbd/leseb | awk '{ SUM += $2 } END { print SUM/1024/1024 " MB" }'
10.6406 MB
```

Et voilà ! Ceph 释放了我们删除的 128 MB。

如果你想自动运行 discard，可以在挂载文件系统的时候加上 discard 选项，这样就可以让文件系统一直检测 discard:

```bash
$ mount -o discard /dev/rbd0 /mnt/

$ mount | grep rbd
/dev/rbd0 on /mnt type xfs (rw,discard)
```

> 注意，使用 discard 的挂载选项会严重影响性能。所以，一般你可以通过一个每天与星的 cron 任务来触发 fstrim 命令。


## 译者补充：
我的实际线上环境是：ubuntu12.04+Ceph 0.67.9

同时对于使用 KRBD 的都有感触，要保障业务不断的同时进行升级是相当麻烦的，但是这个存储系统已经稳定运行两年，Ceph检测到已用空间到了70%，但是实际业务系统层面统计还不到50%，因此执行 fstrim 势在必行。

经过本文作者的提示，我们的方案是在这个集群上使用kvm新建一个 ubuntu14.04.4 的虚拟机，依次将所有 RBD 在该虚拟机上映射后挂载到某一个目录，然后在这个目录上执行 fstrim 命令，达到清理的目的。

这个方案有需要注意：

推荐的步骤是原挂载点先 **umount** ，再在该虚拟机上mount再执行fstrim。（原来的做法没有在原挂载点先umount，而是同一个 RBD 在两个不同节点映射并挂载，这样做存在新写入数据丢失的风险，一部分块被标记删除但是此时又有新数据写入就会有这种风险，再具体的需要对rbd的fstrim的机制进一步深入分析才能确认，再次感谢微信群 **Ceph中国社区**  的大牛 陈晓熹 的指点。；
