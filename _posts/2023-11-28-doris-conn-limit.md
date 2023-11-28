---
title: 小踩坑 - Doris单租户最大连接限制
date: 2023-11-28
categories:
- Doris
tags:
- Doris
---

很早之前团队就开始用的`Doris`，且是用在业务场景，而非大数据的聚合分析场景。  

这天有业务的同学反馈测试环境的`Doris`连不上，且在重启、删除元数据等一系列操作后，还是无法进行正常连接，所以协助排查了一把。最终问题也并不复杂，涉及`Doris`多租户的资源限制问题，官方也有说明这个配置项，但是最初一直以为是`FE`的配置问题，所以一直没找到对应的用户配置项。  

> 不要轻易删元数据，要删也需要备份。。。  

## 问题现象  

其实问题现象很简单，在连接`Doris`时，报出了下面的错误。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/doris_conn_limit/1.png){:height="500" width="500"}

字面意思理解，**到达连接数限制**。  

那么问题点比较明显，接下来就是确定如何修改配置。  

## 问题分析

### FE连接限制排查

这个异常其实之前就已经遇到过，`Doris FE`会限制最大连接数，对应的配置项是`qe_max_connection`，该值默认大小为1024，在连接服务过多或者连接池配置不合理的时候，该连接数限制容易被耗尽。  

由于我们是在业务场景中使用，连接的业务服务较多，单个服务的POD也比较多，所以调大过到4096，最开始推测是不是这个最大连接数到达限制，查一下FE进程建立的连接数就可以知道。  

```
lsof -p {pid} | grep ESTABLISH | wc -l
```

结果和预期不符，数量只有100出头。  

那就不太对了，明显没到`qe_max_connection`的限制，但是同时也可以判断应该是某个配置限制了连接数为100，直接想到反查FE当前的配置项情况。  

### FE配置项反查  

既然是某个配置限制了连接数为100，所以直接去FE配置项里找。  

```
admin show frontend config
```

但是结果也并没有找到可疑的配置项。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/doris_conn_limit/2.png){:height="500" width="500"}

这就有点费解了，于是准备查源码看看。  

> 其实这里方向有些偏了，实际并不是FE的配置问题

### 源码分析

要分析源码，只要顺着抛出的异常信息去源码里追踪即可。  

```
Reach limit of connections
```

顺着这个异常，很快可以找到抛出异常的位置。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/doris_conn_limit/3.png){:height="500" width="500"}

从图中可以看出，在通过上下文注册连接的时候失败了，进入了`else`分支抛出异常，那么具体就看为什么`connectScheduler.registerConnection`会返回`false`。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/doris_conn_limit/4.png){:height="500" width="500"}

共有两处会抛出异常，由于连接数远没有到`qe_max_connection`配置的4096，所以基本可以断定是红框处的判断返回了`false`。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/doris_conn_limit/5.png){:height="500" width="500"}
![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/doris_conn_limit/6.png){:height="500" width="500"}

最终可以查到用户配置里有一个20年新增的配置项`maxConn`，并且默认值是100，同时是通过用户的`max_user_connections`属性来进行配置。  

到这里基本是真想大白了，剩下就差实际验证一把了。  

### 问题验证

可以通过`arthas`查找制定类型的对象，并查看对象属性值，这里我们就找到` org.apache.doris.mysql.privilege.UserProperty`的对象，并查看`maxConn`属性。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/doris_conn_limit/7.png){:height="500" width="500"}

从多个`UserProperty`对象中，找到我们使用的`root`用户，并且查看其`maxConn`值，确实是100。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/doris_conn_limit/8.png){:height="500" width="500"}

然后修改其值，与`qe_max_connection`保持一致，再查看`maxConn`的值。  

![](https://raw.githubusercontent.com/Taaang/blog/master/assets/images/post_imgs/doris_conn_limit/9.png){:height="500" width="500"}

此时值已变化，配置生效，问题也最最终解决。  

> 正常来说，qe_max_connection应该大于所有用户的`maxConn`之和，但是由于测试环境，且只有一个用户在使用，就省事儿配置成一样了。

### 问题总结  

总结来看，问题比较简单，就是对单个用户的最大连接数限制上，除了FE的配置项`qe_max_connection`限制FE的最大连接数外，还有单租户的最大连接限制`max_user_connections`，对应`UserProperty`中的`maxConn`。  

这里其实也有一些其他问题，就是为什么之前没出现这个情况？  

这里其实原因有两个：  
（1）首先FE启动失败，当时我上去排查的时候，发现其实是日志太多，测试主机磁盘满了，但是业务同学不知道，还误删了元数据目录；  
（2）租户配置应该之前运维同学有调整过，但是配置保存后有可能是保存在FE元数据里，但是业务同学误删了元数据，导致重建`Doris`配置回滚为默认的100。  
