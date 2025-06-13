---
title: 从 RAG 角度看向量检索
date: 2025-06-13
categories:
- RAG
- Embedding
- 向量检索
- 学习笔记
tags:
- RAG
- Embedding
- 向量检索
- 学习笔记
---
## 前言  
### LLM应用中的痛点  
#### **幻觉问题/知识局限性**  
最简单的理解就是“一本正经的胡说八道”。  
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rag_embedding_search/1.png?raw=true){:height="500" width="500"}
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rag_embedding_search/2.png?raw=true){:height="500" width="500"}![](image%207.png)
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rag_embedding_search/3.png?raw=true){:height="500" width="500"}  
实际上只有 452 条。  

#### **知识时效性问题**<!-- {"fold":true} -->  
受限于模型的训练、微调数据的时效性。  
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rag_embedding_search/4.png?raw=true){:height="500" width="500"}
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rag_embedding_search/5.png?raw=true){:height="500" width="500"}
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rag_embedding_search/6.png?raw=true){:height="500" width="500"}
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rag_embedding_search/7.png?raw=true){:height="500" width="500"}

#### **敏感数据安全与隐私**  
企业内部敏感数据（会议记录、核心技术文档）  
个人敏感信息（手机号、住址等）  

直接问可能问不出来，但是可以“曲线救国”。  
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rag_embedding_search/8.png?raw=true){:height="500" width="500"}

## 什么是 RAG？  
### RAG的定义  
RAG（Retrieval-Augmented Generation，检索增强生成）是一种结合信息检索与文本生成的人工智能技术，旨在通过动态引用外部知识库提升生成内容的准确性、时效性和专业性。  

其实简单理解，就是提供知识检索能力，基于用户输入搜索相关联的知识，与提示词整合后提交给大模型，即辅助增强生成。  

### RAG的基本流程  
RAG基本流程包含三个阶段：检索阶段、增强阶段、生成阶段。  
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rag_embedding_search/9.png?raw=true){:height="500" width="500"}

### RAG的优势  

* 引入权威的依据（降低幻觉问题出现的可能性）  
* 具备专有领域知识（垂直领域知识定制）  
* 知识本地化（与私有数据结合，数据安全与隐私）  
* 大模型回答具备一定的可解释性（追溯参考的知识内容）  

LLM 的痛点，知识时效性问题能解决吗？  

并没有真正解决，我理解只是转移了问题，依旧要解决知识库的知识时效问题。  
真正解决知识时效性的是`web_search`，利用互联网本身的知识生产能力，获取最新的知识，但是缺点同样很明显，就是知识的可靠性及专业性。  

既然只是转移了问题，  
那么为什么还是普遍基于知识库来解决知识时效性问题？  
**成本**，知识库的更新成本＜模型训练。  

从知识时效性的角度看，什么场景最适合使用 RAG？  

1. 静态知识（低频更新，法律、医疗等）  
2. 天然具备更新机制的知识（同步更新，财务、XX 文档中心、CRM 等等已有业务流的领域）  

## RAG中涉及的核心技术  
### Embedding  
将高维离散数据（如文本、图像等）映射到低维连续向量空间。  
过程中将真实世界的实物以特征化、数据化的角度呈现。  

### 向量搜索  
在给定的向量集合中，找到与查询向量相近的 K 个向量。  

#### 如何定义“向量相近”？  
简单理解，就是两个向量的“距离”很近，即两个相似的向量，它们之间的距离应该很近。  

#### 如何理解“距离”？  
🙋‍♀️🌰  
（二维向量距离）二维坐标轴上的两个点，它们之间的距离。  
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rag_embedding_search/10.png?raw=true){:height="500" width="500"}
（[三维向量距离](https://www.geogebra.org/calculator/bentknwh)）三维坐标轴上的两个点，它们之间的距离。  
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rag_embedding_search/11.png?raw=true){:height="500" width="500"}

#### 如何计算距离（相似度）？  
常见的距离算法：  
1. 曼哈顿距离  
计算两点在直角坐标系中，沿轴移动的路径总和。  
其核心思想是仅允许沿水平或垂直方向移动，模拟城市街道的行走方式。  
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rag_embedding_search/12.png?raw=true){:height="500" width="500"}
[曼哈顿距离-示例](https://www.geogebra.org/calculator/hg95fayh)
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rag_embedding_search/13.png?raw=true){:height="500" width="500"}  
**应用场景**：城市街区导航、棋类游戏  

2. 欧几里得距离  
计算空间中两点之间的直线距离。  
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rag_embedding_search/14.png?raw=true){:height="500" width="500"}
[欧几里得距离](https://www.geogebra.org/calculator/bentknwh)
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rag_embedding_search/15.png?raw=true){:height="500" width="500"}
**应用场景**：物理空间测距、向量距离计算  

3. 余弦相似度  
衡量两个向量方向相似性的指标。  
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rag_embedding_search/16.png?raw=true){:height="500" width="500"}
**应用场景**：向量距离计算、推荐系统相似度计算。  
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rag_embedding_search/17.png?raw=true){:height="500" width="500"}

4. 其他距离  
[距离相似度计算总结](https://zhuanlan.zhihu.com/p/354289511)  

#### 为什么有那么多不同的距离？  
侧重点不同。  
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rag_embedding_search/18.png?raw=true){:height="500" width="500"}
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rag_embedding_search/19.png?raw=true){:height="500" width="500"}
一个关注大小，一个关注方向。  


### 向量搜索中的性能优化  
#### 大规模向量数据集的搜索优化  
在大规模向量数据集的搜索上，会有哪些问题？  
性能  

传统数据库的主要解决思路有哪些？  
索引、缓存  

在向量搜索的场景中，有哪些不同？  
索引  
B树构建的索引结构能解决向量检索问题吗？  
如果不行，需要什么样的索引结构？  
缓存  
向量数据 VS 业务行数据？  
以 Embedding 数据为例，  
OpenAI提供的 Embedding 模型，处理后是 1536 维，按 FP16 处理是 3KB。  
测试环境找了张`t_user`表，平均算下来单行数据是 0.2KB。  
单位数据的大小带来更大的内存需求。  
计算量  
涉及高维数据间的距离计算。  

##### 向量索引  
以我们在用的`Milvus`为例，看看不同的索引算法。  
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rag_embedding_search/20.png?raw=true){:height="500" width="500"}

* FLAT  
暴力检索。  
全量比对计算距离，取距离最近的 TopN。  

* IVF（Inversed File Index）  
倒排文件索引（个人理解，实际上也算不上倒排）。  
先 k-means 聚类，找到距离最近的 M 个聚类，然后再在这些聚类中找到距离最近的 TopN。  
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rag_embedding_search/99.png?raw=true){:height="500" width="500"}
[IVF](https://www.geogebra.org/calculator/vsbhuunp)
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rag_embedding_search/22.png?raw=true){:height="500" width="500"}

* HNSW（Hierarchical Navigable Small World）  
分层可导航小世界，ANN 算法领域中的其中一种。  
核心思想 ：跳表 + 可导航小世界  
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rag_embedding_search/22.png?raw=true){:height="500" width="500"}
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rag_embedding_search/23.png?raw=true){:height="500" width="500"}
🙋‍♀️🌰：  
有一批向量（二维），如何构建整个搜索世界？  
A（-2, 3）、B（-5, 3）、C（-2, 6）、D（-3, 5）、E（-5, 4）、F（-5, 5）  
目标：找到向量G 最相似的 1 个向量  
样例演示：  
[\[GeoGebra\] HNSW 样例演示](https://www.geogebra.org/calculator/jzvrhegk)  
[HNSW算法的基本原理及使用](https://zhuanlan.zhihu.com/p/673027535)  

* 索引算法简单对比  
ANN-Benchmarking  
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rag_embedding_search/24.png?raw=true){:height="500" width="500"}
[glove-100-angular \(k = 10\)](https://ann-benchmarks.com/glove-100-angular_10_angular.html)

* 向量索引本身的性能问题  
以 HNSW 为例，  
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rag_embedding_search/25.png?raw=true){:height="500" width="500"}  
要高性能实现整个索引过程，  
重点在哪？  
会有什么问题？  

做个对比，  
传统数据库，在基于索引进行查询加速的过程中，  
会做什么？  
遇到什么问题？  

索引缓存  
索引本身及部分数据应该在内存中，避免多次磁盘 IO 带来性能下降。  
缓存大小  
需要控制缓存占用的内存大小，本质上是空间换时间。  

有什么不通？  
向量数据的特征之一就是高维。  
高维 * 单维大小带来较大的空间占用，负载压力体现在内存、磁盘、IO。  
高维向量的距离计算带来计算资源占用，负载压力体现在CPU。  

##### 索引中的向量压缩  
索引缓存占用内存大，怎么优化？  
对索引中的向量进行压缩（非数据本身）  
类比理解，就是压缩 MySQL B+树索引中的节点数据。  

向量如何压缩？  
降维、量化  

* 向量降维  
字面理解，减少向量的维数。  

以 Embedding 数据为例，  
OpenAI提供的 Embedding 模型，处理后是 1536 维。  
如果降维到 384 维，那么存储和计算量就能减少 75%。  

那不是会丢失很多信息吗？  
为什么会考虑降维呢？  

🙋‍♀️🌰  
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rag_embedding_search/26.png?raw=true){:height="500" width="500"}  
假设，我们想要描述一个人的体貌特征，我们会采集很多数据，比如**身高，体重，臂展，衣服尺寸、鞋子尺码**等等。  

但对于一个人来说，一个人如果身高越高，往往其体重，臂展，腿长，衣服尺码，鞋子尺码，都会越大，这几个特征是高度相关的。  

换句话讲，一个人的体貌特征，不需要用这么多的特征描述才行，我们完全可以量化成一个综合指标，叫做一个“**体型大小**”。  
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rag_embedding_search/27.png?raw=true){:height="500" width="500"}

那么具体要怎么找到并计算“**体型大小**”的值呢？  
PCA 算法  
[主成分分析（PCA算法）](https://zhuanlan.zhihu.com/p/25116541521)  

* 向量量化  
量化  
量化是一种通用方法，指的是将数据压缩到较小的空间中。  

与降维的区别是什么？  
降维：降低特征维度，1536D -> 384D（减少维度）  
量化：降低数值精度，FP32  -> INT8（减少每个维度的取值范围）  

那向量量化要怎么做呢？  
🙋‍♀️🌰，以 PQ乘积量化（Product Quantization）为例。  
分为两个大阶段：预处理、检索。  

预处理  
将向量数据进行量化（降低数值精度，缩小数值范围）  
分为三步：  
1. **向量分段**。假设原始有 1000 个 128 维的向量，每个向量拆分成 M 个分段，每个分段包含 128/M 维。例如示例中，128 维向量拆分 64 个分段，单个分段 2 维（后续也可以通过 2 维坐标表示，方便直观理解）；  
2. **分段训练**。将所有向量的同一分段放到一个集合中，通过 k-means 进行聚类，得到 k 个质心。例如示例中，1000 个向量的第一段放到一个集合里，并聚类得到 k 个质心，最终所有分段聚类完，得到一个 k * M 的质心矩阵，并对质心进行编号（0～k），其中 k 为单个分段的质心个数、M 为分段总数，示例中单个质心是一个二维向量（坐标）；  
3. **向量编码**。拆分单个向量的不同分段，找到每个分段距离最近的质心，并以质心编号替代分段向量，编码成新的向量。示例中，假设向量的第一段距离 2 号质心最近，则以编号 2 代替第一位向量，原始向量经过整体转化，变为一个 64 维的新向量。  

![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/rag_embedding_search/28.png?raw=true){:height="500" width="500"}


检索  
这个过程相对简单，将查询向量同样做编码，然后与全量样本进行距离计算，找到距离最近的 topN 向量即可。  

整个量化过程中，有哪些优点？  
这个过程中，  
（1）向量维数下降，同时向量单维的数值范围从 32bit -> 8 bit。  
（2）需要的内存空间大小下降，整个**索引**的内存空间占用从 128\*32bit=512bytes，缩减到 64\*8bit=64bytes。  

##### 计算加速  
对比传统数据库查询数据时的字段匹配，向量数据库中向量匹配是一个更加复杂的过程。  
其中，核心步骤需要计算高维向量之间的距离，这个过程的计算量明显更大。  

那么如何优化呢？  
计算的主体是 CPU，  
换个计算方式，SIMD  
换更多的主体来做计算，并行计算，OpenMP && MPI  
换个更擅长并行计算的主体，GPU && NPU && TPU  
