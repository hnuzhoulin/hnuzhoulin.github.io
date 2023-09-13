---
title: 记一次日志所在SSD故障导致的object unfound
date: 2016/04/20 14:43:57
updated: 2016/04/23 17:18:35
categories: ceph
tags: [排错,ssd]
description: 记一次日志所在SSD故障，继而导致导致ceph故障，同时不正确的处理流程导致的object unfound
---
## 1.问题
节点node-65因为其中一个既做系统又做日志分区的SSD寿命到期性能急剧下降导致对应OSD延时超级大，且有OSD无法启动，同时，整个过程依然有虚拟机在对集群进行读写。

在成功通过先调整reweight为0给SSD减压最后替换这个SSD再恢复reweight之后，有一个object 处于unfound状态，同时导致其所在的pg整个处于backfilling状态

## 2.处理流程及总结
### 2.1.pg以及object相关信息
该pg的信息如下：

```bash
ceph pg map 4.438
osdmap e42522 pg 4.438 (4.438) -> up [34,20,30] acting [7,11]

ceph pg 4.438 query
{ "state": "active+recovering+degraded+remapped",
  "epoch": 42479,
  "up": [
        34,
        20,
        30],
  "acting": [
        7,
        11],
  "backfill_targets": [
        "20",
        "30",
        "34"],
  "actingbackfill": [
        "7",
        "11",
        "20",
        "30",
        "34"],
---
"recovery_state": [
        { "name": "Started\/Primary\/Active",
          "enter_time": "2016-03-19 10:24:50.186833",
          "might_have_unfound": [
                { "osd": "6",
                  "status": "osd is down"},
                { "osd": "11",
                  "status": "already probed"},
                { "osd": "20",
                  "status": "already probed"},
                { "osd": "30",
                  "status": "already probed"},
                { "osd": "31",
                  "status": "already probed"},
                { "osd": "34",
                  "status": "already probed"}],
          "recovery_progress": { "backfill_targets": [
                    "20",
                    "30",
                    "34"],
              "waiting_on_backfill": [],
              "last_backfill_started": "0\/\/0\/\/-1",
              "backfill_info": { "begin": "0\/\/0\/\/-1",
                  "end": "0\/\/0\/\/-1",
                  "objects": []},
              "peer_backfill_info": [],
              "backfills_in_flight": [],
              "recovering": [],
              "pg_backend": { "pull_from_peer": [],
                  "pushing": []}},
          "scrub": { "scrubber.epoch_start": "0",
              "scrubber.active": 0,
              "scrubber.block_writes": 0,
              "scrubber.finalizing": 0,
              "scrubber.waiting_on": 0,
              "scrubber.waiting_on_whom": []}},
        { "name": "Started",
          "enter_time": "2016-03-19 10:24:49.182580"}],
  "agent_state": {}}
```

居然up和scting 不一致，应该是调整了osd 的crush reweight导致的。

ubfound object的信息如下：

```bash
ceph pg 4.438 list_missing
{ "offset": { "oid": "",
      "key": "",
      "snapid": 0,
      "hash": 0,
      "max": 0,
      "pool": -1,
      "namespace": ""},
  "num_missing": 1,
  "num_unfound": 1,
  "objects": [
        { "oid": { "oid": "rbd_data.188b9163c78e9.00000000000015f2",
              "key": "",
              "snapid": -2,
              "hash": 2427198520,
              "max": 0,
              "pool": 4,
              "namespace": ""},
          "need": "39188'2314230",
          "have": "39174'2314229",
          "locations": []}],
```

### 2.2.使用旧版或者直接删除该object
按照ceph官网的troubleshoot手册，对于这种问题，可以直接告诉ceph使用已有的旧版本或者直接删除：

```bash
root at node-67:~# ceph pg 4.438 mark_unfound_lost revert
Error EINVAL: pg has 1 unfound objects but we haven't probed all sources,not marking lost

root at node-67:~# ceph pg 4.438 mark_unfound_lost delete
Error EINVAL: pg has 1 unfound objects but we haven't probed all sources,not marking lost
```

看信息貌似是该object的一个副本没有准备好且不处于lost状态，但是实际情况是osd.6因为leveldb故障已经被踢出集群。

> **注意：** 从这里得到的经验是只有在确认该osd没有用之后再使用删除操作，要实现 **迁移数据和该osd不对外提供服务** 的方法最好是使用调整crush reweight的方法。

### 2.3.尝试使用ceph-objectstore-tool
在邮件列表上看到使用ceph-objectstore-tool可以将pg数据导出来在导入集群，因为osd.6的数据分区是依然保存完好的。但是目前使用的ceph版本的ceph-objectstore-tool不支持pg导入导出，F版必须是0.80.11版才有，单独下载ceph-test安装包也因为依赖无法完成安装。（目前集群状态不正常，不敢直接升级到0.80.11）

期间将osd.6的硬盘插入别的服务器中，使用0.80.11版本的ceph-test包中的ceph-objectstore-tool工具成功将gaipg数据导出，但是不能导入也没用。

### 2.4.重建osd.6
仔细检查前面的所有输出，发现之所以不能通过`ceph pg 4.438 mark_unfound_lost revert`恢复object是因为还有osd.6不处于可用状态或者标记为lost状态。因此想到一个折中的方法，我重新加入一个osd.6是不是可行呢？

根据这个思路，为了保证原来的osd.6的数据以备不时之需，换一个新硬盘插到osd.6的位置作为新的osd.6，使用新下面的步骤操作：

- 使用ceph-deploy添加osd，会自动使用目前缺失的6作为序号
- 会进行数据同步，查看ceph -w日志
- 在处理到pg4.438时，日志报告有object丢失，但是不影响该pg其他数据同步
- 执行 `ceph pg 4.438 mark_unfound_lost revert` 
- 再次同步
- 集群OK

## 3.总结
从最近的几次集群故障来看，ceph还是比较健壮的，可能遇到的问题和故障都已经有相应的解决工具（这个应该得益于社区活跃）和方法，我们只需要为这些工具和方法准备所需的前提条件就可以。