---
layout: post
title: TCP连接建立
categories: [网络基础, TCP]
date: 2022-07-07
keywords: tcp
---

TCP三次握手和四次挥手。

本文不讨论SSL的七次握手

## 三次握手

头两次，双方互认身份，确定client和server。第三次，告诉server，我要开始确认了你的身份，我们开始传输吧。

<img src="/assets/images/20220707/tcp_connect_3times.png" width="60%">

第三次的必要性：如果client在server发出SYN后掉线，则认定握手失败，释放TCP链接。（预设：Server的资源远比Client重要）


## 四次挥手

由于TCP连接已经建立，所以中间会存在数据的传输。为了确保server将数据发送完成，所以挥手的次数比握手多一次。

多的一次，就是为了等待Server确认可以关闭了。

<img src="/assets/images/20220707/tcp_goodbye_4times.png" width="60%">

很多时候，操作数据库是，Client连接不释放，从而导致大量的time_wait、或者fin_wait_2。这是因为TCP释放有一个时间段，这个时间段内如果出现大量的过期未主动close的连接，就出导致大量TCP连接状态为time_wait等，进而导致无服务或者服务阻塞现象。

同样的，在高并发情况下，大量连接的释放，会使一段时间内，大量TCP连接处于time_wait的等待期，从而造成网路阻塞。

