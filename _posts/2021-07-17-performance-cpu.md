---
title: 性能分析与排查 - CPU
date: 2021-05-13
categories:
- 性能
tags:
- MySQL
- 性能
---

最近在公司做了一次简单的技术分享，主要讲性能问题的分析与排查，首次主要以 CPU 为主题进行，这里也把 PPT 整理成文章。

本文将会介绍 CPU 性能相关的基础知识，并通过简单的例子介绍常见的 CPU 性能问题排查工具和方法。

## 使用之前先了解 - CPU性能
在聊到 CPU 性能优劣的时候，可能很多人首先想到的就是核数，认为核数越多，性能越强，了解更多的同学可能也会关心频率。

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/performance_cpu/1.png){:height="500" width="500"}  

那么 CPU 的实际性能，要通过什么参数来判断呢，来看看`Inte`官方给出的`i7 8700`性能相关参数。

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/performance_cpu/2.png){:height="500" width="500"}  

与 CPU 性能相关的参数并不少，在我之前的文章中有介绍过，这里就不重复说明。

其中，我们接触最多的应该就是核数与线程数，接下来我们就通过一系列的例子，来看看各种场景下的 CPU 性能问题分析方法。


## 到底忙不忙？ - CPU 资源利用
我们常常要面对这样一个问题，服务、中间件压测，或者定位线上的突发延迟问题，需要判断服务器或者容器的压力，这种情况我们一般会首先看 CPU ，但是要怎么判断 CPU 忙不忙呢？

### 已经跑到 100% ，CPU 忙碌吗？

先来看第一个例子，我们跑了段代码，并通过`top`看到的线程信息如下：

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/performance_cpu/3.png){:height="500" width="500"}  

此时的 CPU 使用率到达 100% ，它忙碌吗？

答案当然是不一定，大家可能会说信息不足，都不知道 CPU 能支持多少线程并发，需要通过`/proc/cpuinfo`来看看 CPU 基础信息。

这里提供一个更快的方法，在`top`命令界面下，按下数字键`1`，可以看到所有核（物理核+逻辑核）的使用情况。

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/performance_cpu/4.png){:height="500" width="500"}  

可以看到只有 1 核在用户态跑满，剩下的 11 核处于空闲状态，所以 CPU 并不忙碌。

### 1200% ！CPU 这回忙了吧

再看第二个例子，

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/performance_cpu/5.png){:height="500" width="500"}  

CPU 使用率已经到 1200%，在看各核的使用情况，

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/performance_cpu/6.png){:height="500" width="500"}  

基本每个核都跑满了，这次 CPU 是真的忙碌了。

那么问题来了，CPU 为什么这么忙，相比第一个例子，为什么本例能跑满 CPU 呢？

要想真正把所有核心利用起来，那么肯定需要多线程的支持，怎么验证呢？

```
1. top -p {pid} //只查看某个进程的信息
2. H   //查看进程下所有子线程的信息
```

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/performance_cpu/7.png){:height="500" width="500"}  

这样一看，原因就很简单了。

进程创建了 12 个线程，每个线程都交由 1 个核心执行，并把核心跑满，因此整个 CPU 都处于忙碌的状态。

### 满载 or 超载？

上面的两种场景相对简单，比较好判断，但是往往我们需要知道一台服务器的负载，CPU 负载是需要衡量的指标之一，那么要怎么判断 CPU 是否超载呢？

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/performance_cpu/8.png){:height="500" width="500"}  

从 CPU 使用率上看跑满了 1200%，但是和第二个例子有什么不同吗，此时的 CPU 是满载，还是超载了呢？

一般想要判断 CPU 负载，可以关注`top`界面中的一个数据，`Load Average`。

`Load Average`（图片右上角）主要描述了在最近1分钟、5分钟和15分钟内CPU 的负载情况，那么要怎么判断呢？

先来了解一下数值的含义，举个栗子：

行车过桥。

单核 CPU （不支持超线程）每个时间点只能执行一个线程，就类似一座桥每个时间点只能通过一辆车，此时桥的`Load Average`就是 1。

如果同时有另外一辆车要过桥，就要等待前车通过后才能前进，那么此时的负载就是 2，即意味一辆车在过桥，一辆车在等待，此时的桥就处于超载状态。

回归例子本身，在当前环境下共有 12 核，而最近 1 分钟的`Load Average`已经到达 24，说明此时每个核心除了在执行的线程外，还有一个额外的线程在等待被执行，此时的 CPU 是超载的。

（对比第二个例子，`Load Average`在 12 左右，是处于满载而非过载的状态）

## 这是在干啥呀 - CPU 行为分析
之前的例子主要帮忙我们判断 CPU 忙不忙，同时你也一定会好奇，CPU 到底在忙什么？

### 飞奔的 CPU 都在干啥

看回前面 CPU 满载的例子，

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/performance_cpu/9.png){:height="500" width="500"}  

整个 CPU 已经跑满了，我们一共跑了 12 个线程，各跑满满一个核心，通过`Load Average`判断当前处于满载状态。

CPU 这么忙，到底在忙什么呢？

要想知道 CPU 在忙什么，就要找到每个线程在做的事情，最常用的方法就是`pstack / jstack`，看到的结果如下：

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/performance_cpu/10.png){:height="500" width="500"}  

可以看到每个线程当前的执行栈快照，所有的线程都在执行`calc()`。

除了`pstack / jstack`命令外，还可以使用调试神奇`gdb`，`attach`到对应的线程，并通过`info threads`查看所有子线程的执行信息，得到的结果如下：

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/performance_cpu/11.png){:height="500" width="500"}  

和`pstack`的结果一致，均在执行`calc()`方法。

在一些相对简单的场景下，可以通过这两种方法快速的看到 CPU 在做什么，结果也一目了然。

### 同样在飞奔，CPU 的忙碌有什么不同？

再来看一个新的例子，

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/performance_cpu/12.png){:height="500" width="500"}  

线程当前的 CPU 使用率是 100% ，此时只有一个核心（`CPU1`）跑满。

同样是跑满一个核心，和文章中第一个例子有什么不同吗？

我们细看`CPU1`的使用情况，发现本例中跑满的 100% 使用率，有 16.5% 是在`us`下，有 83.5% 是在`sy`下，而文章中的第一个例子是 100% 都跑在`us`，这面的`us`和`sy`有何不同？

要想分析清楚，首先需要了解`top`命令中描述 CPU 状态的维度。

|   维度 | 说明 |
| ------- | ------- |
| us | User Time，运行在用户空间的百分比 |
| sy | Kernel Time，运行在内核空间的百分比 |
| ni | Nice Time，运行低优先级线程的百分比 |
| id | Idle Time，空闲的百分比 |
| wa | IO Wait Time，IO等待占用 CPU 的百分比 |
| hi | Hardware Interrupts Time，硬中断占用 CPU 的百分比 |
| si | Software Interrupts Time，软中断占用 CPU 的百分比 |

本例在`sy`中的使用占比更高，即 CPU 更多运行在内核空间，涉及到系统调用和底层系统资源的管理和使用。

那么如何知道 CPU 在内核空间做什么呢？

之前我们有提到，可以通过`pstack / jstack`或者`gdb`来查看，但是正常线程的调用栈很深，并不好找到对应的系统调用行为，这种情况下可以通过`strace`来查看系统调用行为。

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/performance_cpu/13.png){:height="500" width="500"}  

这样可以很直观的看到，线程不停的在执行`write`方法写入两个字符（”a\0”）到文件中，而要文件IO就是在内核空间完成的，所以最终导致 CPU `sy`较高。

> （题外话环节 ↓↓↓）
> 如果要找到问题根源，很多时候需要找到被写入的文件，才能定位到代码排查问题，那么通过上面的信息要怎么做呢？
>  
> 首先，我们要了解`write`方法，后两个参数描述写入的内容，第一个参数是文件描述符`FD`，再通过`lsof`或者`/proc/{pid}/fd`找到`FD`对应的文件。

## 瞬时所见并非真实 - CPU 行为统计
虽然上面介绍了一些方法，可以看到 CPU 在某个时间点在做什么，但是在实际的问题分析中，我们需要观察 CPU 持续一段时间的行为，并进行分析统计才能找到问题的根源所在。

### CPU 大部分时间在忙什么？

下面依然是一颗飞奔的 CPU，

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/performance_cpu/14.png){:height="500" width="500"}  

我们想知道 CPU 在干啥，可以通过`pstack / jstack`或者`gdb info threads`来看，此时你可能看到这样的结果：

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/performance_cpu/15.png){:height="500" width="500"}  

不同的线程在做不同的事情，这种场景下我们无法通过某个时间点的 CPU 行为直接找到占用资源最多的方法。

要想找到 CPU 跑满的根本原因，需要多一段时间的 CPU 行为进行统计，这里介绍 Linux 下的性能分析神奇——`perf`，它是一个综合性的性能分析工具，我们可以通过`perf record`对一段时间内的 CPU 执行堆栈数据进行采样，并通过`perf report`进行统计分析。

```
perf record -F 90 -g -a -p {pid}
perf report
```

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/performance_cpu/16.png){:height="500" width="500"}  

从统计结果中可以直观的看到，`calc3`的 CPU 资源占用达到 59%，是最高的，其次是`calc2`的 26%，这就能知道到底具体是哪个方法占用资源过多。

当然如果你想获得更加全面的 CPU 行为统计信息，可以配合使用`FlameGraph`来生成`on-cpu`火焰图。

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/performance_cpu/17.png){:height="500" width="500"}  

图中不仅能看到各个函数的 CPU 资源使用比例，还能看到调用依赖信息。
（具体不说明了，GitHub FlameGraph 上有详细`on-cpu`火焰图生成的说明，火焰图的食用方法可自行百度）

### CPU 为什么这么闲？

再来看一个新的例子，

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/performance_cpu/18.png){:height="500" width="500"}  

我们新建了很多线程，可是 CPU 的使用率很低，只有少数线程进行。

那么问题来了，我已经新建了这么多线程可以执行， CPU 为什么还会这么闲呢？

线程没有被执行， CPU 资源仍有空余（idle 91.3%），说明是线程本身处于中断状态（Thread Status = R，处于休眠、受阻或者资源等待），此时我们需要关注不再是 CPU 正在做什么，而是线程没有被调度运行，究竟是在等什么。

这种情况下，我们需要关注的是`off-cpu`，可以使用`bcc`提供的`offcputime`进行数据采样，最终配合`FlameGraph`生成`off-cpu`火焰图。

```
// 对PID为21874的进程及其子线程进行数据采样
// -U仅采集用户空间调用栈，-f显示折叠的结果，采集时间 5s
offcputime-bpfcc -p 21874 -f -U 5 > output.folded
// 利用折叠后的结果，生成火焰图
FlameGraph/flamegraph.pl output.folded > output.svg
```

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/performance_cpu/19.png){:height="500" width="500"}  

可以看出， 有接近一半的时间处于`nanosleep`，还有一半的时间处于资源等待`lock_wait`，从源码里我们也可以对应上。

```
pthread_mutex_t mute;
void thread_func() {
        while (1) {
                pthread_mutex_lock(&mute);
                int loop = 10000;
                int x, y, sum = 0;
                for (x = 0; x < loop; ++x) {
                    for (y = 0; y < loop; ++y) {
                        sum += y;
                    }
                }
                pthread_mutex_unlock(&mute);
                sleep(1);
        }
}
```

## 总结

* **CPU 性能**

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/performance_cpu/20.png){:height="500" width="500"}  

和 CPU 性能有关的参数很多，和我们关系最大的应该是核心、频率，在定位性能问题前需要先了解基本的 CPU 硬件性能参数，并通过一些基础命令查看当前 CPU 的运行状态。

* **CPU 资源使用**

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/performance_cpu/21.png){:height="500" width="500"}  

在 CPU 的资源使用率上，我们可以关注整体使用率、各个核心的使用，同时通过 Load Average 来判断当前 CPU 的负载情况。

* **CPU 行为分析**

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/performance_cpu/22.png){:height="500" width="500"}  

在分析 CPU 行为时，常见的两种是用户态忙和内核态忙，都可以通过对应的方法来找到原因。这里其实还有更多情况，比如软中断占用（`sync flood`时会出现，可以通过hping3模拟）、硬中断占用、Steal 占用（遇到过一次，出现在容器内，是宿主机资源超卖时导致），而不同的情况也可能需要配合一些其他的命令排查，常见的包括`lsof`、`dmesg`等。

* **CPU 行为统计**

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/performance_cpu/23.png){:height="500" width="500"}  

两种情况，通过`on-cpu`和`off-cpu`火焰图可以进行分析，其实在 CPU 未被利用时，本质上不是 CPU 在等什么，更多是找线程为什么被阻塞、挂起。
