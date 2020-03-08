---
layout: post
title:  "Nginx与php-fpm通信之Unix domain socket"
date:   2020-3-6 12:00:00
categories: Unix-domain-socket
tags: Nginx php-fpm  Unix-domain-socket
excerpt: Nginx与php-fpm通信之Unix domain socket
mathjax: true
---

* content
{:toc}

# Unix domain socket
网络通信使用Socket，Tcp Socket是我们所熟知的，但是还有另一种Socket，Unix domain socket，它类似于Tcp Socket，用于用一台机器上不同进程之间进行通信，主要使用socket 套接字文件，在linux上表现为以.sock结尾的文件，文件类型为s。
当Tcp Socket用于同一台机器上不同进程之间通信时，需要跨越Tcp层及IP层，一系列的协议交互，当在不同机器的不同进程之间通信时，还需要从网卡将数据发出，经过物理层传输。相比之下，Unix domain socket只能用于同一台机器不同进程之间通信，通过socket文件进行数据传输，效率高于Tcp Socket。
在常用的LNMP架构中，Nginx与php之间通过fastcgi进行通信，那么Nginx与php-fpm之间的通信就有上述两种方式，常用的在nginx中配置将请求代理到本机的某端口，在fpm配置中监听此端口，可进行Tcp Scoket通信，那么Unix domian socket是如何进行的呢？同样的方式，建立socket文件，nginx将请求代理到socket文件，fpm监听此socket文件。
首先我们在/dev/shm目录下新建socket文件
```shell
touch php-fpm.sock
```
Nginx配置
```shell
fastcgi_pass unix:/dev/shm/php-fpm.sock;
```
php-fpm配置
```shell
listen=/dev/shm/php-fpm.sock
```
重启php-fpm和nginx，可以看到php-fpm.sock文件已经由普通文件变成了socket文件
```shell
srwxrwxrwx 1 root root 0 Mar  6 00:24 php-fpm.sock
```
然后我们修改php-fpm.sock文件的权限为777。此时可以成功请求。
我们找一个利用Unix domain socket通信的fpm进程

![php-fpm-process](/img/php-fpm-process.png)

查看进程打开的文件描述符

![proc-fd](/img/proc-fd.png)

查看描述符对应的文件

![fd-unix-domain-socket](/img/fd-unix-domain-socket.png)

可以看到，打开的是socket文件
同样，我们看一下当nginx和fpm之间使用Tcp Socket通信时，文件描述符和对应打开的文件

![tcp-socket](/img/tcp-socket.png)

看到使用的是Tcp Socket进行通信。那么这两种Socket通信的性能如何呢？我们使用ab基准测试测一下

# Tcp socket与Unix domain socket性能比较
我们分别在并发10、总数1000，并发100、总数10000，并发1000、总数100000，并发10000、总数100000下进行测试，我们主要观察请求的平均耗时参数

![TcpSocketandudsperformance](/img/TcpSocketandudsperformance.png)

通过比较我们得出以下结论：
（1）在并发10和100能够正常完成全部请求的情况下，Unix domain Socket的性能优于Tcp Socket。
（2）在并发1000的情况下，Tcp socket可以正常完成全部请求，但是Unix domain socket 有将近一半的失败请求。
（3）并发10000请求总数100000时，两种socket都出现了 Connection reset by peer，由于并发太高，php-fpm超时之后重置连接。
（4）随着并发数和请求总数的提升，Unix domain socket性能提升明显优于Tcp socket。

优于Tcp是面向连接的协议，nginx通过Tcp socket进行代理的时候需要loopback，申请端口以及Tcp相关资源，所以处理性能劣于Tcp socket，但是在并发数大量增加的时候，由于没有面向连接协议的支撑，Unix domain socket很容易出错不返回，因此我们在低并发比如1000以内我们选择 Unix domain socket，因为它比较轻量级；在高并发的时候我们选择Tcp socket。当我们选择了Unix domain socket，而并发数突然增长的时候，我们有没有呢办法优化呢？有，使用backlog，下面我们先了解一下backlog。

# 增大backlog提高并发能力
首先我们看一下socket建立和Tcp三次握手的过程

![setup-socket](/img/setup-socket.png)

有的同学可能好奇了，我们此时使用的是Unix domain socket，为什么要用Tcp socket的过程来分析呢？其实不管哪种socket，建立的过程都是一样的，只不过Tcp socket在bind的时候绑定的是ip和端口，Unix domain socket在bind的时候绑定的是sock文件而已。
server端在bind，listen之后准备好建立socket连接，在底层维护了SYNC_RECEIVED队列和ESTABLISHED队列，当server端接收到client的sync请求时，将请求放入sync_quene中，给client回复sync/ack，当再次收到client的ack之后，将请求从sync_quene移到accept_quene中，所以，当client进行三次握手的sync时，如果server端的队列已满，server端不做任何回应，直到client端超时重试。backlog就是指的accept_quene的大小，系统的sync_quene和accept_quene分别由/proc/sys/net/ipv4/tcp_max_syn_backlog和/proc/sys/net/core/somaxconn指定。所以，
（1）php-fpm的backlog太小，请求根本无法进入accept_quene，会报 "502 bad gateway"，一般我们甚至backlog=qps
（2）php-fpm的backlog太大，导致超过了php-fpm的处理能力，长时间不返回，nginx等待超时，断开连接，报"504 gateway timeout"；当php-fpm处理完成返回数据时，发现连接已断开。
根据以上，我们分别增大nginx和php-fpm的backlog大小到1024和4096，我们再次用ab进行基准测试

![uds-increase-backlog](/img/uds-increase-backlog.png)

此时，可以正常的完成全部请求，增大backlog的操作是有效的。