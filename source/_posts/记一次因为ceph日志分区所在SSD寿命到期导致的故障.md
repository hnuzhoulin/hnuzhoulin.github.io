---
title: 记一次因为ceph日志分区所在SSD寿命到期导致的故障
date: 2016-04-23 15:51:00
categories: ceph #分类
tags: [ceph,故障处理,SSD] #文章标签，可空，多标签请用格式，注意:后面有个空格
description: 一次因为Ceph日志分区所在SSD故障，导致一个节点IO挂起卡住直至重启。本文记录整个分析过程。 #附加一段文章摘要，字数最好在140字以内。
---

## 1.环境
- openstack加ceph，计算存储放在一起，管理节点独立。
- 4个存储节点，每个节点12块3T的SATA硬盘，都做OSD
- sda是120G的Intel S3500既做操作系统分区又分出6个10G做日志
- sdb是120G的Intel S3500分6个10G分区只做日志
- 虚拟机大多是UbuntuKylinDesktop1404，作为开发测试虚拟机
- ceph集群分public和cluster两个网段，都是万兆
- ceph是0.87.9
- 3个mon分布在管理节点和2个存储节点

## 2.问题
某一天管理节点网络无法连接，直连存储节点node-63正常。在机房管理员修复网络后，用户反映桌面无法卡顿，查看集群状态发现node-65节点有OSD状态是down，但是此时node-65已经无法通过ssh连接，处于输入用户名密码后挂起状态。

## 3.思考及处理流程
### 3.1.IPMI远程查看node-65
有IPMI还是方便很多的，无需去机房就可以实现对服务器的所有操作。从IPMI上看到服务器报了一些错，同时点击鼠标键盘无反应，实地去机房接显示器和键盘输入用户名等待很久才提示输入密码，输入密码后一直卡在登陆界面。

> 此时判断应该是服务器出现某种错误导致挂起不能正常提供服务

### 3.2.重启服务器
经过上面的判断，基本上只能重启了，为了防止数据进行大规模迁移，执行下面的命令禁止数据进行backfill和recover：

```bash
ceph osd set nobackfill
ceph osd set norecover
```

重启服务器后能够正常登陆，但是负载很大。命令交互很大延迟。此时频繁出现某一个OSD重启的情况，查看所有OSD性能如下：

```bash
root@node-67:~# ceph osd perf
osdid fs_commit_latency(ms) fs_apply_latency(ms)
    0                  9819                10204
    1                     1                    2
    2                     1                    2
    3                  9126                 9496
    4                     2                    3
    5                     2                    3
    6                     0                    0
    7                     0                    1
    8                     1                    2
    9                 32960                33953
   10                     2                    3
   11                     1                   31
   12                 23450                24324
   13                     0                    1
   14                     1                    2
   15                 18882                19622
   16                     2                    4
   17                     1                    2
   18                     3                   46 
```

其中ID为3，9，12，15的OSD都在node-65上。且此时osd.6一直处于down状态。

> 此时感觉是node-65的负载问题拖累整个集群，但是由于osd.6又正常，因此尝试将延时太高的这4个节点的 `primary-affinity` 设为0。

### 3.3.修改primary-affinity为0.0
通过下面的命令修改延时高的4个OSD的primary-affinity为0.0：

```bash
ceph osd primary-affinity 3 0.0
ceph osd primary-affinity 9 0.0
ceph osd primary-affinity 12 0.0
ceph osd primary-affinity 15 0.0
```

但是实际上并没有什么效果，osd perf显示延时很大，看ceph日志写操作大都处于slow request状态。

> 实际上这一步只是当时瞎投医。因此尝试将这几个OSD的reweight调为0

### 3.4.调整延时大的OSD的reweight为0
使用下面的命令将OSD的reweight设为0：

```bash
ceph osd crush reweight 3 0.0
ceph osd crush reweight 15 0.0
ceph osd crush reweight 9 0.0
ceph osd crush reweight 12 0.0
```

由于此时虚拟机已经基本上都处于瘫痪状态，直接将backfill和recover的优先级、进程数调到最大。等到数据完全同步后，node-65的负载居然恢复正常了。（期间node-65上OSD.0也出现同样问题，再次调整reweight，等待同步）

> 此时来看，应该是OSD服务导致node-65负载过高，此时可以登陆node-65仔细检查。

### 3.5.登陆node-65检查
首先是通过smartctl、dmesg和syslog等查看上述OSD对应的硬盘，没有发现错误。其中OSD.6一直处于down是 **leveldb损坏** ，这个Sage说是目前基本无力回天，只能重建。

> 实际通过调整OSD的reweight后能够正常同步数据来看，OSD所在硬盘应该是没问题的。

通过atop发现node-65的sda一直处于 **busy** 状态，而此时sda仅仅是作为操作系统，因此使用smartctl查看sda的信息，找到了如下输出信息：

```bash
smartctl -a /dev/sda 
---
233 Media_Wearout_Indicator 0x0032   001   001   000    Old_age   Always
---
```

根据CephNote的博客[intel-520-ssd-journal](http://cephnotes.ksperis.com/blog/2015/05/19/intel-520-ssd-journal/)知道这个值表示的是SSD的寿命，输出表示当前SSD可用寿命为1%。

### 3.6.SSD寿命将近的表现
对于smartctl的值还是有所怀疑，于是在sda上运行fio测试，结果如下：

```bash
root@node-65:~# fio -direct=1 -bs=4k -ramp_time=40 -runtime=100
-size=20g -filename=./testfio.file -ioengine=libaio -iodepth=8
-norandommap -randrepeat=0 -time_based -rw=randwrite -name "osd.0 4k
randwrite test"
osd.0 4k randwrite test: (g=0): rw=randwrite, bs=4K-4K/4K-4K/4K-4K,
ioengine=libaio, iodepth=8
fio-2.1.10
Starting 1 process
osd.0 4k randwrite test: Laying out IO file(s) (1 file(s) / 20480MB)
Jobs: 1 (f=1): [w] [100.0% done] [0KB/252KB/0KB /s] [0/63/0 iops] [eta 00m:00s]
osd.0 4k randwrite test: (groupid=0, jobs=1): err= 0: pid=30071: Wed
Mar 30 13:38:27 2016
  write: io=79528KB, bw=814106B/s, iops=198, runt=100032msec
    slat (usec): min=5, max=1031.3K, avg=363.76, stdev=13260.26
    clat (usec): min=109, max=1325.7K, avg=39755.66, stdev=81798.27
     lat (msec): min=3, max=1325, avg=40.25, stdev=83.48
    clat percentiles (msec):
     |  1.00th=[   30],  5.00th=[   30], 10.00th=[   30], 20.00th=[   31],
     | 30.00th=[   31], 40.00th=[   31], 50.00th=[   31], 60.00th=[   36],
     | 70.00th=[   36], 80.00th=[   36], 90.00th=[   36], 95.00th=[   36],
     | 99.00th=[  165], 99.50th=[  799], 99.90th=[ 1221], 99.95th=[ 1237],
     | 99.99th=[ 1319]
    bw (KB  /s): min=    0, max= 1047, per=100.00%, avg=844.21, stdev=291.89
    lat (usec) : 250=0.01%
    lat (msec) : 4=0.02%, 10=0.14%, 20=0.31%, 50=98.13%, 100=0.25%
    lat (msec) : 250=0.23%, 500=0.16%, 750=0.24%, 1000=0.22%, 2000=0.34%
  cpu          : usr=0.18%, sys=1.27%, ctx=22838, majf=0, minf=27
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=143.7%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.1%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued    : total=r=0/w=19875/d=0, short=r=0/w=0/d=0
     latency   : target=0, window=0, percentile=100.00%, depth=8

Run status group 0 (all jobs):
  WRITE: io=79528KB, aggrb=795KB/s, minb=795KB/s, maxb=795KB/s,
mint=100032msec, maxt=100032msec

Disk stats (read/write):
  sda: ios=864/28988, merge=0/5738, ticks=31932/1061860,
in_queue=1093892, util=99.99%
```

看到这里的bw、iops和延时应该很清晰了，当然为了保险起见，我在其他节点运行类型测试，iops可以达到14223,也就是说SSD寿命在1%的时候跟普通SATA硬盘相当，而这样的性能对于既做操作系统又做6个OSD的日志来说负载肯定是相当大的。

### 3.7.替换sda
现在问题已经很清晰了，解决方案也是唯一的：替换sda。这里的操作是使用一个新的SSD，利用dd将数据拷贝到这个新的SSD中，再直接插入node-65原来sda的位置即可。

