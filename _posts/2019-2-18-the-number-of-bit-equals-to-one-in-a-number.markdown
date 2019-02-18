---
layout: post
title:  "算法系列之一个数字中被置位的个数"
date:   2019-2-18 19:00:00
categories: algorithm
tags: algorithm and-operation
excerpt: 算法系列之一个数字中被置位的个数
mathjax: true
---

* content
{:toc}

从这个系列起将会写一些算法相关的文章，这些算法可能比较有趣，可能涉及面试，可能涉及一些数学知识。写算法是一个很好的锻炼逻辑思维能力和抽象能力的过程。
本篇文章所描述的算法比较简单，统计一个数字中被置位的个数，以golang为例，统计一个uint64的值中，值为1的bit位的个数，这是《go程序设计语言》第二章后的习题。
解法一比较快我们称之为查表的方式，可以依次计算uint64的1-8位，9-16位直到57-64位这每一个uint8中置位的个数，然后将结果相加。那么我们就可以先将每一个uint8所含1的个数计算出来，供查表使用。
```golang
var pc1 [256]byte
for i := range pc1 {
    pc1[i] = pc1[i/2] + byte(i&1)
}
```
其中，i中置位的个数等于末尾置位的个数加上将i右移一位后其中置位的个数，这是一种比较快的计算方式。接下来计算uint64中由低到高每8位中所含1的个数，对于一个uint64类型的x，可以
```golang
for i = 0; i < 8; i++ {
    r += pc1[byte(x>>(i*8))]
}
```
byte显示类型转换将只取数字的最低八位。
解法二使用golang自带的库strconv中的FormatUint函数，将uint格式化为指定的进制，再对字符串进行遍历即可
```golang
func PopCount2(x uint64) byte {
    var cnt byte
    bx := strconv.FormatUint(x,2)
    for i,_ := range bx {
        if(string(bx[i]) == "1"){
            cnt++
        }
    }
    return cnt
}
```
解法三也比较好理解，一个数字和1执行与操作判断最后一位是否是1，再结合>>右移操作，可想而知这种方法是比较耗时的
```golang
func PopCount3(x uint64) byte {
    var r byte
    for x > 0 {
        if(x&1 == 1){
            r++
        }
        x = x >> 1
    }
    return r 
}
```
解法四主要使用的是x=x&(x-1),这个操作可以将x的最右一位的1置为0，如何理解呢？
对于一个uint64永远可以写成A1B这种形式，其中A任意，B为若干个0（可以是0个），那么x-1就变成了A0C，其中，C为若干个1，长度与B相等，x&(x-1)就变成了A1B&A0C，结果是A0B，可以看到，将A1B的最右一位1置为了0
```golang
func  PopCount4(x uint64) byte {
    var r byte
    for x > 0 {
        x = x&(x-1)
        r++
    }
    return r 
}
```
定义x uint64 = 67348296478，四种解法耗时如下，单位ns
```golang
1:  1252
2:  3130
3:  955
4:  821
```
可以看到，使用FormatUint转换为二进制再遍历耗时最长，x&(x-1)操作耗时最短，毕竟只是针对bit位为1进行了操作，同时解法一的初始化表还是比较耗时的。