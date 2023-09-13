---
title: 博客更新
date: 2016-04-24 16:53:35
description: 记录博客的已完成和待完成的改进
#top: true
---

### TODO List
1. Yelee主题学习：
2. 实现CND加速

### 20160604
1. 修改 `hexo new` 默认模板，加入description、top、categories等标签
2. 实现本地搜索，启动jQuery UI，安装hexo-generator-search

### 20160507
1. 主域名指向新的vps，[枫叶主机，香港CN2机房](http://www.fyzhuji.com/hk_vps.html)
2. 部署hexo静态博客到该vps，使用apache2，关闭查看目录，设置 `ErrorDocument 404 /404.html` 
3. 向百度和谷歌提交sitemap，感谢[hexo干货系列：（六）hexo提交搜索引擎（百度+谷歌）](http://www.jianshu.com/p/619dab2d3c08)
4. hexo的404指向腾讯公益404
5. 解决没有生成rss.xml和sitemap.xml的问题，`package.json` 中没有加入插件 `hexo-generator-sitemap`

### 20160505
1. 使用rsync这个deploy方法实现在vps上做镜像站点
2. 尝试添加站内搜索，使用swiftype，但是默认css，js与当前主题有部分冲突，导致首页右边文章部分空白

### 20160502
1. 实现置顶
2. 导航底部链接加入github链接
3. 文章底部分享，实际是被adblock屏蔽了。

### 20160501
1. 加入百度、谷歌统计
2. 迁移博客，标题跟leanote的同步，博客URL使用全英文，即 `hexo new post "blog URL"` ，在编辑对应md文件时，其中的title用中文
3. 博客改进记录改为使用博文而不是单页，这样能够生成目录

### 20160424
1. 修改主题为Yelee，自动实现博文内容加入版权信息模块

### 20160423
1. 在Mac上使用hexo和git利用github page实现本静态博客
2. 加入多说评论，需要在主题的 `_config.yaml` 文件中修改
3. hexo有多个tag时： [tag1,tag2,----]
4. 更换为jacman主题，
5. 打开首页时如何设置只预览部分内容，在需要预览的部分后加 `<!--more-->`