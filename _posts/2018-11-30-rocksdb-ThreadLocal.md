---
layout: post
title:  "rocksdb ThreadLocal 源码解析"
categories: rocksdb
---

* content
{:toc}

## 背景
thread-local简单的说就是线程私有变量，在c++11之前，我们可以通过pthread_key_t来实现这个功能，c++11里面增加thread_local关键字，比原来使用方便，对应thread_local变量，每个线程都有自己拷贝。
如果有多个thread_local变量，需要申请多个pthread_key_t或者thread_local，rocksdb里面使用ThreadLocal封装。

## 实现
```
class ThreadLocalPtr {
private:
	static StaticMeta* Instance();
	const uint32_t id_;
};
struct ThreadData {
  explicit ThreadData(ThreadLocalPtr::StaticMeta* _inst) : entries(), inst(_inst) {}
  std::vector<Entry> entries;
  ThreadData* next;
  ThreadData* prev;
  ThreadLocalPtr::StaticMeta* inst;
};
class ThreadLocalPtr::StaticMeta {
private:
	ThreadData head_;
	pthread_key_t pthread_key_;
};
```
static_meta是static变量，全局唯一。一个线程有一个ThreadData，线程通过pthread_key_t获取自己的ThreadData。
对于同一个ThreadLocal，他在ThreadData的数组里面的下标是一样。
## 使用方式
线程需要使用thread_local的时候，直接new ThreadLocalPtr类，然后调用reset方法设置关联的变量，get方法获取关联的变量

rocksdb里面主要有2个地方用到这个：

1. super_version_  版本没有更新的时候线程通过局部变量减少锁冲突。
2. lock_maps_cache_





