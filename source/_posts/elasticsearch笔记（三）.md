---
title: Elasticsearch笔记（三）
date: 2020/02/04
updated: 2020/02/05
comments: true
categories: 
- [笔记, database, elasticsearch]
tags: 
- elasticsearch
- 非关系型数据库
---
## Elasticsearch笔记（三）

### 九、索引管理

#### 1. 创建删除索引
1. 创建索引
```
PUT /my_index
{
  "settings": { ... any settings ... },
  "mappings": {
    "type_one": { ... any mappings ... },
    "type_two": { ... any mappings ... },
    ...
  }
```
    + 可以通过在config/elasticsearch.yml中添加下面的配置来防止自动创建索引`action.auto_create_index: false`
2. 删除索引
    + 使用以下的请求来删除索引`DELETE /my_index`
    + 也可以用下面的方式删除多个索引`DELETE /index_one,index_two` `DELETE /index_*`
    + 甚至可以删除所有索引`DELETE /_all`

#### 2. 索引设置
1. 可以创建只有一个主分片，没有复制分片的小索引(`number_of_shards`定义一个索引的主分片个数，默认值是`5`。这个配置在索引创建后不能修改)
```
PUT /my_temp_index
{
  "settings": {
    "number_of_shards" : 1,
    "number_of_replicas" : 0
  }
}
```
2. 然后，我们可以用update-index-settings API动态修改复制分片个数
```
PUT /my_temp_index/_settings
{
  "number_of_replicas": 1
}
```

#### 3. 配置分析器
1. 第三个重要的索引设置是`analysis`部分，用来配置已存在的分析器或创建自定义分析器来定制化你的索引
2. `standard`分析器是用于全文字段的默认分析器，对于大部分西方语系来说是一个不错的选择。它考虑了以下几点
    + `standard`分词器，在词层级上分割输入的文本
    + `standard`表征过滤器，被设计用来整理分词器触发的所有表征(但是目前什么都没做)
    + `lowercase`表征过滤器，将所有表征转换为小写
    + `stop`表征过滤器，删除所有可能会造成搜索歧义的停用词，如`a` `the` `and` `is`(停用词过滤器默认是被禁用的)
3. 创建一个基于`standard`分析器的自定义分析器，并且设置`stopwords`参数可以提供一个停用词列表，或者使用一个特定语言的预定停用词列表
4. 在下面的例子中，我们创建了一个新的分析器，叫做`es_std`，并使用预定义的西班牙语停用词
```
PUT /spanish_docs
{
  "settings": {
    "analysis": {
      "analyzer": {
        "es_std": {
          "type": "standard",
          "stopwords": "_spanish_"
        }
      }
    }
  }
}
```
5. `es_std`分析器不是全局的，它仅仅存在于我们定义的`spanish_docs`索引中。为了用analyze API来测试它，我们需要使用特定的索引名
```
GET /spanish_docs/_analyze?analyzer=es_std
El veloz zorro marrón
```
6. 下面简化的结果中显示停用词`El`被正确的删除了  
```
{
  "tokens" : [
    { "token" : "veloz", "position" : 2 },
    { "token" : "zorro", "position" : 3 },
    { "token" : "marrón", "position" : 4 }
  ]
}
```

#### 4. 自定义分析器
1. 分析器是三个顺序执行的组件的结合
    1. 字符过滤器(char_filter)：
        + `html_strip`字符过滤器可以删除所有的HTML标签，并且将HTML实体转换成对应的Unicode字符
    2. 分词器(tokenizer)：
        + `standard`分词器将字符串分割成单独的字词，删除大部分标点符号
        + `keyword`分词器输出和它接收到的相同的字符串，不做任何分词处理
        + `whitespace`分词器只通过空格来分割文本
        + `pattern`分词器可以通过正则表达式来分割文本
    3. 表征过滤器(filter)：
        + `lowercase`表征过滤器将所有表征转换为小写
        + `stop`表征过滤器删除所有可能会造成搜索歧义的停用词，如a the and is
        + `stemmer`表征过滤器将单词转化为他们的根形态(root form)
        + `ascii_folding`表征过滤器会删除变音符号，比如从très转为tres
        + `ngram`和`edge_ngram`表征过滤器可以让表征更适合特殊匹配情况或自动完成
2. 创建自定义分析器的格式
```
PUT /my_index
{
  "settings": {
    "analysis": {
      "char_filter": { ... custom character filters ... },
      "tokenizer": { ... custom tokenizers ... },
      "filter": { ... custom token filters ... },
      "analyzer": { ... custom analyzers ... }
    }
  }
}
```
3. 创建自定义字符过滤器
```
"char_filter": {
  "&_to_and": {
    "type": "mapping",
    "mappings": [ "&=> and "]
  }
}
```
4. 创建自定义表征过滤器
```
"filter": {
  "my_stopwords": {
    "type": "stop",
    "stopwords": [ "the", "a" ]
  }
}
```
5. 组合成分析器
```
"analyzer": {
  "my_analyzer": {
    "type": "custom",
    "char_filter": [ "html_strip", "&_to_and" ],
    "tokenizer": "standard",
    "filter": [ "lowercase", "my_stopwords" ]
  }
}
```
6. 用下面的方式可以将以上请求合并成一条
```
PUT /my_index
{
  "settings": {
    "analysis": {
      "char_filter": {
        "&_to_and": {
          "type": "mapping",
          "mappings": [ "&=> and "]
        }
      },
      "filter": {
        "my_stopwords": {
          "type": "stop",
          "stopwords": [ "the", "a" ]
        }
      },
      "analyzer": {
        "my_analyzer": {
          "type": "custom",
          "char_filter": [ "html_strip", "&_to_and" ],
          "tokenizer": "standard",
          "filter": [ "lowercase", "my_stopwords" ]
        }
      }
    }
  }
}
```
7. 测试分析器`GET /my_index/_analyze?analyzer=my_analyzer  The quick & brown fox`
8. 使用分析器
```
PUT /my_index/_mapping/my_type
{
  "properties": {
    "title": {
      "type": "string",
      "analyzer": "my_analyzer"
    }
  }
}
```

#### 5. 类型和映射
1. Lucene中一个文档由一组简单的键值对组成，一个字段至少需要有一个值，但是任何字段都可以有多个值。类似的，一个单独的字符串可能在分析过程中被转换成多个值。Lucene不关心这些值是字符串，数字或日期，所有的值都被当成不透明字节
2. Elasticsearch类型是在这个简单基础上实现的。一个索引可能包含多个类型，每个类型有各自的映射和文档，保存在同一个索引中
    + 因为Lucene没有文档类型的概念，每个文档的类型名被储存在一个叫`_type`的元数据字段上。当我们搜索一种特殊类型的文档时，Elasticsearch简单的通过`_type`字段来过滤出这些文档
    + Lucene同样没有映射的概念。映射是Elasticsearch将复杂JSON文档映射成Lucene需要的扁平化数据的方式
3. 预防类型陷阱
    + 想象一下我们的索引中有两种类型：blog_en表示英语版的博客，blog_es表示西班牙语版的博客。两种类型都有title字段，但是其中一种类型使用english分析器，另一种使用spanish分析器
    + 我们在两种类型中搜索title字段，首先需要分析查询语句，Elasticsearch会采用第一个被找到的title字段使用的分析器，这对于这个字段的文档来说是正确的，但对另一个来说却是错误的
    + 我们可以通过给字段取不同的名字来避免这种错误。比如，用title_en和title_es。或者在查询中明确包含各自的类型名
    ```
    GET /_search
    {
      "query": {
        "multi_match": {
          "query": "The quick brown fox",
          "fields": [ "blog_en.title", "blog_es.title" ]
        }
      }
    }
    ```
    + 为了保证你不会遇到这些冲突，建议在同一个索引的每一个类型中，确保用同样的方式映射同名的字段

#### 6. 根对象
1. 映射的最高一层被称为根对象，它可能包含下面几项
    + 一个`properties`节点，列出了文档中可能包含的每个字段的映射
    + 多个元数据字段，每一个都以下划线开头，例如`_type` `_id`和`_source`
    + 设置项，控制如何动态处理新的字段，例如`analyzer` `dynamic_date_formats`和`dynamic_templates`
    + 其他设置，可以同时应用在根对象和其他`object`类型的字段上，例如`enabled` `dynamic`和`include_in_all`
2. 文档字段和属性的三个最重要的设置
    + `type`：字段的数据类型，例如string和date
    + `index`：字段是否应当被当成全文来搜索(analyzed)，或被当成一个准确的值(not_analyzed)，还是完全不可被搜索(no)
    + `analyzer`：确定在索引和或搜索时全文字段使用的分析器
    
#### 7. 元数据中的source字段
1. 默认情况下，Elasticsearch用JSON字符串来表示文档主体保存在`_source`字段中。像其他保存的字段一样，`_source`字段也会在写入硬盘前压缩
2. 可以用下面的映射禁用`_source`字段
```
PUT /my_index
{
  "mappings": {
    "my_type": {
      "_source": {
      "enabled": false
      }
    }
  }
}
```
3. 在搜索请求中你可以通过限定`_source`字段来请求指定字段
```
GET /_search
{
  "query": { "match_all": {}},
  "_source": [ "title", "created" ]
}
```
4. 在Elasticsearch中，单独设置储存字段不是一个好做法。完整的文档已经被保存在`_source`字段中。通常最好的办法会是使用`_source`参数来过滤你需要的字段

#### 8. 元数据中的all字段
1. 如果你决定不再使用`_all`字段，你可以通过下面的映射禁用它
```
PUT /my_index/_mapping/my_type
{
  "my_type": {
    "_all": { "enabled": false }
  }
}
```
2. 通过`include_in_all`选项可以控制字段是否要被包含在`_all`字段中，默认值是`true`。在一个对象上设置`include_in_all`可以修改这个对象所有字段的默认行为
3. 你可能想要保留`_all`字段来查询所有特定的全文字段。相对于完全禁用`_all`字段，你可以先默认禁用`include_in_all`选项，而选定字段上启用`include_in_all`
```
PUT /my_index/my_type/_mapping
{
  "my_type": {
    "include_in_all": false,
    "properties": {
      "title": {
        "type": "string",
        "include_in_all": true
      },
      ...
    }
  }
}
```
4. 谨记`_all`字段仅仅是一个经过分析的string字段。它使用默认的分析器来分析它的值，而不管这值本来所在的字段指定的分析器。而且像所有string类型字段一样，你可以配置`_all`字段使用的分析器
```
PUT /my_index/my_type/_mapping
{
  "my_type": {
    "_all": { "analyzer": "whitespace" }
  }
}
```

#### 9. 元数据中的id字段
1. 文档唯一标识由四个元数据字段组成
    + `_id`：文档的字符串ID
    + `_type`：文档的类型名
    + `_index`：文档所在的索引
    + `_uid`：`_type`和`_id`连接成的`type#id`
2. `_id`字段有一个你可能用得到的设置：`path`设置告诉Elasticsearch它需要从文档本身的哪个字段中生成`_id`
```
PUT /my_index
{
  "mappings": {
    "my_type": {
      "_id": {
        "path": "doc_id"
      },
      "properties": {
        "doc_id": {
          "type": "string",
          "index": "not_analyzed"
        }
      }
    }
  }
}
```
3. 虽然这样很方便，但是注意它对`bulk`请求有个轻微的性能影响。处理请求的节点将不能仅靠解析元数据行来决定将请求分配给哪一个分片，而需要解析整个文档主体

#### 10. 动态映射
1. 当Elasticsearch遇到一个未知的字段时，它通过动态映射来确定字段的数据类型且自动将该字段加到类型映射中
2. 可以通过`dynamic`设置来控制这些行为，它接受下面几个选项
    + `true`：自动添加字段(默认)
    + `false`：忽略字段
    + `strict`：当遇到未知字段时抛出异常
3. `dynamic`设置可以用在根对象或任何`object`对象上。你可以将`dynamic`默认设置为`strict`，而在特定内部对象上启用它
```
PUT /my_index
{
  "mappings": {
    "my_type": {
      "dynamic": "strict",
      "properties": {
        "title": { "type": "string"},
        "stash": {
          "type": "object",
          "dynamic": true
        }
      }
    }
  }
}
```
4. 将`dynamic`设置成`false`完全不会修改`_source`字段的内容。`_source`将仍旧保持你索引时的完整JSON文档。然而，没有被添加到映射的未知字段将不可被搜索

#### 11. 自定义动态映射
1. 当Elasticsearch遇到一个新的字符串字段时，它会检测这个字段是否包含一个可识别的日期，比如`2014-01-01`。如果它看起来像一个日期，这个字段会被作为`date`类型添加，否则，它会被作为`string`类型添加
2. 日期检测可以通过在根对象上设置`date_detection`为`false`来关闭
```
PUT /my_index
{
  "mappings": {
    "my_type": {
      "date_detection": false
    }
  }
}
```
3. Elasticsearch判断字符串为日期的规则可以通过`dynamic_date_formats`配置来修改
4. 使用`dynamic_templates`，你可以完全控制新字段的映射，你设置可以通过字段名或数据类型应用一个完全不同的映射  
(es:字段名以`_es`结尾需要使用`spanish`分析器 en:所有其他字段使用`english`分析器)
```
PUT /my_index
{
  "mappings": {
    "my_type": {
      "dynamic_templates": [
        { "es": {
          "match": "*_es",
          "match_mapping_type": "string",
          "mapping": {
            "type": "string",
            "analyzer": "spanish"
          }
        }},
        { "en": {
          "match": "*",
          "match_mapping_type": "string",
          "mapping": {
            "type": "string",
            "analyzer": "english"
          }
        }}
      ]
}}}
```
5. `match_mapping_type`允许你限制模板只能使用在特定的类型上，就像由标准动态映射规则检测的一样，(例如string和long)
6. `match`参数只匹配字段名，`path_match`参数则匹配字段在一个对象中的完整路径，所以`address.*.name`规则将匹配一个这样的字段
```
{
  "address": {
    "city": {
      "name": "New York"
    }
  }
}
```
7. `unmatch`和`path_unmatch`规则将用于排除未被匹配的字段

#### 12. 默认映射
1. 通常，一个索引中的所有类型具有共享的字段和设置。用`_default_`映射来指定公用设置会更加方便，而不是每次创建新的类型时重复操作
2. `_default`映射像新类型的模板。所有在`_default_`映射 之后的类型将包含所有的默认设置，除非在自己的类型映射中明确覆盖这些配置
```
PUT /my_index
{
  "mappings": {
    "_default_": {
      "_all": { "enabled": false }
    },
    "blog": {
      "_all": { "enabled": true }
    }
  }
}
```
3. `_default_`映射也是定义索引级别的动态模板的好地方

#### 13. 重新索引数据
1. 虽然你可以给索引添加新的类型，或给类型添加新的字段，但是你不能添加新的分析器或修改已有字段。假如你这样做，已被索引的数据会变得不正确而你的搜索也不会正常工作
2. 修改在已存在的数据最简单的方法是重新索引：创建一个新配置好的索引，然后将所有的文档从旧的索引复制到新的上
3. `_source`字段的一个最大的好处是你已经在Elasticsearch中有了完整的文档，你不再需要从数据库中重建你的索引，这样通常会比较慢
4. 为了更高效的索引旧索引中的文档，使用`scan-scoll`来批量读取旧索引的文档，然后将通过`bulk API`来将它们推送给新的索引
```
GET /old_index/_search?search_type=scan&scroll=1m
{
  "query": {
    "range": {
      "date": {
        "gte": "2014-01-01",
        "lt": "2014-02-01"
      }
    }
  },
  "size": 1000
}
```

#### 14. 索引别名
1. 前面提到的重新索引过程中的问题是必须更新你的应用，来使用另一个索引名。索引别名正是用来解决这个问题的
2. 有两种管理别名的途径：`_alias`用于单个操作，`_aliases`用于原子化多个操作
3. 创建一个索引`my_index_v1`，然后将别名`my_index`指向它
```
PUT /my_index_v1
PUT /my_index_v1/_alias/my_index
```
4. 检测这个别名指向哪个索引，或哪些别名指向这个索引
```
GET /*/_alias/my_index
GET /my_index_v1/_alias/*
```
5. 两者都将返回下列值
```
{
  "my_index_v1" : {
    "aliases" : {
      "my_index" : { }
    }
  }
}
```
6. 然后，我们决定修改索引中一个字段的映射。当然我们不能修改现存的映射，索引我们需要重新索引数据。首先，我们创建有新的映射的索引`my_index_v2`
```
PUT /my_index_v2
{
  "mappings": {
    "my_type": {
      "properties": {
        "tags": {
          "type": "string",
          "index": "not_analyzed"
        }
      }
    }
  }
}
```
7. 然后我们从将数据从`my_index_v1`迁移到`my_index_v2`(原子化操作)
```
POST /_aliases
{
  "actions": [
    { "remove": { "index": "my_index_v1", "alias": "my_index" }},
    { "add": { "index": "my_index_v2", "alias": "my_index" }}
  ]
}
```
8. 在应用中使用别名而不是索引。然后你就可以在任何时候重建索引。别名的开销很小，应当广泛使用

### 十、分片内部原理

#### 1. 使文本可被搜索
+ 传统的数据库每个字段存储单个值，但这对全文检索并不够。文本字段中的每个单词需要被搜索，对数据库意味着需要单个字段有索引多个值的能力(单词)
+ 最好的支持一个字段多个值需求的数据结构是倒排索引(inverted-index)。倒排索引包含一个有序列表，列表包含所有文档出现过的不重复词项，对于每一个词项，包含了它所有曾出现过文档的列表
+ 早期的全文检索会为整个文档集合建立一个很大的倒排索引并将其写入到磁盘。一旦新的索引就绪，旧的就会被其替换，这样最近的变化便可以被检索到

#### 2. 不变性
+ 倒排索引被写入磁盘后是不可改变的，它永远不会修改，不变性有重要的价值
    1. 不需要锁。如果你从来不更新索引，你就不需要担心多进程同时修改数据的问题
    2. 一旦索引被读入内核的文件系统缓存，便会留在哪里，由于其不变性。只要文件系统缓存中还有足够的空间，那么大部分读请求会直接请求内存，而不会命中磁盘。这提供了很大的性能提升
    3. 其它缓存(像filter缓存)，在索引的生命周期内始终有效。它们不需要在每次数据改变时被重建，因为数据不会变化
    4. 写入单个大的倒排索引允许数据被压缩，减少磁盘I/O和需要被缓存到内存的索引的使用量
+ 一个不变的索引也有不好的地方。如果你需要让一个新的文档可被搜索，你需要重建整个索引。这要么对一个索引所能包含的数据量造成了很大的限制，要么对索引可被更新的频率造成了很大的限制

#### 3. 动态更新索引
+ 通过增加新的补充索引来反映新近的修改，而不是直接重写整个倒排索引。每一个倒排索引都会被轮流查询到，从最早的开始，​查询完后再对结果进行合并
+ Elasticsearch基于Lucene，一个Lucene索引我们在Elasticsearch称作分片，一个Elasticsearch索引是分片的集合。一个Lucene索引包含一个提交点和三个段，Lucene这个java库引入了按段搜索的概念，每一段本身都是一个倒排索引，但索引在Lucene中除表示所有段的集合外，还增加了一个列出了所有已知段的文件(提交点)
+ 当一个查询被触发，所有已知的段按顺序被查询。词项统计会对所有段的结果进行聚合，以保证每个词和每个文档的关联都被准确计算。这种方式可以用相对较低的成本将新文档添加到索引
+ 段是不可改变的，所以既不能从把文档从旧的段中移除，也不能修改旧的段来进行反映文档的更新。取而代之的是，每个提交点会包含一个`.del`文件，文件中会列出这些被删除文档的段信息
+ 当一个文档被删除或被更新时(相当于旧版本文档被删除)实际上只是在`.del`文件中被标记删除。一个被标记删除的文档仍然可以被查询匹配到，但它会在最终结果被返回前从结果集中移除(在后续操作中被标记删除的文档最终会被系统移除)

#### 4. 近实时搜索
+ 新文档在几分钟之内即可被检索，但这样还是不够快，磁盘在这里成为了瓶颈，我们需要的是一个更轻量的方式来使一个文档可被搜索
+ 在Elasticsearch和磁盘之间是文件系统缓存。在内存索引缓冲区(在内存缓冲区中包含了新文档的Lucene索引)中的文档会被写入到一个新的段中(缓冲区的内容已经被写入一个可被搜索的段中，但还没有进行提交)
+ 新段会被先写入到文件系统缓存(这一步代价会比较低)，稍后再被刷新到磁盘(​这一步代价比较高)。不过只要文件已经在缓存中，就可以像其它文件一样被打开和读取了
+ 在Elasticsearch中，写入和打开一个新段的轻量的过程叫做`refresh`。默认情况下每个分片会每秒自动刷新一次。这就是为什么我们说Elasticsearch是近实时搜索(文档的变化并不是立即对搜索可见，但会在一秒之内变为可见)
+ 尽管刷新是比提交轻量很多的操作，它还是会有性能开销。当写测试的时候，手动刷新很有用，但是不要在生产环境下每次索引一个文档都去手动刷新。相反，你的应用需要意识到Elasticsearch的近实时的性质，并接受它的不足
1. 手动刷新所有的索引 `POST /_refresh`
2. 手动刷新`blogs`索引 `POST /blogs/_refresh`
3. 设置自动刷新的时间间隔(`refresh_interval`需要一个持续时间值，例如`1s`或`2m`。一个绝对值`1`表示的是1毫秒，无疑会使你的集群陷入瘫痪)  
```
PUT /my_logs
{
  "settings": {
    "refresh_interval": "30s"
  }
}
```
4. 关闭自动刷新(在生产环境中，当你正在建立一个大的新索引时，可以先关闭自动刷新，待开始使用该索引时，再把它们调回来)  
```
PUT /my_logs/_settings
{ "refresh_interval": -1 }
```

#### 5. 持久化变更
+ 如果没有用fsync把数据从文件系统缓存刷(flush)到硬盘，我们不能保证数据在断电甚至是程序正常退出之后依然存在。为了保证Elasticsearch的可靠性，需要确保数据变化被持久化到磁盘
+ 即使通过每秒刷新(refresh)实现了近实时搜索，我们仍然需要经常进行完整提交来确保能从失败中恢复。我们也不希望丢失掉两次提交之间发生变化的文档数据
+ Elasticsearch增加了一个translog，或者叫事务日志，在每一次对Elasticsearch进行操作时均进行了日志记录。通过translog，整个流程看起来是下面这样：
1. 一个文档被索引之后，就会被添加到内存缓冲区，并且追加到了translog
2. 刷新(refresh)使分片处于刷新(refresh)完成后，缓存被清空，但是事务日志不会
3. 这个进程继续工作，更多的文档被添加到内存缓冲区和追加到事务日志
4. 分片每30分钟被自动刷新(flush)，或者在translog太大的时候也会刷新。一个新的translog被创建，并且一个全量提交被执行
+ translog提供所有还没有被刷到磁盘的操作的一个持久化纪录。当Elasticsearch启动的时候，它会从磁盘中使用最后一个提交点去恢复已知的段，并且会重放translog中所有在最后一次提交后发生的变更操作
+ translog也被用来提供实时CRUD。当你试着通过ID查询、更新、删除一个文档，它会在尝试从相应的段中检索之前，首先检查translog任何最近的变更。这意味着它总是能够实时地获取到文档的最新版本
1. 手动刷新`blogs`索引 `POST /blogs/_flush`
2. 手动刷新所有索引，并在其完成后返回结果 `POST /_flush?wait_for_ongoing`
3. 设置translog的fsync间隔(提升一些性能，但是有丢失几秒数据的风险)(默认每次写请求完成后都fsync，默认参数`"index.translog.durability": "request"`)
```
PUT /my_index/_settings
{
  "index.translog.durability": "async",
  "index.translog.sync_interval": "5s"
}
```

#### 6. 段合并
+ 由于自动刷新流程每秒会创建一个新的段，这样会导致短时间内的段数量暴增。而段数目太多会带来较大的麻烦(每一个段都会消耗文件句柄、内存和cpu运行周期)，每个搜索请求都必须轮流检查每个段，段越多，搜索也就越慢
+ Elasticsearch通过在后台进行段合并来解决这个问题。小的段被合并到大的段，然后这些大的段再被合并到更大的段。段合并的时候会将那些旧的已删除文档从文件系统中清除。被删除的文档(或被更新文档的旧版本)不会被拷贝到新的大段中
+ 启动段合并不需要你做任何事，进行索引和搜索时会自动进行。合并大的段需要消耗大量的I/O和CPU资源，如果任其发展会影响搜索性能。Elasticsearch在默认情况下会对合并流程进行资源限制，所以搜索仍然有足够的资源很好地执行
+ 手动合并索引为一个段(可能会消耗很多资源使节点不能正常使用) `POST /logstash-2014-10/_optimize?max_num_segments=1`