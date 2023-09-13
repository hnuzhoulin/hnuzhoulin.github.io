---
title: ceph-deploy
tags: [部署]
top: false
date: 2016-06-04 17:29:51
updated: 2016-06-04 17:29:51
categories: ceph
description: 汇总ceph-deploy的相关学习积累
---

## 1.设置使用本地源安装ceph

两种方式：

- 一种就是如下文所述，直接在命令行中写
- 另一种是写配置文件 `cephdeploy.conf` 见下文：


```bash
[ceph-deploy-install]
repo_url=http://10.1.70.18/ceph.com/debian-firefly
gpg_url=http://10.1.70.18/release.asc
release=firefly
```

**注意：**这里最好是写在 `ceph-deploy-install` 这个sction中，

## 2.使用本地源安装ceph

```bash
ceph-deploy install autoceph3 autoceph4  --release=firefly --repo-url=http://10.1.70.18/ceph.com/debian-firefly --gpg-url=http://10.1.70.18/release.asc
```

执行上述命令会在 `apt-get update` 的时候报错，因为本地源没有32位的，解决方法见下

### 本地只有x86源时的处理方法

如果使用的是本地源，而且仅仅有64位的软件包地址，这样在第二次安装到的时候会出现找不到i386包而报错，处理方法是修改文件 `/usr/local/lib/python2.7/dist-packages/ceph_deploy/hosts/remotes.py` 文件的47行，修改为：

```python
content = 'deb  [arch=amd64] {url} {codename} main\n'.format(
```

即加入arch标签