---
layout: post
title:  "dead lock2"
categories: rocksdb
---

* content
{:toc}

## 背景
最近线上遇到2次报警，登录机器查看，发现所有线程cpu百分比，不响应任何命令，和之前的排查过的[死锁](https://bravoboy.github.io/2018/09/11/rocksdb-deadlock/) 现象。<br/>
直接使用pstack命令查看堆栈信息，发现确实出现死锁。<br/>

## 死锁1

#### 分析过程
```
Thread 16 (Thread 0x7fd90920e700 (LWP 20244)):
#0  0x00007fda6e33d222 in pthread_spin_lock () from /lib64/libpthread.so.0
#1  0x00000000008aa7b5 in lock (this=0x2dc4024) at /home/xiaoju/bigdata-storage/fusion.r2/src/spin_lock.h:16
#2  lock_guard (__m=..., this=<synthetic pointer>) at /opt/gcc-5.4/include/c++/5.4.0/mutex:386
#3  ReplicationController::role (this=0x2dc4000) at /home/xiaoju/bigdata-storage/fusion.r2/src/replication.cpp:939
#4  0x000000000081179f in get_self_status () at /home/xiaoju/bigdata-storage/fusion.r2/src/replication.h:666
#5  cmd_proc (req=req@entry=0x6beab6b40) at /home/xiaoju/bigdata-storage/fusion.r2/src/cmds.cpp:191
#6  0x00000000008cbe4f in work_process (work=0x3246000) at /home/xiaoju/bigdata-storage/fusion.r2/src/resp_server.cpp:1588
#7  0x0000000000ca2cc4 in event_persist_closure (ev=<optimized out>, base=0x31618c0) at event.c:1629
#8  event_process_active_single_queue (base=base@entry=0x31618c0, activeq=0x5666cd20, max_to_process=max_to_process@entry=2147483647, endtime=endtime@entry=0x0) at event.c:1688
#9  0x0000000000ca366f in event_process_active (base=0x31618c0) at event.c:1789
#10 event_base_loop (base=0x31618c0, flags=flags@entry=0) at event.c:2012
#11 0x0000000000ca38b7 in event_base_dispatch (event_base=<optimized out>) at event.c:1823
#12 0x00000000008d109a in worker_loop (data=0x3246000) at /home/xiaoju/bigdata-storage/fusion.r2/src/resp_server.cpp:1671
#13 0x0000000000c85245 in g_thread_proxy (data=0xbb68de0) at gthread.c:778
#14 0x00007fda6e338dc5 in start_thread () from /lib64/libpthread.so.0
#15 0x00007fda6d73e21d in clone () from /lib64/libc.so.6
Thread 15 (Thread 0x7fd902e0d700 (LWP 20245)):
#0  0x00007fda6d73e7f3 in epoll_wait () from /lib64/libc.so.6
#1  0x0000000000cac6f4 in epoll_dispatch (base=0x32ac000, tv=<optimized out>) at epoll.c:465
#2  0x0000000000ca3495 in event_base_loop (base=0x32ac000, flags=flags@entry=0) at event.c:1998
#3  0x0000000000ca38b7 in event_base_dispatch (event_base=<optimized out>) at event.c:1823
#4  0x00000000008d109a in worker_loop (data=0x329a000) at /home/xiaoju/bigdata-storage/fusion.r2/src/resp_server.cpp:1671
#5  0x0000000000c85245 in g_thread_proxy (data=0xbb68e30) at gthread.c:778
#6  0x00007fda6e338dc5 in start_thread () from /lib64/libpthread.so.0
#7  0x00007fda6d73e21d in clone () from /lib64/libc.so.6
```

```
Thread 14 (Thread 0x7fd8fca0c700 (LWP 20246)):
#0  0x00007fda6e33c6d5 in pthread_cond_wait@@GLIBC_2.3.2 () from /lib64/libpthread.so.0
#1  0x0000000000cad515 in evthread_posix_cond_wait (cond_=0x2a7e570, lock_=0x2a7e210, tv=<optimized out>) at evthread_pthread.c:158
#2  0x0000000000c9fde6 in event_del_nolock_ (ev=ev@entry=0x38b3680, blocking=blocking@entry=2) at event.c:2896
#3  0x0000000000ca00b2 in event_del_ (ev=0x38b3680, blocking=2) at event.c:2783
#4  0x0000000000ca012e in event_del (ev=0x38b3680) at event.c:2792
#5  event_free (ev=0x38b3680) at event.c:2233
#6  0x00000000008b0724 in ReplicationController::Promote (this=0x2dc4000) at /home/xiaoju/bigdata-storage/fusion.r2/src/replication.cpp:1039
#7  0x00000000008b4841 in slaveof_impl (ip=<optimized out>, port=<optimized out>) at /home/xiaoju/bigdata-storage/fusion.r2/src/replication.cpp:3002
#8  0x0000000000815d14 in slaveof_cmd (req=0x4a993140) at /home/xiaoju/bigdata-storage/fusion.r2/src/cmds.cpp:5871
#9  0x00000000008904a6 in FusionCommand::process (this=this@entry=0x2dc9758, req=0x4a993140) at /home/xiaoju/bigdata-storage/fusion.r2/src/fusion_command.cpp:19
#10 0x000000000081182f in cmd_proc (req=req@entry=0x4a993140) at /home/xiaoju/bigdata-storage/fusion.r2/src/cmds.cpp:251
#11 0x00000000008cbe4f in work_process (work=0x3288000) at /home/xiaoju/bigdata-storage/fusion.r2/src/resp_server.cpp:1588
#12 0x0000000000ca2cc4 in event_persist_closure (ev=<optimized out>, base=0x32ac2c0) at event.c:1629
#13 event_process_active_single_queue (base=base@entry=0x32ac2c0, activeq=0x5666ce10, max_to_process=max_to_process@entry=2147483647, endtime=endtime@entry=0x0) at event.c:1688
#14 0x0000000000ca366f in event_process_active (base=0x32ac2c0) at event.c:1789
#15 event_base_loop (base=0x32ac2c0, flags=flags@entry=0) at event.c:2012
#16 0x0000000000ca38b7 in event_base_dispatch (event_base=<optimized out>) at event.c:1823
#17 0x00000000008d109a in worker_loop (data=0x3288000) at /home/xiaoju/bigdata-storage/fusion.r2/src/resp_server.cpp:1671
#18 0x0000000000c85245 in g_thread_proxy (data=0xbb68e80) at gthread.c:778
#19 0x00007fda6e338dc5 in start_thread () from /lib64/libpthread.so.0
#20 0x00007fda6d73e21d in clone () from /lib64/libc.so.6
```

#### 结论
event_process_active_single_queue回调用户函数卡在业务代码里面的锁,
业务代码另外一个线程先加锁,然后调用了event_free,这个里面又卡在信号量通知，形成死锁。
针对有资源竞争的场景，尽量放到同一个线程执行。

## 死锁2

#### 分析过程
pstack现场没有及时保存，这里代码文字分析一下。<br/>
rocksdb在启动一个Manual-Compact的时候需要先加全局锁，然后调用CompactRange判断对应的区间里面是否和其他compact存在冲突。如果存在冲突就等待<br/>
```
Status DBImpl::RunManualCompaction(
    ColumnFamilyData* cfd, int input_level, int output_level,
    const CompactRangeOptions& compact_range_options, const Slice* begin,
    const Slice* end, bool exclusive, bool disallow_trivial_move,
    uint64_t max_file_num_to_ignore) {
	//只展示关键代码
	InstrumentedMutexLock l(&mutex_);
	while (!manual.done) {
		assert(HasPendingManualCompaction());
		manual_conflict = false;
		Compaction* compaction = nullptr;
		if (ShouldntRunManualCompaction(&manual) || (manual.in_progress == true) ||
			scheduled ||
			(((manual.manual_end = &manual.tmp_storage1) != nullptr) &&
			 ((compaction = manual.cfd->CompactRange(
				   *manual.cfd->GetLatestMutableCFOptions(), manual.input_level,
				   manual.output_level, compact_range_options, manual.begin,
				   manual.end, &manual.manual_end, &manual_conflict,
				   max_file_num_to_ignore)) == nullptr &&
			  manual_conflict))) {
		  // exclusive manual compactions should not see a conflict during
		  // CompactRange
		  assert(!exclusive || !manual_conflict);
		  // Running either this or some other manual compaction
		  bg_cv_.Wait();
		}
	}
}
```
我们代码里面自定义了Filter类,在过滤的时候会调用write接口,向db写入数据，写操作是需要申请加锁的。<br/>
有一个正在跑着的compact线程，在filter过滤数据的时候，会写数据，卡在全局锁上面。<br/>
新的Manual-Compact持有全局锁，但是在等待其他compact完成，又卡在信号通知上面。<br/>
修改方法就是把自定义Filter类中写操作，放到异步队列当作，不在本线程。