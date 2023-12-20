---
title: 一个容器缺少Capabilities带来的小问题
date: 2023-12-20
categories:
- linux
tags:
- linux
---

前段时间，有团队的同学在边缘服务器上遇到问题，最终定位和`Linux Capabilities`有关，定位过程并不复杂，回顾过往算是第二次遇到类似问题，所以还是打算记录下来。

## 问题背景

业务有部署在客户侧的边缘服务器，我们会将服务通过云端控制下发到边缘服务器上，这些下沉的业务服务均以容器化的方式进行部署和管理。

在一个业务场景中，会在客户侧的边缘服务器插入一个硬件外设，业务服务初始化时调用厂商脚本安装外设驱动，但是在测试的过程中发现驱动安装一直失败，提示如下异常：

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/linux_cap/1.png){:height="500" width="500"}

由于厂商脚本底层SDK未提供源码，无法通过源码进行问题定位，所以研发同学在这个问题上花了不少时间。

## 问题分析

从报错信息来看，问题点很好判断，脚本内会进行`insmod/rmmod`操作失败，提示的异常是`Operation not permitted`。

看到这里，可能首先想到的就是权限不够，但是初步排查即使以`root`用户身份执行该脚本，也提示相同的异常信息，所以我们当时就想到应该是容器本身不具备某些资源或者权限。

但是这就让研发同学比较抓瞎，没有源码也不好判断具体缺少哪个行为或者权限。

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/linux_cap/2.png){:height="500" width="500"}

这就到了掐指一算的环节了。

因为涉及到权限、资源层面的使用，大部分都需要通过系统调用进行操作，抛出的`Operation not permitted`大概率也是系统调用本身返回了无权限，因此只要看哪一个系统调用返回异常就可以了。

那就是时候掏出`strace`了。

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/linux_cap/3.png){:height="500" width="500"}

最终发现是`delete_module`返回了`Operation not permitted`，追踪一下源码，很快可以找到问题根因。

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/linux_cap/4.png){:height="500" width="500"}

执行`delete_module`需要判断是否具备`CAP_SYS_MODULE`能力，如果不具备则返回`-EPERM`，提示`Operation not permitted`。

那么要解决这个问题，我们需要了解两个东西：

（1）`CAP_SYS_MODULE`是什么？  —  Linux Capabilities
（2）如何让容器具备这个能力？   —  Pod Security Context

## Linux Capabilities

为了能够进行权限检查，传统的UNIX将进程分为特权进程和非特权进程，其中`特权进程`可以绕过所有内核权限检查直接执行对应的操作，而`非特权进程`则需要根据进程认证和权限信息进行判断。

从Linux 2.2开始将超级用户的特权拆分成不同的单元，这些权能单元就是`Linux Capabilities`。

> 官方文档
> [capabilities\(7\) - Linux manual page](https://man7.org/linux/man-pages/man7/capabilities.7.html)

`CAP_SYS_MODULE`就是`Linux Capabilities`中的一种，包含模块加载和卸载相关的权限能力。

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/linux_cap/5.png){:height="500" width="500"}

### Capabilities分配

既然将特权拆分成不同的`Capabilities`模块，那么就可以按需进行`Capabilities`分配，配置方式包含线程权能集（`Thread Capabilities Sets`）和文件权能集（`File Capabilities`），而两种权能集各有其分配计算方式、继承规则，最终共同作用决定是否具备权限进行操作。

> 其中，文件权能集从Linux 2.6.24开始支持。

* Thread Capabilities Sets

每个进程都以下的几种权能集合，每个集合包含0到多个不同的权能。

| 类型          | 说明                                                      |
|-------------|---------------------------------------------------------|
| Permitted   | 进程可获取的权限                                                |
| Inheritable | 从父进程继承传递给子进程的权限                                         |
| Effective   | 进程的有效权限，内核基于该集合进行权限检查<br>（可以理解为最终真正生效的权限集）              |
| Bounding    | 进程允许拥有的最大权限集<br>（Since Linux 2.6.25，在此之前是一个系统级属性限制所有线程） |
| Ambient     | 当前生效的权限，应用于非特权程序的当前进程或子进程                               |

其中，通过`fork()`创建的子进程继承父进程的权能集。

* File Capabilities

从Linux 2.6.24后，内核支持为一个可执行文件设置权能集，保存在`security.capability`中。

| 类型          | 说明                                             |
|-------------|------------------------------------------------|
| Permitted   | 当文件执行时，自动添加给进程的权能集                             |
| Inheritable | 该集合与线程Inheritable集做与&操作，以确定进程哪些权能可被继承          |
| Effective   | 非集合，仅是一个bit位。为1时进程Permitted集才可添加到进程Effective集中 |

* 权能计算方式

有了进程和文件权能集，那么实际执行一个新的程序时，也有其对应的权限计算方式。

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/linux_cap/6.png){:height="500" width="500"}

## Pod Security Context

安全上下文（Security Context）定义 Pod 或 Container 的特权与访问控制设置，其中就包含`Linux权能`，能为进程赋予 root 用户的部分特权而非全部特权。

> [Kubernetes 为 Pod 或容器配置安全上下文](https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/security-context/)

可针对Pod或容器层面进行配置，容器层面的官方配置示例如下：

```
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo-4
spec:
  containers:
  - name: sec-ctx-4
    image: xxxx:1.0
    securityContext:
      capabilities:
        add: ["NET_ADMIN", "SYS_TIME"]
```

因此，结合我们的场景，只需要给容器增加对应的权能即可。

## 解决方案

要解决这个问题，只需要给对应的容器增加`SYS_MODULE`权能即可。

```
....
spec:
  containers:
  - name: xxxx
    image: xxxx:1.0
    securityContext:
      capabilities:
        add: ["SYS_MODULE"]
....
```

> 这里注意，源码中的权能名是`CAP_SYS_MODULE`，但是实际在`securityContext`的时候对应权能名位`SYS_MODULE`

## 问题总结

综上所述，实际为容器不具备对应的权能，导致硬件外设驱动安装失败，只要加上`SYS_MODULE`权能即可。

但是实际上，驱动安装这个行为也并不适合在容器中完成，应该在服务器初始化阶段进行安装，将基础环境初始化与服务部署两个行为解耦开。
