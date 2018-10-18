---
layout: post
title:  "如何把自己的项目发布为composer包"
date:   2018-10-17 20:28:00
categories: composer
tags: composer
excerpt: 如何将php项目发布为composer包
mathjax: true
---

* content
{:toc}

写文章前，说一下自己最近的感悟吧。最近学习到一个概念叫做"持续输出"，第一次看到"持续输出"是在微博上一个人讲如何高效阅读，每读完一定量的文字比如一小节或者一章内容需要写读书笔记，将学习到的东西转化为自己的沉淀和思考，如果只是流水一般看过去，可能很快就会忘记。包括最近我在公司的老司机也会跟我讲需要将学习到的东西持续的输出，写文章，如果你还不能把一个东西讲明白说明你还没有学懂，当你能够真正的输出一些东西，哪怕是很小的知识点，记忆就会比你只看过深刻很多倍，当你养成持续输出的好习惯，知识也会成系统。

这段时间在学习shell，今天在《shell脚本学习指南》当中看到对文本块排序的方式，通常一条记录是一行的时候我们使用sort排序即可，但是当一条记录有多行的时候我们需要借助于awk或者其他命令，比如如下的文本
```shell
#SORT KEY zhang san
zhang san
unter den 78
Canada

#SORT KEY qing han
qing han
xierqi haidian
China

#SORT KEY chuchu li
chuchu li
xujiaxu shanghai
China
```
当我们不能像sort一样使用-k指定排序字段时，就要添加额外的标记比如以上记录#开头的行，那么具体如何使用呢。首先我们的思路就是将每条记录由多行变为一行再进行排序，可以
将每行的换行符替换为一个特殊的不可打印字符，可以使用awk，但是awk默认每条记录是一行，对每行进行操作，可以使用RS=""，这就表示记录以空行的方式隔开
```shell
awk -v RS="" '{gsub("\n","^Z");print}'
```
gsub函数表示全局替换。此处还有一个坑，一般会在windows下将待排序文件写好，当放在linux下时，用vim打开，使用:set ff查看文件格式为dos，需要:set ff=unix将文件格式变为unix下的文件，因为unix/win/mac下的文本换行符不同，分别是\n,\r\n,\r。并且此处的不可打印字符^Z的输入方式为 ctrl-v ctrl-z。

通过以上方式将每条记录变为一行后再用sort排序，再将^Z变为\n输出即可，

```shell
cat myfriends|
awk -v RS="" '{gsub("\n","^Z");print}'|
sort -f|
awk -v ORS="\n\n" '{gsub("^Z","\n");print}'|
grep -v "#SORT KEY"
```

其中，ORS="\n\n"，表示空行为输出记录的分隔符，因为在第一个awk之后输出的记录已经以换行分割了。
