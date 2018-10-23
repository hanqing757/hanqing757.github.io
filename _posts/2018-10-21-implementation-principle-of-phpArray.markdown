---
layout: post
title:  "PHP数组实现原理"
date:   2018-10-21 18:01:00
categories: array
tags: array implementation-principle
excerpt: PHP数组的底层实现原理
mathjax: true
---

* content
{:toc}

php中的array应用非常广泛和灵活，其底层实际上使用hashtable来实现的，类似于java中的hashmap，主要结构是数组加链表，通过hash映射到对应数组的位置，通过链表解决hash冲突。首先看一下hashtable的结构定义

 <div align="center">![hashtable](/img/hashtable.png)</div>

其中nTableMask=nTableSize-1，通过nTableMask与hash值按位与取到hash值的后几位，快速定位在hashtable中的位置，nNextFreeElement表示下一个可用的数字索引，arBucket实际指向hashtable里每个元素后的链表，是实际的存储数据的容器，bucket的结构如下

 <div align="center">![bucket](/img/bucket.png)</div>

h表示hash值，对数字索引来说，h就直接是索引值，nKeyLength=0;对字符串索引来说，索引值保存在arKey中，索引长度保存在nKeyLength中。pData指向实际保存的数据，当bucket保存的是一个指针的时候，将指针保存在
pDataPtr中，将pData指向pDataPtr，这样可以减少内存碎片。
hashtable结构图如下，比如对于一个array(1=>1,2=>2,3=>3,4=>4,5=>5)

 <div align="center">![array-hashtable](/img/array-hashtable.png)</div>

hashtable中的pListHead指向线性列表下的第一个元素，pListTail指向线性列表下的最后一个元素，pListNext指向线性结构下的下一个元素，pListLast指向线性结构下的上一个元素。
pInternalPointer指向当前指针的位置，所以在顺序遍历的时候会先从pListHead开始，顺着pListNext/pListLast移动pInternalPointer来实现对所有元素的顺序遍历。
随机访问时，先通过key获取hash值，确定bucket中的头指针的位置，再通过pNext和pLast来找到需要的元素。
当增加元素的时候，会将增加的元素放在线性表的尾部和冲突链表的头部，所以线性遍历的时候会按照元素的插入顺序来遍历，因此foreach会按照元素的插入顺序来遍历。