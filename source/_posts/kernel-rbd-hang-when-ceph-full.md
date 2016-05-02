---
title: 因为ceph波动导致kernel rbd挂起
date: 2016/04/18 10:44:27
updated: 2016/04/18 10:44:27
categories: ceph
tags: [排错,rbd,kernel]
description: 使用kernel rbd时由于 ceph full后导致的操作挂起的解决方式
---
## 1.问题
有一个ceph环境的使用方式如下：

- 在某一个ceph节点上map rbd
- 格式化该rbd
- mount该rbd到本地目录A
- 通过samba共享该目录A

因为集群少数OSD达到full状态，所有通过samba共享出去的目录在客户端节点上都出现不同程度的问题：某些目录可以访问以前的数据，最新的数据无法访问；某些客户端df看不到挂载的samba共享；有些可以看到共享目录但是cd进去提示设备不存在。。。。

## 2.解决过程
### 2.1.解决OSD full问题
这是首当其冲需要解决的问题，关于这个的 解决我不再赘述，参见我以前的博文。这里只记录一下解决过程：

- 首先使用 `ceph osd reweight-by-utilization` 自动调节，将用量85%以上的3歌osd进行了调整
- 上述调整数据同步完成后，又有新的osd达到87%，再次调整 `ceph osd reweight-by-utilization 115` ,注意这里加了一个参数115，又再次调整了两个osd
- 同步完成后又有一个osd用量达到83%，虽然这个值还没有达到ceph的默认告警值，但是我还是手动通过 `ceph osd reweight 39 0.95` 调整了一下 

到此集群恢复正常，但是此时samba共享依然无法访问

### 2.2.查找共享不能读取原因
目前客户那边反馈的情况在第一小节中有提到。而我同样在ceph节点进行了下面的测试：

- 直接进入rbd挂载的本地目录，检查是否能够正常查看文件
- 在其他节点再次mount samba共享验证samba通道是否正常

仅仅针对客户反馈过来的几个共享进行检查，发现结果都是历史数据是可以获取到的，但是要想获取实时数据比如使用quota查看限额信息等都是直接挂起不响应，命令无法通过 `Ctrl+C` 取消。于是通过 `ps aux|grep quota` 命令检查发现进程状态处于 **D** ，进一步查看samba进程的状态，天啦噜，居然绝大部分都是处于D状态。然后通过 `sambastatus -u testuser` 查看testuser用户的samba状态，发现有差不多522个进程处于D状态。

因此问题应该就是因为存储full后停止写导致rbd挂起，进一步的导致samba进程一直处于同步等待IO资源深度睡眠的状态（即ps中显示的D状态）。进一步了解这个D状态发现最好的方法就是重启。

### 2.3.解决rbd hang
对于线上环境来说，重启是最终不得已的方法，但是业务无法使用又是等不起的。嘴周只能一边协调准备重启事宜，一边在疯狂搜索ceph的邮件列表。当然由于搜索所用关键词不准确，并没有搜到有用的。（本文标题的几个词是后面根据邮件列表的标题重新整理的关键词）无奈之下只能直接发问了。好在有一个国外的网友及时回复，在他的指导下找到了根本解决办法。

- 在rbd挂载的节点上执行：`/sys/kernel/debug/ceph -type f -print -exec cat {} \; `
- 文件 `/sys/kernel/debug/ceph/***/osdc` 是否为空，不为空的话多次查看是否内容不变
- 使用 `ceph osd down 37` 重启上述osdc文件中列到的osd
- 观察集群状态，osd恢复后再同样的方法重启下一个，直到所有的都完成。

这里的目录 `/sys/kernel/debug/ceph` 的具体作用的话可以看seb的博客：[See What the Ceph Client Sees](http://www.sebastien-han.fr/blog/2015/07/08/see-what-the-ceph-client-sees/)

**说明：** 这里的这个目录只有通过kernel rbd模块挂载rbd才会生成。

## 3.参考文档
- 关于我提交到邮件列表的提问：[directory hang which mount from a mapped rbd](https://www.mail-archive.com/ceph-users%40lists.ceph.com/msg28454.html)
- Seb的博文：[See What the Ceph Client Sees](http://www.sebastien-han.fr/blog/2015/07/08/see-what-the-ceph-client-sees/)
- [Kernel RBD hang on OSD Failure](https://www.mail-archive.com/ceph-users%40lists.ceph.com/msg25418.html)