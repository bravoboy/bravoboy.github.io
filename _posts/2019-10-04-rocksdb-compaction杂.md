---
layout: post
title:  "rocksdb compaction杂项"
categories: rocksdb
---

* content
{:toc}

## 背景
最近在facebook看到有人提问，学习到了新参数：periodic_compaction_seconds。这个参数的作用是：强制把某个时间点以前的sst文件重新compact，比如说7天以前生成的sst文件。对应的还有其他类型的compaction：ttl compaction, bottommost file compaction, marked file compaction。下面详细介绍一下这几个具体的用处

## 用处

#### periodic_compaction_seconds
如果我们设置了全局ttl过期时间，并且最底层的sst文件范围区间里面并没有读写，那么这部分数据是很难被删除掉的，白白占用了磁盘空间。如果我们触发manual compact的话，会导致compact范围太大，增加磁盘io压力，对读写性能都有比较大的影响。我们可以设置periodic_compaction_seconds这个参数等于全局ttl过期时间，这样能及时的删除过期数据，同时又不会增加io压力。

#### ttl
ttl和periodic_compaction_seconds区别是，ttl compaction不选择最底层sst文件, 感觉可以不需要这个参数

#### bottommost
bottommost file compaction只选择最底层的sst文件，并且文件中有delete key，同时没有快照依赖最低层sst文件。不考虑snapshot，本身compact生成的最底层文件里面是不带delete key，在compact过程中，直接丢弃delete key。但是如果有snapshot的存在就可能会导致delete key和原来的真实key/value不能被compact丢弃。本来最底层sst文件被compact的次数就比较少，如果这种情况出现多了，会导致seek性能变差，以及存储空间放大。bottommost file compaction可以有效的解决这个问题。

#### marked file
用户可以主动设置策略标识compact哪些文件。我们知道delete key多了以后，会影响rocksdb的读性能，那么我们可以设置优先compact 包含delete key多的文件，参考CompactOnDeletionCollector。<br/>

CompactOnDeletionCollector构造函数只有2个参数:sliding_window_size和deletion_trigger。就是说如果在window_size里面delete key个数超过了deletion_trigger，那么就会把这个sst文件标识为need compaction。<br/>

这个滑动窗口实现还是挺有意思的，比如说我们设置窗口大小256，deletion_trigger=16。我们最简单的实现是每256个key，重新计数delete key个数，这样朴素方法无法统计分区临界点数据，比如说前一个窗口末尾有10个delete key，后一个窗口开头有10个delete key，这种场景是不能被标识处理。想要精确统计的话，我们可以申请一个window_size大小的循环数组，每key都映射到数组里面的一个元素，然后每次更新数组以后，都统计一下数组里面当前的delete key数量。rocksdb采用了折中方法，申请了128大小的循环数组，bucket_size = window_size / 128, bucket_size个元素对应到循环数组里面一个元素。
