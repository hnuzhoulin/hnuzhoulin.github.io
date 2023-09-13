---
title: SQLAlchemy的ORM模型操作数据库
date: 2015/12/07 21:11:27
updated: 2016/03/12 09:56:49
categories: python
tags: [sqlalchemy]
description: SQLAlchemy的ORM模型操作数据库
---
SQLAlchemy 是python 操作数据库的一个库。能够进行**ORM**映射

>SQLAlchemy采用简单的Python语言，为高效和高性能的数据库访问设计，实现了完整的企业级持久模型

本文在实例的基础上加上注释来解释如何利用SQLAlchemy的ORM模型像操作类一样实现对数据库的操作。

## 1.初始化操作
```
coding:utf-8
__author__ = 'itzhoulin'

from sqlalchemy import Integer, String, BIGINT, DateTime,         VARCHAR, and_, Table, Sequence
from sqlalchemy import create_engine, MetaData, Column
from sqlalchemy.orm import sessionmaker
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()
```
##2.定义数据库中的表

当使用ORM模式的时候，需要为每一个Table建立一个以Base为基类的类，这个类必不可少的一项就是tablename和至少一个Column(设置为primary_key)

    class host(Base):
    __tablename__ = 'gci_server_orm'
    id = Column(Integer, primary_key = True)
    hostname = Column(String(30), nullable = False)
    ip = Column(String(16), nullable = False)
    fsid = Column(String(60), nullable = False)
    type = Column(String(10),nullable = False)
    
    def __repr__(self):
    return "host(hostname='%s',ip='%s',fsid='%s',type='%s')" % (self.hostname,self.ip,self.fsid,self.type)

这里定义 `__repr__` 函数是为了显示该类，可以自定义，是可选的

>Additionally, Firebird and Oracle require sequences to generate new primary key identifiers, and SQLAlchemy doesn’t generate or assume these without being instructed. For that, you use the Sequence construct:

对于postgresql而言，会自动新建，这里指定是为了设定自增起始值，关于Sequence的详细说明请[点击](http://docs.sqlalchemy.org/en/latest/core/defaults.htmlsqlalchemy.schema.Sequence)
 
```
class fsuser(Base):
    __tablename__ = 'fsusers_orm'
    id = Column(BIGINT, Sequence('fsuser_aid_seq', start=5000, increment=1), primary_key=True)
    user_name = Column(VARCHAR(50),nullable=False)
    user_pwd = Column(VARCHAR(500),nullable=False)
    fsid = Column(String(60),nullable=False)
     mount_point = Column(VARCHAR(50))
     uuid = Column(VARCHAR(50))
     flag = Column(VARCHAR(1))

def __repr__(self):
    return "fsuser(user_name='%s',user_pwd='%s',fsid='%s')" % (self.user_name,self.user_pwd,self.fsid)
```

可以输出看一下定义的类，这里需要注意的是，这里的 `String` 和 `VARCHAR` 都定义了长度的， 对于sqlite和postgresql不定义长度也是可以的，但是其他数据就不行:

>Users familiar with the syntax of CREATE TABLE may notice that the VARCHAR columns were generated without a length; on SQLite and Postgresql, this is a valid datatype, but on others, it’s not allowed. So if running this tutorial on one of those databases, and you wish to use SQLAlchemy to issue CREATE TABLE, a “length” may be provided to the String type as below:
 Column(String(50))

    print host.__table__

## 3.准备连接数据库

首先设置数据库连接，更多数据库连接设置方式参考[官网手册](http://docs.sqlalchemy.org/en/rel_0_9/core/engines.htmldatabase-urls)：

创建一个Engine实例，指向通过db_url设置的数据库，这仅仅表示到数据库的接口已经建立:

    db_url = 'postgresql://testdb:123456@192.168.14.78/testdbdb'

但是并没有连接到数据库，仅仅当执行engine.execute() 或者 engine.connect()时才真正建立连接。

    engine = create_engine(db_url)

定义一个Metadata，它是一个容器对象，能够描述一个数据库或者多个数据库的很多不同特性。Table对象是被连接到Metadata的，Table是由诸多Column组成的，Base类包含有.metadata方法，因此新建Table方法如下，跟不使用ORM的时候一样

    Base.metadata.reflect(engine)

新建/删除上面定义的两个类对应的Table是很容易的：

    Base.metadata.drop_all(engine)
    Base.metadata.create_all(engine)

上面drop和create是通过Base类来实现对其上下文定义的所有Table进行操作的

新建的实例，Base类定义有 `__init__` 方法，会接收值并赋给Table中设置的Column，如下新建插入值：

    host1=host(hostname='host4.test.com',ip='10.1.35.27',fsid='1qazxsw23edcvfr4',type='s3')
    print host1.hostname

这里实际还 **没有写入数据库** ，只是生成一个跟数据库mappered的类的实例，
因此 **主键id并没有实际值** ，但是引用也不会报错，返回None，打印测试一下：

    print host1.id

## 4.连接数据库
连接数据库，ORM模式下跟数据库的实际连接是通过Session

    Session = sessionmaker(bind=engine)

如果新建Session的时候还没有生成engine，可以通过下面的两步先后设置

    Session = sessionmaker()
    Session.configure(bind=engine)

这样每当需要跟数据库进行操作的时候实例化Session就行

    session = Session()

这里新建了实例session，但是也并 **没有打开跟数据库的连接** ，仅当 **第一次开始用时才建立连接** ， 而且仅当我们提交所有的变更或者手动关闭session的时候才断开连接

## 5.插入操作
插入一条记录，以上面的host1为例：

    session.add(host1)

此时依然能够修改其中的某一个记录，同时session知道我们修改了这个数据：

    host1.fsid = 'abcdefghijklmnopqresuvmxyz'
    
通过dirty来判定session里面的数据是否有修改
```
session.dirty
 Out[40]: IdentitySet([host(hostname='host1.test.com',ip='192.168.35.23',fsid='abcdefghijklmnopqresuvmxyz',type='samba')])
```
或者添加多条记录

```
session.add_all([
fsuser(user_name='user1',user_pwd='123456',fsid='abcdefghijklmnopqresuvmxyz'),
fsuser(user_name='user3',user_pwd='123456',fsid='abcdefghijklmnopqresuvmxyz'),
host(hostname='host2.test.com',ip='10.1.35.24',fsid='1qazxsw23edcvfr4',type='samba')])
```

此时通过session的new方法 **获取新加的记录**
```.new
 Out[43]: IdentitySet([
host(hostname='host2.test.com',ip='10.1.35.24',fsid='1qazxsw23edcvfr4',type='samba'),
fsuser(user_name='user1',user_pwd='123456',fsid='abcdefghijklmnopqresuvmxyz'),
fsuser(user_name='user1',user_pwd='123456',fsid='abcdefghijklmnopqresuvmxyz')])
```
***注意***：此时并没有实际写入数据库

仅仅当执行commit之后才会实际写入数据库

    session.commit()

## 6.修改
数据项修改，需要先查询以获得前面定义的Table的实例
`all()`获取所有符合条件的数据项，返回结果是一个列表，`and_`用于定义多个查找匹配项，举例如下：
```
for user2 in session.query(fsuser).filter(and_(fsuser.user_name=='user7',\
fsuser.user_pwd=='itzhoulin',fsuser.fsid=='abcdefghijklmnopqresuvmxyz')).all():
#或者
for user2 in session.query(fsuser).filter(fsuser.user_name=='user7').all():
    print user2
print user2.user_pwd
```

修改后提交到数据库
```
print user2
user2.user_pwd='yifangzhoulin'
print user2
session.commit()
```
## 7.回滚
session的回滚功能，比较赞
```
host1.hostname='host1-2.test.com'
#修改后查询一下
session.query(host).filter(host.ip=='192.168.35.23').all()
 Out[52]: [host(hostname='host1-2.test.com',ip='192.168.35.23',fsid='abcdefghijklmnopqresuvmxyz',type='samba')]
#回滚后再查询一下：
session.rollback()
session.query(host).filter(host.ip=='192.168.35.23').get(1)
 Out[60]: host(hostname='host1.test.com',ip='192.168.35.23',
    fsid='abcdefghijklmnopqresuvmxyz',type='samba')
```

## 8.查询操作

查询的方法很多，一看就能明白，选择合适的
```
下面result的值即为一个string类型的，实际为ip的值，由此可见all()返回列表，取一条记录中的某一个值直接点号
result= session.query(host).filter(and_(host.fsid=='1qazxsw23edcvfr4',host.hostname=='host3')).all()[0].ip

for instance in session.query(fsuser).order_by(fsuser.id):
    print instance.name, instance.fullname

for name, fullname in session.query(fsuser.user_name, fsuser.fullname):
    print name, fullname

for row in session.query(fsuser, fsuser.user_name).all():
    print row.User, row.name

关于`label`还不是很清楚，我的理解就是给某一项一个别名
for row in session.query(fsuser.user_name.label('name_label')).all():
    print(row.name_label)

这里的aliased就是专门给Table设置别名的
from sqlalchemy.orm import aliased
user_alias = aliased(fsuser, name='user_alias')
for row in session.query(user_alias, user_alias.name).all():
    print row.user_alias

query返回的是列表，这个比较好理解，查询返回排序
for u in session.query(fsuser).order_by(fsuser.id)[1:3]:
    print u
```
这里需要注意的就是filter_by和filter两者的区别
```
for name, in session.query(fsuser.user_name).filter_by(fullname='Ed Jones'):
    print name
for name, in session.query(fsuser.user_name).filter(fsuser.fullname=='Ed Jones'):
    print name

for user in session.query(fsuser).filter(fsuser.user_name=='ed').filter(fsuser.fullname=='Ed Jones'):
    print user
```

## 9.删除操作
删除，一样的首先需要query查询获得一个实例
```
host1 = session.query(host).filter(and_(host.hostname=='host3',host.fsid == '1qazxsw23edcvfr4')).first()
session.delete(host1)
session.commit()
```

## 10.关闭连接
最好是在操作完成后关闭数据库连接
```
session.close()
```