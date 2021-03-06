---
layout: post
title:  "rocksdb Log解析"
categories: rocksdb
---

* content
{:toc}

## 内容
对于一个程序，日志是非常重要的，它可以帮忙我们分析程序运行的状态。rocksdb运行的时候会打印很多日志，这里说的log不是WAL文件，从日志中我们可以看到rocksdb的状态，所以我们有必要分析一下rocksdb的日志。rocksdb支持用户自定义log类。我们这里只分析默认的log行为。option里面有几个选项关于log文件，1. max_log_file_size 表示log文件最大值，超过以后会生成新log文件，默认0表示不限制。2. log_file_time_to_roll > 0 表示根据时间分割log。3. info_log_level 定义log级别，默认info。4. db_log_dir log文件存储目录。

```
2019/06/10-16:54:30.163319 7f893640da60 RocksDB version: 6.1.1
2019/06/10-16:54:30.163416 7f893640da60 Git sha rocksdb_build_git_sha:de76909464a99d5ed60e61844e9cd0371ca350fe
2019/06/10-16:54:30.163421 7f893640da60 Compile date Jun 10 2019
2019/06/10-16:54:30.163425 7f893640da60 DB SUMMARY
2019/06/10-16:54:30.163444 7f893640da60 SST files in /tmp/rocksdb_transaction_example dir, Total Num: 0, files: 
2019/06/10-16:54:30.163447 7f893640da60 Write Ahead Log file in /tmp/rocksdb_transaction_example:
```
log文件开头是时间戳，精确到us，后面是thread_id。
启动的时候会打印出rocksdb version，git commit_id，编译时间，sst文件目录，wal文件目录。然后会打印出来option选项值，这个可以方便让我们知道启动rocksdb实例相关参数。
接着会打印出来支持的压缩算法。
```
 10 2019/07/07-17:03:30.730878 7f340004fa60                         Options.error_if_exists: 0
 11 2019/07/07-17:03:30.730881 7f340004fa60                       Options.create_if_missing: 1
 12 2019/07/07-17:03:30.730883 7f340004fa60                         Options.paranoid_checks: 1
 13 2019/07/07-17:03:30.730885 7f340004fa60                                     Options.env: 0xb11c00
 14 2019/07/07-17:03:30.730890 7f340004fa60                                Options.info_log: 0x1a316c0
 15 2019/07/07-17:03:30.730893 7f340004fa60                Options.max_file_opening_threads: 16
 16 2019/07/07-17:03:30.730895 7f340004fa60                              Options.statistics: (nil)
 17 2019/07/07-17:03:30.730897 7f340004fa60                               Options.use_fsync: 0
 18 2019/07/07-17:03:30.730901 7f340004fa60                       Options.max_log_file_size: 1073741824
 19 2019/07/07-17:03:30.730904 7f340004fa60                  Options.max_manifest_file_size: 1073741824
 ...
 77 2019/07/07-17:03:30.731052 7f340004fa60 Compression algorithms supported:
 78 2019/07/07-17:03:30.731056 7f340004fa60   kZSTDNotFinalCompression supported: 0
 79 2019/07/07-17:03:30.731058 7f340004fa60   kZSTD supported: 0
 80 2019/07/07-17:03:30.731061 7f340004fa60   kXpressCompression supported: 0
 81 2019/07/07-17:03:30.731063 7f340004fa60   kLZ4HCCompression supported: 0
 82 2019/07/07-17:03:30.731066 7f340004fa60   kLZ4Compression supported: 0
 83 2019/07/07-17:03:30.731069 7f340004fa60   kBZip2Compression supported: 1
 84 2019/07/07-17:03:30.731071 7f340004fa60   kZlibCompression supported: 1
 85 2019/07/07-17:03:30.731074 7f340004fa60   kSnappyCompression supported: 0
 86 2019/07/07-17:03:30.731082 7f340004fa60 Fast CRC32 supported: Supported on x86
```
接着会打印出来具体的每个cf相关的配置参数
```
 88 2019/07/07-17:03:30.731473 7f340004fa60 [/column_family.cc:477] --------------- Options for column family [default]:
 89 2019/07/07-17:03:30.731483 7f340004fa60               Options.comparator: leveldb.BytewiseComparator
 90 2019/07/07-17:03:30.731485 7f340004fa60           Options.merge_operator: None
 91 2019/07/07-17:03:30.731499 7f340004fa60        Options.compaction_filter: None
 92 2019/07/07-17:03:30.731502 7f340004fa60        Options.compaction_filter_factory: None
 93 2019/07/07-17:03:30.731505 7f340004fa60         Options.memtable_factory: SkipListFactory
 94 2019/07/07-17:03:30.731507 7f340004fa60            Options.table_factory: BlockBasedTable
 95 2019/07/07-17:03:30.731557 7f340004fa60            table_factory options:   flush_block_policy_factory: FlushBlockBySizePolicyFactory (0x1a292f0)
```
rocksdb里面还定义了event_logger，表示某个事件的日志。格式如下，job_id是原子变量，单调递增。
```
2019/07/07-17:03:30.733725 7f340004fa60 EVENT_LOG_v1 {"time_micros": 1562490210733707, "job": 1, "event": "recovery_started", "log_files": [13, 18, 22,27]}
```
rocksdb里面有一个参数Options.stats_dump_period_sec 默认设置了600s，调用一次DBImpl::DumpStats()函数，打印db状态。里面信息有每个cf的数据分布情况和读耗时情况。
```
2019/07/05-15:51:01.096971 7fd3474f3700 [WARN] [db/db_impl.cc:489]
** DB Stats **
Uptime(secs): 618257.1 total(从开始统计到当前时间秒数), 304.1 interval(距离上次统计间隔)
下面这3行是累积值
Cumulative writes: 392M writes(写入总batch数), 41G keys(写入key个数), 392M commit groups(组提交次数), 1.0 writes per commit group(平均每次组提交batch个数), ingest: 3324.02 GB(从开始统计累积写入数据量), 5.51 MB/s(平均每秒写入数据量)
Cumulative WAL: 392M writes(写wal文件次数), 0 syncs(sync方式写入次数), 392027713.00 writes per sync, written: 3321.68 GB(累积写入WAL文件数据量), 5.50 MB/s(平均每秒写入WAL文件数据量)
Cumulative stall: 00:00:0.000 H:M:S, 0.0 percent(因为stalling原因导致写耽搁时间,累积值)
下面这3行是间隔值
Interval writes: 309K writes, 28M keys, 309K commit groups, 1.0 writes per commit group, ingest: 2368.70 MB, 7.79 MB/s
Interval WAL: 309K writes, 0 syncs, 309684.00 writes per sync, written: 2.31 MB, 7.79 MB/s
Interval stall: 00:00:0.000 H:M:S, 0.0 percent

** Compaction Stats [default] **
Level    Files   Size     Score Read(GB)  Rn(GB) Rnp1(GB) Write(GB) Wnew(GB) Moved(GB) W-Amp Rd(MB/s) Wr(MB/s) Comp(sec) Comp(cnt) Avg(sec) KeyIn KeyDrop
----------------------------------------------------------------------------------------------------------------------------------------------------------
  L0      0/0    0.00 KB   0.0      0.0     0.0      0.0       0.1      0.1       0.0   1.0      0.0     16.6         4        95    0.045       0      0
  L1      2/0   618.65 KB   0.0     72.2     0.1     72.1      72.1     -0.0       0.0 1038.4     28.1     28.1      2630      4171    0.631   1492M  2906K
  L2      1/0    1.21 KB   0.0      0.0     0.0      0.0       0.0      0.0       0.0   0.0      0.0      0.0         0         0    0.000       0      0
 Sum      3/0   619.86 KB   0.0     72.2     0.1     72.1      72.1      0.0       0.0 1039.4     28.1     28.0      2634      4266    0.617   1492M  2906K
 Int      0/0    0.00 KB   0.0      0.0     0.0      0.0       0.0      0.0       0.0   0.0      0.0      0.0         0         0    0.000       0      0
Uptime(secs): 618257.1 total, 304.1 interval
Flush(GB): cumulative 0.069(累积flush数据量), interval 0.000(间隔flush数据量)
AddFile(GB): cumulative 0.000(累积Fastload导入数据量), interval 0.000(间隔Fastload导入数据量)
AddFile(Total Files): cumulative 0(累积Fastload导入文件个数), interval 0(间隔Fastload导入文件个数)
AddFile(L0 Files): cumulative 0(累积Fastload导入L0文件个数), interval 0(间隔Fastload导入L0文件个数)
AddFile(Keys): cumulative 0(累积Fastload导入key数), interval 0(间隔Fastload导入key数)
Cumulative compaction: 72.14 GB write(compact写数据量，包括memtable Flush到L0), 0.12 MB/s write, 72.17 GB read(compact读文件数据量，Memtable Flush到L0没有读数据量), 0.12 MB/s read, 2634.2 seconds(compact耗时，包括memtable Flush到L0)
Interval compaction: 0.00 GB write, 0.00 MB/s write, 0.00 GB read, 0.00 MB/s read, 0.0 seconds
Stalls(count): 0 level0_slowdown(发送次数), 0 level0_slowdown_with_compaction, 0 level0_numfiles, 0 level0_numfiles_with_compaction, 0 stop for pending_compaction_bytes, 0 slowdown for pending_compaction_bytes, 0 memtable_compaction, 0 memtable_slowdown, interval 0 total count

** File Read Latency Histogram By Level [default] **
** Level 0 read latency histogram (micros):
Count: 5905 Average: 20.5529  StdDev: 89.68
Min: 0  Median: 4.7716  Max: 2900
Percentiles: P50: 4.77 P75: 6.91 P99: 378.50 P99.9: 1054.75 P99.99: 2704.75
------------------------------------------------------
[       0,       1 )     1198  20.288%  20.288% ####
[       1,       2 )      158   2.676%  22.964% #
[       2,       3 )      101   1.710%  24.674%
[       3,       4 )      470   7.959%  32.633% ##
[       4,       5 )     1329  22.506%  55.140% #####
[       5,       6 )      783  13.260%  68.400% ###
```
说一下上面的直方图统计，是一种近似统计，事前生成槽位数组，然后判断value落到那个区间，对应的区间count计数+1。<br/>
输出格式从左到右分别是[left, right)  count计数，百分比，累积百分比，#字符串长度表示近似百分比。<br/> 
然后会打印rocksdb里面的一些统计，比如说block_cache的命中率等信息，可以帮助我们调整block_cache大小。
```
2019/07/05-15:51:01.100587 7fd3474f3700 [WARN] [db/db_impl.cc:414] STATISTICS:
 rocksdb.block.cache.miss COUNT : 5000579768
rocksdb.block.cache.hit COUNT : 274057516
rocksdb.block.cache.add COUNT : 2992615
rocksdb.block.cache.add.failures COUNT : 0
rocksdb.block.cache.index.miss COUNT : 1702567
rocksdb.block.cache.index.hit COUNT : 190156548
rocksdb.block.cache.index.add COUNT : 1702567
rocksdb.block.cache.index.bytes.insert COUNT : 723660151408
rocksdb.block.cache.index.bytes.evict COUNT : 721723192912
rocksdb.block.cache.filter.miss COUNT : 4646930
rocksdb.block.cache.filter.hit COUNT : 67442573
```
rocksdb还提供了GetProperty(ColumnFamilyHandle* column_family,const Slice& property, std::string* value);函数实时获取db状态。<br/>

```
2019/10/09-16:44:25.842335 7f3b9cb19700 (Original Log Time 2019/10/09-16:44:25.841912) [db/db_impl_compaction_flush.cc:140] [default] Level summary: base level 1 max bytes base 268435456 files[10 147 516 762 344 0 0] max score 27.21 (1386 files need compaction)
```
这个log显示每层文件个数，以及多少个文件需要compaction<br/>

```
2019/10/09-16:44:29.126771 7f3ba931b700 (Original Log Time 2019/10/09-16:44:29.125613) [db/compaction_job.cc:622] [default] compacted to: base level 1 max bytes base 268435456 files[10 146 516 762 344 0 0] max score 26.95 (1383 files need compaction), MB/sec: 61.5 rd, 52.5 wr, level 2, files in(1, 3) out(3) MB in(64.3, 304.4) out(314.7), read-write-amplify(10.6) write-amplify(4.9) OK, records in: 1015689, records dropped: 24192
```
这个log除了显示每层文件个数，还显示了读写byte数，写byte总数就是输出文件总大小，读写放大倍数，写放大倍数=写byte总数/非输出level的byte总数，读写放大倍数=写总数+输入byte总数/非输出level的byte总数，compaction输入key个数，compaction过程中丢弃key个数
