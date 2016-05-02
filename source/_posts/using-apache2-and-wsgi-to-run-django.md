---
title: django项目利用apache2+mod_wsgi运行
date: 2016/03/02 11:44:59
updated: 2016/03/02 11:45:48
categories: python
tags: [django,wsgi]
description: using-apache2-and-wsgi-to-run-django.md
---
## 1.前提
假设django项目名为`myproject`,为了便于处理权限，将项目目录置于`/var/www`目录下。

## 2.安装
安装apache和mod_wsgi：
```
apt-get install apache2 libapache2-mod-wsgi
```

## 3.配置
编辑文件`/etc/apache2/sites-enabled/000-default`，替换为下面的内容：
```
<VirtualHost *:80>
    . . .
    Alias /static /var/www/myproject/static
    <Directory /home/user/myproject/static>
        Require all granted
    </Directory>

    <Directory /var/www/myproject/myproject>
        <Files wsgi.py>
            Require all granted
        </Files>
    </Directory>

    WSGIDaemonProcess myproject python-path=/var/www/myproject:/usr/lib/python2.7/site-packages:/usr/local/lib/python2.7/site-packages
    WSGIProcessGroup myproject
    WSGIScriptAlias / /var/www/myproject/myproject/wsgi.py

</VirtualHost>
```

这里需要说明一下:

1. 第一个alias是将静态文件重定向到指定目录，如果是简单项目不需要的话可以删除
2. 后面的一个`Directory`是针对static的权限的，如果不需要也可以对应删除。
3. 第二个`Directory`就是关键，注意目录结构，文件`wsgi.py`在新建django项目的时候会自动生成。
4. 在WSGIDaemonProcess里面我把python的两个系统库目录加进去了，如果你用的env，就加自己env的地址。

## 4.部分可能的权限问题
```
chmod 664 ~/myproject/db.sqlite3
chown :www-data ~/myproject/db.sqlite3
chown :www-data ~/myproject
```

如果遇到服务器403的错误，那么可能是你的"/"目录被禁止访问（默认）
修改一下apache2.conf文件（位于/etc/apache2/）

把`Require all denied`改为`Allow from all`就可以了

## 5.重启apache2服务即可
收工