---
layout: post
title:  "rocksdb SwitchMemtable 源码解析"
categories: rocksdb
---

* content
{:toc}

## 背景
#### 函数作用
顾名思义，函数的作用是把当前memtable变成immutable，然后切换到新的memtable
#### 调用场景
1. ScheduleFlushes
2. HandleWALFull
3. HandleWriteBufferFull
4. FlushMemTable

## 代码分析
一：调用函数条件：<br/>
1. 当前线程持有db的大锁<br/>
2. 当前线程是写wal文件的leader<br>
如果开启了enable_pipelined_write选项(写wal和写memtable分开), 那么同时要等到成为写memtable的leader<br/>
二：判断当前wal文件是否为空，如果不为空就创建新的wal文件(recycle_log_file_num选项开启就复用)，然后构建新memtable。刷盘当前wal文件<br/>
三：把之前的memtable变成immutable，然后切到新memtable<br>
四：调用InstallSuperVersionAndScheduleWork函数，这个函数会更新super_version_





