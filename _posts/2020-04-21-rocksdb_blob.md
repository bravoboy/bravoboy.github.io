---
layout: post
title:  "rocksdb blob"
categories: rocksdb
---

* content
{:toc}

## 背景
最近线上遇到一个业务，平均value很大，达到50K，写放大很严重。<br/>
blob_db参考了WiscKey的思想，设计的kv存储分离，可以有效的减小写放大。<br/>
LSM树里面只存储key和value的地址，这样后台线程compact的时候可以少读写很多数据。<br/>
rocksdb里面增加一种类型：kTypeBlobIndex 表示value是否blob_db的地址。 <br/>

## 写流程
首先判断value大小是否超过配置，超过了就写blob_db，然后在把offset和文件id做为value写入lsm树。<br/>
否则就是正常的rocksdb写入。如果设置了db最大size，并且磁盘空间超过限制了，就会淘汰删除最老的blob文件。<br/>
blob文件格式：<br/>
head结构 <br/>
|----|----|----|-|-|--------|---------|  <br/>
magic version cf_id flag expiration_start expiration_end <br/>
foot结构 <br/>
|----|--------|--------|---------|----|  <br/>
magic count expiration_start expiration_end crc <br/>

和sst文件一样，blob文件写完以后，不会被更改，只能被删除。<br/>
每个blob文件都有size限制，超过这个限制就会和wal一样，重新打开一个blob文件写入。<br/>
每个blob文件没有类似rocksdb那样的level层级。<br/>

## 读流程
主要的流程还是和普通的db一样，增加GetImplOptions里面is_blob_index选项。<br/>
BlobDBOptions选项里面min_blob_size控制多大的value存储在blob_db中，小于min_blob_size，还是和原来一样，存储在lsm树。<br/>
根据返回的value类型判断，如果是kTypeBlobIndex，那么就需要再从blob_db获取真正的value，可以看到比原来多了一次读。<br/>
先需要解码lsm树里面获取的value,找到对应的blob_db里面的文件和offset,然后再获取真正的数据。<br/>

## gc流程
每个blob文件或者会有几个对应的sst文件，或者对应几个memtable。<br/>
只有blob文件没有关联的sst文件并且blob文件的seq比flush_seq大，才满足被gc删除条件。<br/>
后台线程会周期性的删除无用的blob文件。<br/>
Flush memtable的时候会跟进value类型判断，如果valuekTypeBlobIndex，则会更新文件对应的最早的blob文件。<br/>
Flush完成以后会调用blob的回调函数，建立新的sst文件和blob文件的对应关系。<br/>
Compact完成以后也会调用blob的回调函数，老的sst文件和blob文件映射关系解除，增加新的sst文件和blob文件的映射关系。<br/>

## 限制
1. 只支持单cf

## 参考资料
- https://www.usenix.org/system/files/conference/fast16/fast16-papers-lu.pdf
- https://www.jianshu.com/p/24152a212296