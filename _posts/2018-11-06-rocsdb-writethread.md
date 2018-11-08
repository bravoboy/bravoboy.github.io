---
layout: post
title:  "rocksdb write_thread 解析"
categories: rocksdb
---

* content
{:toc}

## 背景
当多线程并发写入rocksdb，所有的写入操作会构造一个write_batch，但是内部写入wal文件还是串行操作，保证写入操作的顺序性，写memtable可以并发写入，对于同时写入的多个write_batch，rocksdb内部会合并成一个大的batch，提供写入性能。
同时外部写入的时候，可能会发生memtable写满切换以及wal文件写满切换操作，这个时候外部写入操作需要暂停等待切换操作完成(这个也是rocksdb毛刺的原因之一)，这些工作都是由write_thread完成，同时还负责保证写入操作的顺序性。

#### 状态
1. STATE_INIT = 1
2. STATE_GROUP_LEADER = 2
3. STATE_MEMTABLE_WRITER_LEADER = 4
4. STATE_PARALLEL_MEMTABLE_WRITER = 8
5. STATE_COMPLETED = 16
6. STATE_LOCKED_WAITING = 32

初始是init状态，只有leader才有权限发起写操作，STATE_PARALLEL_MEMTABLE_WRITER专门给并发写memtable用的。
#### 细节
内部会维护write group，把多个batch合并一起写入。如果设置了enable_pipelined_write，那么写wal和写memtable分开，可以提高写吞吐。batch如果是follower，那么只能等待leader唤醒。
#### 数据结构
```
struct Writer {
    WriteBatch* batch;
    bool sync;
    bool no_slowdown;
    bool disable_wal;
    bool disable_memtable;
    uint64_t log_used;  // log number that this batch was inserted into
    uint64_t log_ref;   // log number that memtable insert should reference
    WriteCallback* callback;
    bool made_waitable;          // records lazy construction of mutex and cv
    std::atomic<uint8_t> state;  // write under StateMutex() or pre-link
    WriteGroup* write_group;
    SequenceNumber sequence;  // the sequence number to use for the first key
    Status status;            // status of memtable inserter
    Status callback_status;   // status returned by callback->Callback()
    std::aligned_storage<sizeof(std::mutex)>::type state_mutex_bytes;
    std::aligned_storage<sizeof(std::condition_variable)>::type state_cv_bytes;
    Writer* link_older;  // prev指针
    Writer* link_newer;  // next指针
};
    
```

```
struct WriteGroup {
    Writer* leader = nullptr;
    Writer* last_writer = nullptr;
    SequenceNumber last_sequence;
    // before running goes to zero, status needs leader->StateMutex()
    Status status;
    std::atomic<size_t> running;
    size_t size = 0;
};

class WriteThread {
private:
  // 并发写memtable
  const bool allow_concurrent_memtable_write_;
  // 分开写memtable和wal
  const bool enable_pipelined_write_;
  // Points to the newest pending writer. Only
  // leader can remove elements, adding can 
  // be done lock-free by anybody.
  std::atomic<Writer*> newest_writer_;
  // Points to the newest pending memtable writer.
  // Used only when pipelined write is enabled.
  std::atomic<Writer*> newest_memtable_writer_;
};

```


write_group单次批量写入队列，leader指向第一个写入的batch，last_write指向本批次最后一个写入的batch

#### 重点函数
- bool LinkOne(Writer* w, std::atomic<Writer*> * newest_writer);<br/>
  把w插入到newest_writer的list，newest_writer指向w，如果w前面没有元素了，w就是leader，返回true，线程安全
- bool LinkGroup(WriteGroup& write_group, std::atomic<Writer*> * newest_writer);<br/>
  把write group里面所有write的next指针解除，把write group里面的leader链接到newest_writer，如果leader前面没有元素了，返回true，线程安全
- uint8_t AwaitState(Writer* w, uint8_t goal_mask, AdaptationContext* ctx);<br/>
  等待w的状态 & goal_mask不为0
- BlockingAwaitState<br/>
  通过信号量唤醒，等待state状态 & mask != 0。
- void JoinBatchGroup(Writer* w);<br/>
  先调用LinkOne函数，把w插入list，如果w是leader的话直接返回，否则调用AwaitState函数，等待w是leader或者w是finish状态
- size_t EnterAsBatchGroupLeader(Writer* leader, WriteGroup* write_group);<br/>
  构造一个write group出来，可以修改这个函数，不让小batch合并成大batch一起写入，空batch也不能合并
- CreateMissingNewerLinks<br/>
  调用LinkOne只是把双向链表里面的一个prev指针设置了，这个函数的功能是设置next指针
- void ExitAsBatchGroupLeader(WriteGroup& write_group, Status status);<br/>
  1. 如果开启enable_pipelined_write_选项，如果batch不写memtable，那么write操作完成，否则就调用LinkGroup函数，next指针解除，唤醒next_leader，等待leader成为STATE_MEMTABLE_WRITER_LEADER。
  2. 没有开启enable_pipelined_write_，唤醒next_leader，group里面所有的batch变成STATE_COMPLETED
- EnterAsMemTableWriter<br/>
  构造写memtable的write group, 如果有batch里面有merge操作，则不能合并，空batch也不能合并
- ExitAsMemTableWriter<br/>
  唤醒新的STATE_MEMTABLE_WRITER_LEADER，所有batch变成STATE_COMPLETED，~~有个地方没看懂： 为什么leader是最后变成STATE_COMPLETED？~~（__原因是：leader线程持有write group对象，如果leader线程先退出，write group就会被析构释放，其他的batch持有的write group的指针就变成野指针__）
- LaunchParallelMemTableWriters<br/>
  开启并发写memtable，会调用这个函数设置state=STATE_PARALLEL_MEMTABLE_WRITER
- CompleteParallelMemTableWriter<br/>
  最后一个batch return true然后调用ExitAsMemTableWriter，其他batch等待STATE_COMPLETED，最后一个batch(不一定是leader)会唤醒其他batch
- EnterUnbatched<br/>
  插入空batch到，等待成为leader，FlushMemtable，SwitchMemtable等时候会调用，调用这个函数时候线程持有db大锁。
- ExitUnbatched<br/>
  唤醒新的leader
- WaitForMemTableWriters<br/>
  构造一个空batch插入写memtable链表，等待成为leader，调用这个函数时候线程持有db大锁。
  
  
  
  
## 资料
1. http://mysql.taobao.org/monthly/2018/07/04/
2. http://kernelmaker.github.io/Rocksdb_pipeline_write
3. https://yq.aliyun.com/articles/409102
  
   