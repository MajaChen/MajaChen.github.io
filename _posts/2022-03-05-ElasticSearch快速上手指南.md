---
layout: post
title: ElasticSearch快速上手指南
subtitle: es api search
cover-img: /assets/img/16.jpg
thumbnail-img: /assets/img/animals/TawnyFrogmouth.jpg
tags: [es]
---

# ElasticSearch 快速上手指南

&emsp;&emsp;本篇文章系统地介绍了es API的用法与功能特性，以助我在工作中解决es相关的问题，也希望对大家有所帮助。

## 添加、更新、删除文档

```json
PUT /index_name/employee/1
{
    "first_name" : "John",
    "last_name" :  "Smith",
    "age" :        25,
    "about" :      "I love to go rock climbing",
    "interests": [ "sports", "music" ]
}

```
&emsp;&emsp;基于同样的形式，可以使用 `DELETE` 命令来删除文档，以及使用 `HEAD` 指令来检查文档是否存在。如果想更新已存在的文档，只需再次 `PUT` 。

## 检索文档（重点）

### 基于URL

&emsp;&emsp;检索单个文档，指定id

```json
GET /index_name/employee/1?pretty
```

&emsp;&emsp;检索单个文档，指定筛选条件

```json
GET /index_name/employee/_search?q=last_name:Smith&pretty
```

&emsp;&emsp;检索全部文档，_search路径

```
GET /index_name/employee/_search?pretty
```



### 基于请求体（主流）

**检索单个文档，指定筛选条件**

```
GET /index_name/employee/_search?pretty
{
    "query" : {
        "match" : {
            "last_name" : "Smith"
        }
    }
}
```

**检索全部文档**

```json
GET /index_name/employee/_search?pretty
{
    "query" : {
        "match_all" : {}
    }
}

```

**检索特定范围内文档——基于from和size**

```
GET /index_name/employee/_search
{
  "query": { "match_all": {} },
  "sort": [
    { "account_number": "asc" }
  ],
  "from": 10,
  "size": 10
}
```

**排序检索结果——基于sort**

```
GET /index_name/employee/_search
{
  "query": { "match_all": {} },
  "sort": [
    { "account_number": "asc" }
  ]
}
```

**返回部分字段——基于source**

```json
GET /index_name/employee/_search
{
  "query": {
    "match_all": {}
  },
  "_source": ["balance","firstname"]  
}


```

### 三种匹配方式

#### 全文匹配

&emsp;&emsp;全文匹配是将文档字段（text类型）进行分词存入倒排索引，查找字符串同样进行分词，用字符串的分词匹配去匹配倒排索引中的分词。例如：

```json
GET /index_name/employee/_search
{
  "query": { "match": { "address": "mill lane" } }
}
```

&emsp;&emsp;会匹配address字段包含mill或者lane或者二者兼备（具体的分词情况可能并非如此）的文档，例如mill mama。

&emsp;&emsp;全文匹配会计算文档的相关性得分，默认按照得分进行排序，返回前十条匹配文档，它回答的是——文旦与查询的相关性如何（How Well）。

**全文检索加短语匹配（变体）**

&emsp;&emsp;将短词拆解为多个分词，所有分词必须匹配文档中对应的分词，**且分词顺序一致**。

```json
GET /bank/_search
{
  "query": { "match_phrase": { "address": "mill lane" } }
}
```

&emsp;&emsp;会匹配address字段完整包含mill lane且mill与lane（具体的分词情况可能并非如此）的顺序一致的的文档，比如mama mile papa lane。

**全文检索加关键字匹配（变体）**

&emsp;&emsp;相当于是不对查询字段本身进行分词拆解，去倒排索引中查看是否有“990 Mill”的分词存在，返回对应的文档。

```json
GET /index_name/employee/_search
{
  "query": {
    "match": {
      "address.keyword": "mill lane" 
    }
  }
}
```

&emsp;&emsp;文档的address字段必须完整包含keyword"990 Mill"才可以匹配到，例如mama 990 Mill。



#### 精准匹配

&emsp;&emsp;它回答的是——文档是否满足查询要求（YES/NO），使用term来对非text字段进行**过滤**。

```json
GET /index_name/_search
{
  "query": {
    "term": {
      "account_number": 970
    }
  }
}
```



#### 模糊匹配

&emsp;&emsp;在不分词的情况下两个字符串即使不完全相等也可认为是匹配的，包括部分相等、正则匹配等情况。模糊匹配同时适用于全文匹配和精准匹配。

**前缀匹配**

```json
GET /index_xxx/address/_search?pretty
{
    "query": {
        "prefix": {
            "postcode": "W1"
        }
    }
}
```

&emsp;&emsp;文档的postcode字段以W1开头则匹配。

**正则匹配**

 ```json
GET /index_xxx/address/_search?pretty
{
    "query": {
        "wildcard": {
            "postcode": "W?F*HW" 
        }
    }
}

 ```

&emsp;&emsp;?匹配任意字符，  *匹配 0 或多个字符，这个查询会匹配包含 `W1F 7HW` 和 `W2F 8HW` 的文档。

**编辑距离匹配**

编辑距离：字符串A变换到字符串B需要改变的字符数目；编辑距离越小则两个字符越匹配。

```json
GET /index_name/_search
{
  "query": {
    "fuzzy": {
      "text": "surprize"
    }
  }
}
```



### 组合多个查询条件

#### 基于bool

bool分词可以组合多个查询条件

```json
{
   "bool" : {
      "must" :     [],
      "should" :   [],
      "must_not" : [],
      "filter":    []
   }
}
```

- must：所有筛选条件都必须匹配
- should：至少有一个筛选条件匹配
- must_not：所有筛选条件都不匹配
- filter：用于过滤，只有满足filter条件的文档才会被返回

```json
GET /index_name/_search
{
  "query": {
    "term": {
      "account_number": 970
    }
  }
}
```

&emsp;&emsp;must和should会影响文档的相关性得分，相关性得分越高的文档越有可能被返回。must_not和filter都不会影响相关性得分，仅回答YES/NO的问题。在不考虑相关性得分、要求精准匹配的查询场景下，更推荐使用filter，因为它性能更高（原因从略）。


```json
GET index_name/_search?pretty
{
  "query": { 
    "bool": { 
      "must": [
        { "match": { "title":   "Search"        }},
        { "match": { "content": "Elasticsearch" }}
      ],
      "filter": [ 
        { "term":  { "status": "published" }},
        { "range": { "publish_date": { "gte": "2015-01-01" }}}
      ]
    }
  }
}

```

&emsp;&emsp;查询title包含Search、content包含Elasticsearch（全文匹配，考虑分词、考虑分值）、status等于published、publish_date大于2015-01-01（精准匹配，不考虑分词、不考虑分值）的文档。

&emsp;&emsp;所以推荐的动词组合是：

**全文匹配**

<img src="https://gitee.com/xinyuanchen/image_collection/raw/master/image-20220305112554594.png" alt="image-20220305112554594" style="zoom:50%;" />

**精准匹配**

<img src="C:\Users\chxy\AppData\Roaming\Typora\typora-user-images\image-20220305112723205.png" alt="image-20220305112723205" style="zoom:50%;" />



#### 基于query_string

&emsp;&emsp;在query_string中，可以同时指定多个字段和多个查询条件。

```
GET /index_name/_search?pretty
{
  "query": {
    "query_string": {
      "query": "(new york city) OR (big apple)",
      "default_field": "content"
    }
  }
}
'

```

&emsp;&emsp;多字段：支持以通配符方式指定多个查询字段。多个字段同时匹配查询，取得分最高的字段作为匹配字段（best_field策略），取最高得分作为最后相关性得分。

&emsp;&emsp;多查询条件：通过 AND、OR、NOT进行分隔。

&emsp;&emsp;此外，query_string也支持模糊查询。

&emsp;&emsp;个人理解：query_string相比于bool，查询能力明显增强，可以通配符方式支持多个查询字段、以AND、OR连接词指定多个匹配条件、以best_field等策略指定如何确定最后的相关性得分。但是，曲高和寡，目前不深钻。

## 其他

### 检索文档元数据信息

```json
GET /index_name/_mapping
```



