---
layout: post
title:  "rocksdb Cuckoo Hash"
categories: rocksdb
---

* content
{:toc}

## 背景
rocksdb memtable 底层实现里面有Cuckoo，所以学习一下Cuckoo Hash。 <br/>
cuckoo中文名是布谷鸟。这种鸟有一种即狡猾又贪婪的习性，它不肯自己筑巢，而是把蛋下到别的鸟巢里，而且它的幼鸟又会比别的鸟早出生，布谷幼鸟天生有一种残忍的动作，幼鸟会拼命把未出生的其它鸟蛋挤出窝巢，今后以便独享“养父母”的食物。cuckoo hash也是类似的思想，用了多个哈希函数来解决hash冲突，当计算出来所有hash槽位被别人占用的时候，通过BFS(广度优先遍历)查找原来槽位key能否迁移到其他槽位，如果可以的话，就迁移走，空出槽位填入新key。

## 好处
提升空间利用率，具体可以参考[CMU论文](http://www.cs.cmu.edu/~binfan/papers/conext14_cuckoofilter.pdf)。

## 实现
- 查找<br/>
  计算出多个槽位，依次去比较就可以确认key是否存在。
- 插入<br/>
  先去遍历计算出的多个槽位，发现有一个槽位为就直接插入。如果全部都别人占用了，通过BFS(广度优先遍历)依次枚举原来槽位的key，看看它能否迁移出来，直至找到这么一个key。使用BFS原因就是降低迁移成本，找一个需要迁移步骤最少的key。如果还是找不到那么就插入失败。 rocksdb内部实现的代码还挺复杂的，一个小优化：BFS队列去重一下，对于相同的槽位只需要搜索一次就行了。

## 缺点
作为memtable底层存储结构来说，Cuckoo Hash对于seek接口支持不好，性能很差。

## 参考资料
- https://en.wikipedia.org/wiki/Cuckoo_hashing
- https://coolshell.cn/articles/17225.html
- https://yq.aliyun.com/articles/563053
