---
title: 根据本地deb文件生成ubuntu本地安装源
date: 2016-05-24 17:42:01
updated: 2016-05-24 17:43:01
categories: ubuntu
tags: [源,deb]
description: 为了在无外网环境使用apt-get安装软件，根据已经下载好的deb离线文件制作本地源
---

脚本来源于ceph的calamari项目以及diluge，感谢各位大神：

```bash
#!/bin/bash
if [ -e "/root/mysoft" ]
then
rm -rf /root/mysoft
fi
mkdir -p /root/mysoft/conf
mkdir /root/mysoft/soft
cp -p /var/cache/apt/archives/*.deb /root/mysoft/soft
apt-get install reprepro -y
apt-get install rng-tools -y
rngd -r /dev/urandom
gpg --gen-key
PUB_KEY=`gpg --list-keys|grep "^pub"|awk '{print $2}'|awk -F/ '{print $2}'`
SEC_KEY=`gpg --list-keys|grep "^sub"|awk '{print $2}'|awk -F/ '{print $2}'`
echo -n "Codename: precise
Suite: stable
Components: main
Architectures: i386 amd64
Origin: mysoft
Label: mysoft
Description: mysoft system
SignWith: $PUB_KEY
" >> /root/mysoft/conf/distributions
cd /root/mysoft
gpg --output mysoft_pub.gpg --armor --export $PUB_KEY
gpg --output mysoft_sec.gpg --armor --export-secret-key $SEC_kEY
reprepro --ask-passphrase -Vb . includedeb precise soft/*.deb
rm -r soft
echo "Done......."
echo "1)scp to this dir with: scp -r /root/mysoft {calamari IP}:/opt/calamari/webapp/content"
echo "2)client add: deb http://{Calamri IP}/static/mysoft precise main"
echo "3)client:wget -O - http://{Calamri IP}/static/mysoft/mysoft_pub.gpg|apt-key add -"
echo "4)if you just use local,add: deb file:///root/mysoft precise main"
```