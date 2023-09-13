---
title: using-libvirt-create-vm-combine-rbd
tags: [rbd libvirt qemu]
top: false
date: 2018-04-15 11:25:43
updated: 2018-04-15 11:25:43
categories: ceph
description: 使用libvirt创建虚拟机，底层存储直接对接ceph，整个过程全流程介绍
---

## 前情提要
在一台没有桌面环境的远程服务器上需要新建虚拟机测试ceph，中间涉及桥接网络的配置、使用ISO镜像安装系统的xml文件、

## 操作流程
```shell
# 确保qemu，libvirt相关的包已经安装，

## 一步，准备iso镜像，准备硬盘
qemu-img create -f raw debian9.4Template.raw 2G

# 第二步，配置桥接网络
## ---------注意：一旦将某一个网口绑定到桥接网卡，这个口的IP将失效，
## ---------直到IP信息下沉到桥接网卡，下属操作使用其他网卡连接后执行-----------
## 新建br0，再查询桥接网络
root@test-vm:~# brctl addbr br0 
root@test-vm:~# brctl show    
bridge name bridge id STP enabled interfaces
br0 8000.000000000000 no
## 绑定网口
root@test-vm:~# brctl addif bond1 
## 查询br0信息
root@test-vm:/home/zhoulin/vmxml# brctl show
bridge name bridge id STP enabled interfaces
br0 8000.fe5400b23c01 no vnet0
## 激活br0
ifconfig br0 up
ip link set br0 up
## 将IP信息下沉到br0
ip addr del 192.168.1.198/24 dev bond1
ip addr add 192.168.1.198/24 dev br0
ip route add default via 192.168.1.254  #route add default gw 192.168.1.254
## -----注意：上述所有动作都在当前环境有效，配置都没有固化的，一旦重启就全部失效-------

# 第三步，使用最后附件的最精简xml新建虚拟机
virsh create debian9.4Template.xml

# 第四步，查询vnc端口，使用vnc连接，安装系统，然后测试网络，登入后做好配置，删除网卡udev信息，删除网络配置信息，删除虚拟机，作为镜像
netstat -tlp|grep qemu
virsh destroy debian9.4Template

# 第五步，根据上一步制作的模板生成新虚拟机磁盘文件
## 这里参考CeTune给的方法直接在本地挂载复制出来的盘，直接修改配置，实现新建虚拟机后即可用。
## 查看磁盘文件的分区结构
root@test-vm:~# fdisk -lu test-vm.img 
Disk test-vm.img: 2.5 GiB, 2684354560 bytes, 5242880 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x0009a368

Device Boot Start End Sectors Size Id Type
debian9.4Template1 * 2048 5240831 5238784 2.5G 83 Linux
##由上可知，真实的磁盘文件数据是从   (2048*512) 开始的，跳过前面这部分挂载到本地
root@test-vm:~# mount -o loop,offset=1048576 debian9.4Template.raw  /mnt
##此时可以安心做任何文件级别的修改了，比如主机名、网络配置
root@test-vm:~# echo ${hostname} > /mnt/etc/hostname
root@test-vm:~# echo "auto eth0" >> /mnt/etc/network/interfaces
root@test-vm:~# echo "iface eth0 inet static" >> /mnt/etc/network/interfaces
root@test-vm:~# echo "address "${ip} >> /mnt/etc/network/interfaces
root@test-vm:~# echo "netmask 255.255.255.0" >> /mnt/etc/network/interfaces
root@test-vm:~# echo "gateway 10.200.53.254" >> /mnt/etc/network/interfaces

# 第六步，libvirt对接ceph
ceph auth get-or-create client.admin mon 'allow *' osd 'allow *' -o /etc/ceph/ceph.client.admin.keyring
keyring=`cat /etc/ceph/ceph.client.admin.keyring | grep key | awk '{print $3}'`
ceph_cluster_uuid=`ceph -s | grep cluster | awk '{print $2}'`
echo "<secret ephemeral='no' private='no'>" > secret.xml
echo " <uuid>$ceph_cluster_uuid</uuid>" >> secret.xml 
echo " <usage type='ceph'>" >> secret.xml
echo " <name>client.admin secret</name>" >> secret.xml
echo " </usage>" >> secret.xml
echo "</secret>" >> secret.xml
virsh secret-define --file secret.xml
virsh secret-set-value $ceph_cluster_uuid $keyring

# 第七步，生成新虚拟机的xml文件
## 在附件xml文件的基础上作如下修改：
## 1.mac地址
## 2.mem，cpu根据需要修改
## 3.去掉 <boot dev="cdrom"/> 
## 4.去掉有关cdrom部分，即<disk type='file' device='cdrom'>部分
## 5.添加ceph rbd块作为其中一个disk，xml部分间附件2
```

## 附件
### 附件1：libvirt新建开机自动iso安装系统最精简的xml文件
```xml
<domain type="kvm">
    <name>test-vm</name>
    <memory>524288</memory>
    <vcpu>1</vcpu>
    <cputune>
        <vcpupin vcpu="0" cpuset="1"/>
    </cputune>
    <os>
        <type>hvm</type>
        <boot dev="cdrom"/>
        <boot dev="hd"/>
    </os>
    <features>
        <acpi/>
    </features>
    <clock offset="utc">
        <timer name="pit" tickpolicy="delay"/>
        <timer name="rtc" tickpolicy="catchup"/>
    </clock>
    <cpu mode="host-model" match="exact"/>
    <devices>
        <disk type="file" device="disk">
            <driver name="qemu" type="qcow2" cache="none"/>
            <source file="/home/zhoulin/vmxml/vm1.i.nease.net.qcow2" />
            <target bus="virtio" dev="vda"/>
        </disk>
        <disk type='file' device='cdrom'>
            <driver name='qemu' type='raw'/>
            <source file='/var/lib/libvirt/images/debian-9.4.0-amd64-xfce-CD-1.iso'/>
            <target dev='hdb' bus='ide'/>
            <readonly/>
        </disk>
        <interface type="bridge" >
            <source bridge ="br0"/>
            <mac address ="52:54:00:b2:3c:01"/>
        </interface>
        <serial type="pty"/>
        <input type="tablet" bus="usb"/>
        <graphics type="vnc" autoport="yes" keymap="en-us" listen="0.0.0.0"/>
    </devices>
</domain>
```

### 附件2：vm添加rbd作为硬盘的xml
```xml
<disk type='network' device='disk'>
    <driver name='qemu' type='raw' cache='none'/>
    <auth username='admin'>
        <secret type='ceph' uuid='${ceph_uuid}'/>
    </auth>
    <source protocol='rbd' name='${pool_name}/${rbd_name}' />
    <target dev='vdb' bus='virtio'/>
</disk>
```

## 其他问题
### 1.莫名操作libvirt挂了
错误如下，试了一些办法，无效，重启才恢复
```
root@test-vm:/home/zhoulin/vdbs# /etc/init.d/libvirtd restart
[ ok ] Restarting libvirt management daemon: /usr/sbin/libvirtd.
root@test-vm:/home/zhoulin/vdbs# virsh list
error: failed to connect to the hypervisor
error: no valid connection
error: Failed to connect socket to '/var/run/libvirt/libvirt-sock': No such file or directory
```
### 2.单纯virsh create在destroy后vm消失
```
virsh define XXX.xml
virsh list --all
```
上述方法注册之后，即使destroy也只是强关vm，还是依然在的，重新start可以拉起vm