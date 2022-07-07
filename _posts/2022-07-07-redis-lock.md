---
layout: post
title: Redis分布式锁
categories: 技术笔记, 后端技术, Redis
date: 2021-07-07 18:30:00
keywords: redis
---

Redis分布式锁在几种异常场景下的解决方案。

# Redis 分布式锁

Created: July 7, 2021 6:30 PM

参考：

分布式锁主要通过：setnx 和 expire来实现。大致会出现以下问题

## 一、主备切换

主节点加锁成功后宕机，此时从节点没有同步到。切换后，会出现加锁失败的情况。

<img src="/assets/images/20220707/master_lose.png" width="60%">

## 二、集群脑裂

由于网络原因，slave节点无法感知到master节点，进入slave升级为master，导致出现多个master以供set操作。

## 三、死锁

一般出现的情况为：setnx后，expire命令执行失败。

解决方案：使用set key value nx ex seconds_num。

- NX 表示只有当 lock_resource_id 对应的 key 值不存在的时候才能 SET 成功。保证了只有第一个请求的客户端才能获得锁，而其它客户端在锁被释放之前都无法获得锁。
- EX 10 表示这个锁 seconds_num 秒钟后会自动过期，业务可以根据实际情况设置这个时间的大小。

但是这种方式仍然不能彻底解决分布式锁超时问题：

- 锁被提前释放。业务逻辑没有在限定时间内执行完成，锁被自动释放了。
- 锁被误删。锁被提前释放后，被另一个线程B申请了，如果此时当前线程执行DEL操作，则会删除线程B持有的锁。同理的，可能造成连锁反应。

补充的，可以在执行DEL操作前，增加判断逻辑，确认锁的持有者是自身。

锁的提前释放，可以通过引用Redisson框架进行解决，原理是：定时扫描锁在临机释放时，其持有者是否具备释放条件，如果不具备，重新设置一次过期时间。

要点：重设过期时间是当前进程控制的，这样可以防止客户端掉线后导致的永久占有锁。

<img src="/assets/images/20220707/lock_by_lua.png" width="60%">

## 四、分布式情况下的锁丢失

以上情况，都是处置单机下的锁丢失。集群下，如果master在锁同步前异常，由于slave异步同步的特性，锁会丢失。

解决方案：引入redlock。原理是：使用lua，向所有节点请求锁，只有超过半数成功，才算锁定。

除此之外，为了防止节点崩溃后造成锁丢失，可通过延时重启节点来保护锁安全。延时时间应大于锁的过期时间。


## 参考

[分布式锁的实现之 redis 篇](https://xiaomi-info.github.io/2019/12/17/redis-distributed-lock/)

[浅析 Redis 分布式锁解决方案](https://www.infoq.cn/article/dvaaj71f4fbqsxmgvdce)
