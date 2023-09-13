---
title: CeTune抓取性能数据的代码段分析
date: 2016/03/04 11:58:06
updated: 2016/03/17 15:53:11
categories: ceph
tags: [cetune,performance]
description: CeTune抓取性能数据的代码段分析，包含cpu、内存、网络、磁盘和ceph数据；使用工具有sar，iostat，fio和ceph perfcounter
---
## 1.sar数据
### 1.1 抓取命令为：
```bash
common.pdsh(user, nodes, "sar -A %s > %s/`hostname`_sar.txt & echo `date +%s`' sar start' >> %s/`hostname`_process_log.txt" % (monitor_interval, dest_dir, '%s', dest_dir))
```
数据写入`node1_sar.txt`文件，`node1_process_log.txt`文件记录开始时间

### 1.2 停止命令
```bash
common.pdsh(user, nodes, "killall -9 sar; echo `date +%s`' sar stop' >> %s/`hostname`_process_log.txt" % ('%s', dest_dir), option = "check_return")
```
这里只是记录停止时间。

### 1.3数据作用
在文件`/analyzer/analyzer.py`的127行：
```bash
phase_name_map = {"cpu": "sar", "memory": "sar", "nic": "sar", "osd": "iostat", "journal": "iostat", "vdisk": "iostat" }
```
表明cpu、memory和nic的数据来自于sar

### 1.4 数据处理
#### 1.4.1 CPU数据
```bash
stdout = common.bash( "grep ' *CPU *%' -m 1 "+path+" | awk -F\"CPU\" '{print $2}'; cat "+path+" | grep ' *CPU *%' -A 1 | awk '{flag=0;if(NF<=3)next;for(i=1;i<=NF;i++){if(flag==1){printf $i\"\"FS}if($i==\"all\")flag=1};if(flag==1)print \"\"}'" )
```
CPU数据的获取是从node1_sar.txt文件中取出` *CPU *%`所在行及其下一行再利用awk逐行处理，获取all字段后面部分的数据。

#### 1.4.2 内存数据

```bash
stdout = common.bash( "grep 'kbmemfree' -m 1 "+path+" | awk -Fkbmemfree '{printf \"kbmenfree  \";print $2}'; grep \"kbmemfree\" -A 1 "+path+" | awk 'BEGIN{find=0;}{for(i=1;i<=NF;i++){if($i==\"kbmemfree\"){find=i;next;}}if(find!=0){for(j=find;j<=NF;j++)printf $j\"\"FS;find=0;print \"\"}}'" )
```

这里学习到的awk的`BEGIN`是设置起始全局变量，接收输入前设置，找到kbmemfree是第几列，真正的数据就从第几列开始输出。

#### 1.4.3 网卡数据

```bash
stdout = common.bash( "grep 'IFACE' -m 1 "+path+" | awk -FIFACE '{print $2}'; cat "+path+" | awk 'BEGIN{find=0;}{for(i=1;i<=NF;i++){if($i==\"IFACE\"){j=i+1;if($j==\"rxpck/s\"){find=1;start_col=j;col=NF;for(k=1;k<=col;k++){res_arr[k]=0;}next};if($j==\"rxerr/s\"){find=0;for(k=start_col;k<=col;k++)printf res_arr[k]\"\"FS; print \"\";next}}if($i==\"lo\")next;if(find){res_arr[i]+=$i}}}'" )
```

这里的处理更加巧妙，值得学习，将awk的处理语句展开如下：

```bash
{
    for(i=1;i<=NF;i++)
    {
        if($i=="IFACE")
        {
            j=i+1;
            if($j=="rxpck/s")
            {
                find=1;
                start_col=j;
                col=NF;
                for(k=1;k<=col;k++)
                {
                    res_arr[k]=0;
                }
                next
             };
             if($j=="rxerr/s")
             {
                find=0;
                for(k=start_col;k<=col;k++)
                    printf res_arr[k]""FS; 
                print "";
                next
             }
        }
        if($i=="lo")
            next;
        if(find)
        {
            res_arr[i]+=$i
        }
    }
}
```

先是找到合适的数据行，即包含`IFACE`且其后字段是`rxpck/s`；然后初始化一个数组为0，立马处理下一行，求取所有网卡对应字段值的和；等到`IFACE`后的字段是`rxerr/s`时开始输出结果，并置标志位为0.

**说明：**下面说明各字段的大概含义：

- **rxpck/s** ： 每秒接收的数据包大小
- **txpck/s** ：每秒发送的数据包大小
- **rxkB/s** ： 每秒接受的字节数
- **txkB/s** ： 每秒发送的字节数
- **rxcmp/s** ： 每秒接受的压缩数据包
- **txcmp/s** ： 每秒发送的压缩数据包
- **rxmcst/s** ： 每秒接受的多播数据包

## 2. fio数据
数据来源主要是fio输出的两行：
```bash
stdout, stderr = common.bash("grep \" *io=.*bw=.*iops=.*runt=.*\|^ *lat.*min=.*max=.*avg=.*stdev=.*\" "+path, True)
```
将所有数据求和即可。

## 3. iostat数据
### 3.1抓取命令
在run函数中iostat的命令为：

```bash
common.pdsh(user, nodes, "iostat -p ALL -dxm %s > %s/`hostname`_iostat.txt & echo `date +%s`' iostat start' >> %s/`hostname`_process_log.txt" % (monitor_interval, dest_dir, '%s', dest_dir))
```

### 3.2数据处理
首先是获取osd、journal以及vdisk的列表，然后分别使用下面的数据处理脚本处理：

数据处理的awk函数如下：

```bash
BEGIN
{
    split(dev,dev_arr,\" \");
    dev_count=0;  #设备总数
    for(k in dev_arr)
    {
        count[k]=0;  #某一个设备的行数
        dev_count+=1
    };
    for(i=1;i<=line;i++)
        for(j=1;j<=NF;j++)
        {
            res_arr[i,j]=0
        }
}
{
    for(k in dev_arr)
        if(dev_arr[k]==$1) #如果第一列的值等于我们要求的设备名
        {
            cur_line=count[k];   #设置一个行数指针指向某一个设备的行数
            for(j=2;j<=NF;j++)
            {
                res_arr[cur_line,j]+=$j;  #从第二列开始赋值给二维数组
            }
            count[k]+=1; #行数加1
            col=NF  #列数
        }
}

END
{
    for(i=1;i<=line;i++)
    {
        for(j=2;j<=col;j++)
            printf (res_arr[i,j]/dev_count)\"\"FS; 
            print \"\"
    }
}
```

其中的dev和line是通过-v命令传入的参数，dev是所有osd或者journal磁盘以空格分隔。是生成一个二维数组存储，是总行数和列数。

**注意:**巧妙的是利用设备行数和行数指针实现，同一次的数据所有osd或者journal的数据求和。**建议最后的结果不除以设备数**

## 4.运行时间
处理文件`node1_process_log.txt`获取各个工具的开始和结束时间，并计算其和fio/cosbench测试的时间差，已确保fio等运行前，性能收集工具已经运行。

## 5.perfcounter数据收集
关于ceph自己的perf数据，各个项所代表的实际含义还有待深入探索，这里只是检查CeTune收集了哪些数据。
### 5.1抓取命令
在run函数中对启动perf数据收集的语句进行分割：

```bash
echo `date +%s`' perfcounter start' >> %s/`hostname`_process_log.txt; 
for i in `seq 1 %d`; do 
    find /var/run/ceph -name '*osd*asok' | while read path; do 
        filename=`echo $path | awk -F/ '{print $NF}'`;
        res_file=%s/`hostname`_${filename}.txt;
    echo `ceph --admin-daemon $path perf dump`, >> ${res_file} & 
        done; 
    sleep %s; 
done; 
echo `date +%s`' perfcounter stop' >> %s/`hostname`_process_log.txt;"
```
这路第一个for循环是`time = int(self.benchmark["runtime"]) + int(self.benchmark["rampup"]) + self.cluster["run_time_extend"]`；而sleep时间为设置的监控间隔 `monitor_interval = self.cluster["monitoring_interval"]`。**注意：这里每次取完数据还加了一个逗号作为分隔符**

### 5.2数据处理
- 先将搜集到的所有数据项的实际数据由数值变为列表，以osd的op_latency为例举例如下：

```json
(u'op_latency', OrderedDict([(u'avgcount', [1913, 1915, 1915, 1918, 1919, 1920, 1923, 1924, 1925, 1929, 1930, 1932, 1937, 1939, 1941, 1945, 1949, 1954, 1958, 1960, 1968, 1973, 1977, 1980, 1981, 1984, 1992, 1998, 2004, 2010, 2016, 2019, 2024, 2027, 2030, 2037, 2047, 2050, 2053, 2060, 2064, 2073, 2076, 2082, 2086, 2091, 2097, 2104, 2108, 2113, 2117, 2124, 2130, 2132, 2141, 2145, 2147, 2148, 2151, 2154, 2158, 2165, 2168, 2171, 2177, 2180, 2183, 2189, 2195, 2199, 2206, 2206, 2211, 2217, 2222, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223, 2223]), (u'sum', [163.617276, 163.678138, 163.678138, 163.716475, 163.733006, 163.749852, 163.827178, 163.830007, 163.843426, 163.956236, 163.976961, 164.012076, 164.071066, 164.094675, 164.098881, 164.220935, 164.290138, 164.366159, 164.410334, 164.453903, 164.609118, 164.697958, 164.747356, 164.799038, 164.823, 164.87076, 165.010713, 165.116875, 165.240544, 165.324972, 165.389481, 165.395939, 165.434573, 165.46386, 165.505794, 165.606881, 165.6955, 165.725082, 165.762864, 165.847311, 165.897523, 166.001776, 166.054834, 166.144204, 166.170836, 166.240923, 166.350937, 166.436625, 166.493355, 166.521452, 166.593893, 166.647407, 166.7904, 166.828572, 167.009988, 167.051504, 167.056824, 167.087444, 167.121147, 167.155003, 167.197488, 167.35159, 167.410595, 167.439328, 167.499467, 167.530351, 167.539261, 167.555903, 167.605426, 167.634956, 167.735829, 167.735829, 167.761924, 167.823874, 167.882077, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737, 167.913737])])), 
```

- 对于这种有avgcount和sum的数据，实际的op_latency值应该是`data['sum'][i]-last_sum)/(data['avgcount'][i]-last_avgcount`，即两个相邻值的差再做除法。对于avgcount不变的，最终值设为0。如果没有avgcount和sum则无需此运算。
- 再将上述运算后的值作为列表作为最终数据中op_latency索引的值。

### 5.3 filestore收集参数列表
主要是源码文件：`src/os/filestore/FileStore.cc`中有说明
- queue_transaction_latency_avg:Store operation queue latency,
- bytes:Data written to store
- ops:Operations written to store
- op_queue_bytes:Size of writing to FS queue
- op_queue_ops:Operations in writing to FS queue
- journal_bytes:Total operations size in journal
- journal_latency:Average journal queue completing latency
- journal_wr:Journal write IOs
- journal_wr_bytes:Journal data written
- journal_ops:Total journal entries written
- apply_latency:Apply latency
- commitcycle:Commit cycles
- commitcycle_latency:Average latency of commit
- commitcycle_interval:Average interval between commits
- committing:Is currently committing

### 5.4 OSD收集的参数列表
- op:Client operations,ops
- op_latency:Latency of client operations (including queue time)
- op_in_bytes:Client operations total write size
- op_out_bytes:Client operations total read size
- op_process_latency:Latency of client operations (excluding queue time)
- op_wip:Replication operations currently being processed (primary)
- op_r:Client read operations
- op_r_out_bytes:client read out bytes
- op_r_latency:Latency of read operation (including queue time) // client read latency
- op_r_process_latency:Latency of read operation (excluding queue time  // client read process latency
- op_w:Client write operations // client writes
- op_w_in_bytes:Client data written// client write in bytes
- op_w_latency:Latency of write operation (including queue time  // client write latency
- op_w_rlat:Client write operation readable/applied latency  // client write readable/applied latency
- op_w_process_latency:Latency of write operation (excluding queue time // client write process latency
- subop:Suboperations
- subop_in_bytes:Suboperations total size  // subop in bytes
- subop_latency:Suboperations latency
- subop_w_in_bytes:Replicated written data size // replicated write in bytes
- subop_w_latency:Replicated writes latency // replicated write latency
- loadavg:CPU load
- stat_bytes_used:stat_bytes_used,"Used space"
- stat_bytes_avail:stat_bytes_avail, "Available space"

## 6.汇总
上述所有数据搜集完成后存入result中交给Visualizer去画图以及生成最终的展示页面：

```python
view = visualizer.Visualizer(result, dest_dir)
output = view.generate_summary_page()
```

**重要：**最后的报告首页summary中有几个**SN开头的参数** 解释如下：
1. SN_IOPS：先求某一个节点所有OSD在runtime期间底层iostat抓取的IOPS数据的平均值，再把所有节点的这个平均值相加，求得实际底层OSD的总IOPS；
2. SN_BW(MB/s)：计算方法同上，先求某一个节点所有OSD的速度的平均值，再求所有节点的和，得到实际底层OSD的速度；
3. SN_Latency(ms)： 这个值的计算先是求某一个节点所有OSD的延时的平均值，在求所有节点的延时平均值，以反映整个集群的平均值。