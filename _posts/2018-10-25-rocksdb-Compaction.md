---
layout: post
title:  "rocksdb Compaction"
categories: rocksdb
excerpt: rocksdb把随机写转化成顺序写，需要gc回收垃圾，compaction的工作就是gc数据
---

* content
{:toc}

## 背景
rocksdb把随机写转化成顺序写，需要gc回收垃圾，compaction的工作就是gc数据。线上有一个集群，之前是没有开启ttl过期的，数据积累很多了，占用磁盘85%了，再增长下去非常危险。和业务方沟通以后可以删除30天以前的数据，只保留最近30天的数据，果断开启了ttl功能(会自动删除大于30天的数据)。开启以后，观察发现数据删除很慢，每天新增的数据比删除的数据还多，磁盘占用率还在上升。top -Hp命令发现一般只有一个rocksdb后台线程在compact工作。

## Compact
rocksdb中存在两种类型的compaction：minor compaction和major compaction。minor compaction是指的把内存中的immutable dump到L0的sst文件，一般我们也叫做flush。major compaction是指的L0以下的sst文件规整。

rocksdb里面有2种compaction策略: [Universal Compaction](https://github.com/facebook/rocksdb/wiki/Universal-Compaction)和[Leveled Compaction](https://github.com/facebook/rocksdb/wiki/Leveled-Compaction)

## 主要函数
compaction的调度都是MaybeScheduleFlushOrCompaction函数里面。

```
void DBImpl::MaybeScheduleFlushOrCompaction() {
  mutex_.AssertHeld();
  if (!opened_successfully_) {
    // Compaction may introduce data race to DB open
    return;
  }
  if (bg_work_paused_ > 0) {
    // we paused the background work
    return;
  } else if (shutting_down_.load(std::memory_order_acquire)) {
    // DB is being deleted; no more background compactions
    return;
  }
  auto bg_job_limits = GetBGJobLimits();
  bool is_flush_pool_empty =
    env_->GetBackgroundThreads(Env::Priority::HIGH) == 0;
  while (!is_flush_pool_empty && unscheduled_flushes_ > 0 &&
         bg_flush_scheduled_ < bg_job_limits.max_flushes) {
    unscheduled_flushes_--;
    bg_flush_scheduled_++;
    env_->Schedule(&DBImpl::BGWorkFlush, this, Env::Priority::HIGH, this);
  }

  // special case -- if high-pri (flush) thread pool is empty, then schedule
  // flushes in low-pri (compaction) thread pool.
  if (is_flush_pool_empty) {
    while (unscheduled_flushes_ > 0 &&
           bg_flush_scheduled_ + bg_compaction_scheduled_ <
               bg_job_limits.max_flushes) {
      unscheduled_flushes_--;
      bg_flush_scheduled_++;
      env_->Schedule(&DBImpl::BGWorkFlush, this, Env::Priority::LOW, this);
    }
  }

  if (bg_compaction_paused_ > 0) {
    // we paused the background compaction
    return;
  }

  if (HasExclusiveManualCompaction()) {
    // only manual compactions are allowed to run. don't schedule automatic
    // compactions
    return;
  }

  while (bg_compaction_scheduled_ < bg_job_limits.max_compactions &&
         unscheduled_compactions_ > 0) {
    CompactionArg* ca = new CompactionArg;
    ca->db = this;
    ca->m = nullptr;
    bg_compaction_scheduled_++;
    unscheduled_compactions_--;
    env_->Schedule(&DBImpl::BGWorkCompaction, ca, Env::Priority::LOW, this,
                   &DBImpl::UnscheduleCallback);
  }
}
```

看上面代码知道：根据bg_job_limits限制来决定生成多少个compact任务。
GetBGJobLimits函数

```
DBImpl::BGJobLimits DBImpl::GetBGJobLimits() const {
  mutex_.AssertHeld();
  return GetBGJobLimits(immutable_db_options_.max_background_flushes,
                        mutable_db_options_.max_background_compactions,
                        mutable_db_options_.max_background_jobs,
                        write_controller_.NeedSpeedupCompaction());
}

DBImpl::BGJobLimits DBImpl::GetBGJobLimits(int max_background_flushes,
                                           int max_background_compactions,
                                           int max_background_jobs,
                                           bool parallelize_compactions) {
  BGJobLimits res;
  if (max_background_flushes == -1 && max_background_compactions == -1) {
    // for our first stab implementing max_background_jobs, simply allocate a
    // quarter of the threads to flushes.
    res.max_flushes = std::max(1, max_background_jobs / 4);
    res.max_compactions = std::max(1, max_background_jobs - res.max_flushes);
  } else {
    // compatibility code in case users haven't migrated to max_background_jobs,
    // which automatically computes flush/compaction limits
    res.max_flushes = std::max(1, max_background_flushes);
    res.max_compactions = std::max(1, max_background_compactions);
  }
  if (!parallelize_compactions) {
    // throttle background compactions until we deem necessary
    res.max_compactions = 1;
  }
  return res;
}
```

我们线上已经调整过max_background_compactions=20。parallelize_compactions参数是write_controller_控制，这个是限制compact太多导致影响写性能。
如下条件有其中一条满足的时候，会触发stop write.<br/>
1: immutable 个数大于 max_write_buffer_number<br/>
2: disable_auto_compactions=false(默认是false, TransactionDB=true) 并且 L0 文件个数大于 level0_stop_writes_trigger<br/>
3: disable_auto_compactions=false(默认是false, TransactionDB=true) 并且 预估compact字节数大于hard_pending_compaction_bytes_limit<br/>

如下条件有其中一条满足的时候，会触发stalling write.<br/>
1: immutable 个数大于 max_write_buffer_number - 1 并且 max_write_buffer_number > 3<br/>
2: disable_auto_compactions=false(默认是false, TransactionDB=true) 并且 L0 文件个数大于 level0_slowdown_writes_trigger<br/>
3: disable_auto_compactions=false(默认是false, TransactionDB=true) 并且 预估compact字节数大于soft_pending_compaction_bytes_limit<br/>

还有一种情况会导致compact任务增加:
1: L0文件个数大于min(2*level0_file_num_compaction_trigger, level0_file_num_compaction_trigger + level0_slowdown_writes_trigger/4 - level0_file_num_compaction_trigger/4)<br/>
2: 预估compact字节数大于soft_pending_compaction_bytes_limit / 4 
修改点：设置res.max_compactions = max_background_compactions

以上3种情况触发会设置parallelize_compactions=true，compact任务数会变多。<br/>
#### 修改点：上面这种场景历史数据堆积比较多，所以不会触发parallelize_compactions=true的条件，所以临时强制设置parallelize_compactions=true。
## 效果

20分钟就把磁盘文件大小从3.2T降低到1.5T。 效果非常明显，不过这个也是有代价的，这20分钟时间内ioutil飙升到了100%。cpu占有率也非常高，至此完成数据清理问题。


