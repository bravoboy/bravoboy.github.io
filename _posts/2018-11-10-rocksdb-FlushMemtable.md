---
layout: post
title:  "rocksdb FlushMemtable 源码解析"
categories: rocksdb
---

* content
{:toc}

## 背景
rocksdb数据写入的时候，先写入Wal文件和memtable。等memtable大小达到限制以后，就把memtable变成immutable，后台flush线程会把immutable刷到磁盘L0文件，释放immutable内存。多个cf对应同一个wal文件，wal文件只能串行写入。


#### 函数调用
![](/images/call.png)

1. HandleWALFull
   多cf模式，总的wal文件大小超过(max_total_wal_size>0)||(4 * write_buffer_size * max_write_buffer_number)，需要写入新的wal文件，写memtable会对应到一个wal文件，wal文件要切换了memtable也要切换
   
2. SwitchMemtable
   如果设置了enable_pipelined_write等待成为写memtable的leader，如果当前wal文件不为空，那么创建一个新的wal文件，后续所有的写操作写到新的wal文件里面。遍历所有的cf，cf的memtable为空的设置log_number为新的值。

#### 数据结构
```
class MemTable {
enum FlushStateEnum { FLUSH_NOT_REQUESTED, FLUSH_REQUESTED, FLUSH_SCHEDULED };
int refs_; //引用计数
unique_ptr<MemTableRep> table_; //实际存储数据的table
std::atomic<uint64_t> data_size_;  //内存大小
std::atomic<uint64_t> num_entries_; //key个数
std::atomic<uint64_t> num_deletes_; //delete类型key个数
std::atomic<SequenceNumber> first_seqno_; //第一个插入key的seq号
std::atomic<SequenceNumber> earliest_seqno_; //创建这个memtable的时候db的seq号
SequenceNumber creation_seq_;  //创建这个memtable的时候db的seq号，如果memtable里面没有数据的时候这个会被更新到最新seq号
std::atomic<FlushStateEnum> flush_state_; //flush状态
//FLUSH_NOT_REQUESTED 不需要flush
//FLUSH_REQUESTED 需要被flush但是还没有轮到
//FLUSH_SCHEDULED 已经计划flush
};
```
#### 配置解析
write_buffer_size 控制单个memtable大小
db_write_buffer_size 控制所有的memtable总和大小





WriteBufferManager 管理memtable总大小。db_write_buffer_size = 0表示不限制总的memtable大小。

