---
layout: post
title:  "rocksdb memtable_prefix_bloom_size_ratio 研究调优"
date:   2018-09-10 20:41:00
categories: rocksdb
excerpt: rocksdb的memtable_prefix_bloom_size_ratio参数是控制memtable的filter大小
mathjax: true
---
* content
{:toc}

## 布隆过滤器

filter是一个布隆过滤器，如果一个key不在memtable中，通过布隆过滤器是可以检查出来的，但是不能验证key存在

## filter

memtable_prefix_bloom_bits表示filter占用总的bit数<br/>

>memtable_prefix_bloom_bits = write_buffer_size * memtable_prefix_bloom_size_ratio * 8

write_buffer_size如果取值128M的话，memtable_prefix_bloom_size_ratio取值0.15，那么memtable_prefix_bloom_bits等于161061273，memtable_prefix_bloom_bits这个还会对齐取整，
实际配置内存大小
$$size=\lfloor{memtable\_prefix\_bloom\_bits}\rfloor/8$$
<br/>默认的hash函数是BloomHash，一个key在布隆过滤器散列6次(写死的)。

## 测试

如果memtable_prefix_bloom_size_ratio>0，插入的成本会增加，需要多插入一次布隆过滤器，查询的时候会先去布隆过滤器读取是否存在。如果不存在就直接return，可能存在再去读memtable。<br/>
写入2M个key,  key大小是10个字节。<br/>
<table>
        <tr>
            <th>情况</th>
            <th>开启bloom，使用自带的hash函数</th>
            <th>开启bloom，使用murmur3 hash函数</th>
            <th>不开启bloom</th>
        </tr>
        <tr>
            <th>插入耗时</th>
            <th>2636毫秒</th>
            <th>3109毫秒</th>
            <th>2446毫秒</th>
        </tr>
        <tr>
            <th>查询不存在的key</th>
            <th>2416毫秒</th>
            <th>823毫秒</th>
            <th>2229毫秒</th>
        </tr>
        <tr>
            <th>查询存在的key</th>
            <th>2786毫秒</th>
            <th>3261毫秒</th>
            <th>2601毫秒</th>
        </tr>
    </table>

## 结论

自带的bloomhash速度比murmur3 hash快，但是散列非常不均衡，murmur3 hash key冲突的概率 < 1%<br/>开启bloom，使用自带的hash函数效果不如不开启<br/>使用murmur3 hash函数，查找不存在的key速度比不开启快，插入和查找存在的key速度比不开启慢

