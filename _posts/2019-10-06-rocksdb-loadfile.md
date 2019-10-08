---
layout: post
title:  "rocksdb load file"
categories: rocksdb
---

* content
{:toc}

## 背景
LSM树的结构是MVCC多版本存在，这种结构非常方便快速导入数据文件。新数据事先合并成底层存储文件，通过接口直接导入，读取的时候优先读取新数据就可以保证数据准确性。Hbase也是LSM树结构，它具备blukload 快速导入数据功能。rocksdb同样也具有这个功能。今天研究一下rocksdb是怎么实现这个功能的。

## 使用
```
virtual Status IngestExternalFile(
      ColumnFamilyHandle* column_family,
      const std::vector<std::string>& external_files,
      const IngestExternalFileOptions& ingestion_options) override;
```
使用IngestExternalFile接口，参数是column_family, 文件名和options就可以导入文件到db中，使用方式很简单。<br/>
重点看一下option里面有哪些配置项
```
// IngestExternalFileOptions is used by IngestExternalFile()
struct IngestExternalFileOptions {
  //导入文件和rocksdb数据目录在同一个文件系统的时候，一般设置为true
  bool move_files = false;
  // 设置为true，可以保证导入的key的seq比db中已有的快照都大
  bool snapshot_consistency = true;
  // 如果设置false，文件自带的seq不能用的时候，那么直接返回导入失败
  bool allow_global_seqno = true;
  // 如果设置false，导入的文件如果和memtable有overlap，那么直接返回导入失败
  bool allow_blocking_flush = true;
  // 如果设置为true，那么会把文件导入到最低层level，前提是和已有文件范围不冲突，冲突就返回导入失败，适合空db初始化导入数据
  bool ingest_behind = false;
  //设置为true，会打开导入的sst文件，然后修改seq号，这个参数涉及到版本兼容问题，rocksdb未来会设置默认false
  bool write_global_seqno = true;
  //设置为true，会打开每个导入文件，对每个block进行crc校验，会影响导入速度
  bool verify_checksums_before_ingest = false;
};
```

对于一个空的db，全量导入数据的时候，可以所有的sst文件的seq都是0。ingest_behind可以设置为true，write_global_seqno设置false


## 实现
1. 参数检查
2. 申请文件号，rocksdb里面所有的文件名都是唯一的，并且单调递增。这个过程会对整个rocksdb加锁。
3. 读取文件找到最大和最小的key, 分配文件号
4. 会等待成为write leader
5. 判断和memtable是否overlap，如果有的话，并且allow_blocking_flush设置false，那么导入失败
6. 如果有overlap，那么需要先flush memtable
7. 寻找合适的level存放导入的sst文件
8. 设置seq号，整个sst文件所有的key都共享一个seq号
9. db最新的seq+1，更新version信息 
