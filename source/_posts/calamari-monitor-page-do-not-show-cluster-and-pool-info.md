---
title: calamari的监控页面没有集群和Pool信息
date: 2016/01/31 22:56:22
updated: 2016/02/01 00:06:41
categories: ceph
tags: [calamari]
description: calamari的监控页面看不到集群和Pool的信息，但是节点的监控信息可以看到的一种可能原因
---
## 一.问题：
### 1.dashboard没有IOPS和Usage信息
在calamari的dashboard页面，IOPS和Usage两项没有数据，如下图：
![](https://cloud.githubusercontent.com/assets/4092006/12389397/95791f32-be2c-11e5-8f08-14a3c7967376.png)
### 2.监控页面没有集群和Pools信息
在监控页面，能够看到节点的监控信息，但是cluster和Pools两大类里面没有监控数据，如下图：
![](https://cloud.githubusercontent.com/assets/4092006/12389269/c7afbeda-be2b-11e5-8e79-7e2eb6ab0691.png)
### 3.日志信息

```
2016-01-30 21:00:20,166 - metric_access - django.request No graphite data for ceph.cluster.52ae92c8-fd8e-47d8-9404-26051b344c23.pool.72.num_bytes
2016-01-30 21:00:20,179 - metric_access - django.request No graphite data for ceph.cluster.52ae92c8-fd8e-47d8-9404-26051b344c23.pool.73.num_objects
2016-01-30 21:00:20,190 - metric_access - django.request No graphite data for ceph.cluster.52ae92c8-fd8e-47d8-9404-26051b344c23.pool.73.num_bytes
2016-01-30 21:00:35,155 - metric_access - django.request No graphite data for ceph.cluster.52ae92c8-fd8e-47d8-9404-26051b344c23.df.total_used
2016-01-30 21:00:35,167 - metric_access - django.request No graphite data for ceph.cluster.52ae92c8-fd8e-47d8-9404-26051b344c23.df.total_space
2016-01-30 21:00:35,177 - metric_access - django.request No graphite data for ceph.cluster.52ae92c8-fd8e-47d8-9404-26051b344c23.df.total_avail
2016-01-30 21:00:39,503 - metric_access - django.request No graphite data for ceph.cluster.52ae92c8-fd8e-47d8-9404-26051b344c23.df.total_used
2016-01-30 21:00:39,522 - metric_access - django.request No graphite data for ceph.cluster.52ae92c8-fd8e-47d8-9404-26051b344c23.pool.0.num_objects
2016-01-30 21:00:39,526 - metric_access - django.request No graphite data for ceph.cluster.52ae92c8-fd8e-47d8-9404-26051b344c23.df.total_space
2016-01-30 21:00:39,549 - metric_access - django.request No graphite data for ceph.cluster.52ae92c8-fd8e-47d8-9404-26051b344c23.df.total_avail
2016-01-30 21:00:39,558 - metric_access - django.request No graphite data for ceph.cluster.52ae92c8-fd8e-47d8-9404-26051b344c23.pool.0.num_bytes
2016-01-30 21:00:39,579 - metric_access - django.request No graphite data for ceph.cluster.52ae92c8-fd8e-47d8-9404-26051b344c23.pool.1.num_objects
2016-01-30 21:00:39,601 - metric_access - django.request No graphite data for ceph.cluster.52ae92c8-fd8e-47d8-9404-26051b344c23.pool.1.num_bytes
2016-01-30 21:00:39,621 - metric_access - django.request No graphite data for ceph.cluster.52ae92c8-fd8e-47d8-9404-26051b344c23.pool.2.num_objects
2016-01-30 21:00:39,640 - metric_access - django.request No graphite data for ceph.cluster.52ae92c8-fd8e-47d8-9404-26051b344c23.pool.2.num_bytes
```
## 二.解决
有节点信息，但是没有集群的信息，如集群用量和pool信息等，以前是有的，经过和正常工作的环境对比，代码没有问题，查看源码，所看文件为`/usr/share/diamond/collectors/ceph/ceph.py`

```
    def _collect_cluster_stats(self, path):
        """
        If this service is a mon and it is the leader of a quorum, then
        publish statistics about the cluster.
        cluster_name, service_type, service_id = self._parse_socket_name(path)
        """
        if service_type != 'mon':
            return

        # We have a mon, see if it is the leader
        mon_status = self._admin_command(path, ['mon_status'])
        if mon_status['state'] != 'leader':
            return
        fsid = mon_status['monmap']['fsid']
        # We are the leader, gather cluster-wide statistics

        self.log.debug("mon leader found, gathering cluster stats for cluster '%s'" % cluster_name)

        def publish_pool_stats(pool_id, stats):
            # Some of these guys we treat as counters, some as gauges
            delta_fields = ['num_read', 'num_read_kb', 'num_write', 'num_write_kb', 'num_objects_recovered','num_bytes_recovered', 'num_keys_recovered']

            for k, v in stats.items():
                self._publish_cluster_stats(cluster_name, fsid, "pool.{0}".format(pool_id), {k: v},counter=k in delta_fields)

        # Gather "ceph pg dump pools" and file the stats by pool
        for pool in self._mon_command(cluster_name, ['pg', 'dump', 'pools']):
            publish_pool_stats(pool['poolid'], pool['stat_sum'])
        all_pools_stats = self._mon_command(cluster_name, ['pg', 'dump', 'summary'])['pg_stats_sum']['stat_sum']

        publish_pool_stats('all', all_pools_stats)  
        # Gather "ceph df"
        df = self._mon_command(cluster_name, ['df'])
        self._publish_cluster_stats(cluster_name, fsid, "df", df['stats'])
```

由上可知，集群的信息是通过 `leader mon` 获取的，尝试在存储节点上执行上述mon_status命令发现，集群的三个mon节点都没有asok文件，于是乎**kill调原mon进程，使用service命令重启mon**.

> 这里直接重启服务不一定会生效

------

**备注：** 图片引用自calamari的这个[issue](https://github.com/ceph/calamari/issues/384)