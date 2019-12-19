---
layout: post
title:  "redis中的简单动态字符串"
date:   2019-12-19 12:00:00
categories: simple dynamic string
tags: simple dynamic string
excerpt: redis中的简单动态字符串
mathjax: true
---

* content
{:toc}

redis中的字符串并没有使用C默认的char* 类型，因为char* 的功能单一，抽象简单，不能高效支持redis中一些命令。如果使用char* ，那strlen命令的时间复杂度将是*O(N)* ，每对字符串执行一次append命令就会有一次的内存分配。因此，redis中使用了Sds（Simple Dynamic String，简单动态字符串）来表示字符串，每个字符串对象都包含一个Sds值。

那么在redis中哪些地方使用了字符串对象呢？redis的键总是字符串对象，当redis的值保存的类型是字符串时，也使用的字符串对象。只有当字符串对象保存的是字符串的时候，才包含有一个sds值。
sds的组成结构如下
```C
struct sdshdr
{
    //buf已占用长度
    int len;

    //buf剩余长度
    int free;

    //字符串数据保存位置
    char buf[];
};
```
比如保存"hello world"字符串的sdshdr结构：
```C
struct sdshdr
{
    len = 11;

    free = 0;

    buf = "hello world\0";   //实际长度是len+1;
};
```
通过sdshdr中的len属性，实现复杂度为*O(1)*的字符串长度计算。另一方面，buf中会预留一些额外空间，free记录未使用空间的大小，以此来减小内存分配次数，下面我们详细讨论这一点。
内存分配的伪代码如下：
```c
def sdsMakeRoomFor(sdshdr, required_len){
    if(sdshdr.free >= required_len){
        return sdshdr;
    }

    newlen = sdshdr.len + required_len;

    if(newlen < SDS_MAX_PREALLOC){
        newlen *= 2;
    }else{
        newlen += SDS_MAX_PREALLOC;
    }

    newsh = zrelloc(sdshdr, sizeof(struct sdshdr)+newlen+1);

    newsh.free = newlen - sdshdr.len;

    return newsh;
}
```
目前redis版本中SDS_MAX_PREALLOC的值为1024\*1024，也就是1MB字符串大小。流程图表示如下：

![sds-mem-alloc](/img/sds-mem-alloc.png)

通常预分配内存不会释放，除非键被删除或者redis重启。