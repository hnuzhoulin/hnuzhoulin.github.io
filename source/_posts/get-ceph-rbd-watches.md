---
title: ceph下查看使用rbd的客户端
date: 2016/02/03 14:51:33
updated: 2016/02/03 14:55:40
categories: ceph
tags: [rbd]
description: 查看ceph的rbd两种使用方式下的客户端
---
## 一.背景
不知道挂载点这个表述是否准确。真实的需求是有个时候删除rbd块会提示当前rbd正在使用中，如法删除。对于linux下是文件或者目录可以通过`lsof`查看当前文件或目录的使用进程。对于ceph也希望有一个合适的方式查看当前rbd块在哪个主机上被使用。

## 二.解决方法
对于rbd块的使用，目前主要有两种形式：内核模块map后再mount和通过libvirt等供给虚拟机使用。两种方式不太一样。

### 2.1 内核模块map
查看方法如下：

```
##查看rbd池下的rbd
root@node-62:~# rbd ls
gci

##查看gci的挂载点
root@node-62:~# rados -p rbd listwatchers gci.rbd
watcher=10.1.41.205:0/1642395845 client.5493237 cookie=2

##去挂载点检查
root@node-67:~# rbd showmapped |grep gci
2  rbd  gci          -    /dev/rbd2  
```

### 2.1 虚拟机通过libvirt使用rbd
查看方法如下：
```
##查看该虚拟机卷的信息
root@node-67:~# rbd info volume-997d91e8-40f3-4ede-affb-8b61775fccde -p volumes
rbd image 'volume-997d91e8-40f3-4ede-affb-8b61775fccde':
        size 300 GB in 76800 objects
        order 22 (4096 kB objects)
        block_name_prefix: rbd_data.10475c6f9a52
        format: 2
        features: layering
        parent: volumes/volume-4461e760-8c3d-4e7f-9537-c27b301a74c2@snapshot-d17bc874-c589-4118-b6af-058f1c6e3a02
        overlap: 300 GB
        
##查看该卷的挂载点
root@node-67:~# rados -p volumes listwatchers rbd_header.10475c6f9a52
watcher=10.1.41.201:0/1001261 client.4449991 cookie=1

## 通过nova show验证该卷是否在物理机10.1.41.201(node-63)上
root@node-67:~# nova show zhoulin_UbuntuKylin_0824
+--------------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------+
| Property                             | Value                                                                                                                                            |
+--------------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------+
| OS-DCF:diskConfig                    | AUTO                                                                                                                                             |
| OS-EXT-AZ:availability_zone          | nova                                                                                                                                             |
| OS-EXT-SRV-ATTR:host                 | node-63                                                                                                                                          |
| OS-EXT-SRV-ATTR:hypervisor_hostname  | node-63                                                                                                                                          |
| OS-EXT-SRV-ATTR:instance_name        | instance-00000086                                                                                                                                |
| OS-EXT-STS:power_state               | 1                                                                                                                                                |
| OS-EXT-STS:task_state                | -                                                                                                                                                |
| OS-EXT-STS:vm_state                  | active                                                                                                                                           |
| OS-SRV-USG:launched_at               | 2015-08-24T02:14:34.000000                                                                                                                       |
| OS-SRV-USG:terminated_at             | -                                                                                                                                                |
| accessIPv4                           |                                                                                                                                                  |
| accessIPv6                           |                                                                                                                                                  |
| config_drive                         |                                                                                                                                                  |
| created                              | 2015-08-24T02:14:28Z                                                                                                                             |
| flavor                               | m1.large (903620b6-e4e2-49a7-bcca-38139b9d4e30)                                                                                                  |
| hostId                               | 73f6a8472a61da98b5ecbf7a5b7a94791991bdc69a94f3fe984f58dc                                                                                         |
| id                                   | 3b5a16f5-b2a3-4e74-95c0-bd3e2114016a                                                                                                             |
| image                                | Attempt to boot from volume - no image supplied                                                                                                  |
| key_name                             | ubuntu                                                                                                                                           |
| metadata                             | {}                                                                                                                                               |
| name                                 | zhoulin_UbuntuKylin_0824                                                                                                                         |
| novanetwork network                  |                                                                                                                                                  |
| os-extended-volumes:volumes_attached | [{"id": "997d91e8-40f3-4ede-affb-8b61775fccde"}, {"id": "5f9ba54a-c7e1-441a-b28c-180995accec1"}, {"id": "8256288a-fca8-4d70-9d8e-f1a0a7821a1a"}] |
| progress                             | 0                                                                                                                                                |
| security_groups                      | default                                                                                                                                          |
| status                               | ACTIVE                                                                                                                                           |
| tenant_id                            | 26d260968b1e4a8caa9356aafff090e6                                                                                                                 |
| updated                              | 2016-01-31T01:51:12Z                                                                                                                             |
| user_id                              | c18a490fffb846fc9d52aa62b4d07ae2                                                                                                                 |
+--------------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------+
                     
```

## 三.总结
基本都是利用`rados listwatchers`去查看，只是两种方式需要读取的文件不一样。