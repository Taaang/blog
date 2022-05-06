---
title: 由浅入深了解半/全连接队列（一） - 基础篇
date: 2022-05-06
categories:
- 网络
tags:
- TCP
- 半连接队列
- 全连接队列
---

最近项目上遇到一个问题，运维反馈有一个服务会不定时出现健康检查异常，于是去排查了一下。问题最终发现和半连接、全连接队列大小有关，所以打算好好研究一下。  

（本文涉及的Linux内核源码以v3.10为主）  

## 一个偶发的线上问题  

该项目通过WebSocket对外提供服务，每次出现异常时，都会伴随一些现象发生：  

1. 健康检查提示请求超时。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/syn_accept_queue/1.png){:height="500" width="500"}  

2. 通过监控发现在异常时间段，有大量状态码是101的HTTP请求，并且upstream_time时间较长。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/syn_accept_queue/2.png){:height="500" width="500"}  

可以看出，当时有大量客户端同时进行WebSocket建连，而该项目的健康检查机制是通过TCP进行端口建连来判断服务是否健康，很大可能是实例短时无法承载大量建连导致异常。  

和并发建连有关的参数，首先想到的就是半连接、全连接队列，于是登录到容器中通过`netstat -s`查看两个队列是否有溢出。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/syn_accept_queue/3.png){:height="500" width="500"}  

可以看到半连接、全连接队列均有溢出的情况，所以可以初步推断是这两个队列大小的问题，最终通过扩大全连接队列的大小也解决了这个问题。  

那么，到底什么是半/全连接队列呢？  

## 什么是半/全连接队列

在TCP三次握手进行建连的过程中，Linux内核会通过这两个队列来进行连接过程状态的缓存和维护。  

*半连接队列*，也称SYN队列，当服务端收到客户端发来的`SYN`请求后，就会将该连接存储到半连接队列中，并向客户端回复`SYN+ACK`。  

*全连接队列*，也称ACCEPT队列，当服务端收到`ACK`回复，完成三次握手后，就会将该连接从半连接队列中移除，放入全连接队列中，等待应用进程调用`accept()`取走连接。  

对应的过程如下：  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/syn_accept_queue/4.png){:height="500" width="500"}    

## 半连接队列  

### 半连接队列大小  

主要受两个参数的影响，`backlog`和`tcp_max_syn_backlog`。  

在Linux内核源码中，半连接队列的空间分配在`net/core/request_socket.c`中的`reqsk_queue_alloc()`方法里进行，队列大小计算逻辑如下：  

```
//其中，nr_table_entries记录最终队列大小
nr_table_entries = min_t(u32, nr_table_entries, sysctl_max_syn_backlog);
nr_table_entries = max_t(u32, nr_table_entries, 8);
nr_table_entries = roundup_pow_of_two(nr_table_entries + 1);
```

队列大小受两个参数影响：  

* `backlog`。源码中，`nr_table_entries`的初始值为服务端`listen()`时传入的`backlog`；  
* `sysctl_max_syn_backlog`。该参数即是系统参数`tcp_max_syn_backlog `，通过`sysctl`可以查看和设置（源码中默认为256）。  

队列大小会取两者的最小值，同时必须大于等于8，并将得到的结果进行`roundup_pow_of_two`运算，该操作会找到当前数值的二进制最高位数n，然后以1循环右移n次作为结果集，简单理解就是按2的倍数向上取整。  

举个栗子，如果`backlog`设置为128，`sysctl_max_syn_backlog`配置为64，则计算过程简化如下：  

```
半连接队列大小 = roundup_pow_of_two(min(128, 64)) = 128
```

### 半连接队列溢出的条件  

在内核源码`net/ipv4/tcp_ipv4.c`的`tcp_v4_conn_request`方法中，半连接队列有两个主要的场景：  

1. 半连接队列满了，且未开启`syn_cookies`  

```
if (inet_csk_reqsk_queue_is_full(sk) && !isn) {
	want_cookie = tcp_syn_flood_action(sk, skb, "TCP");
	if (!want_cookie)
		goto drop;
}
```
（关于`syn_cookies`，我们之后单独介绍）   

2. 全连接队列满了，且半连接队列中待回复`SYN+ACK`的连接超过1个  

```
if (sk_acceptq_is_full(sk) && inet_csk_reqsk_queue_young(sk) > 1) {
	NET_INC_STATS_BH(sock_net(sk), LINUX_MIB_LISTENOVERFLOWS);
	goto drop;
}
```
可以看到这种情况下，不仅半连接队列溢出+1，全连接队列溢出也会+1。  

### 半连接溢出的场景  

1. 短时间内大量并发建连，队列空间不足以存放收到的SYN包；  
2. 全连接队列已满，半连接队列直接丢弃新的SYN包（即使当前仍有空间）；  
3. 建连时间过长，半连接队列积压导致溢出。  

## 全连接队列  

### 全连接队列大小  

全连接队列大小的分配在`net/socket.c`中，计算逻辑如下：  

```
somaxconn = sock_net(sock->sk)->core.sysctl_somaxconn;
if ((unsigned int)backlog > somaxconn) {
	backlog = somaxconn;
}
```

队列大小受两个参数影响：  

* `backlog`。由服务端`listen()`时传入的`backlog`决定；  
* `sysctl_somaxconn`。该参数同样为系统参数，通过`sysctl`可以查看和设置（源码中默认为128）。  

全连接队列大小会设置为`backlog`和`somaxconn`中较小的值。  

### 全连接队列溢出的条件  

全连接对出的判断就比较简单了，在`net/ipv4/tcp_ipv4.c`中进行了判断：  

```
if (sk_acceptq_is_full(sk))
	goto exit_overflow;
```

即在完成了三次握手，得到一个有效的`synack`时，如果发现全连接队列满了，则将其丢弃。  

### 全连接队列溢出的场景  

1. 短时间内大量连接完成三次握手，队列放不下；  
2. 业务层`accept()`速度较慢，全连接队列积压。  

## 如何判断队列是否溢出  

判断的方法很多，例如`ss`、`netstat`、`/proc/net/netstat`，这里主要介绍`netstat`，通过`-s`参数可以查看到所有网络统计信息。  

```
netstat -s

...
    85336 times the listen queue of a socket overflowed  //全连接队列溢出的累计次数
    206804 SYNs to LISTEN sockets dropped  //半连接队列溢出的累计次数
...
```

> `netstat`是如何统计这些信息的？
>  
> 看了一眼`netstat`的源码，也是从`/proc/net/netstat`中读取的。
> 其中，`/proc/net/netstat`中的数据来源于内核计数，相关定义在内核源码的`net/ipv4/proc.c`中，当内核发现半/全连接队列满，会相应的计数+1，对应`LINUX_MIB_LISTENDROPS`和`LINUX_MIB_LISTENOVERFLOWS`。

## 问题如何解决  

既然知道了是两个队列均有溢出，那么就先从队列大小看起。  

前文有提到，半/全连接队列大小与几个参数有关，`tcp_max_syn_backlog`、`somaxconn`和端口监听时传入的`backlog`。  

先来看看系统参数：  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/syn_accept_queue/5.png){:height="500" width="500"}  

可以看到系统参数配置中，半/全连接队列的大小均是配置为4096，对于我们的服务来说并不小。  

那么再看代码中传入的`backlog`，该服务是前端Node实现的连接层，listen时没有传入`backlog`参数，源码中会默认设置为511。  

此时，    
半连接队列大小 = roundup_pow_of_two(min(4096, 511)) = 512    
全连接队列大小 = min(4096, 511) = 511    
`Socket`中两个队列的实际大小都偏小。  

最终我们将`backlog`参数设置为与系统一致的4096后，就没有再出现队列溢出的问题。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/syn_accept_queue/6.png){:height="500" width="500"}    

> 当两个队列均没有溢出时，`netstat`的统计并不会有这两个结果，想要进一步确认也可以通过`/proc/net/stat`进行判断。

## 总结
如果了解半/全连接及基础的溢出判断方法，那么对于这一类问题的定位还是比较好解决的，先通过 `netstat -s`看半/全连接队列是否有溢出，再通过系统参数和`backlog`入参判断两个队列的实际大小是否合理。  

但是，实际上遇到连接队列溢出的问题时并不好判断，本文更多通过一个简单的线上问题介绍半/全连接队列的概念。  

想要了解更多半/全连接队列的问题，可以期待之后的实战及周边知识文章，持续更新中~  
