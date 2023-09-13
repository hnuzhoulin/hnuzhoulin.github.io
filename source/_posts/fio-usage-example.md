---
title: fio应用
date: 2016/04/01 14:13:24
updated: 2016/04/06 10:51:03
categories: tool
tags: [linux,fio]
description: fio的具体应用实例，比如如何输出过程曲线并画图
---
## 1.输出bw，lat和iops数据并画图
fio安装完后自带有一个高级脚本`fio_generate_plots`能够根据fio输出的数据进行画图。操作流程如下：

### 1.1设置fio输出详细日志
fio的输出日志主要包含三种：**bw**，**lat**和**iops**，设置这三种的参数如下：

```bash
write_bw_log=rw
write_lat_log=rw
write_iops_log=rw
```

这里需要强调的一点是，后面接的参数**rw**，是输出日志文件名的**prefix**，如最终会生成的日志文件如下：

```bash
rw_iops.log
rw_clat.log
rw_slat.log
rw_lat.log
rw_bw.log
```

这个参数在后面画图的时候也要用到。

### 1.2 画图
前提是还需要安装好`gnuplot`，然后使用下面的命令即可自动画图：

```bash
root@ubuntu:/tmp> fio_generate_plots bw
```

发现没有，`fio_generate_plots`接受的唯一参数就是这个日志文件名的prefix。

本例中生成的图片文件有：

```
bw-bw.svg  
bw-clat.svg  
bw-iops.svg  
bw-lat.svg  
bw-slat.svg
```