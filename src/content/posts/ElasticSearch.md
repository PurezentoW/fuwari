---
title: ElasticSearch
published: 2024-04-28
description: ElasticSearch 分布式搜索引擎入门，涵盖倒排索引原理、安装配置与 SpringBoot 集成
image: ''
tags: [ElasticSearch, 搜索引擎, JAVA]
category: '技术分享'
draft: false
lang: zh-CN
---
# ElasticSearch

## ES分布式搜索引擎

- elasticsearch

  一个开源的分布式搜索引擎，可以用来实现搜索、日志统计、分析、系统监控等功能

- elastic stack（ELK）

  是以elasticsearch为核心的技术栈，包括beats、Logstash、kibana、elasticsearch

- Lucene

  是Apache的开源搜索引擎类库，提供了搜索引擎的核心API

![image](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202404282322381.png)

## 初识elasticsearch



### 1. 倒排索引

倒排索引的概念是基于MySQL这样的正向索引而言的。

#### 1.1 正向索引

> 设置了索引的话挺快的，但要是模糊查询则就很慢！

那么什么是正向索引呢？例如给下表（tb_goods）中的id创建索引：

![image](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202404282322588.png)

如果是根据id查询，那么直接走索引，查询速度非常快。

但如果是基于title做模糊查询，只能是逐行扫描数据，流程如下：

1）用户搜索数据，条件是title符合`"%手机%"`

2）逐行获取数据，比如id为1的数据

3）判断数据中的title是否符合用户搜索条件

4）如果符合则放入结果集，不符合则丢弃。回到步骤1

逐行扫描，也就是全表扫描，随着数据量增加，其查询效率也会越来越低。当数据量达到数百万时，就是一场灾难。

#### 1.2 倒排索引

倒排索引中有两个非常重要的概念：

- 文档（`Document`）：用来搜索的数据，其中的每一条数据就是一个文档。例如一个网页、一个商品信息
- 词条（`Term`）：对文档数据或用户搜索数据，利用某种算法分词，得到的具备含义的词语就是词条。例如：我是中国人，就可以分为：我、是、中国人、中国、国人这样的几个词条

**创建倒排索引**是对正向索引的一种特殊处理，流程如下：

- 将每一个文档的数据利用算法分词，得到一个个词条
- 创建表，每行数据包括词条、词条所在文档id、位置等信息
- 因为词条唯一性，可以给词条创建索引，例如hash表结构索引

如图：

![image](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202404282322957.png)

倒排索引的**搜索流程**如下（以搜索"华为手机"为例）：

1）用户输入条件`"华为手机"`进行搜索。

2）对用户输入内容**分词**，得到词条：`华为`、`手机`。

3）拿着词条在倒排索引中查找，可以得到包含词条的文档id：1、2、3。

4）拿着文档id到正向索引中查找具体文档。

如图：

![image](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202404282323206.png)

虽然要先查询倒排索引，再查询倒排索引，但是无论是词条、还是文档id都建立了索引，查询速度非常快！无需全表扫描。

#### 1.3 正向和倒排对比

概念区别：

- **正向索引**是最传统的，根据id索引的方式。但根据词条查询时，必须先逐条获取每个文档，然后判断文档中是否包含所需要的词条，是**根据文档找词条的过程**。
- 而**倒排索引**则相反，是先找到用户要搜索的词条，根据词条得到保护词条的文档的id，然后根据id获取文档。是**根据词条找文档的过程**。

优缺点：

**正向索引**：

- 优点：
  - 可以给多个字段创建索引
  - 根据索引字段搜索、排序速度非常快
- 缺点：
  - 根据非索引字段，或者索引字段中的部分词条查找时，只能全表扫描。

**倒排索引**：

- 优点：
  - 根据词条搜索、模糊搜索时，速度非常快
- 缺点：
  - 只能给词条创建索引，而不是字段
  - 无法根据字段做排序

### 2. ES数据库基本概念

elasticsearch中有很多独有的概念，与mysql中略有差别，但也有相似之处。

#### 2.1.文档和字段

> 一个文档就像数据库里的一条数据，字段就像数据库里的列

elasticsearch是面向**文档（Document）**存储的，可以是**数据库中的一条商品数据**，一个订单信息。文档数据会被序列化为json格式后存储在elasticsearch中：

![image](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202404282323258.png)

而Json文档中往往包含很多的**字段（Field）**，类似于**mysql数据库中的列**。

#### 2.2.索引和映射

> 索引就像数据库里的表，映射就像数据库中定义的表结构

**索引（Index）**，就是相同类型的文档的集合【**类似mysql中的表**】

例如：

- 所有用户文档，就可以组织在一起，称为用户的索引；
- 所有商品的文档，可以组织在一起，称为商品的索引；
- 所有订单的文档，可以组织在一起，称为订单的索引；

![image](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202404282323183.png)

因此，我们可以把索引当做是数据库中的表。

数据库的表会有约束信息，用来定义表的结构、字段的名称、类型等信息。因此，索引库中就有**映射（mapping）**，是索引中文档的字段约束信息，类似表的结构约束。

#### 2.3.mysql与elasticsearch

> 各自长处：
>
> - Mysql：擅长事务类型操作，可以确保数据的安全和一致性
> - Elasticsearch：擅长海量数据的搜索、分析、计算

我们统一的把**mysql与elasticsearch的概念做一下对比**：

| **MySQL** | **Elasticsearch** | **说明**                                                     |
| --------- | ----------------- | ------------------------------------------------------------ |
| Table     | Index             | 索引(index)，就是文档的集合，类似数据库的表(table)           |
| Row       | Document          | 文档（Document），就是一条条的数据，类似数据库中的行（Row），文档都是JSON格式 |
| Column    | Field             | 字段（Field），就是JSON文档中的字段，类似数据库中的列（Column） |
| Schema    | Mapping           | Mapping（映射）是索引中文档的约束，例如字段类型约束。类似数据库的表结构（Schema） |
| SQL       | DSL               | DSL是elasticsearch提供的JSON风格的请求语句，用来操作elasticsearch，实现CRUD |

在企业中，往往是两者结合使用：

- 对安全性要求较高的写操作，使用mysql实现
- 对查询性能要求较高的搜索需求，使用elasticsearch实现
- 两者再基于某种方式，实现数据的同步，保证一致性

![image](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202404282323149.png)

### 3. 安装es、kibana、分词器

> 分词器的作用是什么？
>
> - 创建倒排索引时对文档分词
> - 用户搜索时，对输入的内容分词
>
> IK分词器有几种模式？
>
> - ik_smart：智能切分，粗粒度
> - ik_max_word：最细切分，细粒度
>
> IK分词器如何拓展词条？如何停用词条？
>
> - 利用config目录的IkAnalyzer.cfg.xml文件添加拓展词典和停用词典
> - 在词典中添加拓展词条或者停用词条

#### 3.1.安装es

[Elasticsearch 7.17.1 | Elastic](https://www.elastic.co/cn/downloads/past-releases/elasticsearch-7-17-1)

直接启动E:\ElasticSearch\elasticsearch-7.17.1\bin\elasticsearch.bat

![image-20240421192414582](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202404282323302.png)

![image-20240421192003315](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202404282323226.png)

#### 3.2.安装kibana

[Kibana 7.17.0 | Elastic](https://www.elastic.co/cn/downloads/past-releases/kibana-7-17-0)

直接启动E:\ElasticSearch\kibana-7.17.1-windows-x86_64\bin\kibana.bat

![image-20240421192233220](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202404282324902.png)

![image-20240421192115416](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202404282323583.png)

#### 3.3.安装分词器

[Releases · infinilabs/analysis-ik](https://github.com/infinilabs/analysis-ik/releases/)

下载分词器：elasticsearch-analysis-ik-7.17.1.zip，解压到ES目录E:\ElasticSearch\elasticsearch-7.17.1\plugins\analysis-ik，重新启动

IK分词器包含两种模式：

- `ik_smart`：最少切分
- `ik_max_word`：最细切分

在kibana的Dev tools中输入以下代码：

> "analyzer" 就是选择分词器模式

![image-20240427151058997](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202404282325331.png)

扩展词词典

![image-20240427151613988](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202404282325682.png)

![image-20240427151618472](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202404282325060.png)

重启ES以后

![image-20240427151846796](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202404282325316.png)



## 索引库操作

索引库就类似数据库表，mapping映射就类似表的结构。

我们要向es中存储数据，必须先创建"库"和"表"。

### 1. Mapping映射属性

### 2. 索引库的CRUD



## 文档操作

### 1. 文档的CRUD



## RestAPI

### 1.API操作索引库

### 2.API操作文档



## ES搜索引擎

### 1. DSL设置查询条件

### 2. 设置搜索结果

### 3.RestClient查询文档



## 自动补全



[ElasticSearch入门篇（保姆级教程） - 知乎](https://zhuanlan.zhihu.com/p/451571598)

[ElasticSearch (ES从入门到精通一篇就够了) - 不吃紫菜 - 博客园](https://www.cnblogs.com/buchizicai/p/17093719.html)

[TheKingOfBigData/note/中间件/从 0 到 1 学习 elasticsearch ，这一篇就够了！(建议收藏).md at master · BigDataScholar/TheKingOfBigData](https://github.com/BigDataScholar/TheKingOfBigData/blob/master/note/%E4%B8%AD%E9%97%B4%E4%BB%B6/%E4%BB%8E%200%20%E5%88%B0%201%20%E5%AD%A6%E4%B9%A0%20elasticsearch%20%EF%BC%8C%E8%BF%99%E4%B8%80%E7%AF%87%E5%B0%B1%E5%A4%9F%E4%BA%86%EF%BC%81%28%E5%BB%BA%E8%AE%AE%E6%94%B6%E8%97%8F%29.md)

[ELK搭建（一）：手把手教你搭建分布式微服务日志监控 - 掘金](https://juejin.cn/post/7088314722432319524)
