---
layout: post
title:  "rocksdb perf"
categories: rocksdb
---

* content
{:toc}

## 背景
对于基础存储服务来说，性能是至关重要的，偶尔的抖动都会在业务层放大很多倍。linux perf工具可以帮助我们定位服务热点和瓶颈。对于LSM树，天然存在读写放大的问题，可能一次读操作，需要多次io才能找到我们想要的数据。对于这种情况，外部工具并不方便观察，rocksdb自带的perf工具可以详细的输出各种路径上的操作次数和耗时。

## 使用方法
```
get_perf_context()->Reset();

your code

print get_perf_context()->ToString()
```
输出的内容很多，有key的比较次数，block_cache命中次数等，可以根据输出内容分析出数据是来自于memtable, 还是block cache读取还是磁盘读取, 还可以分析出操作耗时，比如说在memtable上seek耗时和get耗时。耗时统计这个地方也很巧妙，构造了临时对象，析构的时候会调用stop函数，统计耗时
```
class PerfStepTimer {
public:
  ~PerfStepTimer() {
    Stop();
  }
}
// Declare and set start time of the timer
#define PERF_TIMER_GUARD(metric)                                  \
  PerfStepTimer perf_step_timer_##metric(&(perf_context.metric)); \
  perf_step_timer_##metric.Start();

```
另外还有一个观察磁盘读写的监控，使用方法也是和上面类似
```
get_iostats_context()->Reset();

your code

print get_iostats_context()->ToString();
```
可以观察一次put/get操作，实际底层的磁盘读写情况，put操作大概率观察不到，因为数据一般先写到内存buffer(写wal也是这样)，除非配置了direct_IO模式。
