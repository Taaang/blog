---
title: RocketMQ groupChannelTable内存泄露引发的线上问题
date: 2024-11-07
categories:
- RocketMQ
tags:
- RocketMQ
---

这次遇到的问题比较特殊，是在某个特定场景下会触发`RocketMQ`的问题。  

虽然问题发生时影响可控，知道原因后还是吓出一身汗，极端情况下可能导致大面积的消费延迟。  

让我们一起来看看这次的问题。  

## 问题现象  

业务反馈`Topic`下消息的消费出现延迟。  

![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rocketmq_mem_leak/1.png?raw=true){:height="500" width="500"}

而且从消费位点上看，消费者是阻塞住的，然后突然某个时间消费一批，然后又阻塞住，整个消费行为是一阵一阵的。  

最初怀疑是业务消费能力不够，但是发现`Consumer`侧负载不高，且扩容后依旧无效，怀疑是`broker`本身有问题，但是监控显示正常。  

![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rocketmq_mem_leak/2.png?raw=true){:height="500" width="500"}
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rocketmq_mem_leak/3.png?raw=true){:height="500" width="500"}

这就让人困惑了，消息两头的监控数据显示都正常，那么问题到底出在哪呢？  

## 问题分析  

业务消费端扩容无效，那么大概率问题还是在`RocketMQ`的`Broker`侧，先抓一下线程堆栈，看看有没有出现线程阻塞或等待现象。  

![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rocketmq_mem_leak/4.png?raw=true){:height="500" width="500"}

从其中一个有消费延迟`topic`的`Broker`中，有出现阻塞的线程只有`HeartbeatThread`，看上去好像不太相关，但是这个线程池的线程应该是主要处理心跳包的，应该也不会阻塞住才对，而且整个`HeartbeatThread`线程池中的所有线程均阻塞在这个方法调用上，这就比较奇怪了。  

### 被`synchronized`方法锁住的对象  

无奈之下，抱着试一试的态度，先翻了一下源码。  

![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rocketmq_mem_leak/5.png?raw=true){:height="500" width="500"}

`registerProducer`方法被`synchronized`修饰，也就是说当方法被调用时，锁住的是`ProducerManager`这个类的对象实例（拥有这个方法的对象）。  

看到这里，就有了一个猜想，有没有可能其他线程调用了这个对象的某个`synchronized`方法？顺着堆栈，搜了一下这个锁住的对象。  

![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rocketmq_mem_leak/6.png?raw=true){:height="500" width="500"}
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rocketmq_mem_leak/7.png?raw=true){:height="500" width="500"}

从 两个不同的`Broker`线程堆栈上看，对应的对象锁当前被`NettyEventExecutor`线程占用，该线程处理I/O相关的事件，且堆栈都指向在`doChannelCloseEvent()`方法中。  

也就意味着抽选两个`Broker`中，`ProducerManager`的对象锁都被`NettyEventExecutor`线程被获取，并用于执行`doChannelCloseEvent()`，这就有点凑巧了。  

那么这个方法主要在做什么呢？  

### 处理客户端关闭的`doChannelCloseEvent`  

看一看源码。  

![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rocketmq_mem_leak/8.png?raw=true){:height="500" width="500"}

我们也可以看到该方法被`synchronized`修饰，这就与我们看到的堆栈相同，当`doChannelCloseEvent`方法执行时，`ProducerManager`的其他`synchronized`会BLOCKED的。  

这个方法逻辑很简单，`groupChannelTable`里保存了所有的`ProvicerGroup`与`Client`的关系，该方法就是遍历所有的producer group，并尝试从中移除对应的client channel，但是两个`Broker`的堆栈都同时显示对象锁都被获取并用于执行`doChannelCloseEvent`方法，有没有可能这个方法耗时比较久，导致其他线程获取不到锁？  

讲道理应该不会，这个情况只有在`Broker`上的producer group数量非常多的时候才会出现，但是实在找不到其他原因了，抱着试一试的态度先查查看。  

### 庞大的`groupChannelTable`  

我们找到对应的`Broker`机器，通过arthas看了一下`ProducerManager`中的`groupChannelTable`，结果令人脑壳疼。  

![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rocketmq_mem_leak/9.png?raw=true){:height="500" width="500"}

怎么会有1.5万个producer group，正常情况下不应该，紧接着细看了一下里面的数据，发现里面有一系列很特殊的group，名字为IotRuleEngine-{timestamp}，导出来搜索计数一下。  

![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rocketmq_mem_leak/10.png?raw=true){:height="500" width="500"}
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rocketmq_mem_leak/11.png?raw=true){:height="500" width="500"}

这下原因基本就清楚了，**确实是由于`groupChannelTable`中的group过多，导致遍历耗时长占用了锁**，最终heartbeat线程被全部阻塞。  

> 而后续和业务确认，发现他们确实会周期性的新建producer group。  

聊到这里，可能你心里还有两个疑问：  

1. heartbeat线程阻塞为什么会导致消费延迟？  
2. 为什么旧的producer group仍然滞留在groupChannelTable没被删除？  

接下来我们一一分析。  

### 心跳检测失败的Consumer  

为什么heartbeat线程阻塞会导致消息堆积？  

![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rocketmq_mem_leak/12.png?raw=true){:height="500" width="500"}

看上去好像两者应该没有关联，我们可以先看看顺着heartbeat方法看看里面做了什么。  

![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rocketmq_mem_leak/13.png?raw=true){:height="500" width="500"}

在heartbeat中，会处理接受到的心跳数据，里面会包含consumer、provider的注册信息，其中发生阻塞的调用就是`registerProducer()`方法，此时如果所有的heartbeat线程池线程均处于BLOCKED，那么也就意味着无法处理新进入的心跳请求并最终超时，而实际上也是这个情况。  

![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rocketmq_mem_leak/14.png?raw=true){:height="500" width="500"}

此时consumer端的心跳请求超时失败，导致consumer注册失败，没有consumer client进而导致消费延迟。  

> 和实际情况也相符合。  
> 实际情况中，会出现消息偶尔突然被消费一批后又中止，就是有consumer心跳请求正巧碰上某个线程获取到锁并处理了这个请求，就会出现短时消费行为。  


### `groupChannelTable`泄露问题  

问题定位到这里，原因知道了，但是这也不符合逻辑，业务周期性新建producer group后，旧group中就没有client了，理论上应该移除这个group才对，那么为什么没有移除呢？  

如果要排查没有移除的原因，就需要顺着源码往回找，转念一想这种情况出现的应该不少，rocketmq社区说不定有人遇到相同的问题，所以先去了github上搜了一下，就找到了一位同学提交的bug。  

![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rocketmq_mem_leak/15.png?raw=true){:height="500" width="500"}

**~[\[Bug\] The producer groupChannelTable memory leak](https://github.com/apache/rocketmq/issues/8673#top)~**

现象、复现方法完全一致，顺着作者提交的issue，就基本能了解原因。  

![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rocketmq_mem_leak/16.png?raw=true){:height="500" width="500"}

**~[\[ISSUE \#8673\] fix producer groupChannelTable memory leak](https://github.com/apache/rocketmq/pull/8672/files#top)~**

这里主要增加的逻辑，就是在producer group对应的client列表为空时，移除掉了`groupChannelTable`中的group key，这也就意味着在此之前`RocketMQ`源码中被没有在client列表为空时移除对应的group key，导致该场景下的`groupChannelTable`出现泄露。  


## 问题根因

总体来看，问题由两个因素共同导致：  

1. `RocketMQ`在producer group的client列表为空时，并没有移除对应的producer group；  
2. 业务侧时间周期性新建producer group，导致空client的producer group数量较多。  

两者结合导致了大量无效的producer group滞留在`groupChannelTable`里，而在收到客户端关闭请求并遍历`groupChannelTable`时，遍历庞大的table耗时较长，导致阻塞了heartbeat线程，consumer周期性心跳注册就会超时，导致consumer下线，消息堆积无法被消费。  

## 解决方案  

### 1. 短期方案 - 业务优化  

我们优先采用了这个方案，业务侧做了优化，不再周期性创建producer group，并重启`RcoketMQ`重置`groupChannelTable`，操作后恢复正常。  

![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rocketmq_mem_leak/17.png?raw=true){:height="500" width="500"}

### 2. 长期方案 - 升级RocketMQ至5.x版本  

**~[\[ISSUE \#8673\] fix producer groupChannelTable memory leak](https://github.com/apache/rocketmq/pull/8672/files#top)~**

翻了一下源码，该issue已经被合入`RocketMQ`主分支，并在5.0.0版本中已经加入，可以通过升级解决。  

![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rocketmq_mem_leak/18.png?raw=true){:height="500" width="500"} 
