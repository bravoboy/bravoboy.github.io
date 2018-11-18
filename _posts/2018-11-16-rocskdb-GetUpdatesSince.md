---
layout: post
title:  "rocksdb GetUpdatesSince 源码解析"
categories: rocksdb
---

* content
{:toc}

## wal文件格式
文件按照block划分，每个block默认是32k。<br/>
batch在wal文件格式：![](/images/wal_record.png)
1. 如果batch能完整放在block剩余空间中，类型是kFullType，
2. 如果batch比较大的时候，block剩余空间不能完整放下，<br/>
   先放部分数据在这个block(类型是kFirstType)，<br/>
   剩余数据放在下一个block，最后部分数据类型是kLastType，<br/>
   中间数据类型是kMiddleType。
## GetUpdatesSince作用

GetUpdatesSince可以通过seq获取wal里面对应的batch.<br/>
函数步骤: <br/>

1. 	先获取当前所有的wal文件
2. 读取每个文件开头第一条batch数据的seq号
3. 排序，2分查找把前面不符合范围的文件去掉，
4. 打开第一个文件，从头遍历找到>=seq的第一个batach，返回TransactionLogIterator迭代器

## 问题1
这个函数执行是没有加锁的，那么就有并发问题，可能第1步获取的文件被其他的compact线程删除了，然后第2步就会出错。这个时候是不能区分被删除的文件的seq是否大于调用GetUpdatesSince的seq号，如果小于调用GetUpdatesSince的seq号，那么被删除应该没问题。

## 问题2
TransactionLogIterator执行next函数的时候会判断seq号是否连续，如果不连续(部分操作没有写wal文件，disableWAL=true)会从文件头开始重新遍历文件，效率低下

## 修改

#### 问题1修改
0. 维护一个全局map，key是wal文件名，value是wal文件对应的第一个seq号
1. 每次创建新log文件的时候，写一行记录的时候就对map里面增加一行记录
2. 删除wal文件就从map里面删除对应的记录
3. GetUpdatesSince遍历这个map就可以了

对全局map的操作都是需要加锁

#### 问题2修改
对于不是连续的seq号，如果seq还在文件范围内，直接忽略原来的从头开始遍历的逻辑




