---
title: Elasticsearch笔记（四）
date: 2020/02/05
updated: 2020/02/05
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