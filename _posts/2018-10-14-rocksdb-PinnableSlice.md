---
layout: post
title:  "rocksdb PinnableSlice"
categories: rocksdb
excerpt: PinnableSlice相对于Slice可以减少内存拷贝
---

* content
{:toc}

## 什么是PinnableSlice
PinnableSlice继承Slice同时又继承Cleanable
先介绍Cleanable， 这个类析构的时候会调用注册到类里面的clean函数，释放相关资源, 因此PinnableSlice析构的时候也会释放相关资源

## 使用方式
调用BlockBasedTableReader::Get()方法从sst文件获取数据的时候，会先从block_cache里面查找是否有缓存，如果有直接返回，没有的话从磁盘读取文件，然后把block缓存到block_cache里面。block里面找到对应的key以后，会调用
```c++
get_context->SaveValue(parsed_key, biter.value(), &biter)
```
这个方法里面会直接把这个block在block_cache里面的内存地址赋值给PinnableSlice，这样就减少了一次内存拷贝。可能大家会有疑问：如果block在block_cache中被淘汰了，那么对应的内存地址获取到值就不对。

## 原理
重新在说说Cleanable，Cleannable子类对象A在调用PinSlice方法的时候会把B注册的clean方法拿过来，然后B析构的时候就不会调用clean方法了，B的clean方法全部委托给A。而是等到A析构时才会调用clean方法。<br/>BlockIter和PinnableSlice这2个类都继承Cleanable类，PinnableSlice调用PinSlice的时候就把BlockIter类的clean方法拿过来。当然本身block_cache里面的数据都是有引用计数的，BlockIter的clean方法把引用计数减一，如果引用计数为0才释放对应的内存。
## 收益
这个别人测试的[收益](https://github.com/facebook/rocksdb/pull/1756#issuecomment-286201693), 可以看到对于大value收益比较明显
## 补充
这里说一下block在block_cache中的key生成方法：
```
  long version = 0;
  result = ioctl(fd, FS_IOC_GETVERSION, &version);
  
  if (result == -1) {
    return 0;
  }
  uint64_t uversion = (uint64_t)version;

  char* rid = id;
  rid = EncodeVarint64(rid, buf.st_dev);
  rid = EncodeVarint64(rid, buf.st_ino);
  rid = EncodeVarint64(rid, uversion);
  assert(rid >= id);
  return static_cast<size_t>(rid - id);
```
对于一个block来说 key: 设备id + inode节点号 + inode版本 + offset