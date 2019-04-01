---
layout: post
title:  "rocksdb SingleDelete"
categories: rocksdb
---

* content
{:toc}

## 背景
在rocksdb里面所有的写操作(包括更新操作)都转换成append操作，这也是rocksdb写性能牛逼原因，对于普通的delete操作是转换成特殊的append写操作。所以rocksdb天然就是mvcc(多版本控制，同一个key存在多个版本)。查询的时候根据LSM树的结构默认会查询的最新版本。对于历史数据会在后台启动compact线程延迟整理数据，对于level n进行compact的时候，如果是最底层的话，那么可以把delete操作的key直接忽略掉，否则必须保留，因为有可能在更下面层有相同的key存在，为了不让用户查询到已经删除的key，所以必须要保留delete操作的key。我们线上曾经遇到过一次问题：有大量的delete类型的key存在，导致迭代器seek(要过滤掉delete类型数据)的时候耗时很大。

## 好处
SingleDelete为了消灭delete key产生，在compact的时候可以直接删除delete key和以前的版本。前提是满足删除条件(下面代码分析讲解)。这样快速可以消灭tombstones，节省磁盘空间，加快迭代器seek速度。

## 限制
- 不能和delete，merge操作混合用
- 不能在SingleDelete之前write多次，原因是compact的时候找到一个以前版本就认为找全了。
- 不能SingleDelete同一个key多次

## 实现
写入时候的行为和普通的delete没有区别，主要区别是在compact的时候行为不同。下面是具体代码分析(代码精简过):

```
ProcessKeyValueCompaction(SubcompactionState* sub_compact) {
  auto c_iter = new CompactionIterator(input...)
  while (status.ok() && !cfd->IsDropped() && c_iter->Valid()) {
    c_iter->Next();
  }
}
void CompactionIterator::Next() {
  NextFromInput();
}
void CompactionIterator::NextFromInput() {
  valid_ = false;
  while (!valid_ && input_->Valid() && !IsShuttingDown()) {
    key_ = input_->key();
    value_ = input_->value();
    if (ikey_.type == kTypeSingleDeletion) {
      ParsedInternalKey next_ikey;
      input_->Next();

      // Check whether the next key exists, is not corrupt, and is the same key
      // as the single delete.
      if (input_->Valid() && ParseInternalKey(input_->key(), &next_ikey) &&
          cmp_->Equal(ikey_.user_key, next_ikey.user_key)) {
      	if (key_ 和 next_key 之间没有快照) {
          if (key_ 之前没有快照) {
          } else {
            valid_ = true;
            //这个要保留的原因是在事务场景里面，我们需要知道在事务开始以后，到提交的时候有没有其他事务修改了这个key，用来做写冲突检查
            //这种情况就是在next_key之前有快照
            //next_key可以被compact干掉，因为永远不会被用户读到
          }
        } else {
          valid_ = true;
        }
      } else {
        valid_ = true;
      }
    }
  }
}
```

重点分析NextFromInput里面内容，如下几种情况必须保留:
1. next_key和当前key不相等，说明之前写入key和single delete key不在同一个compact任务里面
2. 如果有快照在next_key版本和当前key版本之间，为了快照读到数据一致，也必须保留2个key 
3. 如果有事务在next_key版本之前开始，乐观事务在提交的时候需要做写冲突检查，所以要保留delete key，之前的put key可以被删除

## 参考资料
- https://github.com/facebook/rocksdb/wiki/Single-Delete
- https://github.com/facebook/rocksdb/wiki/Delete-A-Range-Of-Keys
