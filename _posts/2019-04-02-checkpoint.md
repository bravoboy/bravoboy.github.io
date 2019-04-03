---
layout: post
title:  "rocksdb CheckPoint"
categories: rocksdb
---

* content
{:toc}

## 背景
rocksdb支持打快照备份，如果快照在同一个文件系统，就可以使用硬链接(文件别名，对应同一个inode)，否则就必须要拷贝了。myrocks备份就使用这个checkpoint功能。

## 实现
```
CheckpointImpl::CreateCheckpoint(const std::string& checkpoint_dir, uint64_t log_size_for_flush) {
    Status s = db_->GetEnv()->FileExists(checkpoint_dir);
    if (s.ok()) {
        return Status::InvalidArgument("Directory exists");
    } else if (!s.IsNotFound()) {
        assert(s.IsIOError());
        return s;
    }

    db_->DisableFileDeletions();
    CreateCustomCheckpoint();
    db_->EnableFileDeletions(false);
}
```
调用CreateCheckpoint首先会检查目录是否存在，存在就直接报错，然后设置compact过程不删除文件。
同一个文件系统，用硬链接方式拷贝sst文件，MANIFEST和CURRENT以及最后一个wal文件采用拷贝方式，之前的wal文件用硬链接拷贝方式

## 参考资料
- https://github.com/facebook/rocksdb/wiki/Checkpoints
- http://mysql.taobao.org/monthly/2017/02/02/
