---
layout: post
title:  "mysql8.0 autoinc 重复排查"
categories: innodb
---

* content
{:toc}

## 问题背景
业务在线上数据库里面偶然发现有部分数据，刚insert插入，然后select出来发现值和自己入的对不上。业务逻辑里面并没有update操作，解析mysql binlog文件，发现数据确实是被update更新过了，数据库版本是mysql8.0。那么这个update操作到底是哪里来的？

## 排查过程
#### 分析问题
简化一下表结构，创建下面t1表，其中id是主键，并且是自增值，存储引擎是innodb
```
create table t1 (
id in not null auto_increment,
name varchar(1024),
age int,
primary key(id)) engine = innodb
```
insert sql语句
```
insert into t1 (id, name, age) values (...) on duplicate update age = values(age)
```
除了insert语句，并没有其他的update语句，看到sql语句，那么可以猜测走到on duplicate update逻辑里面了。但是这个表里面除了id主键以外，没有其他unique的索引，那么只能是id重复冲突了，问题就变成了auto_increment字段为什么会重复冲突。

#### 代码解析
先大致梳理一下innodb插入数据的过程
```
1. 先调用update_auto_increment获取autoinc值，更新table->autoinc
2. 再调用row_insert_for_mysql插入数据
3. 最后调用set_max_autoinc尝试更新table->autoinc
```
第一步分配autoinc值是要对table->autoinc_mutex加锁，保证并发的插入的时候不会分配出来重复的id。<br/>
第二步row_insert_for_mysql里面会有duplicate key检查。<br/>
那么按理说不会出现autoinc重复的问题。再次翻看业务的sql语句发现有一些insert里面id字段对应的是null，有一些insert语句id字段对应的值不是null，是用户传入的一个值。
对于传入null字段，系统是会自动分配递增的autoinc <br/>
如果是用户传入了一个id值，系统就不会再分配了，所以猜测问题是用户传入的值和系统分配的值冲突了，导致触发了update逻辑 <br/>

## 验证猜测
假设当前表的autoinc值最大值是2，下一个分配的值是3，用2个session同时插入数据证实我的猜测
```
session1插入insert into t1 (id, name, age) values (null, ‘c’, 3);
session2插入insert into t1 (id, name, age) values (3, ‘cc’, 33);
```
需要使用gdb调试多线程，网上资料很多，我就不再赘述。然后在ha_innobase::write_row函数开始加断点。这里我们可以观察到2个session都在卡在断点。
1. 我们先让session2执行，因为自带了id值，所以不需要系统分配，执行完row_insert_for_mysql，观察返回值err，发现是成功的。
2. 再然后我们让session1执行，在第一步系统分配出来的autoinc值确实就是3，在执行row_insert_for_mysql前切回到session2线程
3. 我们让session2继续执行，释放锁。
4. 切回session1线程，执行row_insert_for_mysql，观察返回值err发现是DB_DUPLICATE_KEY，证明了上面的猜测。

## 解决方案
和业务沟通发现以后，发现业务的数据有2部分来源
1. 历史库老数据，对于这部分数据，insert的时候带id，同时希望插入到新库里面保持id不变
2. 线上生产的新数据，这部分数据，insert不带id，使用系统分配自增值

针对这种问题，只需要保证这两部分数据id不冲突就好了，那么可以先查一下历史库最大的id值多少，然后调用alter命令修改表的auto inc值，保证比历史库最大的值还要大1，就可以避免冲突。

