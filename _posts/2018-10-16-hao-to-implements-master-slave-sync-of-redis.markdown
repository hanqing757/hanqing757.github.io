---
layout: post
title:  "如何在一台机器上实现redis的主从同步"
date:   2018-12-01 18:00:00
categories: redis
tags: redis master-slave datasync
excerpt: 在一台机器上实现redis的主从同步
mathjax: true
---

* content
{:toc}

在学习《redis入门指南》这本书的过程中，第八章主要讲redis集群。其中在一台机器上新建两个redis实例，一个作为主库(master)，一个作为从库(slave)，并实现了主从同步。我也在机器上实现了一下，过程梳理如下。

首先启动一个redis实例有以下两种方式
```shell
1 redis-server path/to/redis.conf
2 redis-server --port 6380 
```
第一种是载入配置文件的方式，第二种是在6380端口新起一个redis实例，具体可参考https://redis.io/topics/config。其中说明，启动redis实例常用的是载入配置文件的方式，在测试和开发的时候也可以以传递参数的方式来启动，通过命令行传递参数的格式和redis.conf文件的参数是一致的，只是以--开头，这些参数在内部生成临时配置文件。根据redis.conf配置文件中参数的格式，我们还可以这样写
```shell
redis-server --port 6380 --slaveof 127.0.0.1 6379
```
表示在6380端口新起一个redis实例作为6379端口redis的从库，如果redis主库有密码，再加上 --masterauth xxx即可。

或者我们不想在命令行写参数，新建一个配置文件，将port slaveof masterauth 这三个参数写入，以载入配置文件的方式启动redis，同样可以新建主从redis库，并实现主从同步。
