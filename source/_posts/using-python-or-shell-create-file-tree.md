---
title: 文件目录树的python和shell生成方法
date: 2015/12/23 12:07:17
updated: 2015/12/23 12:09:45
categories: program
tags: [python,shell]
description: 分别使用python和shell生成文件目录树，可自定义层级
---

## 1.背景
背景很明确，需要遍历一个目录下的所有文件和目录，生成目录树，输出样式为相对路径显示目录和目录下的文件

## 2.python实现

### 2.1.基础版

    import os
    currentDir = '/home/ubuntu/data/user-config/Downloads/ceph'
    for dirName,subdirList,fileList in os.walk(currentDir):
        print('%s' % dirName)
        for fname in fileList:
            abspath=dirName+os.sep+fname
            print(abspath)

输出如下：

    /home/ubuntu/data/user-config/Downloads/ceph
    /home/ubuntu/data/user-config/Downloads/ceph/.gitignore
    /home/ubuntu/data/user-config/Downloads/ceph/SUMMARY.md
    /home/ubuntu/data/user-config/Downloads/ceph/chapter1.md
    /home/ubuntu/data/user-config/Downloads/ceph/README.md
    /home/ubuntu/data/user-config/Downloads/ceph/.git
    /home/ubuntu/data/user-config/Downloads/ceph/.git/config
    /home/ubuntu/data/user-config/Downloads/ceph/.git/description
    /home/ubuntu/data/user-config/Downloads/ceph/.git/branches
    /home/ubuntu/data/user-config/Downloads/ceph/.git/refs
    /home/ubuntu/data/user-config/Downloads/ceph/.git/refs/tags
    /home/ubuntu/data/user-config/Downloads/ceph/.git/refs/remotes
    /home/ubuntu/data/user-config/Downloads/ceph/.git/refs/remotes/origin

### 2.2.加入筛选功能版
这里额外多了三个参数：

- pattern是匹配模式，寻找匹配这个匹配模式的文件和目录；
- single_level指是否只显示定义目录根目录下的文件和目录，不递归遍历；
- yield_folders：输出结果是否需要包含目录名

详细代码如下：

    import os,fnmatch

    def all_files(root,patterns='*',single_level=False,yield_folders=True):
        """
        Expand patterns from semicolon-separated string to list
        pattern：use patterns to select files and dirs
        single_level:if only show the root dir
        yield_folders:the output contain dirs?
        """
        patterns = patterns.split(';')
        for path,subdirs,files in os.walk(root):
            if yield_folders:
                files.extend(subdirs)
            for name in files:
                for pattern in patterns:
                    if fnmatch.fnmatch(name,pattern):
                        yield os.path.join(path,name)
                break
            if single_level:
                break

    thefiles = list(all_files('/home/ubuntu/data/user-config/Downloads/ceph'))
    thefiles.sort()
    for i in thefiles:
        print (i)

输出为：

    /home/ubuntu/data/user-config/Downloads/ceph/.git
    /home/ubuntu/data/user-config/Downloads/ceph/.git/HEAD
    /home/ubuntu/data/user-config/Downloads/ceph/.git/branches
    /home/ubuntu/data/user-config/Downloads/ceph/.git/config
    /home/ubuntu/data/user-config/Downloads/ceph/.git/description
    /home/ubuntu/data/user-config/Downloads/ceph/.git/logs/refs
    /home/ubuntu/data/user-config/Downloads/ceph/.git/logs/refs/heads
    /home/ubuntu/data/user-config/Downloads/ceph/.git/logs/refs/heads/master
    /home/ubuntu/data/user-config/Downloads/ceph/.git/logs/refs/remotes
    /home/ubuntu/data/user-config/Downloads/ceph/.git/logs/refs/remotes/origin
    /home/ubuntu/data/user-config/Downloads/ceph/.gitignore
    /home/ubuntu/data/user-config/Downloads/ceph/README.md
    /home/ubuntu/data/user-config/Downloads/ceph/SUMMARY.md

### 2.3.版本3：获取文件的最近修改时间

    import os,fnmatch

    def all_files(root,patterns='*',single_level=False,yield_folders=True):
        """
        Expand patterns from semicolon-separated string to list
        pattern：use patterns to select files and dirs
        single_level:if only show the root dir
        yield_folders:the output contain dirs?
        """
        patterns = patterns.split(';')
        for path,subdirs,files in os.walk(root):
            if yield_folders:
                files.extend(subdirs)
            for name in files:
                for pattern in patterns:
                    if fnmatch.fnmatch(name,pattern):
                        file1=path+os.sep+name
                        statinfo=os.stat(file1)
                        name=name+' '+str(statinfo.st_mtime)
                        yield os.path.join(path,name)
                break
            if single_level:
                break

    thefiles = list(all_files('/home/ubuntu/data/user-config/Downloads/ceph'))
    thefiles.sort()
    for i in thefiles:
        print (i)

### 2.4.os.walk函数详解
使用 os.walk 方便很多了.这个方法返回的是一个三元tupple(dirpath, dirnames, filenames)：

- 第一个为起始路径，是一个string，代表目录的路径,
- 第二个为起始路径下的文件夹,是一个list，包含了dirpath下所有子目录的名字,
- 第三个是起始路径下的文件，是一个list，包含了非目录文件的名字.这些名字不包含路径信息,如果需要得到全路径,需要使用 os.path.join(

详细的介绍看官方帮助： `help (os.walk)`

## 3.shell实现

    #!/bin/bash
    #This script is used to list all files in a directory
    # including the sub-directory and the last modified time
    # then write this into files.

    if [ $# -eq 0 ]&&[ $# -lt 3 ];then
        echo "listfiles.sh: running as listfiles.sh <directory> <timestamp> <level> "
        echo""
        exit
    fi

    filelist=index_$2_$3
    declare statinfo=197900000000

    function ergostat(){
        statinfo=`stat $1|grep Modify|awk -F. '{print $1}'
        | awk '{print $2$3}'| awk -F- '{print $1$2$3}' | awk -F: '{print $1$2$3}'`
        # echo -n $statinfo
    }

    function ergodic(){
        for file in `ls $1`
        do
        if [ -d $1"/"$file ]
        then
        echo $1"/"$file>>$filelist

        ergodic $1"/"$file
        else
        local path=$1"/"$file
        ergostat $path
        echo "$path $statinfo" >>$filelist
        fi
        done
    }

    #这个必须要，否则会在文件名中有空格时出错
    IFS=$'n' 
    #INIT_PATH=".";
    ergodic $1 $2 $3[/bash]