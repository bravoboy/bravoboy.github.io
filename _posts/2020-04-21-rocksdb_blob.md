---
layout: post
title:  "rocksdb blob"
categories: rocksdb
---

* content
{:toc}

## 背景
最近线上遇到一个业务，平均value很大，达到50K，写放大很严重。
blob_db参考了WiscKey的思想，设计的kv存储分离，可以有效的减小写放大。
LSM树里面只存储key和value的地址，这样后台线程compact的时候可以少读写很多数据。
增加一种类型：kTypeBlobIndex 表示value是否blob_db的地址。

## 写流程
首先写blob_db，然后判断在把offset和文件id做为value写入lsm树。
如果设置了db最大size，并且超过限制了，就会淘汰删除最老的blob文件。


## 读流程
主要的流程还是和普通的db一样，增加GetImplOptions里面is_blob_index选项。
BlobDBOptions选项里面min_blob_size控制多大的value存储在blob_db中，小于min_blob_size，还是和原来一样，存储在lsm树。
根据返回的value类型判断，如果是kTypeBlobIndex，那么就需要再从blob_db获取真正的value，可以看到比原来多了一次读。
先需要解码lsm树里面获取的value,找到对应的blob_db里面的文件和offset,然后再获取真正的数据。

## gc流程
这块是重点，先占个坑，有空再看。

## 限制
1. 只支持单cf

## 参考资料
- https://www.usenix.org/system/files/conference/fast16/fast16-papers-lu.pdf
- https://www.jianshu.com/p/24152a212296