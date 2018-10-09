---
layout: post
title:  "rocksdb seekforpre bug总结"
date:   2018-09-12 00:06:05
categories: rocksdb
tags: 线上使用遇到rocksdb seekforpre的bug
---
* content
{:toc}

## 背景
这次bug是sst文件查找导致的，所以关于memtable的内容这里就不介绍了。<br/>先介绍一下sst文件格式，我们线上使用的BlockBasedTable格式，继承leveldb的格式，后续就只说这种格式。<br/>下面是rocksb官方给的格式说明：

```
<beginning_of_file>
[data block 1]
[data block 2]
...
[data block N]
[meta block 1: filter block]                  (see section: "filter" Meta Block)
[meta block 2: stats block]                   (see section: "properties" Meta Block)
[meta block 3: compression dictionary block]  (see section: "compression dictionary" Meta Block)
[meta block 4: range deletion block]          (see section: "range deletion" Meta Block)
...
[meta block K: future extended block]  (we may add more meta blocks in the future)
[metaindex block]
[index block]
[Footer]                               (fixed size; starts at file_size - sizeof(Footer))
<end_of_file>
```

data block里面存储实际数据，meta block存储是filter数据，index等数据，metaindex block保存是meta索引信息，index block保存是data block的索引信息，Footer区保存的是整个文件结尾信息，前面各个block的offset和length。<br/>
先介绍data block，默认大小是4k，block_size_deviation默认10，表示当前block大小到设置大小的90%以后预估一下新插入key是否超过限制，超过了就写新的block。内部的key按照字典序从小到大排序，有相等前缀的时候就只保存后面不同的部分，减少存储量，默认16个key会重新从头开始保存key, 这个点叫重启点。<br/>block结构如下：
![](/images/5.png)

## sst文件查找
看了上面sst文件结构介绍，接下来介绍在sst文件里面查找key过程。L0层sst文件范围互相交叉。L1以及以下的文件是按照key的字典序排序，不会有交叉覆盖问题。对于L0查找，需要在所有的文件里面查找是否存在key，L1以及以下层首先要2分查找，找到对应的文件。然后在文件里面再2分查找，找到对应的block。线上开启了hashindex，所以会先查找hashindex，然后在查找data block。如果开启了total_order_seek选项，那么就不会走hashindex逻辑，直接2分查找. 最底层的是data blockiter, 然后是index blockiter, 再然后是LevelFileNumIterator定位文件. 线上开启了optimize_filters_for_hits=true，所以底层sst文件没有filter，seekforpre的时候会调用seek，然后seek的hash里面没有, 直接return了，会导致TwoLevelIterator里面的seekforpre直接把游标移到到最后.
#### 修改1
TwoLevelIterator里面的seekforpre函数修改：
```
void TwoLevelIterator::SeekForPrev(const Slice& target) {
  if (state_->check_prefix_may_match && !state_->PrefixMayMatch(target)) {
    SetSecondLevelIterator(nullptr);
    return;
  }
  first_level_iter_.SeekForPrev(target);
  if (!first_level_iter_.Valid()) {
      // The only case that we can't get a valid iterator, ignoring the case of empty blocks,
      // is that the value of the first index is greater than `target`.
      first_level_iter_.SeekToFirst();
  }
  while (first_level_iter_.Valid() && first_level_iter_.key().compare(target) < 0) {
    first_level_iter_.Next();
  }
  //first_level_iter_.Seek(target);
  InitDataBlock();
```
修改点：first_level_iter_.SeekForPrev(target); 代替 first_level_iter_.Seek(target);<br/>
理由: SeekForPrev 不会走hashindex
跑了一段时间后再次发现bug

#### 修改2
修改点：恢复first_level_iter_.Seek(target); 同时上层传入total_order_seek=true<br/>
排查过程：初看上面代码没有问题，也实在是很难找到bug. 线上偶尔会出现SeekForPrev返回值为空，不是必现。<br/>
问题就在于: <br/>
>first_level_iter_.key().compare(target)

这个地方是直接字符串比较， 但是first_level_iter_的key有可能是带有seq号的，会导致比较不准，导致找到的data block不准。<br/>index_iter和datablock_iter里面key比较是去掉最后8位的seq number的。<br/>
~~first_level_iter_ 这个地方封装的迭代器不固定，不能保证key一定是带seq number，所以不能简单的调用ExtractUserKey来修复~~

