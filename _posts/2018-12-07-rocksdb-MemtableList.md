---
layout: post
title:  "rocksdb MemtableList 源码解析"
categories: rocksdb
---

* content
{:toc}

## 背景
MemtabeList存储是immutable。immutable刷盘到L0文件可以并发执行

## 实现
```
class MemTableList {
private:
	std::atomic<bool> imm_flush_needed;  //是否需要刷盘
	int num_flush_not_started_;  //需要刷盘的immutable个数
	bool flush_requested_;
	MemTableListVersion* current_;
	//刷盘最少的memtable合并文件数
	const int min_write_buffer_number_to_merge_;
	//选取memtable进行flush,从后向前遍历memlist
	PickMemtablesToFlush(autovector<MemTable*>* ret)
};
class MemTableListVersion {
private:
  // Immutable MemTables that have not yet been flushed. 最新的memtable插入队列头部，查找的时候也是从头开始找，最新的数据在最前面,获取最小的seq号就从list最后一个元素查询
  std::list<MemTable*> memlist_;
  // MemTables that have already been flushed
  // (used during Transaction validation)
  std::list<MemTable*> memlist_history_;
};
struct MemTableOptions {
  size_t write_buffer_size;  //memtable大小
  size_t arena_block_size;   
  uint32_t memtable_prefix_bloom_bits;  //memtable里面前缀bloom_filter比率
  size_t memtable_huge_page_size;
  bool inplace_update_support;
  size_t inplace_update_num_locks;

  size_t max_successive_merges;  //最大merge个数，超过了就变成set
  Statistics* statistics;
  MergeOperator* merge_operator;
  Logger* info_log;
};
class MemTable {
private:
//memtable状态
enum FlushStateEnum { FLUSH_NOT_REQUESTED, FLUSH_REQUESTED, FLUSH_SCHEDULED };
  // These are used to manage memtable flushes to storage
  bool flush_in_progress_; // started the flush
  bool flush_completed_;   // finished the flush
  uint64_t file_number_;    // filled up after flush is complete
  //判断memtable当前大小和write_buffer_size大小关系决定是否变成immutable
  bool ShouldFlushNow() const;
};
```
2019年继续加油！！！





