---
title: Elastic Search查询和聚合
published: 2024-07-30
description: ElasticSearch 查询语法、聚合查询、索引管理及 SpringBoot 集成实战
image: ''
tags: [ElasticSearch, JAVA]
category: '技术分享'
draft: false
lang: zh-CN
---
# Elastic Search API

## Elastic Search查询和聚合

### 1.导入数据

#### 这是ES官方提供的测试数据

https://github.com/elastic/elasticsearch/blob/v6.8.18/docs/src/test/resources/accounts.json

```json
//格式是json
{
  "account_number": 0,
  "balance": 16623,
  "firstname": "Bradshaw",
  "lastname": "Mckenzie",
  "age": 29,
  "gender": "F",
  "address": "244 Columbus Place",
  "employer": "Euron",
  "email": "bradshawmckenzie@euron.com",
  "city": "Hobucken",
  "state": "CO"
}
```

#### 上传

```bash
//上传上去
http://localhost:9200/account/_bulk?pretty=&refresh=
```

![image-20240722222543387](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202407302142263.png)

#### 数据

![image-20240722222606405](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202407302142358.png)

### 2.查询

#### 查询所有

`match_all`表示查询所有的数据，`sort`即按照什么字段排序

```bash
GET /account/_search
{
  "query": { "match_all": {} },
  "sort": [
    { "account_number": "asc" }
  ]
}
```

#### 结果

![image-20240722223050906](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202407302142844.png)

相关字段解释

- `took` – ElasticSearch运行查询所花费的时间（以毫秒为单位）
- `timed_out` –搜索请求是否超时
- `_shards` - 搜索了多少个碎片，以及成功，失败或跳过了多少个碎片的细目分类。
- `max_score` – 找到的最相关文档的分数
- `hits.total.value` - 找到了多少个匹配的文档
- `hits.sort` - 文档的排序位置（不按相关性得分排序时）
- `hits._score` - 文档的相关性得分（使用match_all时不适用）

#### 分页查询(from+size)

**from** 相当于PageNum

**size** 相当于PageSize

```bash
GET /account/_search
{
  "query": { "match_all": {} },
  "sort": [
    { "account_number": "asc" }
  ],
  "from": 10,
  "size": 2
}
```

![image-20240722223817353](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202407302143563.png)

#### 指定字段查询：match

如果要在字段中搜索特定字词，可以使用`match`; 如下语句将查询address 字段中包含 mill 或者 lane的数据

```bash
GET /account/_search
{
  "query": { "match": { "address": "mill lane" } }
}
```

![image-20240722224108930](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202407302143731.png)

#### 查询段落匹配：match_phrase

如果我们希望查询的条件是 address字段中包含 "mill lane"，则可以使用`match_phrase`

```bash
GET /account/_search
{
  "query": { "match_phrase": { "address": "mill lane" } }
}
```

#### 多条件查询: bool

如果要构造更复杂的查询，可以使用`bool`查询来组合多个查询条件。

例如，以下请求在bank索引中搜索40岁客户的帐户，但不包括居住在爱达荷州（ID）的任何人

```bash
GET /account/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "age": "40" } }
      ],
      "must_not": [
        { "match": { "state": "ID" } }
      ]
    }
  }
}
```

#### 查询条件：query or filter

```bash
GET /account/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "state": "ND"
          }
        }
      ],
      "filter": [
        {
          "term": {
            "age": "40"
          }
        },
        {
          "range": {
            "balance": {
              "gte": 20000,
              "lte": 30000
            }
          }
        }
      ]
    }
  }
}
```

**query 上下文的条件是用来给文档打分的，匹配越好 _score 越高**

**filter 的条件只产生两种结果：符合与不符合，后者被过滤掉，没有_score**

### 3.聚合查询：Aggregation

#### 简单聚合

比如我们希望计算出account每个州的统计数量， 使用`aggs`关键字对`state`字段聚合，被聚合的字段无需对分词统计，所以使用`state.keyword`对整个字段统计

```bash
GET /account/_search
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state.keyword"
      }
    }
  }
}
```

![image-20240722225619153](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202407302143746.png)

#### 嵌套聚合

ES还可以处理个聚合条件的嵌套。

在对state分组的基础上，嵌套计算avg(balance):

```bash
GET /account/_search
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state.keyword"
      },
      "aggs": {
        "average_balance": {
          "avg": {
            "field": "balance"
          }
        }
      }
    }
  }
}
```

#### 对聚合结果排序

可以通过在aggs中对嵌套聚合的结果进行排序

对嵌套计算出的avg(balance)，这里是average_balance，进行排序

```bash
GET /account/_search
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state.keyword",
        "order": {
          "average_balance": "desc"
        }
      },
      "aggs": {
        "average_balance": {
          "avg": {
            "field": "balance"
          }
        }
      }
    }
  }
}
```

## 索引管理

### 1.索引管理的引入

```bash
PUT /customer/_doc/1
{
  "name": "John Doe"
}
```

- 默认是自动创建索引

```json
{
  "mappings": {
    "_doc": {
      "properties": {
        "name": {
          "type": "text",
          "fields": {
            "keyword": {
              "type": "keyword",
              "ignore_above": 256
            }
          }
        }
      }
    }
  }
}
```

- 禁止自动创建索引

```bash
action.auto_create_index: false
```

### 2.索引管理

#### 创建索引

```bash
PUT /test-index-users//创建索引
{
  "settings": {
		"number_of_shards": 1,//分片
		"number_of_replicas": 0//副本
	},
  "mappings": {//这部分定义了索引的映射，即文档的字段和字段类型。
    "properties": {//定义了文档的字段
      "name": {//定义了一个名为name的字段
        "type": "text",//指定字段类型为text，表示这个字段可以存储文本数据
        "fields": {//为name字段定义了多字段（multi-fields）
          "keyword": {//定义了一个名为keyword的子字段
            "type": "keyword",//指定子字段类型为keyword，表示这个字段适合用于精确匹配和聚合
            "ignore_above": 256//指定如果字符串长度超过256个字符，则不索引该字段
          }
        }
      },
      "age": {//字段
        "type": "long"
      },
      "remarks": {//字段
        "type": "text"
      }
    }
  }
}
```

![image-20240728154007893](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202407302144884.png)

##### **插入测试数据**

```bash
POST /test-index-users/_doc
{
  "name": "王呈现",
  "age": "1",
  "remarks": "冲冲冲"
}
```

![image-20240728154430382](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202407302144799.png)

#### 打开/关闭索引

```bash
POST /test-index-users/_close
POST /test-index-users/_open
```

#### 删除索引

```bash
DELETE /test-index-users
```

#### 查看索引

```bash
GET /test-index-users/_mapping
```



## 索引模板

在创建索引之前可以先配置模板，这样在创建索引（手动创建索引或通过对文档建立索引）时，模板设置将用作创建索引的基础

模板有两种类型：**索引模板**和**组件模板**。

1. **组件模板**是可重用的构建块，用于配置映射，设置和别名；它们不会直接应用于一组索引。
2. **索引模板**可以包含组件模板的集合，也可以直接指定设置，映射和别名。

### 1.索引模板中的优先级 索引模板中的优先级

1. 可组合模板优先于旧模板。如果没有可组合模板匹配给定索引，则旧版模板可能仍匹配并被应用。
2. 如果使用显式设置创建索引并且该索引也与索引模板匹配，则创建索引请求中的设置将优先于索引模板及其组件模板中指定的设置。
3. 如果新数据流或索引与多个索引模板匹配，则使用优先级最高的索引模板。

### 2.内置索引模板

Elasticsearch具有内置索引模板，每个索引模板的优先级为100，适用于以下索引模式：

1. `logs-*-*`
2. `metrics-*-*`
3. `synthetics-*-*`



## 集成springboot

### 导入es依赖

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
```

### 添加配置类

```yaml
spring:
  elasticsearch:
    uris: localhost:9200
    read-timeout: 30s
    connection-timeout: 5s
```

### 创建实体类

```java
//indexName 指定索引名称
@Document(indexName = "lx-sd")
@Data
public class Article {
    @Id
    @Field(index = false,type = FieldType.Integer)
    private Integer id;
    /**
     * index:是否设置分词  默认为true
     * analyzer：储存时使用的分词器
     * searchAnalyze:搜索时使用的分词器
     * store：是否存储  默认为false
     * type：数据类型  默认值是FieldType.Auto
     *
     */
    @Field(analyzer = "ik_smart",searchAnalyzer = "ik_smart",store = true,type = FieldType.Text)
    private String title;
    @Field(analyzer = "ik_smart",searchAnalyzer = "ik_smart",store = true,type = FieldType.Text)
    private String context;
    @Field(store = true,type = FieldType.Integer)
    private Integer hits;
}
```

### 创建dao层

```java
@Component
public interface ArticleDao extends ElasticsearchRepository<Article,Integer> {

    /**
     * 根据标题查询
     * @param title
     * @return
     */
    List<Article> findByTitle(String title);

    /**
     * 根据标题或内容查询
     * @param title
     * @param context
     * @return
     */
    List<Article> findByTitleOrContext(String title,String context);

    /**
     * 根据标题或内容查询（含分页）
     * @param title
     * @param context
     * @param pageable
     * @return
     */
    List<Article> findByTitleOrContext(String title, String context, Pageable pageable);
}
```

### 测试类



#### 1.储存一条数据

```java

//通过springboot es向elasticsearch数据库储存一条数据
@Test
public void testSave() {
    //创建文档
    Article article = new Article();
    article.setId(1);
    article.setTitle("es搜索");
    article.setContext("成功了吗");
    //保存文档
    articleDao.save(article);
}

```



![image-20240730232827942](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202407302337106.png)

#### 2.查询

```java
//根据标题查询
@Test
public void testFindByTitle(){
    List<Article> es = articleDao.findByTitle("es");
    for (Article e : es) {
        System.out.println(e);
    }
}
```



![image-20240730232727281](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202407302338751.png)

#### 3.修改

```java
//修改
@Test
public void testUpdate() {
    //判断数据库中是否有你指定的id的文档，如果没有。就进行保存，如果有，就进行更新
    //创建文档
    Article article = new Article();
    article.setId(1);
    article.setTitle("es搜索1");
    article.setContext("成功了吗1");
    //保存文档
    articleDao.save(article);
}
```

![image-20240730233008373](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202407302338790.png)

#### 4.删除

```java
//删除
@Test
public void testDelete() {
//根据主键删除
    articleDao.deleteById(1);
}
```

![image-20240730233335605](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202407302338109.png)

#### 5.分页查询

```java
//重新构建数据
@Test
public void makeData(){
    for (int i = 1; i <= 10; i++) {
        //创建文档
        Article article = new Article();
        article.setId(i);
        article.setTitle("es搜索"+i);
        article.setContext("成功了吗"+i);
        article.setHits(100+i);
        //保存数据
        articleDao.save(article);
    }
}

//分页查询
@Test
public void testFindAllWithPage(){
    //设置分页条件
    //page代表页码，从0开始
    PageRequest pageRequest = PageRequest.of(1, 3);

    Page<Article> all = articleDao.findAll(pageRequest);
    for (Article article : all) {
        System.out.println(article);
    }
}
```

![image-20240730233607441](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202407302338132.png)

6.排序查询

```java
//排序查询
@Test
public void testFindWithSort(){
    //设置排序条件
    Sort sort = Sort.by(Sort.Order.desc("hits"));
    Iterable<Article> all = articleDao.findAll(sort);
    for (Article article : all) {
        System.out.println(article);
    }
}
```

![image-20240730233719052](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202407302338449.png)







## 复合查询

在查询中会有多种条件组合的查询，在ElasticSearch中叫复合查询。它提供了5种复合查询方式：

- **bool query(布尔查询)**
- **boosting query(提高查询)**
- **constant_score（固定分数查询）**
- **dis_max(最佳匹配查询）**
- **function_score(函数查询）**



### 1.bool query(布尔查询)

> 通过布尔逻辑将较小的查询组合成较大的查询。

Bool查询语法有以下特点

- 子查询可以任意顺序出现
- 可以嵌套多个查询，包括bool查询
- 如果bool查询中没有must条件，should中必须至少满足一条才会返回结果。

bool查询包含四种操作符，分别是must,should,must_not,filter。他们均是一种数组，数组里面是对应的判断条件。

- `must`： 必须匹配。贡献算分
- `must_not`：过滤子句，必须不能匹配，但不贡献算分
- `should`： 选择性匹配，至少满足一条。贡献算分
- `filter`： 过滤子句，必须匹配，但不贡献算分

### 2.boosting query(提高查询)

> 不同于bool查询，bool查询中只要一个子查询条件不匹配那么搜索的数据就不会出现。而boosting query则是降低显示的权重/优先级（即score)。

比如搜索逻辑是 name = 'apple' and type ='fruit'，对于只满足部分条件的数据，不是不显示，而是降低显示的优先级（即score)

### 3.constant_score（固定分数查询）

查询某个条件时，固定的返回指定的score；显然当不需要计算score时，只需要filter条件即可，因为filter context忽略score。

### 4.dis_max(最佳匹配查询）

分离最大化查询（Disjunction Max Query）指的是： 将任何与任一查询匹配的文档作为结果返回，但只将最佳匹配的评分作为查询的评分结果返回 。

### 5.function_score(函数查询）

简而言之就是用自定义function的方式来计算_score。
