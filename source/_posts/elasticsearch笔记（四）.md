---
title: Elasticsearch笔记（四）
date: 2020/02/05
updated: 2020/02/06
comments: true
categories: 
- [笔记, database, elasticsearch]
tags: 
- elasticsearch
- 非关系型数据库
---
## Elasticsearch笔记（四）

### 十一、结构化搜索

#### 1. 精确值查找
1. 制造数据
```
POST /my_store/products/_bulk
{ "index": { "_id": 1 }}
{ "price" : 10, "productID" : "XHDK-A-1293-#fJ3" }
{ "index": { "_id": 2 }}
{ "price" : 20, "productID" : "KDKE-B-9947-#kL5" }
{ "index": { "_id": 3 }}
{ "price" : 30, "productID" : "JODL-X-1937-#pV7" }
{ "index": { "_id": 4 }}
{ "price" : 30, "productID" : "QQPX-R-3956-#aD8" }
```
2. `term`查询数字
    + SQL语句
    ```
    SELECT document
    FROM   products
    WHERE  price = 20
    ```
    + Elasticsearch的查询表达式(query DSL)
    ```
    GET /my_store/products/_search
    {
      "query" : {
        "constant_score" : {
          "filter" : {
            "term" : {
              "price" : 20
            }
          }
        }
      }
    }
    ```
3. `term`查询文本
    + SQL语句
    ```
    SELECT product
    FROM   products
    WHERE  productID = "XHDK-A-1293-#fJ3"
    ```
    + Elasticsearch的查询表达式(query DSL)
    ```
    GET /my_store/products/_search
    {
      "query" : {
        "constant_score" : {
          "filter" : {
            "term" : {
              "productID" : "XHDK-A-1293-#fJ3"
            }
          }
        }
      }
    }
    ```
    + 发现查询不到，查看`productID`分词情况(因为字段被分词所以无法查询到精确值)
    ```
    GET /my_store/_analyze
    {
      "field": "productID",
      "text": "XHDK-A-1293-#fJ3"
    }
    ```
    + 重建索引，设置`productID`不分词(删除索引，重建索引，重新导入数据)
    ```
    DELETE /my_store
    PUT /my_store
    {
      "mappings" : {
        "products" : {
          "properties" : {
            "productID" : {
              "type" : "string",
              "index" : "not_analyzed"
            }
          }
        }
      }
    }
    ```
    + 重新查询即可查询到
4. 内部过滤器的操作
    + 在内部，Elasticsearch会在运行非评分查询的时执行多个操作
    1. 查找匹配文档
    2. 创建bitset(一个包含0和1的数组，匹配文档标1，没匹配标0)
    3. 迭代bitsets(一般来说先迭代稀疏的bitset，可以排除掉大量的文档)
    4. 增量使用计数，只缓存那些在将来会被再次使用的查询，以避免资源的浪费
    + 从概念上记住非评分计算是首先执行的，这将有助于写出高效又快速的搜索请求

#### 2. 组合过滤器
1. 布尔过滤器(一个`bool`过滤器由三部分组成，分别与`AND`、`NOT`和`OR`等价)
```
{
   "bool" : {
      "must" :     [],
      "should" :   [],
      "must_not" : []
   }
}
```
    + SQL语句
    ```
    SELECT product
    FROM   products
    WHERE  (price = 20 OR productID = "XHDK-A-1293-#fJ3")
      AND  (price != 30)
    ```
    + Elasticsearch的查询表达式(query DSL)
    ```
    GET /my_store/products/_search
    {
      "query" : {
        "filtered" : {
          "filter" : {
            "bool" : {
              "should" : [
                { "term" : {"price" : 20}},
                { "term" : {"productID" : "XHDK-A-1293-#fJ3"}}
              ],
              "must_not" : {
                "term" : {"price" : 30}
              }
            }
          }
        }
      }
    }
    ```
2. 嵌套布尔过滤器
    + SQL语句
    ```
    SELECT document
    FROM   products
    WHERE  productID      = "KDKE-B-9947-#kL5"
      OR (     productID = "JODL-X-1937-#pV7"
           AND price     = 30 )
    ```
    + Elasticsearch的查询表达式(query DSL)
    ```
    GET /my_store/products/_search
    {
      "query" : {
        "filtered" : {
          "filter" : {
            "bool" : {
              "should" : [
                { "term" : {"productID" : "KDKE-B-9947-#kL5"}},
                {
                  "bool" : {
                    "must" : [
                      {"term" : {"productID" : "JODL-X-1937-#pV7"}},
                      {"term" : {"price" : 30}}
                    ]
                  }
                }
              ]
            }
          }
        }
      }
    }
    ```

#### 3. 查找多个精确值
```
GET /my_store/products/_search
{
  "query" : {
    "constant_score" : {
      "filter" : {
        "terms" : {
          "price" : [20, 30]
        }
      }
    }
  }
}
```

#### 4. 包含，而不是相等
+ 一定要了解term和terms是包含(contains)操作，而非等值(equals)
+ 例如`{ "term" : { "tags" : "search" } }`过滤器会匹配到以下两个文档
```
{ "tags" : ["search"] }
{ "tags" : ["search", "open_source"] }
```
+ 如果一定期望整个字段完全相等，最好的方式是增加并索引另一个字段，这个字段用以存储该字段包含词项的数量，同样以上面提到的两个文档为例，现在我们包括了一个维护标签数的新字段
```
{ "tags" : ["search"], "tag_count" : 1 }
{ "tags" : ["search", "open_source"], "tag_count" : 2 }
```
+ 可以构造一个`constant_score`查询，来确保结果中的文档所包含的词项数量与要求是一致的
```
GET /my_index/my_type/_search
{
  "query": {
    "constant_score" : {
      "filter" : {
        "bool" : {
          "must" : [
            { "term" : { "tags" : "search" } },
            { "term" : { "tag_count" : 1 } }
          ]
        }
      }
    }
  }
}
```

#### 5. 范围
+ `range`查询可同时提供包含(inclusive)和不包含(exclusive)这两种范围表达式，可供组合的选项如下
    + `gt`:`>`大于(greater than)
    + `lt`:`<`小于(less than)
    + `gte`:`>=`大于或等于(greater than or equal to)
    + `lte`:`<=`小于或等于(less than or equal to)
1. 数字范围
    + SQL语句
    ```
    SELECT document
    FROM   products
    WHERE  price BETWEEN 20 AND 40
    ```
    + Elasticsearch的查询表达式(query DSL)
    ```
    GET /my_store/products/_search
    {
      "query" : {
        "constant_score" : {
          "filter" : {
            "range" : {
              "price" : {
                "gte" : 20,
                "lt"  : 40
              }
            }
          }
        }
      }
    }
    ```
    + 如果想要范围无界(比方说大于20)，只须省略其中一边的限制
    ```
    "range" : {
      "price" : {
        "gt" : 20
      }
    }
    ```
2. 日期范围
    + `range`查询同样可以应用在日期字段上
    ```
    "range" : {
      "timestamp" : {
        "gt" : "2014-01-01 00:00:00",
        "lt" : "2014-01-07 00:00:00"
      }
    }
    ```
    + `range`查询支持对日期计算(date math)进行操作(例：查找时间戳在过去一小时内的所有文档)
    ```
    "range" : {
        "timestamp" : {
            "gt" : "now-1h"
        }
    }
    ```
    + 日期计算还可以被应用到某个具体的时间，并非只能是一个像`now`这样的占位符(只要在某个日期后加上一个双管符号`||`)
    ```
    "range" : {
        "timestamp" : {
            "gt" : "2014-01-01 00:00:00",
            "lt" : "2014-01-01 00:00:00||+1M"
        }
    }
    ```
3. 字符串范围
    + `range`查询同样可以处理字符串字段，字符串范围可采用字典顺序(lexicographically)或字母顺序(alphabetically)
    + 在倒排索引中的词项就是采取字典顺序排列的，例：`5,50,6,B,C,a,ab,abb,abc,b`
    + 查找从a到b(不包含)的字符串
    ```
    "range" : {
        "title" : {
            "gte" : "a",
            "lt" :  "b"
        }
    }
    ```
    + 数字和日期字段的索引方式使高效地范围计算成为可能。但字符串却并非如此，要想对其使用范围过滤，Elasticsearch实际上是在为范围内的每个词项都执行`term`过滤器，这会比日期或数字的范围过滤慢许多
    + 字符串范围在过滤低基数(low cardinality)字段(即只有少量唯一词项)时可以正常工作，但是唯一词项越多，字符串范围的计算会越慢

#### 6. 处理NULL值
+ 如果一个字段没有值(包括`""` `[]` `null` `[null]`)，那么它不会被存入倒排索引中
1. 存在查询
    + 制造数据
    ```
    POST /my_index/posts/_bulk
    { "index": { "_id": "1"}}
    { "tags" : ["search"]}
    { "index": { "_id": "2"}}
    { "tags" : ["search", "open_source"]}
    { "index": { "_id": "3"}}
    { "other_field" : "some data"}
    { "index": { "_id": "4"}}
    { "tags" : null}
    { "index": { "_id": "5"}}
    { "tags" : ["search", null]}
    ```
    + 以上文档集合中`tags`字段对应的倒排索引如下
    |Token|DocIDs|
    |:-:|:-:|
    |open_source|2|
    |search|1,2,5|
    + SQL语句
    ```
    SELECT tags
    FROM   posts
    WHERE  tags IS NOT NULL
    ```
    + Elasticsearch的查询表达式(query DSL)
    ```
    GET /my_index/posts/_search
    {
      "query" : {
        "constant_score" : {
          "filter" : {
            "exists" : { "field" : "tags" }
          }
        }
      }
    }
    ```
2. 缺失查询
    + SQL语句
    ```
    SELECT tags
    FROM   posts
    WHERE  tags IS NULL
    ```
    + Elasticsearch的查询表达式(query DSL)
    ```
    GET /my_index/posts/_search
    {
      "query" : {
        "constant_score" : {
          "filter": {
            "missing" : { "field" : "tags" }
          }
        }
      }
    }
    ```
3. 对象上的存在与缺失
    + 不仅可以过滤核心类型，`exists`和`missing`查询还可以处理一个对象的内部字段
    ```
    {
      "name" : {
        "first" : "John",
        "last" :  "Smith"
      }
    }
    ```
    + 在映射中，如上对象的内部是个扁平的字段与值(field-value)的简单键值结构
    ```
    {
      "name.first" : "John",
      "name.last"  : "Smith"
    }
    ```
    + 执行下面这个过滤的时候
    ```
    {
      "exists" : { "field" : "name" }
    }
    ```
    + 实际执行的是
    ```
    {
      "bool": {
        "should": [
          { "exists": { "field": "name.first" }},
          { "exists": { "field": "name.last" }}
        ]
      }
    }
    ```
4. `null`值设置
    + 有时候我们需要区分一个字段是没有值，还是它已被显式的设置成了`null`。默认的行为是无法做到这点的，数据被丢失了。但是可以选择将显式的`null`值替换成我们指定占位符(placeholder)
    + 当insert或update数据遇到空值时，将使用该值，这个显式的空值会对其进行索引，以便于搜索
    + 在为字符串(string)、数字(numeric)、布尔值(Boolean)或日期(date)字段指定映射时，同样可以为之设置`null_value`空值，用以处理显式`null`值的情况。不过即使如此，还是会将一个没有值的字段从倒排索引中排除
    + 当选择合适的`null_value`空值的时候，需要保证以下几点：
        1. 它会匹配字段的类型，我们不能为一个date日期字段设置字符串类型的`null_value`
        2. 它必须与普通值不一样，这可以避免把实际值当成`null`空的情况
    + 例：
    ```
    PUT /my_index/_mapping/my_type
    {
      "properties": {
        "status_code": {
          "type": "string",
          "null_value": "NULL"
        }
      }
    }
    ```
    
#### 7. 关于缓存
+ 过滤器的内部操作核心实际是采用一个bitset记录与过滤器匹配的文档。Elasticsearch积极地把这些bitset缓存起来以备随后使用，bitset可以复用任何已使用过的相同过滤器，而无需再次计算整个过滤器
+ 这些bitsets缓存是智能的，它们以增量方式更新。当我们索引新文档时，只需将那些新文档加入已有bitset，而不是对整个缓存一遍又一遍的重复计算。和系统其他部分一样，过滤器是实时的，我们无需担心缓存过期问题
1. 独立的过滤器缓存
    + 属于一个查询组件的bitsets是独立于它所属搜索请求其他部分的。一旦被缓存，一个查询可以被用作多个搜索请求。bitsets并不依赖于它所存在的查询上下文。这样使得缓存可以加速查询中经常使用的部分，从而降低较少易变的部分所带来的消耗
    + 同样，如果单个请求重用相同的非评分查询，它缓存的bitset可以被单个搜索里的所有实例所重用
2. 自动缓存行为
    + Elasticsearch会基于使用频次自动缓存查询。如果一个非评分查询在最近的256次查询中被使用过(次数取决于查询类型)，那么这个查询就会作为缓存的候选
    + 但是，并不是所有的片段都能保证缓存bitset。只有那些文档数量超过10000(或超过总文档数量的3%)才会缓存bitset。因为小的片段可以很快的进行搜索和合并，这里缓存的意义不大
    + 一旦缓存了，非评分计算的bitset会一直驻留在缓存中直到它被剔除。剔除规则是基于LRU的：一旦缓存满了，最近最少使用的过滤器会被剔除
    
### 十二、全文搜索

#### 1. 全文搜索两个重要方面
1. 相关性(Relevance) 
它是评价查询与其结果间的相关程度，并根据这种相关程度对结果排名的一种能力，这种计算方式可以是TF/IDF方法、地理位置邻近、模糊相似，或其他的某些算法
2. 分析(Analysis)  
它是将文本块转换为有区别的、规范化的token的一个过程，目的是为了创建倒排索引以及查询倒排索引
+ 一旦谈论相关性或分析这两个方面的问题时，我们所处的语境是关于查询的而不是过滤

#### 2. 基于词项与基于全文
1. 基于词项的查询  
如`term`或`fuzzy`这样的底层查询不需要分析阶段，它们对单个词项进行操作。用`term`查询词项`Foo`只要在倒排索引中查找准确词项，并且用TF/IDF算法为每个包含该词项的文档计算相关度评分`_score`
2. 基于全文的查询  
像`match`或`query_string`这样的查询是高层查询，它们了解字段映射的信息
    + 如果查询日期(date)或整数(integer)字段，它们会将查询字符串分别作为日期或整数对待
    + 如果查询一个(not_analyzed)未分析的精确值字符串字段，它们会将整个查询字符串作为单个词项对待
    + 但如果要查询一个(analyzed)已分析的全文字段，它们会先将查询字符串传递到一个合适的分析器，然后生成一个供查询的词项列表
+ 一旦组成了词项列表，这个查询会对每个词项逐一执行底层的查询，再将结果合并，然后为每个文档生成一个最终的相关度评分
+ 当我们想要查询一个具有精确值的`not_analyzed`未分析字段之前，需要考虑，是否真的采用评分查询，或者非评分查询会更好
+ 单词项查询通常可以用是、非这种二元问题表示，所以更适合用过滤，而且这样做可以有效利用缓存
```
GET /_search
{
  "query": {
    "constant_score": {
      "filter": {
        "term": { "gender": "female" }
      }
    }
  }
}
```

#### 3. 匹配查询
1. 索引一些数据(只设置一个主分片，防止在数据量少的时候发生相关性破坏)
```
DELETE /my_index

PUT /my_index
{ "settings": { "number_of_shards": 1 }}

POST /my_index/my_type/_bulk
{ "index": { "_id": 1 }}
{ "title": "The quick brown fox" }
{ "index": { "_id": 2 }}
{ "title": "The quick brown fox jumps over the lazy dog" }
{ "index": { "_id": 3 }}
{ "title": "The quick brown fox jumps over the quick dog" }
{ "index": { "_id": 4 }}
{ "title": "Brown fox brown dog" }
```
2. 单个词查询
```
GET /my_index/my_type/_search
{
  "query": {
    "match": {
      "title": "QUICK!"
    }
  }
}
```
+ Elasticsearch执行上面这个`match`查询的步骤是
    1. 检查字段类型  
    标题`title`字段是一个`string`类型(analyzed)已分析的全文字段，这意味着查询字符串本身也应该被分析
    2. 分析查询字符串  
    将查询的字符串`QUICK!`传入标准分析器中，输出的结果是单个项`quick`。因为只有一个单词项，所以`match`查询执行的是单个底层`term`查询
    3. 查找匹配文档  
    用`term`查询在倒排索引中查找`quick`然后获取一组包含该项的文档，本例的结果是文档：1、2和3
    4. 为每个文档评分  
    用`term`查询计算每个文档相关度评分`_score`，这是种将词频(词quick在相关文档的title字段中出现的频率)和反向文档频率(词quick在所有文档的title字段中出现的频率)，以及字段的长度(即字段越短相关度越高)相结合的计算方式

#### 4. 多词查询
1. `match`多词查询
```
GET /my_index/my_type/_search
{
  "query": {
    "match": {
      "title": "BROWN DOG!"
    }
  }
}
```
2. 提高精度  
`match`查询还可以接受`operator`操作符作为输入参数，默认情况下该操作符是`or`。我们可以将它修改成`and`让所有指定词项都必须匹配
```
GET /my_index/my_type/_search
{
  "query": {
    "match": {
      "title": {
        "query":    "BROWN DOG!",
        "operator": "and"
      }
    }
  }
}
```
3. 控制精度  
`match`查询支持`minimum_should_match`最小匹配参数，可以指定必须匹配的词项数。可以将其设置为某个具体数字，更常用的做法是将其设置为一个百分数
```
GET /my_index/my_type/_search
{
  "query": {
    "match": {
      "title": {
        "query":                "quick brown dog",
        "minimum_should_match": "75%"
      }
    }
  }
}
```

#### 5. 组合查询
+ 在组合过滤器中，`bool`过滤器通过`and`、`or`和`not`逻辑组合将多个过滤器进行组合。在查询中，`bool`查询有类似的功能
+ 与过滤器一样，`bool`查询也可以接受`must`、`must_not`和`should`参数下的多个查询语句
```
GET /my_index/my_type/_search
{
  "query": {
    "bool": {
      "must":     { "match": { "title": "quick" }},
      "must_not": { "match": { "title": "lazy"  }},
      "should": [
                  { "match": { "title": "brown" }},
                  { "match": { "title": "dog"   }}
      ]
    }
  }
}
```
+ 评分计算
    + `bool`查询会为每个文档计算相关度评分`_score`，再将所有匹配的`must`和`should`语句的分数`_score`求和，最后除以`must`和`should`语句的总数
    + `must_not`语句不会影响评分，它的作用只是将不相关的文档排除
    + 默认情况下，没有`should`语句是必须匹配的。只有一个例外，那就是当没有`must`语句的时候，至少有一个`should`语句必须匹配
+ 控制精度  
通过`minimum_should_match`参数控制需要匹配的`should`语句的数量，它既可以是一个绝对的数字，又可以是个百分比
```
GET /my_index/my_type/_search
{
  "query": {
    "bool": {
      "should": [
        { "match": { "title": "brown" }},
        { "match": { "title": "fox"   }},
        { "match": { "title": "dog"   }}
      ],
      "minimum_should_match": 2
    }
  }
}
```

#### 6. 如何使用布尔匹配
+ 多词`match`查询只是简单地将生成的`term`查询包裹在一个`bool`查询中
1. 使用默认的`or`操作符，每个`term`查询都被当作`should`语句，这样就要求必须至少匹配一条语句
```
{
  "match": { "title": "brown fox"}
}
{
  "bool": {
    "should": [
      { "term": { "title": "brown" }},
      { "term": { "title": "fox"   }}
    ]
  }
}
```
2. 使用`and`操作符，所有的`term`查询都被当作`must`语句，所以所有(all)语句都必须匹配
```
{
  "match": {
    "title": {
      "query":    "brown fox",
      "operator": "and"
    }
  }
}
{
  "bool": {
    "must": [
      { "term": { "title": "brown" }},
      { "term": { "title": "fox"   }}
    ]
  }
}
```
3. 如果指定参数`minimum_should_match`，它可以通过`bool`查询直接传递，使以下两个查询等价
```
{
  "match": {
    "title": {
      "query":                "quick brown fox",
      "minimum_should_match": "75%"
    }
  }
}
{
  "bool": {
    "should": [
      { "term": { "title": "brown" }},
      { "term": { "title": "fox"   }},
      { "term": { "title": "quick" }}
    ],
    "minimum_should_match": 2
  }
}
```
+ 通常将这些查询用`match`查询来表示，但是如果了解`match`内部的工作原理，就能根据自己的需要来控制查询过程。有些时候单个`match`查询无法满足需求，比如为某些查询条件分配更高的权重

#### 7. 查询语句提升权重
1. `bool`查询
```
GET /_search
{
  "query": {
    "bool": {
      "must": {
        "match": {
          "content": {
            "query":    "full text search",
            "operator": "and"
          }
        }
      },
      "should": [
        { "match": { "content": "Elasticsearch" }},
        { "match": { "content": "Lucene"        }}
      ]
    }
  }
}
```
2. 让包含`Lucene`的有更高的权重，并且包含`Elasticsearch`的语句比`Lucene`的权重更高
```
GET /_search
{
  "query": {
    "bool": {
      "must": {
        "match": {
          "content": {
            "query":    "full text search",
            "operator": "and"
          }
        }
      },
      "should": [
        { "match": {
          "content": {
            "query": "Elasticsearch",
            "boost": 3
          }
        }},
        { "match": {
          "content": {
            "query": "Lucene",
            "boost": 2
          }
        }}
      ]
    }
  }
}
```
3. `boost`参数被用来提升一个语句的相对权重(`boost`值大于1)或降低相对权重(`boost`值处于0到1之间)，但是这种提升或降低并不是线性的，如果一个`boost`值为2，并不能获得两倍的评分`_score`
4. 如果不基于TF/IDF要实现自己的评分模型，我们就需要对权重提升的过程能有更多控制，可以使用`function_score`查询操纵一个文档的权重提升方式而跳过归一化这一步骤

#### 8. 控制分析
+ 查询只能查找倒排索引表中真实存在的项，所以保证文档在索引时与查询字符串在搜索时应用相同的分析过程非常重要，这样查询的项才能够匹配倒排索引中的项
+ 为`my_index`新增一个字段
```
PUT /my_index/_mapping/my_type
{
  "my_type": {
    "properties": {
      "english_title": {
        "type":     "string",
        "analyzer": "english"
      }
    }
  }
}
```
+ 通过使用`analyze`API来分析单词`Foxes`，进而比较`english_title`字段和`title`字段在索引时的分析结果
```
GET /my_index/_analyze
{
  "field": "my_type.title",
  "text": "Foxes"
}

GET /my_index/_analyze
{
  "field": "my_type.english_title",
  "text": "Foxes"
}
```
+ 也可以通过`validate-query`API查看这个行为
```
GET /my_index/my_type/_validate/query?explain
{
  "query": {
    "bool": {
      "should": [
        { "match": { "title":         "Foxes"}},
        { "match": { "english_title": "Foxes"}}
      ]
    }
  }
}
```
+ 默认分析器
    + Elasticsearch会按照以下顺序依次处理，直到它找到能够使用的分析器，索引时的顺序如下
        1. 字段映射里定义的`analyzer`
        2. 索引设置中名为`default`的分析器
        3. 默认为`standard`标准分析器
    + 在搜索时，顺序有些许不同
        1. 查询自己定义的`analyzer`
        2. 字段映射里定义的`analyzer`
        3. 索引设置中名为`default`的分析器
        4. 默认为`standard`标准分析器
    + 考虑额外参数，一个搜索时的完整顺序会是下面这样
        1. 查询自己定义的`analyzer`
        2. 字段映射里定义的`search_analyzer`
        3. 字段映射里定义的`analyzer`
        4. 索引设置中名为`default_search`的分析器
        5. 索引设置中名为`default`的分析器
        6. 默认为`standard`标准分析器
+ 分析器配置实践  
    + 多数字符串字段都是`not_analyzed`精确值字段，比如标签(tag)或枚举(enum)，而且更多的全文字段会使用默认的`standard`分析器或`english`或其他某种语言的分析器
    + 这样只需要为少数一两个字段指定自定义分析：或许标题`title`字段需要以支持输入即查找(find-as-you-type)的方式进行索引
    + 可以在索引级别设置中，为绝大部分的字段设置你想指定的`default`默认分析器。然后在字段级别设置中，对某一两个字段配置需要指定的分析器

#### 9. 被破坏的相关度！
+ 由于性能原因，Elasticsearch不会计算索引内所有文档的IDF。相反，每个分片会根据该分片内的所有文档计算一个本地IDF
+ 因为文档是均匀分布存储的，两个分片的IDF是相同的。如果数据在一个分片里非常普通(所以不那么重要)，但是在另一个分片里非常出现很少(所以会显得更重要)。这些IDF之间的差异会导致不正确的结果
+ 在实际应用中，这并不是一个问题，本地和全局的IDF的差异会随着索引里文档数的增多渐渐消失，在真实世界的数据量下，局部的IDF会被迅速均化，所以上述问题并不是相关度被破坏所导致的，而是由于数据太少
+ 数据量少的测试环境，可以通过两种方式解决这个问题
    1. 只在主分片上创建索引，如果只有一个主分片，那么本地的IDF就是全局的IDF
    2. 在搜索请求后添加`?search_type=dfs_query_then_fetch`，`dfs`是指分布式频率搜索，它告诉Elasticsearch先分别获得每个分片本地的IDF，然后根据结果再计算整个索引的全局IDF
+ 不要在生产环境上使用`dfs_query_then_fetch`。完全没有必要，只要有足够的数据就能保证词频是均匀分布的。没有理由给每个查询额外加上DFS这步