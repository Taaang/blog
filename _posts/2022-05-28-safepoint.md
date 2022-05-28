---
title: 带你彻底了解JVM SafePoint
date: 2022-05-28
categories:
- JVM
tags:
- JVM
- SafePoint
---

之前听了大佬的JVM工作坊，其中有提到`SafePoint`，一直大概知道这个概念，理解很模糊，也比较好奇JVM是如何实现的，所以打算好好研究一把。  

（本文涉及的源码及解析以jdk1.8为主）  

## 从GC说起  

我们知道在GC的过程中，JVM使用可达性分析来判断对象是否有被使用，分析的起点就是`GC Root`。  

以CMS为例。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/safepoint/1.png){:height="500" width="500"}  

其中，为了能准确的找到`GC Root`关联的引用对象，需要一个机制来保证遍历过程中不会出现引用关系变化，这个机制就是`STW`。  

*STW (Stop The World)*，通过字面很容易猜到，该操作仿佛会暂定整个JVM，类似做了一次快照，此时JVM会暂停所有的应用线程，对象引用、线程栈和寄存器、静态对象等均不会被修改变化。  

那么问题来了，  
STW是如何实现的呢？  
要怎样才能在需要的时候通知所有应用线程暂停？  
应用线程又是如何知道需要暂停？  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/safepoint/2.png){:height="500" width="500"}  

答案就是，*SafePoint*，  
今天就带你深入底层，  
真正了解什么是`SafePoint`。  

## SafePoint  

### 什么是SafePoint  


先来看看HotSpot术语表是如何描述的：  

```
A point during program execution at which all GC roots are known and all heap object contents are consistent. From a global point of view, all threads must block at a safepoint before the GC can run. (Except thread running JNI code)
```

在`SafePoint`时，所有的 GC Roots 和堆对象内容将宝支持一致。从全局来看，所有的线程需要在该时刻阻塞，以便于执行 GC。  

### 在哪些场景会进入SafePoint  

在JVM中，存在一个单例的原始线程`VM Thread`，它会产生创建所有其他JVM线程，同时也会被其他线程用于执行一些成本较高的`VM Operions`。这些`VM Operations`会保存在`VMOperationQueue`中，通过自轮询方法`loop()`进行处理。  

（可参考源码`/hotspot/src/share/vm/runtime/vmThread.cpp`）  

而在`VM Thread`退出及轮询处理`VM Operations`的过程中，需要`STW`来保证结果的准确性，就调用`SafepointSynchronize::begin()`进入`SafePoint`。而处理完`VM Operations`后，通过调用`SafepointSynchronize::end()`退出`SafePoint`。  

* `VM Thread`退出时  

意味着JVM即将关闭，此时会进入`SafePoint`，进行退出前的检查，并等待native thread执行完毕，最终关闭整个JVM。  

> 通过源码可以发现，此时的`SafePoint`过程只有begin，没有end。因为JVM已经要关闭了，没有必要再退出。

* 处理`VM Operations`时  

在处理一些JVM行为时，会进入`SafePoint`，这些JVM行为定义在`/hotspot/src/share/vm/runtime/vm_operations.hpp`中。  

```
#define VM_OPS_DO(template)                       \
  template(Dummy)                                 \
  template(ThreadStop)                            \
  template(ThreadDump)                            \
  template(PrintThreads)                          \
  template(FindDeadlocks)                         \
  template(ForceSafepoint)                        \
  template(ForceAsyncSafepoint)                   \
  template(Deoptimize)                            \
  template(DeoptimizeFrame)                       \
  template(DeoptimizeAll)                         \
  template(ZombieAll)                             \
  template(UnlinkSymbols)                         \
  template(Verify)                                \
  template(PrintJNI)                              \
  template(HeapDumper)                            \
  template(DeoptimizeTheWorld)                    \
  template(CollectForMetadataAllocation)          \
  template(GC_HeapInspection)                     \
  template(GenCollectFull)                        \
  template(GenCollectFullConcurrent)              \
  template(GenCollectForAllocation)               \
  template(ParallelGCFailedAllocation)            \
  template(ParallelGCSystemGC)                    \
  template(CGC_Operation)                         \
  template(CMS_Initial_Mark)                      \
  template(CMS_Final_Remark)                      \
  template(G1CollectFull)                         \
  template(G1CollectForAllocation)                \
  template(G1IncCollectionPause)                  \
  template(DestroyAllocationContext)              \
  template(EnableBiasedLocking)                   \
  template(RevokeBias)                            \
  template(BulkRevokeBias)                        \
  template(PopulateDumpSharedSpace)               \
  template(JNIFunctionTableCopier)                \
  template(RedefineClasses)                       \
  template(GetOwnedMonitorInfo)                   \
  template(GetObjectMonitorUsage)                 \
  template(GetCurrentContendedMonitor)            \
  template(GetStackTrace)                         \
  template(GetMultipleStackTraces)                \
  template(GetAllStackTraces)                     \
  template(GetThreadListStackTraces)              \
  template(GetFrameCount)                         \
  template(GetFrameLocation)                      \
  template(ChangeBreakpoints)                     \
  template(GetOrSetLocal)                         \
  template(GetCurrentLocation)                    \
  template(EnterInterpOnlyMode)                   \
  template(ChangeSingleStep)                      \
  template(HeapWalkOperation)                     \
  template(HeapIterateOperation)                  \
  template(ReportJavaOutOfMemory)                 \
  template(JFRCheckpoint)                         \
  template(Exit)                                  \
  template(LinuxDllLoad)                          \
  template(RotateGCLog)                           \
  template(WhiteBoxOperation)                     \
  template(ClassLoaderStatsOperation)             \
  template(JFROldObject)                          \
```

从中看到一些我们熟悉的身影，包括CMS中的两次STW（CMS_Initial_Mark、CMS_Final_Remark）、JVM thread dump时的STW（ThreadDump）等等。  

总体可以归结在以下几个行为类别中：  

（1）GC行为  
（2）偏向锁相关行为  
（3）调试行为（threaddump、heapdump、stacktrace等）  
（4）反优化行为  
（5）部分JNI相关行为  

### SafePoint的进入过程  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/safepoint/3.png){:height="500" width="500"}  

### 如何进入SafePoint  

这就是一个比较复杂的过程了，JVM线程可能在处理不同的事务，处于不同的状态，为了能够让它们能够进入`SafePoint`，JVM设计了不同的机制。  

|  线程状态  | 机制 |
| ------- | ------- |
| 解释执行中 | 修改解释器dispatch table，在下一条字节码执行前，强制检查`SafePoint`条件 |
| 本地方法代码段执行中 | 当线程从本地方法调用返回时，必须检查`SafePoint`条件|
| 编译执行中 | JIT会在编译后的指令中插入检查点，检查点会进行`SafePoint`条件判断|
| 阻塞中| 已处于阻塞中，使线程保持阻塞状态即可|
| 状态转换中| 此时会等待线程完成状态转换后阻塞其自身|

### 底层原理  

看到这里你可能会好奇，线程会在不同的时机检查自己是否需要进入`SafePoint`，但是这个过程是怎样的呢？  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/safepoint/4.png){:height="500" width="500"}  

对于解释执行来说，只需要告诉解释器在下一条字节码执行前，做条件检查即可，而JNI方法调用结束时，会默认进行`SafePoint`检查。  

但是对于JIT编译执行的代码段，已经全部编译生成为可执行的机器码，要怎么判断是否要进入`SafePoint`呢？  

在回答这个问题之前，我们先来了解一些概念。  

* *Polling Page*  

`Polling Page`可以理解为是JVM申请的一块页空间，是专门用于`SafePoint`检测。  

在JVM初始化时，会通过mmap申请一块大小为`Linux:page_siez()`（即4K）的空间作为`Polling Page`。  

```
polling_page = (address) ::mmap(NULL, Linux::page_size(), PROT_READ, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
```

在初始化时，页空间会设置为可读。  

* *SIGSEGV*  

这是一个内核发送出来的信号，当线程尝试访问不被允许访问的内存区域，或以错误的类型访问内存区域时，会发出该信号。  

* *JVM线程中的信号处理*  

JVM线程会通过函数`JVM_handle_linux_signal`来处理收到的信号，在函数内会根据信号的类型进行不同的处理。  

当收到`SIGSEGV`信号，且接收信号的线程是应用线程，就会判断发生段错误的地址是不是`Polling Page`。  

```
if (thread->thread_state() == _thread_in_Java) {
  if (sig == SIGSEGV && os::is_poll_address((address)info->si_addr)) {
    stub = SharedRuntime::get_poll_stub(pc);
  }
}

static bool is_poll_address(address addr)  {
    return addr >= _polling_page && addr < (_polling_page + os::vm_page_size());
}
```

此时，如果发生段错误的地址高于`_polling_page`地址，且在以`_polling_page`地址为起始的一个页空间内，则认为是`Polling Page`发生了段错误，触发`stub`地址对应的注册函数进行处理，而这个注册函数，就是`SafepointSynchronize::handle_polling_page_exception`。  

* *JIT埋入的test指令*  

我们都知道，JIT会将热点代码编译为本地指令，以加速执行过程。在这个过程中，JIT会在代码段的不同位置埋入`test`指令，该指令会对读取两个参数进行与运算，并将结果保存在标志寄存器中。  

* *底层实现*  

问题的答案很简单，只要把上面的过程串联起来。  

`test`指令就是实现`SafePoint`检查的关键，指令的其中一个参数会设置为`Polling Page`的起始地址。  

（1）当不需要进入`SafePoint`时，由于`Polling Page`会初始化为可读状态，所以`test`执行会执行成功，线程继续执行；  

（2）当需要进入`SafePoint`时，JVM会将`Polling Page`设置为不可读，此时`test`命令由于尝试读取一块并不允许访问的内存区域而触发段错误，内核发出`SIGSEGV`信号，此时线程收到信号后，就会调用注册的`SafepointSynchronize::handle_polling_page_exception`函数进行处理，最终进入`SafePoint`。  

### 如何验证

扯了这么多，谁知道你是不是瞎扯，怎么验证呢？  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/safepoint/5.png){:height="500" width="500"}  

方法很简单，我们找一个业务上用到的JAR包来试试看。  

在启动时，带上参数`-XX:+PrintAssembly`，让JVM在执行过程中打印出原始的汇编指令，搜索`test`指令可以看到下面的结果。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/safepoint/6.png){:height="500" width="500"}  

第一个参数不重要，重点看第二个参数，（%rip）表示需要使用RIP相对寻址，实际的地址空间需要通过指令地址及偏移量计算出来，以第一条为例：  

```
实际地址 = 0x00007f478d183022 + 0x171ca0d8 = 0x7f47a434d0fa
```

此时也能会发现，通过下面的其他`test`指令，算出来的实际地址都指向了同一个内存地址`0x7f47a434d0fa`。  

`pmap`一把，看看这块虚拟地址空间。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/safepoint/7.png){:height="500" width="500"}  

可以看到是一块4K大小的匿名空间，且处于可读状态。这块区域大概率就是我们要找的`Polling Page`，但是怎么验证进入`SafePoint`时的状态变化呢？  

是时候掏出神器`systemtap`，看看对应的内核函数参数就知道了。Linux内核对VMA的状态修改使用的是`mprotect`中的`mprotect_fixup`，我们来hook一把。  

先来看看`mprotect_fixup`方法。  

```
int
mprotect_fixup(struct vm_area_struct *vma, struct vm_area_struct **pprev,
	unsigned long start, unsigned long end, unsigned long newflags)
```

其中，`start`和`end`分别是要修改区域的起始和结束位置，`newflags`就是更改后的VMA Mode。相对应的systemtap脚本如下：  

```
probe kernel.function("mprotect_fixup").call {
        time_now = gettimeofday_ms();
        target_address = strtol(@1, 16);
        if ($start == target_address) {
                printf("[%ld] mprotect target address : [%lx - %lx] - %lx\n", time_now, $start, $end, $newflags);
        }
}
```

脚本会传入要监控的虚拟地址，在`mprotect_fixup`调用时，检查是否对指定地址就行变更，如果是，则打印日志输出`newflags`。  

为了更明显的看到效果，我们通过GC时的`SafePoint`检查来进行验证，同时在日志中打印了GC时间和mprotect时间，以此来对应数据。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/safepoint/8.png){:height="500" width="500"}  

先来看时间，`mprotect`被执行的时间是1653707324018，即2022-05-28 11:08:44.015，和上面的GC时间吻合，意味着在这次YGC时，使用了`mprotect`对地址`0x7f47a434d000 ~ 0x7f47a434e000`进行了状态修改，大小4K，与我们之前找到的`Polling Page`地址段一致。  

在这个过程中，`Polling Page`的状态做了两次调整，先修改为了`8000070`，之后立刻修改为了`8000071`，对应状态的含义可以在内核源码`mm.h`中找到。  

```
#define VM_READ		0x00000001

#define VM_MAYREAD	0x00000010
#define VM_MAYWRITE	0x00000020
#define VM_MAYEXEC	0x00000040

#define VM_MERGEABLE	0x80000000
```

VMA的mode状态是由不同的状态位或运算的结果，`8000071`表示该内存区域可进行合并，可配置读写及执行权限，当前处于可读状态；`8000070`则是在此基础之上关闭了当前的读状态。  

由此可见，在进行GC之前，JVM将`Polling Page`设置为`8000070`的不可读状态，此时`test`指令读取地址内容失败，触发段错误，最终进入`SafePoint`。待GC结束后，将`Polling Page`设置为可读状态，结束`SafePoint`。  

## 小结

综上所述，JVM通过`SafePoint`来通知所有线程需要阻塞，以此来实现`STW`，在底层实现上方法也比较巧妙，通过控制内存页空间的状态来触发段错误，在处理`SIGSEGV`信号时来实现线程进入`SafePoint`。  
