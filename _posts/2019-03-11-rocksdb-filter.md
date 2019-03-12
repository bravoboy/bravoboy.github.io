---
layout: post
title:  "rocksdb filter"
categories: rocksdb
---

* content
{:toc}

## 背景
filter 顾名思义是布隆过滤器，用来排除不存在的key，减少磁盘读。<br/>
rocksdb里面有几个参数和filter有关。<br/>
- cache_index_and_filter_blocks <br/>
  这个参数控制index和filter是否放在block cache中，cache_index_and_filter_blocks=true index和filter放在block cache，反之就不会放入，我们只能控制block cache大小，如果filter不在block cache中，内存就不好控制。<br/>
- optimize_filters_for_hits <br/>
  这个参数=true可以控制rocksdb对于最底层level文件不生成filter，如果数据量很大到达TB级别的时候，filter数据量也会很大，线上预估真正data和filter比例关系是100:4左右，如果数据量到达TB，filter大概有40G，在cache_index_and_filter_blocks=false的时候，filter占用内存不受block cache控制，很容易导致oom。
- cache_index_and_filter_blocks_with_high_priority <br/>
  这个参数在cache_index_and_filter_blocks=true时候，控制index和filter在block cache优先级，block cache中高优先级数据被逐出概率小
- pin_l0_filter_and_index_blocks_in_cache <br/>
  这个参数控制L0文件的index和filter是否提前加载到内存 

## 使用场景
- 服务初始化打开文件时候 <br/>
  rocksdb刚启动打开sst文件同时会缓存文件句柄到table_cache中，如果cache_index_and_filter_blocks=true并且level = 0，index和filter会被读到内存。
skip_filter这个参数可以控制打开sst文件时候是否控制filter。如果skip_filter=true，filter将被忽略(即使filter被读到内存)，如果文件没有被compact和table_cache逐出的话，filter将一直不会生效。看下面图片: <br/>
![filter.png](/images/filter.png)
- Get查询 <br/>
  查询的时候也可以设置skip_filter，默认是false，如果optimize_filters_for_hits=true并且文件的level是当前db最大的，skip_filter就会变成true。如果skip_filter=true那么就会忽略filter直接查询data block。但是skip_filter=false的时候也不一定会利用filter，原因在打开文件如果设置skip_filter=true，那么即使查询skip_filter=false也不会使用filter。 
