---
title: 将Ceph集群从Ubuntu迁移到RHEL
date: 2016-05-28 14:44:48
categories: ceph
tags: [翻译,tool]
description: 将基于Ubuntu的Ceph集群迁移到RHEL系统，整个过程在线热迁移，不进行存储数据的手动迁移，不影响集群已连接的客户端的数据访问。
---
英文出处：[http://www.sebastien-han.fr/blog/2016/03/07/migrate-ceph-cluster-from-one-distro-to-another/) 

欢迎加入 [Ceph中国社区翻译小组](http://7xj5dz.com1.z0.glb.clouddn.com/qun.png)

![migrate-ceph-cluster-from-ubuntu-to-rhel.jpg](http://sebastien-han.fr/images/migrate-ceph-cluster-from-ubuntu-to-rhel.jpg)

最近我经历的一个使用案例是将基于Ubuntu的Ceph集群迁移到RHEL系统。对此我们有严格的要求，不希望有任何数据迁移。这是Ceph特别是OSD的另一种美妙之处，那就是基本上他们能够在任何机器上运行。假设你有一个OSD，你可以拔出磁盘并无缝地插入另一台机器上。这里采取的方法有点不同，但依赖于这种能力。

我使用Ansible构建自动化。过程相当简单，可以解耦为下列步骤。请注意所有任务都是串行的。

对于monitors:

- 压缩monitor store
- 打包monitor store
- 拷贝上面一步打好的包到Ansible服务器
- 停止monitor服务
- 关闭服务器并 **使用你自己的方法** 安装RHEL，这里playbook只会执行 `重启` 
- 服务器安装完成并启动后，拷贝上面打好的包到这个新的系统上( **确保已经安装好相同版本的CEph** )
- 解包并将文件拷回原来的目录
- 启动monitor服务，并等待跟其他mon之间同步好数据
- 跟对剩余的monitors重复以上步骤

对于OSDs:

- 设置 `noout` 标记，这样就不会触发recovery
- 打包OSD bootstrap key 和 `ceph.conf`
- 拷贝这个打包文件到Ansible服务器
- 停止OSDs服务
- 关闭服务器并 **使用你自己的方法** 安装RHEL，这里playbook只会执行 `重启` 
- 服务器安装完成并启动后，拷贝上面打好的包到这个新的系统上( **确保已经安装好相同版本的Ceph** )
- 解包并将文件拷回原来的目录
- 启动OSD并等待PG的状态变为clean，在它们的状态都变成 `active+clean` 之后开始针对其他OSD节点执行上面的步骤

在测试环境，你只需要下面的步骤：

```bash
$ git clone https://github.com/ceph/ceph-ansible.git
$ cd ceph-ansible
$ ansible-playbook cluster-os-migration.yml
```

>愉快的迁移之旅！最后再提一句，整个过程不触发任何数据迁移，所以这些操作都能够在线执行，逐步推进，并且不会影响已连接的客户端。