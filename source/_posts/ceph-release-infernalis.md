---
title: Ceph V9.2.0版本(代号INFERNALIS)已发布
date: 2015/11/24 13:05:15
updated: 2015/12/28 15:40:05
categories: ceph
tags: [infernalis,release]
description: ceph的infernalis版更新记录
---

本文由Ceph中国社区-AmosG翻译 Thomas校稿

英文出处：[官网releases栏](http://ceph.com/releases/v9-2-0-infernalis-released/) 欢迎加入[Ceph中国社区翻译组](http://7xj5dz.com1.z0.glb.clouddn.com/qun.png)



![](http://ceph.com/release_images/infernalis-460x148.png)

此次的主发版本将是Ceph下一个稳定系列的基石。自Hammer v0.94.x发布后，我们做了重大修改，这个更新过程很不简单。请仔细阅读以下的发行说明。

## 自Hammer版本以来所做的重要修改 ##

- **General**
	- Ceph的守护进程现已通过systemd来管理(Ubuntu Trusty是个例外，仍沿用upstart来管理)
	- Ceph的守护进程以ceph用户而非root运行
	- 在Red Hat的发行版本中，也存在SeLinux Policy
- **RADOS**
	- RADOS的cache层现在可通过代理将写操作发送到持久层(base tier)，允许直接处理写操作无需先强制将对象移至cache层
	- 对SHEC 纠删码的支持已不再标记为实验性的。SHEC通过消耗一些额外的存储空间来换取更快的修复速度。
	- 为（优化）客户端IO、数据恢复、数据清理、快照裁剪提供统一的队列。
	- 对低层级的修复工具（ceph-objectstore-tool）做了许多改进
	- 为方便使用新的存储后端（如NewStore），内部ObjectStore API已做了重大的修改
- **RGW**
	- Swift API现已支持对象过期机制
	- Swift API的兼容性方面做出了很大改善
- **RBD**
	- 通过rbd du命令显示实际的使用（容量）信息（当object-map处于enabled状态时该操作很快）
	- 对object-map特性的稳定性做了许多改善。
	- object-map和exclusive-lock特性可动态的开启或者关闭。
	- 现在你可以依据个体镜像(individual image)来存储用户的元数据并设置持久的librbd选项。
	- 新的深度扁平（deep-flatten）特性允许对一个克隆及其所有的快照进行扁平化处理（在此之前快照不能被扁平化。）
	- 更快的export-diff命令（使用了aio）。另外新增了fast-diff特性。
	- 可通过单位后缀指定-size参数大小（如--size 64G）。
    - 有一个新的rbd status命令，现在可通过该命令来显示谁打开/映射了某个镜像。
- **CephFS**
	- 现已可对快照进行重命名操作
	- 在管理、诊断、检查和修复工具上已作出了持续的改善
	- 32位主机上，ceph-fuse客户端表现得更好

## 发行版的兼容性 ##

我们已决定放弃对多个旧的Linux发行版的支持，这样一来我们就可以转用更新的编译器工具链（如C++11）。尽管通过安装向后兼容的开发工具来在旧版本中生成Ceph还是可能的，但我们不会在ceph.com上为这些发行版生成并发布release版本的安装包。

我们现在为以下的版本生成安装包：

- CentOS 7或更高的版本。我们已放弃对CentOS 6（以及其它RHEL 6的衍生系列，如Scientific Linux 6）的支持
- Debian Jessie 8.x或后续版本。 Debian Wheezy 7.x的g++对C++11的支持并不完整（而且没有systemd）
- Ubuntu Trusty 14.04或后续版本版本。 Ubuntu Precise 12.04已不再受支持
- Fedora 22或后续版本版本

## 从Firefly更新至Infernalis ##

我们不推荐从Firefly v0.80.z直接更新。虽然直接更新是可能的，但会存在停机时间。我们推荐先更新至Hammer v0.94.4或之后的v0.94.z的发行版，只有那样才能做到线上更新至Infernalis 9.2.z(见下文)。

若需线下直接从Firefly版更新至Infernalis版，那么在任一Infernalis OSD允许启动之前，必须停止所有的Firefly OSD并将其标记为down状态。通过Infernalis的监控器来保证这种隔离机制，因此，使用类似如下的升级步骤

1. 在monitor主机上更新Ceph
2. 重启所有的ceph-mon守护进程
3. 在所有的OSD主机上更新Ceph
4. 停止所有的ceph-osd守护进程
5. 将所有的OSD标记为down，所用的执行命令类似于：`ceph osd down seq 0 1000`
6. 重启所有的ceph-osd守护进程
7. 升级并重启余下的守护进程(ceph-mds, radosgw)

## 从Hammer更新至Infernalis ##

- 对支持systemd(CentOS 7, Fedora, Debian Jessie 8.x, OpenSUSE)的所有Linux发行版而言，Ceph守护进程现在使用原生的systemd文件而不是遗留的sysvinit脚本来进行管理。比如：
```
	systemctl start ceph.target   # start all daemons
	systemctl start ceph.target   # start all daemons
```
值得注意的是，主流的发行版中，Ubuntu trusty 14.04还未使用systemd。下一个Ubuntu LTS（即16.04）中，将会使用systemd替换upstart。

- 现在，默认情况下，Ceph守护进程将以ceph用户和组的身份运行。 在Fedora和Debian（以及其他诸如RHEL/CentOS和Ubuntu等衍生发行版）中ceph用户被赋予了一个静态UID。在SUSE中ceph用户会在其创建的时候被动态的赋予一个UID。

	如果你的系统中已有ceph用户，那么升级安装包的过程中将会出现问题。我们建议在升级前，先移除或重命名已有的ceph用户或ceph组。

	升级过程中，管理员有两个选择：
	
	1. 将如下行添加至所有主机的ceph.conf中：
		`setuser match path = /var/lib/ceph/$type/$cluster-$id`
		
		这将使得Ceph的守护进程以root身份运行（即：未放弃特权及切换至ceph用户），如果Ceph守护进程的数据目录的所有者仍是root。新部署的守护进程将以ceph用户来创建数据目录，并以非特权运行，但升级的守护进程仍以root身份运行。
	2. 升级过程中修复数据的所有权。这是我们所偏好的选择，但这个选择工作量大而且耗时。对每个主机，需做如下步骤的操作：
	
		1). 升级ceph安装包。这就创建了ceph用户和组。如：
	   ```
   		ceph-deploy install --stable infernalis HOST
	   ```
		2). 停止守护进程
	   ```
		service ceph stop           # fedora, centos, rhel, debian
		stop ceph-all               # ubuntu
	   ```
		3). 修复所有权
		```
		chown -R ceph:ceph /var/lib/ceph
		```
		4). 重启守护进程
		```
		start ceph-all                # ubuntu
		systemctl start ceph.target   # debian, centos, fedora, rhel
		```
		另一个可行的方法是:相同的过程可以用单一的守护进程类型来完成，比如：只停止监控器并改变/var/lib/ceph/mon的所有者。

- 处于实验阶段的KeyValueStore OSD后端所采用的磁盘格式(on-disk format)已发生了变化。如果你所要升级的测试集群中用到了它，那么你需要移除所有用到该后端的OSD。
- 与集群满一样，当达到存储池的配额时，librados操作将会无限期阻塞。（而之前的版本中会返回-ENOSPC）。默认状态下，当一个集群或pool满了，就会发生阻塞。如果你的librados应用能优雅地处理ENOSPC或EDQUOT等错误，那么你可以通过使用lirados中新的OPERATION_FULL_TRY标记来获取错误返回码。
- librbd的rbd_aio_read和Image::aio_read API方法在成功时将不再返回已读的字节数，而是在成功时返回0，失败时返回一个负值。
- `ceph scrub`, `ceph compact` 和 `ceph sync force` 已弃用，取而代之的是 ‘ceph mon scrub’, ‘ceph mon compact’ 和 ‘ceph mon sync force’。
- `ceph mon_metadata` 现在应写成 `ceph mon metadata`。没有必要弃用这个命令（自引入以来一直存在）。
- osdmaptool命令的–dump-json已由–dump json替换。
- pg ls-by-{pool,primary,osd} 和 pg ls 这两命令中的recovery参数改为了recovering, 用来包含所列出的pg中正处于recovering状态的pg。


更多变化内容请详见[发行说明](http://ceph.com/releases/v9-2-0-infernalis-released/)


转载自[cph中国社区](http://bbs.ceph.org.cn/article/20)