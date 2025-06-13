---
title: 从NLP的角度看Embedding
date: 2025-04-25
categories:
- NLP
- Embedding
- 学习笔记
tags:
- NLP
- Embedding
- 学习笔记
---

## 前言

大模型已经逐渐开始渗透到生活中的不同领域，辅助人类完成现实世界中的任务。  

![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/nlp_embedding/1.png?raw=true){:height="500" width="500"}

要能辅助人完成任务，就需要将现实世界的信息输入给模型。  
**模型是如何感知现实世界的？**  

### 模型是如何感知世界？  
人通过五官来感知世界。  
眼睛-视觉、鼻子-嗅觉、耳朵-听觉、嘴-味觉、触觉  
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/nlp_embedding/2.png?raw=true){:height="500" width="500"}

那么模型呢？  


模型通过数据归纳和演绎世界。  
数据：  
01010101011101  
白色 - \#FFFFFF   
黑色 - \#000000  

![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/nlp_embedding/3.png?raw=true){:height="500" width="500"}

在深度学习模型中，输入给模型的数据是一连串的数字。  
[0.259, 1.391, -0.448, … , 0.681]  

### 从现实世界到数据的映射  
模型的输入是一连串的数字，即向量。  
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/nlp_embedding/4.png?raw=true){:height="500" width="500"}  
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/nlp_embedding/5.png?raw=true){:height="500" width="500"}
文字“我爱上班” -> ？-> 向量  
花  ->  图片 -> ？-> 向量  

现实世界转化到数据的过程：  
特征化、向量化、Embedding、特征提取、…  

简单理解这个过程，就是编码。  
对现实世界的信息做编码，然后转换成计算机可以理解的数据。  

假设对语言词汇进行编码，  
我 -> 0001  
爱 -> 0010  
学习 -> 0100  
（很类似 RGB 不同颜色的定义）  
再将内容输入给模型，模型是不是也能实现对应的任务呢？  

答案当然是能。  
会有什么问题？  

* 离散高维数据，计算量大。  
汉字有8.5W个，一个汉字需要85000维，  
一次点积运算需要85000次乘法。  
如果含词汇，将会更大。  

* 语义丢失  
两个有关系的词，无法描述其相似性或关联性。  
苹果 -> 0001  
梨子 -> 1000  
但两者同属于水果，且大小相近。  

* 上下文中的多义字/词  
上下文不同，词含义不同。  
苹果 -> 0001  
水果中的苹果  
手机中的苹果  

如何将现实世界的事物，更真实（关系、特征…）的转化成数据？  


## Embedding  
### 定义  
将高维离散数据（如文本、图像等）映射到低维连续向量空间  

高维数据 -> 低维数据  
bge-m3，568M  
100+语言 -> 1024维向量  

离散数据 -> 连续向量  
商品，  
离散数据型商品类目，手表、watch（传统自然语言处理中）  
连续数据型商品类目，[0.5, -1.3, 2.2, 1,7, -0.2]  

### 为什么要做Embedding  
#### 从特征化的角度  
不仅是对物体进行编号或者标识，更是对其特征的抽象和编码。  
举个例子，相亲  
1 号男嘉宾，[1, 0, 0, 0]  
VS  
男嘉宾，[1.78, 33, xx, xxx]  

卷积神经网络，上卷提取图片局部特征  

![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/nlp_embedding/6.png?raw=true){:height="500" width="500"}

#### 从量化的角度  
具备了数据可操作性，能进行向量加减、点积等运算，线性计算描述关系。  
举个经典案例，  
国王（King）: [0.7,0.9,0.3]  
男性（Man）: [0.8,0.1,0.3]  
女性（Woman）: [−0.8,0.1,0.3]  
王后（Queen）: [−0.7,0.9,0.3]  
-> 国王 - 男性 + 女性 ≈ 王后  
-> 国王与王后、男性与女性在部分维度上相似，但又有不同  

#### 其他角度  
隐含更多信息，关系、属性、环境…  

## NLP Embedding 模型  
MTEB（Massive Text Embedding Benchmark），评估文本嵌入模型性能的权威基准  
涵盖8类任务（如检索、分类、聚类等）的56个数据集，支持112种语言  

![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/nlp_embedding/7.png?raw=true){:height="500" width="500"}

以上狂拽炫酷的模型，  

我们一一都不介绍。  
来看一些更简单的 NLP Embedding 算法/模型，简单入门。  

### One-Hot  
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/nlp_embedding/8.png?raw=true){:height="500" width="500"}

### Word2Vec  

#### Skip-gram 单个词预测上下文  

![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/nlp_embedding/9.png?raw=true){:height="500" width="500"}  

#### CBOW 上下文预测中心词  

![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/nlp_embedding/10.png?raw=true){:height="500" width="500"}
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/nlp_embedding/11.png?raw=true){:height="500" width="500"}

增加了局部上下文  

### BERT Embedding  
Bidirectional Encoder Representations from Transformers  
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/nlp_embedding/12.png?raw=true){:height="500" width="500"}
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/nlp_embedding/13.png?raw=true){:height="500" width="500"}
词向量+位置向量+段落向量  

#### BERT 模型详解  

[小白也能听懂的 bert模型原理解读 预训练语言模型_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1Km41117VG/?spm_id_from=333.337.search-card.all.click&vd_source=0293e7888278ef6fd7189c922c8a7229)

### Transformers Input Embedding Layer  

![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/nlp_embedding/14.png?raw=true){:height="500" width="500"}
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/nlp_embedding/15.png?raw=true){:height="500" width="500"}
词向量矩阵 + 位置信息  

## NLP Embedding 中要解决的问题  
### 文本寓意偏见 Bias  
医生、保姆、男生、女生、他、她    

![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/nlp_embedding/16.png?raw=true){:height="500" width="500"}
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/nlp_embedding/17.png?raw=true){:height="500" width="500"}


### 如何理解训练集中没有的词  
王刚是一位菜农。  
{张三}是一位果农。  

![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/nlp_embedding/18.png?raw=true){:height="500" width="500"}

## NLP Embedding 模型实战  
### 以 bge-m3 为例  
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/nlp_embedding/19.png?raw=true){:height="500" width="500"}
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/nlp_embedding/20.png?raw=true){:height="500" width="500"}
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/nlp_embedding/21.png?raw=true){:height="500" width="500"}

### 土豆、马铃薯、洋芋的embedding 相似度如何？  
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/nlp_embedding/22.png?raw=true){:height="500" width="500"}
### 相关性高的词，Embedding 相似度如何？  
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/nlp_embedding/23.png?raw=true){:height="500" width="500"}
### 如何对 Embedding 模型进行评测？  
#### MTEB  
MTEB - Massive Text Embedding Benchmark  
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/nlp_embedding/24.png?raw=true){:height="500" width="500"}
衡量文本嵌入模型的评估指标合集，它涵盖了112种语言的58个数据集，并针对8种任务进行了综合评测。  

不同的任务类型，  
* 文本语义相似度（STS）  
* 检索（Retrieval，Reranking）  
* 聚类（Clustering）  
* 分类（Classification，语言检测、句子对分类）  
* 双语文本挖掘（翻译句子对）  
* 摘要（Summarization）  

中文 Embedding 评测基准，C-MTEB。  
针对中文文本向量的基准测试，包括35个数据集和6种任务的评测。  

基准测试，初步判断模型性能。  

#### 不同类型的任务，如何进行评测？  
##### 文本语义相似度（STS）  
如何判断 STS 是否准确？  

使用 Spearman 系数及 Pearson 系数。  
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/nlp_embedding/25.png?raw=true){:height="500" width="500"}
r = 协方差/标准差  
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/nlp_embedding/26.png?raw=true){:height="500" width="500"}
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/nlp_embedding/27.png?raw=true){:height="500" width="500"}

##### 检索（Retrieval）  
如何判断检索结果是否准确？  
内容对不对？排序对不对？  

**nDCG@10** （Normalized Discounted Cumulative Gain at 10）  
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/nlp_embedding/28.png?raw=true){:height="500" width="500"}

![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/nlp_embedding/29.png?raw=true){:height="500" width="500"}
![](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/nlp_embedding/30.png?raw=true){:height="500" width="500"}
