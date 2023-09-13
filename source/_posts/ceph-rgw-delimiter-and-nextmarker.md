---
title: ceph-rgw-delimiter-and-nextmarker
tags: [rgw]
top: false
date: 2018-11-05 13:26:29
updated: 2018-11-05 13:26:29
categories: ceph
description: ceph的rgw服务在list大于1000个object时delimiter参数会影响返回值是否有NextMarker参数
---

## rgw的list-bucket
ceph rgw的list-bucket接口默认只返回前面1000个object，对于大于1000个object的场景，在第一次请求的response中会有下面两个参数来指示获取下一个1000个object，类似分页：

- NextMarker: 下一个1000个object（也就是下一页）的第一个object的名字
- isTruncated: 标记当前是否是最后一页
- delimiter：按照目录层级的方式返回，标记目录层级的分隔符，比如linux下的目录分隔符为`/`
- Prefix：object的名字前缀，可以理解为目录名

## list-bucket的delimiter问题复现及解决

### 使用boto执行list-bucket

脚本如下：

```python

import boto
from boto.s3.connection import S3Connection
import datetime
import sys
reload(sys)
sys.setdefaultencoding('utf-8')

# get config from environment or ...
endpoint = os.environ["s3_endpoint"]
bucket_name = "hdg2-gcp-bossweb"

access_key = os.environ["access_key"]
secret_key = os.environ["secret_key"]

conn = S3Connection(ak, sk, host=host, calling_format=boto.s3.connection.OrdinaryCallingFormat(), is_secure=False)
#print conn.get_all_buckets()
bucket = conn.get_bucket(bucketname)

marker=""
while True:
    filelist = bucket.get_all_keys(marker=marker)
    for file in filelist:
        print("%s\t%s\t%s" %(file.last_modified, file.name, file.size))
    if filelist.is_truncated == False:
        break

    marker = filelist.next_marker

    print(marker)

```

抓包请求：
![](https://ws4.sinaimg.cn/large/006tNbRwgy1fwx4l1lbg4j30fd03qwew.jpg)

响应如下：
```xml
<ListBucketResult xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
    <Name>hdg2-gcp-bossweb</Name>
    <Prefix />
    <Marker />
    <MaxKeys>1000</MaxKeys>
    <IsTruncated>true</IsTruncated>
    <EncodingType>url</EncodingType>
    <Contents>
        <Key>bosswebfiles%2Fattachment%2F1495115517299a0165c74-4ca4-4c2f-9d66-2e89fbaeef8c.jpg</Key>
        <LastModified>2017-05-18T13:51:57.000Z</LastModified>
        <ETag>&quot;50d0c18045f9b92ae52a66cf4d82701b&quot;</ETag>
        <Size>402987</Size>
        <StorageClass>STANDARD</StorageClass>
        <Owner>
            <ID>p-gcp</ID>
            <DisplayName>p-gcp</DisplayName>
        </Owner>
    </Contents>
...
    <Contents>
        <Key>bosswebfiles%2Fattachment%2F152828637501198245d2a-e1a4-46e6-9502-648959a7045a.jpg</Key>
        <LastModified>2018-06-06T11:59:35.000Z</LastModified>
        <ETag>&quot;7cbcd767f1a282f48b6ed0f7ec2f4fa2&quot;</ETag>
        <Size>160286</Size>
        <StorageClass>STANDARD</StorageClass>
        <Owner>
            <ID>p-gcp</ID>
            <DisplayName>p-gcp</DisplayName>
        </Owner>
    </Contents>
</ListBucketResult>
```

**总结：**
- 没有指定delimiter表示直接按照object的key输出，不人为构建目录层级
- marker为空表示从第一个object开始
- Response的IsTruncated的值为True表示不是最后一页
- Response中却没有NextMarker这一属性，故而无法获取下一页的object

### 使用s3cmd尝试

#### 访问顶级目录

请求如下
![](https://ws4.sinaimg.cn/large/006tNbRwgy1fwx4o3w4wqj30fd03fq38.jpg)

响应如下：
```xml
<ListBucketResult xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
    <Name>hdg2-gcp-bossweb</Name>
    <Prefix />
    <Marker />
    <MaxKeys>1000</MaxKeys>
    <Delimiter>/</Delimiter>
    <IsTruncated>false</IsTruncated>
    <CommonPrefixes>
        <Prefix>bosswebfiles/</Prefix>
    </CommonPrefixes>
</ListBucketResult>
```
**总结：**

- s3cmd默认所有请求都指定delimiter为`/`，故会根据object的名字去切分来构建目录层级
- marker为空表示从第一个object开始
- Response的IsTruncated的值为false表示是最后一页
- Response中没有NextMarker这一属性，表示该页的所有object按照delimiter切分分级后没有文件，CommonPrefixes表示下一级目录

#### s3cmd访问子目录

请求如下：
![](https://ws2.sinaimg.cn/large/006tNbRwgy1fwx4ojhkz7j30eu03ngly.jpg)

响应如下：
```xml
<ListBucketResult xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
    <Name>hdg2-gcp-bossweb</Name>
    <Prefix>bosswebfiles/</Prefix>
    <Marker />
    <MaxKeys>1000</MaxKeys>
    <Delimiter>/</Delimiter>
    <IsTruncated>false</IsTruncated>
    <CommonPrefixes>
        <Prefix>bosswebfiles/attachment/</Prefix>
    </CommonPrefixes>
</ListBucketResult>
```
**总结：**

- s3cmd默认所有请求都指定delimiter为`/`，故会根据object的名字去切分来构建目录层级
- marker为空表示从第一个object开始
- Prefix为第一个请求返回的CommonPrefixes的值，也就是第一级目录的名字，即为获取该目录下的文件
- 此时Response的IsTruncated的值为false表示是最后一页
- Response中也没有NextMarker这一属性，表示该页的所有object按照delimiter进一步切分分级后没有文件，CommonPrefixes表示下一级目录

#### s3cmd访问文件所在目录

请求如下：
![](https://ws2.sinaimg.cn/large/006tNbRwgy1fwx4osdm9ej30g203s3yw.jpg)

部分响应如下：
```xml
<ListBucketResult xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
    <Name>hdg2-gcp-bossweb</Name>
    <Prefix>bosswebfiles/attachment/</Prefix>
    <Marker />
    <NextMarker>bosswebfiles/attachment/152828637501198245d2a-e1a4-46e6-9502-648959a7045a.jpg</NextMarker>
    <MaxKeys>1000</MaxKeys>
    <Delimiter>/</Delimiter>
    <IsTruncated>true</IsTruncated>
    <Contents>
        <Key>bosswebfiles/attachment/1495115517299a0165c74-4ca4-4c2f-9d66-2e89fbaeef8c.jpg</Key>
        <LastModified>2017-05-18T13:51:57.000Z</LastModified>
        <ETag>&quot;50d0c18045f9b92ae52a66cf4d82701b&quot;</ETag>
        <Size>402987</Size>
        <StorageClass>STANDARD</StorageClass>
        <Owner>
            <ID>p-gcp</ID>
            <DisplayName>p-gcp</DisplayName>
        </Owner>
    </Contents>
...
    <Contents>
        <Key>bosswebfiles/attachment/152828637501198245d2a-e1a4-46e6-9502-648959a7045a.jpg</Key>
        <LastModified>2018-06-06T11:59:35.000Z</LastModified>
        <ETag>&quot;7cbcd767f1a282f48b6ed0f7ec2f4fa2&quot;</ETag>
        <Size>160286</Size>
        <StorageClass>STANDARD</StorageClass>
        <Owner>
            <ID>p-gcp</ID>
            <DisplayName>p-gcp</DisplayName>
        </Owner>
    </Contents>
</ListBucketResult>
```


**总结：**

- s3cmd默认所有请求都指定delimiter为`/`，故会根据object的名字去切分来构建目录层级
- marker为空表示从第一个object开始
- Prefix为前面两次请求返回的CommonPrefixes拼起来的路径
- 此时Response的IsTruncated的值为True表示不是最后一页
- 此时Response终于有NextMarker属性值，表示下一页的第一个object
- Response中Content部分就是所有object的列表
- 这里可以看到第一页1000个object的最后一个object名字是`bosswebfiles/attachment/152828637501198245d2a-e1a4-46e6-9502-648959a7045a.jpg`，与前面boto的返回结果一致。

#### s3cmd自动获取下一页object
请求如下：
![](https://ws4.sinaimg.cn/large/006tNbRwgy1fwx4p21yerj30vm03hwf9.jpg)

响应如下：
```xml
<ListBucketResult xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
    <Name>hdg2-gcp-bossweb</Name>
    <Prefix>bosswebfiles/attachment/</Prefix>
    <Marker>bosswebfiles/attachment/152828637501198245d2a-e1a4-46e6-9502-648959a7045a.jpg</Marker>
    <MaxKeys>1000</MaxKeys>
    <Delimiter>/</Delimiter>
    <IsTruncated>false</IsTruncated>
    <Contents>
        <Key>bosswebfiles/attachment/15282863820674cab8349-5b34-46df-9f92-c03349080d3e.png</Key>
        <LastModified>2018-06-06T11:59:42.000Z</LastModified>
        <ETag>&quot;fdf990d0c7e8abe0e4e251cce657d7bf&quot;</ETag>
        <Size>1353613</Size>
        <StorageClass>STANDARD</StorageClass>
        <Owner>
            <ID>p-gcp</ID>
            <DisplayName>p-gcp</DisplayName>
        </Owner>
    </Contents>
...
</ListBucketResult>
```
**总结：**

- s3cmd默认所有请求都指定delimiter为`/`，故会根据object的名字去切分来构建目录层级
- marker为`bosswebfiles/attachment/152828637501198245d2a-e1a4-46e6-9502-648959a7045a.jpg`是上一次请求的NextMarker的值，表示这一页的第一个object
- Prefix依然为最开始两次目录请求返回的CommonPrefixes拼起来的路径
- 此时Response的IsTruncated的值为False表示是最后一页，且Response也没有NextMarker属性值，两者结果匹配



## 问题分析

从上面两个的返回结果可以知道，对于object在逻辑上处于子目录下时，如果不使用`delimiter和prefix=dir`的话是只返回前面1000个文件，而且`IsTruncated=True`，但是`没有返回值NextMarker`

**在luminous版本测试发现已经没有这个问题了。**

## delimiter作用示例
在文件`ceph/src/rgw/rgw_rados.cc`的函数`RGWRados::Bucket::List::list_objects`的前面注释有如下说明：
>  * delim: do not include results that match this string.
> * Any skipped results will have the matching portion of their name
> * inserted in common_prefixes with a "true" mark.

官方文档说明：（http://docs.ceph.com/docs/master/radosgw/s3/bucketops/）
> delimiter：The delimiter between the prefix and the rest of the object name.

`delimiter=“/”` 的实际表现就是：将object名字以`/`作为目录分隔符，去按照目录层级显示；因此如果将其设置为`-`，即表示按照短横线作为目录分隔符去层级显示，如下例：



![](https://ws2.sinaimg.cn/large/006tNbRwgy1fwx4pa5tghj30iv044aaj.jpg)

```xml
<ListBucketResult xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
    <Name>hdg2-gcp-bossweb</Name>
    <Prefix />
    <Marker />
    <NextMarker>bosswebfiles/attachment/152828637501198245d2a-</NextMarker>
    <MaxKeys>1000</MaxKeys>
    <Delimiter>-</Delimiter>
    <IsTruncated>true</IsTruncated>
    <EncodingType>url</EncodingType>
    <CommonPrefixes>
        <Prefix>bosswebfiles/attachment/1495115517299a0165c74-</Prefix>
    </CommonPrefixes>
    <CommonPrefixes>
        <Prefix>bosswebfiles/attachment/1495115925293a32229a9-</Prefix>
    </CommonPrefixes>
...
</ListBucketResult>
```
继续一级一级执行如下：
```python
python list_bucket.py bosswebfiles/attachment/152828637501198245d2a-
DIR bosswebfiles/attachment/152828637501198245d2a-e1a4-
python list_bucket.py bosswebfiles/attachment/152828637501198245d2a-e1a4- 
DIR bosswebfiles/attachment/152828637501198245d2a-e1a4-46e6-
python list_bucket.py bosswebfiles/attachment/152828637501198245d2a-e1a4-46e6- 
DIR bosswebfiles/attachment/152828637501198245d2a-e1a4-46e6-9502-
python list_bucket.py bosswebfiles/attachment/152828637501198245d2a-e1a4-46e6-9502- 
2018-06-06 11:59:35+00:00    bosswebfiles/attachment/152828637501198245d2a-e1a4-46e6-9502-648959a7045a.jpg    160286    "7cbcd767f1a282f48b6ed0f7ec2f4fa2"
```
