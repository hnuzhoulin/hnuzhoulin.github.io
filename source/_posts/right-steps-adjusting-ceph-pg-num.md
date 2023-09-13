---
title: 调整pg数量的步骤
date: 2016/01/06 16:59:18
updated: 2016/03/26 11:18:06
categories: ceph
tags: [排错,pg]
description: 调整pg数量的步骤
---
# 1.计算合适的pg数
关于pg数值的合理值的计算参考 `http://ceph.com/pgcalc/` 。但是请谨记，在你真正还是调整pg前，请确保集群状态是健康的。 

# 2.调整前确保状态ok
如果 `ceph -s` 命令显示的集群状态是OK的，此时就可以动态的增大pg的值。

**注意：** 增大pg有几个步骤，同时必须比较平滑的增大，不能一次性调的太猛。对于生产环境格外注意。

一次性的将pg跳到一个很大的值会导致集群大规模的数据平衡，因而可能导致集群出现问题而临时不可用。

# 3.调整数据同步参数，减少数据同步时对业务的影响
当调整 PG/PGP 的值时，会引发ceph集群的 `backfill` 操作，数据会以最快的数据进行平衡，因此可能导致集群不稳定。 因此首先设置 backfill ratio 到一个比较小的值，通过下面的命令设置：

    ceph tell osd.* injectargs '--osd-max-backfills 1'
    ceph tell osd.* injectargs '--osd-recovery-max-active 1'
    
还包括下面这些：

    osd_backfill_scan_min = 4 
    osd_backfill_scan_max = 32 
    osd recovery threads = 1 
    osd recovery op priority = 1 

# 4.首先调整pg
然后，先增长pg的值，我们推荐的增长幅度是按照 **2的幂** 进行增长，如原来是 **64**个，第一次调整先增加到**128**个， 设置命令如下：    

    ceph osd pool set <poolname> pg_num <new_pgnum>

你可以通过 `ceph -w` 查看集群的变化。

# 5.等到状态再次恢复正常再调整pgp
在pg增长之后，通过下面的命令增加pgp，pgp的值需要与pg的值一致： 

     ceph osd pool set <poolname> pgp_num <new_pgnum>

此时，通过 `ceph -w` 可以看到集群状态的详细信息，可以看到数据的再平衡过程。

# 6.关于pg和pgp的区别在另外的文章中再详述

# 7.关于pg分裂的详细讲解见有云官方博客
[Ceph pg分裂流程及可行性分析](https://www.ustack.com/blog/ceph-pg-fenlie/)
 