---
layout: post
title:  "CompactionIterator"
categories: rocksdb
excerpt: rocksdb里面compact和flush memtable到sst文件这2个地方会用到这个CompactionIterator迭代器
---

* content
{:toc}

## 背景
DB后台在过滤数据的时候用户可以继承CompactionFilter类实现自定义过滤数据，比如说ttlDB过滤数据的时候就可以根据用户写入key的create_time(create_time保存在用户value里面)去过滤数据。




先埋个坑以后在写

