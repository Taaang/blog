---
title: 一个线上问题看各家线程池拒绝策略
date: 2022-05-19
categories:
- 线程池
tags:
- 线程池
- 拒绝策略
---

又到了烧高香的季节，业务旺季来了。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/reject_policy/1.png){:height="500" width="500"}  

之前线上项目偶发出现线程池耗尽的问题，最近终于有空能好好研究一把，问题实际并不复杂，也得益于Dubbo的线程池拒绝策略才能很快找到大致的原因。  

通过这个问题，也有些好奇各家使用的线程池拒绝策略是怎样的，刨刨坑、挖挖土，一起来看看吧~  

## 问题背景  

之前线上偶发出现线程池耗尽问题，现象如下：  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/reject_policy/2.png){:height="500" width="500"}  

在调用下游`Dubbo`接口时，提示`Server`端的线程池耗尽。  

最开始以为是有突发流量，但是监控显示流量稳定，并且扩容后发现问题依然存在，渐渐意识到问题并不简单。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/reject_policy/3.png){:height="500" width="500"}    

## 问题分析

既然有异常日志和堆栈，先看看到底什么场景下会出现这个异常。在`Dubbo`源码中，我们可以找到这一段提示出现在`AbortPolicyWithReport`中。  

```
public class AbortPolicyWithReport extends ThreadPoolExecutor.AbortPolicy
```

`AbortPolicyWithReport`继承自`java.util.concurrent.ThreadPoolExecutor.AbortPolicy`，是一种线程池拒绝策略，当线程池中的缓冲任务队列满，且线程数量达到最大时，就会触发拒绝策略，调用拒绝策略的`rejectedExecution()`方法进行处理。  

那么，有哪些不同的拒绝策略呢？  

### JDK线程池拒绝策略  

在`java.util.concurrent.ThreadPoolExecutor`，我们可以找到JDK预设置的四种拒绝策略：  

* **CallerRunsPolicy - 调用者线程处理**  

该策略下，如果线程池未关闭，则交由当前调用者线程进行处理，否则直接丢弃任务。  

```
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    if (!e.isShutdown()) {
        r.run();
    }
}
```

* **AbortPolicy - 抛出异常**  

如果不配置拒绝策略的话，线程池会默认使用该策略，直接抛出`rejectedExecution`，交由上层业务处理。  

```
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    throw new RejectedExecutionException("...");
}
```

* **DiscardPolicy - 丢弃当前任务**  

最简单的处理方法，直接丢弃。  

```
//实际方法体就是空的，即该场景下不处理，直接丢弃
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
}
```

* **DiscardOldestPolicy - 丢弃最老的任务**  

该策略是丢弃队列中最老的任务（其实就是下一个要执行的任务），并尝试执行当前任务。  

```
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    if (!e.isShutdown()) {
        e.getQueue().poll();
        e.execute(r);
    }
}
```

### Dubbo线程池拒绝策略  

那么`Dubbo`的拒绝策略是怎样的呢？  

其实从名字就能看出来，`AbortPolicyWithReport`。  

```
public class AbortPolicyWithReport extends ThreadPoolExecutor.AbortPolicy {

@Override
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    String msg = String.format("Thread pool is EXHAUSTED!" + ...);
    logger.warn(msg);
    dumpJStack();
    dispatchThreadPoolExhaustedEvent(msg);
    throw new RejectedExecutionException(msg);
}

}
```

`Dubbo`的拒绝策略是抛出异常`RejectedExecutionException`，同时还会做一件事情 - `dumpJStack()`，记录下当时的JVM线程堆栈。  

### dumpJStack  

先来看看源码。  

```
private void dumpJStack() {

   //一些dump时间间隔和并发控制
   ...

    //新建单线程池用于dump堆栈
    ExecutorService pool = Executors.newSingleThreadExecutor();
    pool.execute(() -> {
        ...
		  try (FileOutputStream jStackStream = new FileOutputStream(
        new File(dumpPath, "Dubbo_JStack.log" + "." + dateStr))) {
            JVMUtil.jstack(jStackStream);
        } catch (Throwable t) {
            logger.error("dump jStack error", t);
        }
        ...
    });
    ...
}
```

做法其实很简单，最终调用`JVMUtil.jstack`把当前JVM的线程堆栈dump下来，而这样做有一个很大的好处，就是能知道当时其他线程到底在做什么，帮助分析线程池溢出的原因。  

## 原因分析  

有线程堆栈就好办了，看看当时线程都在做什么。  

`Dubbo`底层使用`Netty`实现网络通信，涉及的线程池包括IO线程池（boss、worker）和业务线程池（处理业务事件）。通过之前的日志可以看到是Server端的业务线程池，即`DubboServerHandler`耗尽，那么统计一把，看看线程都在做什么。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/reject_policy/4.png){:height="500" width="500"}  

很明显，大量的线程都阻塞在获取DB连接上，DB连接池里连接数不够了。那接下来就好办了，可以看看同时间段是不是有慢查询长时间占住了连接，或者是真的连接池小了，线程池和连接池配比不对，分析至此就不继续展开（并不是讨论重点哈哈）。  

## 不同的拒绝策略  

可以看到，`Dubbo`通过重写了拒绝了策略，来帮助异常场景下进行问题定位，带来了很大的帮助。  

那么其他主流组件是怎么做的呢？  

### RocketMQ  

以`Broker`为例，其中包含了非常多的线程池用于处理不同的消息处理场景，包含send、put、pull、query等等。  

在线程池的使用上，`RocketMQ`通过`BrokerFixedThreadPoolExecutor`继承封装了一层`ThreadPoolExecutor`，上层可以自行传入参数，其中也包含了可配置的`RejectedExecutionHandler`。  

实际在`Broker`创建消息处理的不同线程池时，并没有指定特殊的拒绝策略，所以使用的是默认的`AbortPolicy`，即丢弃消息。  

```
this.sendMessageExecutor = new BrokerFixedThreadPoolExecutor(
    this.brokerConfig.getSendMessageThreadPoolNums(),
    this.brokerConfig.getSendMessageThreadPoolNums(),
    1000 * 60,
    TimeUnit.MILLISECONDS,
    this.sendThreadPoolQueue,
    new ThreadFactoryImpl("SendMessageThread_")
    //并没有设置拒绝策略
);
```

同时为了避免任务溢出，为每个线程池默认设置了较大的任务队列大小。  

```
private int sendThreadPoolQueueCapacity = 10000;
private int putThreadPoolQueueCapacity = 10000;
private int pullThreadPoolQueueCapacity = 100000;
private int replyThreadPoolQueueCapacity = 10000;
private int queryThreadPoolQueueCapacity = 20000;
private int clientManagerThreadPoolQueueCapacity = 1000000;
private int consumerManagerThreadPoolQueueCapacity = 1000000;
private int heartbeatThreadPoolQueueCapacity = 50000;
private int endTransactionPoolQueueCapacity = 100000;
```

综上，`RocketMQ`的拒绝策略使用了`AbortPolicy`，即抛出异常，同时为了避免任务队列溢出，设置了较大的任务队列。  

### Netty  

以`EventLoopGroup`为例，线程池的拒绝策略默认使用`RejectedExecutionHandlers`，通过单例模式提供`Handler`进行处理。  

```
public final class RejectedExecutionHandlers {
    private static final RejectedExecutionHandler REJECT = new RejectedExecutionHandler() {
        @Override
        public void rejected(Runnable task, SingleThreadEventExecutor executor) {
            throw new RejectedExecutionException();
        }
    };

    private RejectedExecutionHandlers() { }

    public static RejectedExecutionHandler reject() {
        return REJECT;
    }

    ...
}
```

可以看出，`Netty`的拒绝策略默认也是抛出异常，与`RocketMQ`对比的不同的点在于，任务队列的大小会取`max(16, maxPendingTasks)`，`io.netty.eventLoop.maxPengdingTasks`可通过环境变量进行配置。  

### Doris  

团队内一直在用`Doris`，属于计算存储分离、MPP架构的分析型存储组件，看了一眼FE中的拒绝策略，官方实现了两种：  

**LogDiscardPolicy**  

```
static class LogDiscardPolicy implements RejectedExecutionHandler {

    private static final Logger LOG = LogManager.getLogger(LogDiscardPolicy.class);

    private String threadPoolName;

    public LogDiscardPolicy(String threadPoolName) {
        this.threadPoolName = threadPoolName;
    }

    @Override
    public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
       LOG.warn("Task " + r.toString() + " rejected from " + threadPoolName + " " + executor.toString());
    }
}
```

可以理解就是`DiscardPolicy`，丢弃任务，同时记录warn日志。  

**BlockedPolicy**  

```
static class BlockedPolicy implements RejectedExecutionHandler {
    private String threadPoolName;
    private int timeoutSeconds;

    public BlockedPolicy(String threadPoolName, int timeoutSeconds) {
        this.threadPoolName = threadPoolName;
        this.timeoutSeconds = timeoutSeconds;
    }

    @Override
    public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
        try {
            executor.getQueue().offer(r, timeoutSeconds, TimeUnit.SECONDS);
        } catch (InterruptedException e) {
            LOG.warn("Task " + r.toString() + " wait to enqueue in " + threadPoolName + " " + executor.toString() + " failed");
        }
    }
}
```

这种策略会特殊一些，它会阻塞住当前线程，尽最大努力尝试将任务放入队列中。如果超过指定的阻塞时间`timeoutSeconds`（默认60s），仍然无法将任务放入队列中，则记录warn日志，并丢弃任务。  

这两种策略在`Doris`中都有实际使用到，同时线程池的任务队列大小默认设置为10。  

### ElasticSearch  

`ES`的拒绝策略相对复杂一些，其自定义实现了两种拒绝策略。  

* **EsAbortPolicy**

```
public class EsAbortPolicy extends EsRejectedExecutionHandler {

    @Override
    public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
        if (r instanceof AbstractRunnable) {
            if (((AbstractRunnable) r).isForceExecution()) {
                BlockingQueue<Runnable> queue = executor.getQueue();
                if ((queue instanceof SizeBlockingQueue) == false) {
                    throw new IllegalStateException("forced execution, but expected a size queue");
                }
                try {
                    ((SizeBlockingQueue<Runnable>) queue).forcePut(r);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    throw new IllegalStateException("forced execution, but got interrupted", e);
                }
                return;
            }
        }
        incrementRejections();
        throw newRejectedException(r, executor, executor.isShutdown());
    }
}
```

其实本质上就是`AbortPolicy`，但是会进行一些特殊处理，包括`forceExecution`强制执行的判断、任务拒绝次数统计，最终抛出异常。  

> ES中，线程池的`forceExecution`是指什么？
>  
> 在满足条件的情况下，即使用了ES自定义的`AbstractRunnable`进行任务封装、`SizeBlockingQueue`作为任务队列时，可以根据任务配置判断是否强制放入任务队列。对于一些比较重要的任务，不能丢弃时，可以将`forceExecution `设置为true。
>  
> 强制放入任务队列带来的效果取决于`SizeBlockingQueue`中封装的队列类型，如果封装的是`ArrayBlockingQueue`，则会阻塞等待队列有空余；如果封装的是`LinkedTransferQueue`，由于队列大小无限，且put使用的是ASYNC模式，所以会立刻放入队列并返回。

* **ForceQueuePolicy**  

```
static class ForceQueuePolicy extends EsRejectedExecutionHandler {

        ...

        @Override
        public void rejectedExecution(Runnable task, ThreadPoolExecutor executor) {
            if (rejectAfterShutdown) {
                if (executor.isShutdown()) {
                    reject(executor, task);
                } else {
                    put(executor, task);
                    if (executor.isShutdown() && executor.remove(task)) {
                        reject(executor, task);
                    }
                }
            } else {
                put(executor, task);
            }
        }

        private static void put(ThreadPoolExecutor executor, Runnable task) {
            final BlockingQueue<Runnable> queue = executor.getQueue();
            // force queue policy should only be used with a scaling queue
            assert queue instanceof ExecutorScalingQueue;
            try {
                queue.put(task);
            } catch (final InterruptedException e) {
                assert false : "a scaling queue never blocks so a put to it can never be interrupted";
                throw new AssertionError(e);
            }
        }

        private void reject(ThreadPoolExecutor executor, Runnable task) {
            incrementRejections();
            throw newRejectedException(task, executor, true);
        }
    }

}
```

该策略在线程池未关闭，且使用了ES自定义的`ExecutorScalingQueue`的任务队列时，会强制将任务放入线程池队列中。其中，`ExecutorScalingQueue`也是继承自`LinkedTransferQueue`，最终调用put方法以ASYNC模式放入任务队列中。  

> 看上去也是`forceExecution`，而且最终都是使用`LinkedTransferQueue`的put方法以ASYNC模式非阻塞入队列。那么`EsAbortPolicy`和`ForceQueuePolicy`有什么不同呢？
>  
> 两者有很多相似点，都有`forceExecution`的判断，而且拒绝时都是抛出`RejectedExecutionException`。
>  
> 不同点在于，`ForceQueuePolicy`默认采用强制执行模式，且在线程池关闭时依然可能往队列放入任务。

### 其他

在GitHub上随意翻了一下，也有看到用策略链的方式，实现也很简单，可以随意组合配置不同的策略。  

```
public class RejectedExecutionChainPolicy implements RejectedExecutionHandler {

    private final RejectedExecutionHandler[] handlerChain;

    @Override
    public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
        for (RejectedExecutionHandler handler : handlerChain) {
            handler.rejectedExecution(r, executor);
        }
    }
}
```


## 总结  

拒绝策略主要应用在线程池出现资源溢出的情况下，除了常见的由JDK提供的四种拒绝策略外，不同的组件也会尝试使用不同的拒绝策略来应用。  

**JDK提供的拒绝策略**  

|  类型  |  说明 |
| ------- | ------- |
| CallerRunsPolicy | JDK提供，调用者线程处理 |
| AbortPolicy | JDK线程池默认使用，抛出`RejectedExecutionException`异常|
| DiscardPolicy | JDK提供，丢弃当前任务 |
| DiscardOldestPolicy | JDK提供，丢弃下一个要执行的任务 |

**自定义拒绝策略**  

|  组件 | 类型  |  说明 |
| ------- | ------- | ------ |
| RocketMQ | AbortPolicy | 使用的线程池默认拒绝的略，即`AbortPolicy`|
| Dubbo | AbortPolicyWithReport | 抛出`RejectedExecutionException`异常，并报告溢出，记录下JVM线程堆栈|
| Netty | RejectedExecutionHandlers | 逻辑与`AbortPolicy`一致，抛出异常，封装为单例Handler使用|
| Doris | LogDiscardPolicy | 逻辑与`DiscardPolicy`一致，丢弃任务，并记录warn日志|
| Doris | BlockedPolicy | 尽最大努力尝试将任务放入队列执行，最多等待60s，超时后记录warn日志，并丢弃任务|
| Elastic | EsAbortPolicy | 正常情况下与`AbortPolicy`一致，如果线程标记强制执行，则强制执行或放入任务队列，实际入队列的表现取决于队列类型，可能阻塞或立即返回|
| Elastic | ForceQueuePolicy | 默认强制执行任务或放入任务队列，异步非阻塞 |
| 其他 | PolicyChain | 策略链，包含多种拒绝策略，根据条件与节点处理结果决定最终表现 |
