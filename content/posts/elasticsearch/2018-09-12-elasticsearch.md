---
title:      "Elasticsearch学习"
date:       2018-09-12 18:52:59
author:     "bruce"
toc: true
tags:
    - elasticsearch
    - 搜索
---

![](https://raw.githubusercontent.com/heyuan110/static-source/master/cover/es.jpg)


Elasticsearch是一个基于Apache Lucene(TM)的开源搜索引擎(以下简称ES),是目前全文搜索引擎的首选。它可以快速存储、搜索和分析海量数据，Github，StackOverflow都在采用它。


## 一、ES组成

ES对照RMDB快速了解ES基本组成，它可以包含多个索引(indices)（数据库），每一个索引可以包含多个类型(types)（表），每一个类型包含多个文档(documents)（行），然后每个文档包含多个字段(Fields)（列）,简化如下:

索引  **->**  数据库
类型  **->**  表
文档  **->**  行
字段  **->**  列

## 二、常用查询命令

### 1. 查看_cat相关命令

> `GET /_cat/` 

结果:

```
>
➜  ~ curl -i -XGET http://192.168.11.119:9200/_cat/
HTTP/1.1 200 OK
content-type: text/plain; charset=UTF-8
content-length: 493

>=^.^=
/_cat/allocation
/_cat/shards
/_cat/shards/{index}
/_cat/master
/_cat/nodes
/_cat/tasks
/_cat/indices
/_cat/indices/{index}
/_cat/segments
/_cat/segments/{index}
/_cat/count
/_cat/count/{index}
/_cat/recovery
/_cat/recovery/{index}
/_cat/health
/_cat/pending_tasks
/_cat/aliases
/_cat/aliases/{alias}
/_cat/thread_pool
/_cat/thread_pool/{thread_pools}
/_cat/plugins
/_cat/fielddata
/_cat/fielddata/{fields}
/_cat/nodeattrs
/_cat/repositories
/_cat/snapshots/{repository}
/_cat/templates
```

### 2.查看集群健康

> `GET /_cat/health?v`

结果：

```
➜  ~ curl -XGET http://192.168.11.119:9200/_cat/health\?v
epoch      timestamp cluster       status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1533717572 08:39:32  elasticsearch yellow          1         1    315 315    0    0      315             0                  -                 50.0%
```

green：每个索引的primary shard和replica shard都是active状态的 
yellow：每个索引的primary shard都是active状态的，但是部分replica shard不是active状态，处于不可用的状态 
red：不是所有索引的primary shard都是active状态的，部分索引有数据丢失了

为什么现在会处于一个yellow状态？

我们现在就一台服务器，就启动了一个es进程，相当于就只有一个node。现在es中有一个index，就是kibana自己内置建立的index。由于默认的配置是给每个index分配5个primary shard和5个replica shard，而且primary shard和replica shard不能在同一台机器上（为了容错）。现在kibana自己建立的index是1个primary shard和1个replica shard。当前就一个node，所以只有1个primary shard被分配了和启动了，但是一个replica shard没有第二台机器去启动。

### 3. 查看集群有哪些索引

>`GET /_cat/indices\?v`

结果:

```
➜  ~ curl -i -XGET http://192.168.11.119:9200/_cat/indices\?v
HTTP/1.1 200 OK
content-type: text/plain; charset=UTF-8
content-length: 8840

>health status index                                    uuid                   pri rep docs.count docs.deleted store.size pri.store.size
yellow open   es_es_category_products                  u2TdPYcXS5yyFF8P3a3jYQ   5   1      95311         7103      156mb          156mb
yellow open   web_product_ar_new                       8qhhh9C7QvuwEEu-YYrIgA   5   1      37610           77     55.6mb         55.6mb
yellow open   en_27_category_product                   VtVXVTuHQ3-xyNw4txpEXg   5   1      41206           20       68mb           68mb
yellow open   ar_27_category_product                   Id43cmuDQnKYkhaCepxrIg   5   1      41206           17     67.9mb         67.9mb
yellow open   it_28_category_product                   Gltx9R80Qn6PI22i6-Mflg   5   1      12659           25     22.5mb         22.5mb
yellow open   db_search                                WKYGbjjLSZmh0s_LyuT2tQ   5   1     230133            0     28.7mb         28.7mb
yellow open   de_28_category_product                   IUCYcmTIR6K4AzUpAWJmHg   5   1      12659           27     22.5mb         22.5mb
```

### 4. 创建索引

`PUT /test_index?pretty`

结果：

```
➜  ~ curl -i -XPUT http://192.168.11.119:9200/test_index\?pretty
HTTP/1.1 200 OK
content-type: application/json; charset=UTF-8
content-length: 60
{
  "acknowledged" : true,
  "shards_acknowledged" : true
```


### 5.删除索引

`DELETE /test_index?pretty`


### 6. 新增文档并建立索引

语法格式：

```
PUT /index/type/id
{
    "json数据"
}
```

index索引名、type类型名、id数据的id

```
PUT /test_index/user/1
{
    "name": "小明",
    "email": "xiaoming@test.com",
    "tags": ["篮球","游泳"]
}
```

结果如下：

```
➜  ~ curl -i -XPUT http://192.168.11.119:9200/test_index/user/1 -d '{
    "name": "小明",
    "email": "xiaoming@test.com",
    "tags": ["篮球","游泳"]
}'

>HTTP/1.1 201 Created
Location: /test_index/user/1
Warning: 299 Elasticsearch-5.5.2-b2f0c09 "Content type detection for rest requests is deprecated. Specify the content type using the [Content-Type] header." "Wed, 08 Aug 2018 08:58:29 GMT"
content-type: application/json; charset=UTF-8
content-length: 143

>{"_index":"test_index","_type":"user","_id":"1","_version":1,"result":"created","_shards":{"total":2,"successful":1,"failed":0},"created":true}%
```

### 6.1 查询新增的文档

`GET /索引/类型/字段值`

例如：

```
➜  ~ curl -i -XGET http://192.168.11.119:9200/test_index/user/1\?pretty
HTTP/1.1 200 OK
content-type: application/json; charset=UTF-8
content-length: 232
{
  "_index" : "test_index",
  "_type" : "user",
  "_id" : "1",
  "_version" : 1,
  "found" : true,
  "_source" : {
    "name" : "小明",
    "email" : "xiaoming@test.com",
    "tags" : [
      "篮球",
      "游泳"
    ]
  }
}
```

### 7.修改文档

修改分为全部修改或部分修改，全部修改就是直接替换，需要带上全部字段才能修改，例如：

```
➜  ~  curl -i -XPUT http://192.168.11.119:9200/test_index/user/1 -d '{
    "name": "小明",
    "email": "xiaoming@test.com",
    "tags": ["篮球","游泳","足球"]
}'
HTTP/1.1 200 OK
Warning: 299 Elasticsearch-5.5.2-b2f0c09 "Content type detection for rest requests is deprecated. Specify the content type using the [Content-Type] header." "Wed, 08 Aug 2018 09:15:45 GMT"
content-type: application/json; charset=UTF-8
content-length: 144
{"_index":"test_index","_type":"user","_id":"1","_version":2,"result":"updated","_shards":{"total":2,"successful":1,"failed":0},"created":false}
```

注意全部修改用的是PUT方法.
部分修改就是只更新部分,用的POST方法，参数部分增加了一个doc的key，例如:

```
➜  ~ curl -i -XPOST http://192.168.11.119:9200/test_index/user/1/_update -d '{
        "doc":{
            "email": "xiaoming@demo.com"
        }
}'
HTTP/1.1 200 OK
Warning: 299 Elasticsearch-5.5.2-b2f0c09 "Content type detection for rest requests is deprecated. Specify the content type using the [Content-Type] header." "Wed, 08 Aug 2018 09:18:26 GMT"
content-type: application/json; charset=UTF-8
content-length: 128
{"_index":"test_index","_type":"user","_id":"1","_version":3,"result":"updated","_shards":{"total":2,"successful":1,"failed":0}}
```

### 8.删除文档

`DELETE /test_index/user/1`

例如：

```
➜  ~ curl -i -XDELETE http://192.168.11.119:9200/test_index/user/2
HTTP/1.1 200 OK
content-type: application/json; charset=UTF-8
content-length: 141
{"found":true,"_index":"test_index","_type":"user","_id":"2","_version":2,"result":"deleted","_shards":{"total":2,"successful":1,"failed":0}}
```

### 9.查询字符串

`GET /test_index/user`

例如：

```
➜  ~ curl -i -XGET http://192.168.11.119:9200/test_index/user/_search\?pretty
HTTP/1.1 200 OK
content-type: application/json; charset=UTF-8
content-length: 793
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "hits" : {
    "total" : 2,
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "test_index",
        "_type" : "user",
        "_id" : "2",
        "_score" : 1.0,
        "_source" : {
          "name" : "小王",
          "email" : "xiaowang@test.com",
          "tags" : [
            "游泳"
          ]
        }
      },
      {
        "_index" : "test_index",
        "_type" : "user",
        "_id" : "1",
        "_score" : 1.0,
        "_source" : {
          "name" : "小明",
          "email" : "xiaoming@demo.com",
          "tags" : [
            "篮球",
            "游泳",
            "足球"
          ]
        }
      }
    ]
  }
}
```

查询返回值参数说明

```
took：耗费了几毫秒
timed_out：是否超时，这里是没有
_shards：数据拆成了5个分片，所以对于搜索请求，会打到所有的primary shard（或者是它的某个replica shard也可以）
hits.total：查询结果的数量，3个document
hits.max_score：score的含义，就是document对于一个search的相关度的匹配分数，越相关，就越匹配，分数也高
hits.hits：包含了匹配搜索的document的详细数据
```

搜索名字为bruce的用户，而且按照email倒序

```
➜  ~ curl -i -XGET http://192.168.11.119:9200/test_index/user/_search\?pretty\&q=name:'bruce'&sort=email:desc
[1] 26574
HTTP/1.1 200 OK
content-type: application/json; charset=UTF-8
content-length: 479
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "hits" : {
    "total" : 1,
    "max_score" : 1.1727304,
    "hits" : [
      {
        "_index" : "test_index",
        "_type" : "user",
        "_id" : "4",
        "_score" : 1.1727304,
        "_source" : {
          "name" : "Bruce",
          "email" : "bruce@test.com",
          "tags" : [
            "Hello"
          ]
        }
      }
    ]
  }
}
[1]  + 26574 done       curl -i -XGET
```

通过这个例子发现这样搜索是不区分大小写的.适用于临时的在命令行使用一些工具，比如curl，快速的发出请求，来检索想要的信息；但是如果查询请求很复杂，是很难去构建,在实际的生产环境中，几乎很少使用查询字符串.

### 10. 查询索引的表和字段定义

 查询es所有的表和字段定义
`GET /_mapping`

查询某个索引的表定义
`GET /test_index/_mapping`

查询某个索引的表的字段定义
`GET /test_index/user/_mapping`

例如：
```
➜  ~ curl -i -XGET http://192.168.11.119:9200/test_index/_mapping\?pretty
HTTP/1.1 200 OK
content-type: application/json; charset=UTF-8
content-length: 1267
{
  "test_index" : {
    "mappings" : {
      "user" : {
        "properties" : {
          "email" : {
            "type" : "text",
            "fields" : {
              "keyword" : {
                "type" : "keyword",
                "ignore_above" : 256
              }
            }
          },
          "name" : {
            "type" : "text",
            "fields" : {
              "keyword" : {
                "type" : "keyword",
                "ignore_above" : 256
              }
            }
          },
          "tags" : {
            "type" : "text",
            "fields" : {
              "keyword" : {
                "type" : "keyword",
                "ignore_above" : 256
              }
            }
          }
        }
      },
      "role" : {
        "properties" : {
          "flag" : {
            "type" : "text",
            "fields" : {
              "keyword" : {
                "type" : "keyword",
                "ignore_above" : 256
              }
            }
          },
          "name" : {
            "type" : "text",
            "fields" : {
              "keyword" : {
                "type" : "keyword",
                "ignore_above" : 256
              }
            }
          }
        }
      }
    }
  }
}
```


### 11.查询DSL(Domain Specified Language，特定领域的语言 )

http request body：请求体，可以用json的格式来构建查询语法，比较方便，可以构建各种复杂的语法，比查询字符串肯定强大多了

- **11.1查询所有文档**

```
➜  ~ curl -i -XGET http://192.168.11.119:9200/test_index/user/_search\?pretty -d '
{
  "query": {
                "match_all": {
                }
  }
}
'
HTTP/1.1 200 OK
Warning: 299 Elasticsearch-5.5.2-b2f0c09 "Content type detection for rest requests is deprecated. Specify the content type using the [Content-Type] header." "Wed, 08 Aug 2018 12:58:15 GMT"
content-type: application/json; charset=UTF-8
content-length: 1895
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "hits" : {
    "total" : 6,
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "test_index",
        "_type" : "user",
        "_id" : "5",
        "_score" : 1.0,
        "_source" : {
          "name" : "bruce",
          "email" : "bruce@test.com",
          "tags" : [
            "游泳1"
          ]
        }
      },
      {
        "_index" : "test_index",
        "_type" : "user",
        "_id" : "3",
        "_score" : 1.0,
        "_source" : {
          "name" : "Alex",
          "email" : "alex@test.com",
          "tags" : [
            "吃饭"
          ]
        }
      }
    ]
  }
}
```

注意match_all是包含在query字典里的，query处于root节点位置

- **11.2查询包含输入字符的文档**

query还是处于root节点，增加一个键值sort排序与query同级，示例：

```
➜  ~ curl -i -XGET http://192.168.11.119:9200/test_index/user/_search\?pretty -d '
{
  "query": {
         "match": {
            "name" : "br"
          }
  },
  "sort": [
           {
             "email" : "desc"
           }
  ]
}
'
HTTP/1.1 200 OK
Warning: 299 Elasticsearch-5.5.2-b2f0c09 "Content type detection for rest requests is deprecated. Specify the content type using the [Content-Type] header." "Wed, 08 Aug 2018 13:03:30 GMT"
content-type: application/json; charset=UTF-8
content-length: 193
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "hits" : {
    "total" : 0,
    "max_score" : null,
    "hits" : [ ]
  }
}
```

查询包含Br字符的文档（行），并对结果以email倒序。第一次运行上面语句时报错`Fielddata is disabled on text fields by default. Set fielddata=true on [email] in order to load fielddata in memory by uninverting the inverted index. Note that this can however use significant memory."`,经查询资料，应该是5.x后对排序、聚合相关操作用单独的数据结构fileddata缓存到内存里，需调接口开启使用到的字段，[官方解释](https://www.elastic.co/guide/en/elasticsearch/reference/current/fielddata.html), 执行下面的操作开启:

```
➜  ~ curl -i -XPUT http://192.168.11.119:9200/test_index/_mapping/user\?pretty -d '
{
  "properties": {
    "email": {
      "type": "text",
      "fielddata": true
    }
  }
}'
HTTP/1.1 200 OK
Warning: 299 Elasticsearch-5.5.2-b2f0c09 "Content type detection for rest requests is deprecated. Specify the content type using the [Content-Type] header." "Wed, 08 Aug 2018 13:11:11 GMT"
content-type: application/json; charset=UTF-8
content-length: 28
{
  "acknowledged" : true
}
```

很多查询出来结果集很大，需要做分页，用DSL很简单，和query同级增加from和size键值，分表表示起始值和步长，示例

```
curl -i -XGET http://192.168.11.119:9200/test_index/user/_search?pretty -d '
{
  "query": {
   		"match_all": {
   		} 
  },
  "from" : 1,
  "size" : 2,
  "_source" : ["email"],
  "sort": [
  		{
  			"email" : "asc"
  		}
  ]
}
'
```

- **11.3查询过滤器**

搜索商品名包含Rhinestone，售卖价格小于3大于等于1的商品，结果按售卖价升序，构造DSL语句：

```
curl -i -XGET http://192.168.11.119:9200/en_es_category_products/product/_search?pretty -d '
{
  "query": {
   		"bool": {
   			"must" : {
   				"match" : {
   					"product_name" : "Rhinestone"
   				}
   			},
   			"filter" : {
   				"range" : {
   					"store_price" : {
   					   "gte" :  1
   						"lt" : 3
   					}
   				}
   			}
   		}
  },
  "_source" : [
  		"product_id",
  		"product_name",
  		"store_price",
  		"icon"
  ],
    "sort": [
  		{
  			"store_price" : "asc"
  		}
  ]
}
'
```

range操作符包含:

    * gt :: 大于
    * gte:: 大于等于
    * lt :: 小于
    * lte:: 小于等于

查询结果:

```
HTTP/1.1 200 OK
Warning: 299 Elasticsearch-5.5.2-b2f0c09 "Content type detection for rest requests is deprecated. Specify the content type using the [Content-Type] header." "Wed, 08 Aug 2018 13:15:52 GMT"
content-type: application/json; charset=UTF-8
content-length: 1141
{
  "took" : 2,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "hits" : {
    "total" : 2,
    "max_score" : null,
    "hits" : [
      {
        "_index" : "en_es_category_products",
        "_type" : "product",
        "_id" : "22100",
        "_score" : null,
        "_source" : {
          "product_id" : 22100,
          "icon" : "http://patpatdev.s3.amazonaws.com/Product/22100/1688I-SL-003-00008-001.jpg/1464845443.jpg",
          "store_price" : "2.99",
          "product_name" : "U-shape Silver Faux Perarl & Rhinestone Clip"
        },
        "sort" : [
          2.99
        ]
      },
      {
        "_index" : "en_es_category_products",
        "_type" : "product",
        "_id" : "354460",
        "_score" : null,
        "_source" : {
          "product_id" : 354460,
          "icon" : "http://patpatdev.s3.us-west-1.amazonaws.com/product/000766000119/5b0e5b0e49e8f.jpg",
          "store_price" : "2.99",
          "product_name" : "Pretty Star Decor Rhinestone Stud Hairband for Women"
        },
        "sort" : [
          2.99
        ]
      }
    ]
  }
}
```

注意参数嵌套了好几层，很容易写错，query、_source、sort都处于root级，query/bool下包含must、filter两级


