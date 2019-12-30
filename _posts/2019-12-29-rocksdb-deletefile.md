---
layout: post
title:  "rocksdb delete file"
categories: rocksdb
---

* content
{:toc}

## 背景
2019年最后一篇文章，今天来说一下rocksdb里面文件删除相关逻辑。rocksdb的多版本key存在，决定了需要gc功能，这就少不了删除文件。<br/>
主要删除文件有2个地方：
1. flush memtable以后删除wal文件 <br/>
2. compact以后删除老的sst文件 <br/>

## 删WAL
为了保证数据不丢失，wal文件删除必须在flush memtable以后操作。<br/>
先说一下哪些条件会触发flush:
1. memtable超过设置大小限制
2. 主动调用flush接口
3. WAL目录超过设置大小限制
4. 超过db_write_buffer_size设置大小 <br/>

SwitchMemtable函数会判断是否需要创建新的wal文件，依据是当前的wal文件是否为空，为空的就复用。否则就创建新文件<br/>
新文件会append到logs_和alive_log_files_这2个数据结构里面。
先来说一下这2个数据结构：他们保存的都是活跃wal文件，好像除了2pc场景以后，没有区别，有知道的朋友可以告诉我。<br/>
每个ColumnFamily(简称cf)都会存储自己的最小log文件号，当flush memtable以后会更新这个文件号<br/>
每个memtable可以对应多个wal文件，原因是memtable没有写满，可能其他cf正在做switchmemtable。<br/>
![pic2](/images/rocksdb_delete2.png)

## FindObsoleteFiles
这个函数的功能是扫描需要删除的文件，然后交给PurgeObsoleteFiles删除。如果rocksdb构建checkpoint的时候，会在这个函数里面直接return。<br/>
**这个函数会在全局锁的保护范围内，所以这个函数不能做太重的事情，否则会影响rocksdb的读写性能** <br/>
rocksdb默认6个小时会扫描db的全量文件，这个全局扫描在文件数特别多场景下会有性能问题。<br/>
下图是写了一个简单c程序扫描目录文件统计耗时:
![pic1](/images/rocksdb_delete1.png)

可以看到当文件数达到百万级别的时候耗时就非常大了，所以不推荐单个rocksdb存储特别多的数据，当然可以通过调整文件大小来减少文件数，但是这样做也会有其他的性能问题。<br/>
至于FindObsoleteFiles函数为什么要全量扫描文件，我想这块更多的是为了防止代码出现bug导致文件不能被删除。<br/>
同时在这个函数里面会获取每个cf最小的log文件号，然后判断logs_和alive_log_files_里面是否有文件需要删除。<br/>

## PurgeObsoleteFiles
这个函数会二次校验文件是否可以删除，同时也支持异步删除。<br/>

## CleanupIteratorState
迭代器释放的时候会调用这个函数释放资源，比如说删除sst文件。<br/>
可以设置background_purge_on_iterator_cleanup=true异步的释放资源，减少阻塞用户请求时间 <br/>
具体的调用逻辑看下图:
![pic3](/images/rocksdb_delete3.png)

