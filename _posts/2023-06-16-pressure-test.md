---
title: 对，没错，又是压测惹的祸
date: 2023-06-16
categories:
- 压测
tags:
- 压测
- Influxdb
- JMeter
---

最近有一段时间没搞压测了，团队里的同学也在搭建线上压测平台，想着今年七、八月份能够在业务压测中线上实战一把。   

这段时间正好在验证新压测系统，也和旧的压测平台做一些对比验证。讲道理旧系统已经比较稳定，应该不会出什么问题。。  

然鹅，当你以为没有问题的时候，问题就来了。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/pressure_test/1.png){:height="500" width="500"}

全链路压测小队和测试的同学反馈，用旧系统压着压着，就会出现超时的情况，中间没有任何特殊操作。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/pressure_test/2.png){:height="500" width="500"}

这就很奇怪了，旧系统已经稳定操作那么久，我自己也没遇到这个情况。  

那么，这背后到底发生了什么呢？  

## 问题现象  

最直接的问题现象，就是这个异常。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/pressure_test/3.png){:height="500" width="500"}

可以看出，`JMeter`在测试结束时会将请求相关的`metrics`写入到`InfluxDB`，而在`getResult`等到返回时，出现了3s的`SocketTimeoutExcetpion`。  

中间没有任何特殊变更，就是一直在做压测对比实验。  

## 问题分析  

这么看来，中间出现问题的地方可能有很多，从肉鸡请求到数据持久化到`InfluxDB`中，可能是网络、SLB、连接池、慢SQL、InfluxDB负载等。  

既然可能性比较多，那就从最直接的异常点开始反查。  

### InfluxDB

既然出现3s超时，而且是在`getResult`阶段，那么比较大概率是InfluxDB的问题，先排查该时间段InfluxDB的状态。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/pressure_test/4.png){:height="500" width="500"}

从基础的负载指标来看，好像一切正常，基本可以判断不是负载的问题。  

再顺着监控往下查，突然看到这样一段内容。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/pressure_test/5.png){:height="500" width="500"}

emmmmm。。。。  

`Series`类似于关系型数据库里的数据行，每一条`Series`类似于数据表中某个tag在某个时间点的数据，有4000W+倒是有可能，毕竟是压测环境在用，压测数据比较多，逻辑上能接受。  

但是这个`Measurements`类似于关系型数据库里的表，有。。。1600W+。。。这好像就不太对劲了。。  

抱着试一试的态度，我把`InfluxDB`的旧库删了，重建了压测库，初始化完也只有2个`Measurements`，这才符合预期。  

那么问题来了，肉鸡和压测控制面同样的代码啥都没改，之前一点问题都没有，这次怎么搞出这么多`Measurements`呢？  

先来简单了解一下`InfluxDB`的基础知识。  

### InfluxDB中的数据

`InfluxDB`是一种时间序列数据库，每一条数据都是一个数据点，数据点由不同的部分组成。  

 | 名称 | 必选/可选 | 描述 |
| - | --- | -------- | --- |
| `<measurement>` | 必选 | 表示测量值的名称，类似于传统关系型数据库中表的概念。 |
| `<tag_set>` | 可选 | 表示一组 tag 键值对，描述数据的元数据，用于过滤和聚合数据，例如 tag1=value1,tag2=value2，每个标签值对应于一个数据库索引，使得可以快速查询和过滤数据。 |
| `<field_set>` | 可选 | 表示一组 field 键值对，用于描述每个数据点的度量值，例如 field1=value1,field2=value2，可通过处理和聚合字段数据来计算各种统计信息、平均值、总和等。 |
| `[timestamp]` | 可选 | 表示数据点的时间戳，可以是一个 Unix 时间戳（整数）或一个 InfluxDB 时间格式字符串，例如 2015-08-18T00:00:00Z。如果未指定时间戳，则默认使用当前时间。 |

这么看可能有点难以理解，举个栗子。  

例如，在监控环境中，用一个`measurement`来存储服务器监控数据，其中每个数据点使用`tag`来标识服务器的名称、操作系统版本、数据中心等信息，使用多个`field`来记录CPU、内存、网络等指标。这样，可以方便地查询、过滤和计算特定标签和字段的数据。  

### Measurements

了解的`InfluxDB`的基础知识后，顺着上面的疑问，先看看这1000多万个`Measurements`都是啥。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/pressure_test/6.png){:height="500" width="500"}

？？？？  

怎么`Measurement`的名字都是JSON格式的字符串，而且内容还不完整？  

但是不管怎么说，至少有了方向，在InfluxDB数据写入模块的代码里搜搜JSON字符串中的关键字，应该就能找到问题触发的代码，debug一把应该就能知道原因了。  

但是。。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/pressure_test/7.png){:height="500" width="500"}

难道和代码无关？  

### JMeter脚本

再翻看了所有可能和压测有关的代码、脚本后，最终我找到了这段内容的原始出处。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/pressure_test/8.png){:height="500" width="500"}

对，在JMeter的断言脚本里，脚本逻辑也很简单，在程序返回非2XX时，会把responseBody记录下来，格式化后并打印出来，记录到`AssertionResult`里。  

好像也没什么不妥，但是通过异常信息反推，还是能发现一些端倪。  

* 最终异常和InfluxDB有关，InfluxDB会记录**请求异常**时的结果数据，中间能产生关联的方式只有通过`AssertionResult`。    

* 异常的Measurement名称与格式化后的字符串并不完全相同，缺少了`”assertion failed! response data: %s`，并且还多了一个`}`。    

我们一个一个来看。  

### JMeter InfluxDB模块  

先来看第一点，JMeter执行完一次采样请求后，会处理其结果。在我们的场景中，因为使用的`InfluxDB`模块记录采样数据，最终会进入到`InfluxdbBackendListenerClient.handleSampleResults()`中。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/pressure_test/9.png){:height="500" width="500"}

该方法会将采样结果中的关键信息进行提取和保存。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/pressure_test/10.png){:height="500" width="500"}

在提取采样结果的过程中，会区分采样请求是否成功，如果不成功，就会创建`ErrorMetric`。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/pressure_test/11.png){:height="500" width="500"}

在构造过程中，`responseCode`还未设置，值为空字符串，而我们在JMeter断言脚本中，采样请求失败时设置了`AssertionResult.setFailureMessage`，因此会走入第一个逻辑分支，`responseCode`和`responseMessage`均会被赋值。  

最终，创建成功的`ErrorMetric`会被放入`HttpMetricSender`中进行请求，发送给InfluxDB。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/pressure_test/12.png){:height="500" width="500"}

在这个过程中，你会发现存入`HttpMetricSender`中的待发送metric是一个拼接好的字符串，而里面就有`responseCode`和`responseMessage`。  

那这就奇怪了，为什么要组装成一个字符串交给`HttpMetriceSender`呢？  

### InfluxDB Line Protocol

`InfluxDB Line Protocol`是一种轻量级文本协议，用于向InfluxDB中写入数据。相对于其他协议，Line Protocol更加简单易用，以单行形式描述一个数据点，适用于以时间序列方式进行的数据采集和存储场景。  

其语法也比较简单，以约定好的格式进行单行写入即可。  

```
// Syntax
<measurement>[,<tag_key>=<tag_value>[,<tag_key>=<tag_value>]] <field_key>=<field_value>[,<field_key>=<field_value>] [<timestamp>]

// Example
myMeasurement,tag1=value1,tag2=value2 fieldKey="fieldValue" 1556813561098000000
```

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/pressure_test/13.png){:height="500" width="500"}

所以，上文提到的`ErrorMetric`会拼接成一个`Line Protocol`协议格式的文本数据，由`HttpMetriceSender`发送给InfluxDB进行写入。  

再回到下面这张图，我们可以看看具体写入的`tag`和`field`是怎样的。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/pressure_test/14.png){:height="500" width="500"}

```
fagent,application={application},transaction={transaction},responseCode={responseCode},responseMessage={responseMessage},userTag={userTag} metric_count={count} {timestamp}
```

其中，`tag`部分到userTag截止，`field`只有metricCount，最后加上时间戳，这样看好像也并没有什么问题。  

那我们再回到这张异常`measurement`图。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/pressure_test/15.png){:height="500" width="500"}

里面的异常`measurement`的名字，都是前文`JMeter`断言脚本中的一部分。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/pressure_test/16.png){:height="500" width="500"}

而被截断的部分，正好是在断言脚本中`response data: %s`的后面。  

::那么，有没有一种可能，`response data`记录的是接口的实际返回数据，而接口返回的是格式化后的`json`数据，如果该数据中有换行符，又被整合到拼接到`ErrorMetric`生成的字符串中，按照`InfluxDB`的`Line Protocol`定义，就会被当成多条数据写入。::  

举个栗子，顺着上面的数据格式，我们把`responseMessage`带入进去，即`responseMesssage`中的`response data`是一个格式化后的`json`数据。  

假设`responseMessage`的值为`assertion failed! response data: {\n    "code":0,\n    "message":"ok"\n}, not contains: xxx, token: xxx, traceId: xxx`  

```
fagent,application={application},transaction={transaction},responseCode={responseCode},responseMessage=assertion failed! response data: {
    "code":0,
    "message":"ok"
}, not contains: xxx, token: xxx, traceId: xxx ,userTag={userTag} metric_count={count} {timestamp}
```

带入后，由于格式化`json`中有换行符，最终原本单行的数据，被拆分成4行，而在最后一行中，其格式正好是符合`Line Protocol`，变成了一条单独写入的行数据，即下面的内容：  

```
}, note contains: xxx, token: xxx, traceId: xxx ,userTag={userTag} metric_count={count} {timestamp}
```

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/pressure_test/17.png){:height="500" width="500"}

这正好和我们的现象一致，基本可以断定就是这个问题导致的。  

## 问题根因  

综上看来，根本原因是我们的断言脚本中，会在请求状态码非200时，将请求的返回的`json`异常信息记录到`failureMessage`中，而`JMeter`的会将采样异常的请求记录为`ErrorMetric`，其中包含了断言结果中的错误信息`AssertionResult.failureMessage`，以`Line Protocol`写入到`InfluxDB`的时候，由于格式错误导致新建了大量的`measurement`，影响了查询性能，导致大量查询3s超时。  


## 解决方法  

解决方法其实也很简单。  

由于测试的同学确实希望记录采样请求异常时的`failureMessage`，所以这个内容需要保留。在保留的同时，对`failureMessage`进行处理，移除其中的换行符，保证以`Line Protocol`写入`InfluxDB`时不会出现格式错误就行。最终问题也彻底解决。  

## 总结  

其实问题并不复杂，主要是中间涉及到的细节点比较多，如果了解`InfluxDB`的`Line Protocol`，确实很难想到会和这里的写入格式错误有关。  

同时，这里也涉及`JMeter`底层对于异常数据的采集和聚合问题，从逻辑上来说`failureMessage`可以用作`InfluxDB`的数据行`tag`，但是如果对其底层不了解的话，也容易带来数据聚合无法收敛的问题，例如异常中包含了唯一键信息。  
