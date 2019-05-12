---
layout: post
title:  "rocksdb RateLimit"
categories: rocksdb
---

* content
{:toc}

## 背景
一台机器部署多个rocksdb实例的时候，这个时候我们需要控制rocksdb对磁盘的写入速度，不然会影响磁盘读性能

## 实现
调用NewGenericRateLimiter函数构造一个RateLimiter对象，然后通过调用Request函数来控制速度。<br/>

```
//每次请求生成一个Req
struct GenericRateLimiter::Req {
  explicit Req(int64_t _bytes, port::Mutex* _mu)
      : request_bytes(_bytes), bytes(_bytes), cv(_mu), granted(false) {}
  int64_t request_bytes;
  int64_t bytes;
  port::CondVar cv;
  bool granted;
};
class GenericRateLimiter : public RateLimiter {
private:  
  mutable port::Mutex request_mutex_;  //临界区数据控制, Request函数进入就会加锁

  const int64_t kMinRefillBytesPerPeriod = 100;

  const int64_t refill_period_us_;    //填充周期间隔时间 比如说100ms

  int64_t rate_bytes_per_sec_;        //限速值，单位字节
  // This variable can be changed dynamically.
  std::atomic<int64_t> refill_bytes_per_period_; //每个周期填充字节数 = rate_bytes_per_sec_ / fairness_

  bool stop_;
  port::CondVar exit_cv_;
  int32_t requests_to_wait_;

  int64_t total_requests_[Env::IO_TOTAL];
  int64_t total_bytes_through_[Env::IO_TOTAL];
  int64_t available_bytes_; // 表示可以写入byte
  int64_t next_refill_us_;  //下一次填充时间戳

  int32_t fairness_; // 1/n的概率优先响应low级别请求

  Req* leader_; //leader req
  std::deque<Req*> queue_[Env::IO_TOTAL];   //请求队列，根据io级别分成low, high
}

void GenericRateLimiter::Request(int64_t bytes, const Env::IOPriority pri,
                                 Statistics* stats) {
  MutexLock g(&request_mutex_);
  if (available_bytes_ >= bytes) {
    available_bytes_ -= bytes;
    return;
  }
  Req r(bytes, &request_mutex_);
  queue_[pri].push_back(&r);
  do {
    bool timedout = false;
    if (leader_ == nullptr &&
        ((!queue_[Env::IO_HIGH].empty() &&
            &r == queue_[Env::IO_HIGH].front()) ||
         (!queue_[Env::IO_LOW].empty() &&
            &r == queue_[Env::IO_LOW].front()))) {
      leader_ = &r;
      int64_t delta = next_refill_us_ - NowMicrosMonotonic(env_);
      delta = delta > 0 ? delta : 0;
      if (delta == 0) {
        timedout = true;
      } else {
        int64_t wait_until = env_->NowMicros() + delta;
        RecordTick(stats, NUMBER_RATE_LIMITER_DRAINS);
        ++num_drains_;
        timedout = r.cv.TimedWait(wait_until);
      }
    } else {
      // Not at the front of queue or an leader has already been elected
      r.cv.Wait();
    }

    // request_mutex_ is held from now on
    if (stop_) {
      --requests_to_wait_;
      exit_cv_.Signal();
      return;
    }
    if (leader_ == &r) {
      // Waken up from TimedWait()
      if (timedout) {
        // Time to do refill!
        Refill();

        leader_ = nullptr;

        // Notify the header of queue if current leader is going away
        if (r.granted) {
          assert((queue_[Env::IO_HIGH].empty() ||
                    &r != queue_[Env::IO_HIGH].front()) &&
                 (queue_[Env::IO_LOW].empty() ||
                    &r != queue_[Env::IO_LOW].front()));
          if (!queue_[Env::IO_HIGH].empty()) {
            queue_[Env::IO_HIGH].front()->cv.Signal();
          } else if (!queue_[Env::IO_LOW].empty()) {
            queue_[Env::IO_LOW].front()->cv.Signal();
          }
          break;
        }
      } else {
        assert(!r.granted);
        leader_ = nullptr;
      }
    } else {
      assert(!timedout);
    }
  } while (!r.granted);
}
```
Request函数进入会先请求全局mutex，如果当前的available_bytes_ > 请求的byte，那么可以直接返回。<br/>
否则进入队列排队，如果不是leader，就简单的wait，等待唤醒。<br/>
如果是leader，那么就判断一下next_refill_us_减去当前时间戳的值，如果是正值那么表示要timewait等待超时唤醒或者被别人唤醒，如果是负值则表示第一次有请求进入或者是上次请求距离现在已经超过间隔期时间timeout=true，如果timeout=true那么就需要调用Refill函数，对available_bytes_增加refill_bytes_per_period字节数，然后去队列里面依次处理之前缓存的请求，直到请求处理完或者available_bytes_=0。<br/>
如果请求字节数能被允许的话，那么granted=true，调用循环，否则就继续请求等待。<br/>

## 参考资料
- https://github.com/facebook/rocksdb/wiki/Rate-Limiter
