---
title: 由浅入深了解半/全连接队列（二） - 溢出篇
date: 2022-05-12
categories:
- 网络
tags:
- TCP
- 半连接队列
- 全连接队列
---

上一篇文章中，我们主要介绍了半/全连接队列的基础知识和溢出判断的方法。但是在实际场景中，队列溢出有不同的可能性，我们也通过例子来简单复现，看看不同的溢出场景下的现象和判断方式。  

## 溢出条件  

回顾一下两个队列的溢出条件：

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/syn_accept_queue_overflowed/1.png){:height="500" width="500"}  

由此可见，溢出的情况包含以下两种：  

（1）**仅半连接队列溢出，全连接队列空余**，即`SYN`阶段半连接队列溢出；  
（2）**全连接队列溢出，半连接队列空余**，即`SYN`或`ACK`阶段，发现全连接队列满，则进行溢出处理；  
（3）**半连接和全脸接队列均溢出**，即在`SYN`阶段半/全连接丢列均有溢出，在`ACK`阶段全连接队列溢出。   

我们通过一个例子来复现这三个场景，实际测试一把。同时，为了更好的测试这两种场景，我们先关闭`syn_cookies`（系统默认情况是开启的），避免该参数对半连接队列溢出的影响。  

## 溢出时的表现  

* **半连接队列溢出**  

意味着短时间内客户端发来的`SYN`包过多，半连接队列无法存放。    
此时，服务端溢出的`SYN`包将被丢弃，不做处理（前提是`syn_cookies`关闭）。客户端由于没有收到返回的`SYN+ACK`，将重发`SYN`包，重发间隔时间以指数级增加，重发次数和`tcp_syn_retries`有关，默认重发6次。  

* **全链接队列溢出**  

短时间内完成三次握手的连接较多，而应用层`accept()`连接的速度慢于完成建连的速度，全连接队列无法放下这些连接，最终溢出。    
此时，溢出连接的处理方式取决于系统参数`tcp_abort_on_overflow`，默认为**0**（FALSE），表示当全连接队列满时，会默认丢弃三次握手中的最后一次`ACK`；设置为1（TRUE）时，则服务端会发送一个`RST`给客户端，表示连接重置。  

## 如何模拟溢出  

要模拟队列溢出并不难，只要有大量客户端并发建连就可以复现溢出的场景。  

*服务端（Kotlin）*  

```
val server = ServerSocket(7788, backlog)
val connectionQueue = LinkedBlockingQueue<Socket>()

while (true) {
    val newConnection = server.accept()
    connectionQueue.add(newConnection)
}

```

*客户端（Go）*  

```
for index := 0; index < connCount; index++ {
    fmt.Println("Connection ", index)
    go tcpConnect()
}

func tcpConnect() {
    conn, err := net.Dial("tcp", "xxx.xxx.xxx.xxx:7788")
    if err != nil {
        fmt.Println("TCP connect error.", err)
    }
}
```

其中，服务端开启监听端口，客户端选择通过Go的协程来模拟短时间内的并发建连。  

> 为了便于控制队列大小，可以把代码中`listen()`时的`backlog`设置的非常大，同时只通过`tcp_max_syn_backlog`和`somaxconn`来控制半/全连接队列大小。  
>  
> 同时，这里也可以用`hping3`以`flood`模式进行SYN攻击，也能模拟客户端并发建连，测试半连接队列溢出。  


## 不同队列大小时的溢出表现  

### 仅半连接队列溢出  

这种场景下，我们把保证全连接队列大小足够大，半连接队列设置的较小。    

```
tcp_max_syn_backlog=15  //实际半连接队列大小会设置为16
somaxconn=20480
```

此时，再通过客户端并发建连2万次，来看看实际的效果。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/syn_accept_queue_overflowed/2.gif)

可以看到`SYNs to LISTEN sockets dropped`发生了变化，从**307904**增加到了**308069**，说明整个过程半连接队列溢出了165次，之后通过几轮的重试，最终完成2万个连接的建立。  

> 因为是在局域网内的两台服务器上进行测试，建连速度非常快，所以很小的半连接队列也能支撑较大的并发建连  

同时，这个过程我们可以看到溢出次数发生了3次变化，这是因为第一批`SYN`包中存在溢出的情况，客户端不会收到`SYN+ACK`，于是短暂间隔后进行重试，而连接重试的时间点也十分相近，导致队列再次溢出。  

通过抓包的内容，我们也可以抽取一个重试的连接来证明。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/syn_accept_queue_overflowed/3.png){:height="500" width="500"}

该连接由于队列溢出，建连`SYN`包被丢弃，之后进行了2次重传，分别间隔1s、2s、4s，最终完成建连。  

我们单独取出TCP SYN重传的请求，也能得到如下请求分布。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/syn_accept_queue_overflowed/4.png){:height="500" width="500"}    

可以看到队列多次溢出丢弃SYN包，导致在不同的时间点进行了SYN重传。  

### 仅全连接队列溢出  

我们将半连接队列大小设置的足够大，将全连接队列改小。  

```
tcp_max_syn_backlog=10240  
somaxconn=128
```

同时，为了能够更好的看到全连接队列溢出的现场，我把`tcp_abort_on_overflow`改为1，所以在全连接队列溢出时，客户端会收到`RST`包，出现`connection reset by peer`。  

我们通过并发建连1万次，来测试溢出的情况。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/syn_accept_queue_overflowed/5.gif)

从结果上看，全连接和半连接队列均有溢出：  

**半连接队列溢出次数** = 382320 - 376158 = 6162  
**全连接队列溢出次数** = 161085 - 154923 = 6162  

此时，由于全连接队列溢出，直接导致新的`SYN`包不会进入半连接队列而被直接丢弃，这种情况下也会记录半连接队列溢出次数+1，所以最终两者均有溢出，且溢出次数相同。  

通过第一个窗口，我们可以看到被Accept的连接共计7632个（从0开始的），那么此时被`RST`连接数也可以通过抓包看到。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/syn_accept_queue_overflowed/6.png){:height="500" width="500"}   

从服务端共发出2368个`RST`包，两者相加正好是1万次。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/syn_accept_queue_overflowed/7.png){:height="500" width="500"}    

单独看`RST`包的统计分布，可以看到在最初建连和第一次`SYN`重传时，均有出现全连接队列溢出。  

> 为什么两个队列的溢出次数会相同？  
>  
> 因为只要全连接队列溢出，那么半连接队列溢出次数也会相应+1。这个场景下，均是全连接队列溢出，所以两者溢出次数相同。全连接队列溢出的判断分别在内核源码的`tcp_v4_syn_recv_sock()`和`tcp_v4_conn_request()`中，`LINUX_MIB_LISTENDROPS`和`LINUX_MIB_LISTENOVERFLOWS`分别记录了半连接队列和全连接队列的溢出次数。  

> 为什么溢出次数大于`RST`次数？  
>  
> 全连接队列溢出会在收到第一个`SYN`和第三个`ACK`的时候进行判断。同时，如果前者阶段溢出，那么只会把`SYN`丢弃，只有后者阶段会返回`RST`，所以溢出次数会大于等于`RST`次数。（实际抓包也会发现确实有很多`SYN`被丢弃，导致后续重传）  

### 半/全连接队列均有溢出  

这次我们将两个队列都设置的比较小。  

```
tcp_max_syn_backlog=128  
somaxconn=128
```

同样通过1万次并发建连进行测试。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/syn_accept_queue_overflowed/8.gif)  

其中，  

**半连接队列溢出次数** = 438858 - 430394 = 8464  
**全连接队列溢出次数** = 206315 - 199758 = 6557  

可以看到两个队列均有溢出，且次数并不相同，我们可以抓到的`RST`包数量进一步分析。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/syn_accept_queue_overflowed/9.png){:height="500" width="500"}   

服务端一共发出了2308个`RST`包，即在第三个`ACK`包收到，完成三次握手阶段，全连接队列溢出了2308次。  

那么，可以得到这样一个分析：  

**ACK阶段全连接队列溢出次数** = 2308 次  
**SYN阶段全连接队列溢出次数** = 6557 - 2308 = 4249 次  
**SYN阶段仅半连接队列溢出次数** = 8464 - 4249 = 4215 次  

## 小结  

综上所述，半/全连接队列的溢出由于队列大小和溢出阶段的不同，会呈现不同的溢出结果。  

|  场景  | 说明 |
| ------- | ------- |
| 仅半连接队列溢出 | 只有半连接队列溢出次数增加 |
| 仅全连接队列溢出 | 半/全连接队列溢出次数均增加，且增加的次数相同|
| 两个队列均溢出 | 半/全连接队列溢出次数均增加，但增加的次数并不相同 |

其中，  

**半连接队列溢出**，主要发生在`SYN`阶段，当半连接队列满或者全连接队列满时，均会导致半连接队列溢出.  

**全连接队列溢出**，包含两种可能性：  
（1）发生在`SYN`阶段，如果发现全连接队列已满，则直接丢弃新的`SYN`包，并记录半/全队列均溢出；  
（2）发生在`ACK`阶段，此时已经完成了三次握手建连，但由于全连接队列满，依然进行溢出处理，同时记录半/全队列均溢出。  
