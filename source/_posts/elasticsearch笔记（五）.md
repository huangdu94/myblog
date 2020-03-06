---
title: Elasticsearch笔记（五）
date: 2020/02/26
updated: 2020/02/26
comments: true
categories: 
- [笔记, database, elasticsearch]
tags: 
- elasticsearch
- 非关系型数据库
---
## Elasticsearch笔记（五）

### 十三、多字段搜索

#### 1. 多字符串查询
1. 最简单的多字段查询可以将搜索项映射到具体的字段。例如`War and Peace`是标题，`Leo Tolstoy`是作者，很容易就能把两个条件用`match`语句表示，并将它们用`bool`查询组合起来
```
GET /_search
{
  "query": {
    "bool": {
      "should": [
        { "match": { "title":  "War and Peace" }},
        { "match": { "author": "Leo Tolstoy"   }}
      ]
    }
  }
}
```
2. `bool`查询采取匹配越多越好的方式，所以每条`match`语句的评分结果会被加在一起，从而为每个文档提供最终的分数`_score`。能与两条语句同时匹配的文档比只与一条语句匹配的文档得分要高
3. 并不是只能使用`match`语句，也可以用`bool`查询来包裹组合任意其他类型的查询，甚至包括其他的`bool`查询。可以在上面的示例中添加一条语句来指定译者版本的偏好
```
GET /_search
{
  "query": {
    "bool": {
      "should": [
        { "match": { "title":  "War and Peace" }},
        { "match": { "author": "Leo Tolstoy"   }},
        { "bool":  {
          "should": [
            { "match": { "translator": "Constance Garnett" }},
            { "match": { "translator": "Louise Maude"      }}
          ]
        }}
      ]
    }
  }
}
```
4. `bool`查询运行每个`match`查询，再把评分加在一起，然后将结果与所有匹配的语句数量相乘，最后除以所有的语句数量。处于同一层的每条语句具有相同的权重。在前面这个例子中，包含`translator`语句的`bool`查询，只占总评分的三分之一。如果将`translator`语句与`title`和`author`两条语句放入同一层，那么`title`和`author`语句只贡献四分之一评分
5. 可以使用`boost`参数提升`title`和`author`字段的权重，为它们分配的`boost`值大于1
```
GET /_search
{
  "query": {
    "bool": {
      "should": [
        { "match": {
            "title":  {
              "query": "War and Peace",
              "boost": 2
        }}},
        { "match": {
            "author":  {
              "query": "Leo Tolstoy",
              "boost": 2
        }}},
        { "bool":  {
            "should": [
              { "match": { "translator": "Constance Garnett" }},
              { "match": { "translator": "Louise Maude"      }}
          ]
        }}
      ]
    }
  }
}
```
6. `title`和`author`语句的`boost`值为2，嵌套`bool`语句默认的`boost`值为1
7. 要获取`boost`参数“最佳”值，较为简单的方式就是不断试错。设定`boost`值，运行测试查询，如此反复。`boost`值比较合理的区间处于1到10之间，当然也有可能是15。如果为`boost`指定比这更高的值，将不会对最终的评分结果产生更大影响，因为评分是被归一化的(normalized)

#### 2. 单字符串查询
1. `bool`查询是多语句查询的主干。它的适用场景很多，特别是当需要将不同查询字符串映射到不同字段的时候
2. 有些用户期望将所有的搜索项堆积到单个字段中，并期望应用程序能为他们提供正确的结果。对于多词(multiword)、多字段(multifield)查询来说，不存在简单的万能方案。为了获得最好结果，需要了解我们的数据，并了解如何使用合适的工具
3. 当用户输入了单个字符串查询的时候，通常会遇到以下三种情形
    1. 最佳字段  
    当搜索词语具体概念的时候，比如“brown fox”，词组比各自独立的单词更有意义。像`title`和` body`这样的字段，尽管它们之间是相关的，但同时又彼此相互竞争。文档在相同字段中包含的词越多越好，评分也来自于最匹配字段
    2. 多数字段  
    为了对相关度进行微调，常用的一个技术就是将相同的数据索引到不同的字段，它们各自具有独立的分析链。主字段可能包括它们的词源、同义词以及变音词或口音词，被用来匹配尽可能多的文档。相同的文本被索引到其他字段，以提供更精确的匹配。一个字段可以包括未经词干提取过的原词，另一个字段包括其他词源、口音，还有一个字段可以提供词语相似性信息的瓦片词(shingles)。其他字段是作为匹配每个文档时提高相关度评分的信号，匹配字段越多则越好
    3. 混合字段  
    对于某些实体，我们需要在多个字段中确定其信息，单个字段都只能作为整体的一部分。在这种情况下，我们希望在任何这些列出的字段中找到尽可能多的词，这有如在一个大字段中进行搜索，这个大字段包括了所有列出的字段
    ```
    Person：first_name、last_name
    Book：title、author、description
    Address：street、city、country、postcode
    ```
4. 上述所有都是多词、多字段查询，但每个具体查询都要求使用不同策略

#### 3. 最佳字段
1. 假设有个网站允许用户搜索博客的内容，以下面两篇博客内容文档为例
```
PUT /my_index/my_type/1
{
  "title": "Quick brown rabbits",
  "body":  "Brown rabbits are commonly seen."
}

PUT /my_index/my_type/2
{
  "title": "Keeping pets healthy",
  "body":  "My quick brown fox eats rabbits on a regular basis."
}
```
2. 用户输入词组“Brown fox”然后点击搜索按钮。事先，我们并不知道用户的搜索项是会在`title`还是在`body`字段中被找到，但是，用户很有可能是想搜索相关的词组。用肉眼判断，文档2的匹配度更高，因为它同时包括要查找的两个词。现在运行以下`bool`查询
```
{
  "query": {
    "bool": {
      "should": [
        { "match": { "title": "Brown fox" }},
        { "match": { "body":  "Brown fox" }}
      ]
    }
  }
}
```
3. 查询的结果是文档1的评分更高。文档1的两个字段都包含`brown`这个词，所以两个`match`语句都能成功匹配并且有一个评分。文档2的`body`字段同时包含`brown`和`fox`这两个词，但`title`字段没有包含任何词。这样，`body`查询结果中的高分，加上`title`查询中的0分，然后乘以二分之一，就得到比文档1更低的整体评分
4. `title`和`body`字段是相互竞争的关系，所以就需要找到单个最佳匹配的字段(将最佳匹配字段的评分作为查询的整体评分)
5. 不使用`bool`查询，可以使用`dis_max`即分离最大化查询(Disjunction Max Query)。分离(Disjunction)的意思是或(or)，这与可以把结合(conjunction)理解成与(and)相对应。分离最大化查询(Disjunction Max Query)指的是：将任何与任一查询匹配的文档作为结果返回，但只将最佳匹配的评分作为查询的评分结果返回
```
{
  "query": {
    "dis_max": {
      "queries": [
        { "match": { "title": "Brown fox" }},
        { "match": { "body":  "Brown fox" }}
      ]
    }
  }
}
```
6. 得到我们想要的结果，文档2的评分更高

#### 4. 最佳字段查询调优
1. 当用户搜索“quick pets”时会发生什么呢？在前面的例子中，两个文档都包含词quick，但是只有文档2包含词pets，两个文档中都不具有同时包含两个词的相同字段
2. 一个简单的`dis_max`查询会采用单个最佳匹配字段，而忽略其他的匹配
```
{
  "query": {
    "dis_max": {
      "queries": [
        { "match": { "title": "Quick pets" }},
        { "match": { "body":  "Quick pets" }}
      ]
    }
  }
}
```
3. 结果是两个评分是完全相同的。我们可能期望同时匹配`title`和`body`字段的文档比只与一个字段匹配的文档的相关度更高，但事实并非如此，因为`dis_max`查询只会简单地使用单个最佳匹配语句的评分`_score`作为整体评分
4. 可以通过指定`tie_breaker`这个参数将其他匹配语句的评分也考虑其中
```
{
  "query": {
    "dis_max": {
      "queries": [
        { "match": { "title": "Quick pets" }},
        { "match": { "body":  "Quick pets" }}
      ],
      "tie_breaker": 0.3
    }
  }
}
```
5. 结果是文档2的相关度比文档1略高
6. `tie_breaker`参数提供了一种`dis_max`和`bool`之间的折中选择，它的评分方式为：获得最佳匹配语句的评分`_score`；将其他匹配语句的评分结果与`tie_breaker`相乘；对以上评分求和并规范化。有了tie_breaker，会考虑所有匹配语句，但最佳匹配语句依然占最终结果里的很大一部分
7. `tie_breaker`可以是0到1之间的浮点数，其中0代表使用`dis_max`最佳匹配语句的普通逻辑，1表示所有匹配语句同等重要。最佳的精确值需要根据数据与查询调试得出，但是合理值应该与零接近(处于0.1-0.4之间)，这样就不会颠覆`dis_max`最佳匹配性质的根本

#### 5. multi_match查询
1. `multi_match`查询为能在多个字段上反复执行相同查询提供了一种便捷方式。`multi_match`多匹配查询的类型有多种，其中的三种恰巧与了解我们的数据中介绍的三个场景对应，即：`best_fields`、`most_fields`和`cross_fields`(最佳字段、多数字段、跨字段)
2. 默认情况下，查询的类型是`best_fields`，这表示它会为每个字段生成一个`match`查询，然后将它们组合到`dis_max`查询的内部，如下
```
{
  "dis_max": {
    "queries":  [
      {
        "match": {
          "title": {
            "query": "Quick brown fox",
            "minimum_should_match": "30%"
          }
        }
      },
      {
        "match": {
          "body": {
            "query": "Quick brown fox",
            "minimum_should_match": "30%"
          }
        }
      },
    ],
    "tie_breaker": 0.3
  }
}
```
3. 上面这个查询用`multi_match`重写成更简洁的形式(best_fields类型是默认值，可以不指定)
```
{
  "multi_match": {
    "query":                "Quick brown fox",
    "type":                 "best_fields",
    "fields":               [ "title", "body" ],
    "tie_breaker":          0.3,
    "minimum_should_match": "30%"
  }
}
```
4. 字段名称可以用模糊匹配的方式给出：任何与模糊模式正则匹配的字段都会被包括在搜索条件中，例如可以使用以下方式同时匹配`book_title`、`chapter_title`和`section_title`(书名、章名、节名)这三个字段
```
{
  "multi_match": {
    "query":  "Quick brown fox",
    "fields": "*_title"
  }
}
```
5. 可以使用`^`字符语法为单个字段提升权重，在字段名称的末尾添加`^boost`，其中`boost`是一个浮点数(`chapter_title`这个字段的`boost`值为2，而其他两个字段`book_title`和`section_title`字段的默认`boost`值为1)
```
{
  "multi_match": {
    "query":  "Quick brown fox",
    "fields": [ "*_title", "chapter_title^2" ]
  }
}
```

#### 6. 多数字段
1. 全文搜索被称作是召回率(Recall,返回所有的相关文档)与精确率(Precision,不返回无关文档)的战场，目的是在结果的第一页中为用户呈现最为相关的文档。为了提高召回率的效果，我们扩大搜索范围，不仅返回与用户搜索词精确匹配的文档，还会返回我们认为与查询相关的所有文档
2. 如果一个用户搜索“quick brown box”，一个包含词语fast foxes的文档被认为是非常合理的返回结果。如果包含词语fast foxes的文档是能找到的唯一相关文档，那么它会出现在结果列表的最上面，但是，如果有100个文档都出现了词语quick brown fox，那么这个包含词语fast foxes的文档当然会被认为是次相关的，它可能处于返回结果列表更下面的某个地方。当包含了很多潜在匹配之后，我们需要将最匹配的几个置于结果列表的顶部
3. 提高全文相关性精度的常用方式是为同一文本建立多种方式的索引，每种方式都提供了一个不同的相关度信号signal。主字段会以尽可能多的形式的去匹配尽可能多的文档。举个例子，我们可以进行以下操作：
	1. 使用词干提取来索引jumps、jumping和jumped样的词，将jump作为它们的词根形式。这样即使用户搜索jumped，也还是能找到包含jumping的匹配的文档
	2. 将同义词包括其中，如jump、leap和hop
	3. 移除变音或口音词：如ésta、está和esta都会以无变音形式esta来索引
4. 尽管如此，如果我们有两个文档，其中一个包含词jumped，另一个包含词jumping，用户很可能期望前者能排的更高，因为它正好与输入的搜索条件一致。为了达到目的，我们可以将相同的文本索引到其他字段从而提供更为精确的匹配。一个字段可能是为词干未提取过的版本，另一个字段可能是变音过的原始词，第三个可能使用shingles提供词语相似性信息。这些附加的字段可以看成提高每个文档的相关度评分的信号signals，能匹配字段的越多越好
5. 一个文档如果与广度匹配的主字段相匹配，那么它会出现在结果列表中。如果文档同时又与signal信号字段匹配，那么它会获得额外加分，系统会提升它在结果列表中的位置
6. 稍后对同义词、词相似性、部分匹配以及其他潜在的信号进行讨论，但这里只使用词干已提取(stemmed)和未提取(unstemmed)的字段作为简单例子来说明这种技术
7. 首先要做的事情就是对我们的字段索引两次：一次使用词干模式以及一次非词干模式。为了做到这点，采用`multifields`来实现，已经在`multifields`有所介绍
```
DELETE /my_index
PUT /my_index
{
  "settings": { "number_of_shards": 1 },
  "mappings": {
    "my_type": {
      "properties": {
        "title": {
          "type":     "string",
          "analyzer": "english",
          "fields": {
            "std":   {
              "type":     "string",
              "analyzer": "standard"
            }
          }
        }
      }
    }
  }
}
```
8. 接着索引一些文档
```
PUT /my_index/my_type/1
{ "title": "My rabbit jumps" }

PUT /my_index/my_type/2
{ "title": "Jumping jack rabbits" }
```
9. 这里用一个简单`match`查询`title`标题字段是否包含jumping rabbits
```
GET /my_index/_search
{
  "query": {
    "match": {
      "title": "jumping rabbits"
    }
  }
}
```
10. 因为有了english分析器，这个查询是在查找以jump和rabbit这两个被提取词的文档。两个文档的`title`字段都同时包括这两个词，所以两个文档得到的评分也相同
11. 如果只是查询`title.std`字段，那么只有文档2是匹配的。尽管如此，如果同时查询两个字段，然后使用bool查询将评分结果合并，那么两个文档都是匹配的(`title`字段的作用)，而且文档2的相关度评分更高(`title.std`字段的作用)
```
GET /my_index/_search
{
  "query": {
    "multi_match": {
      "query":  "jumping rabbits",
        "type":   "most_fields", (1)
        "fields": [ "title", "title.std" ]
    }
  }
}
```
12. 我们希望将所有匹配字段的评分合并起来，所以使用`most_fields`类型。这让`multi_match`查询用`bool`查询将两个字段语句包在里面，而不是使用`dis_max`查询
13. 结果文档2现在的评分要比文档1高。用广度匹配字段`title`包括尽可能多的文档以提升召回率，同时又使用字段`title.std`作为信号将相关度更高的文档置于结果顶部
14. 每个字段对于最终评分的贡献可以通过自定义值`boost`来控制。比如，使`title`字段更为重要，这样同时也降低了其他信号字段的作用
```
GET /my_index/_search
{
  "query": {
    "multi_match": {
      "query":       "jumping rabbits",
      "type":        "most_fields",
      "fields":      [ "title^10", "title.std" ] (1)
    }
  }
}
```

#### 7. 跨字段实体搜索
1. 现在讨论一种普遍的搜索模式：跨字段实体搜索(cross-fields entity search)。在如person、 product或address这样的实体中，需要使用多个字段来唯一标识它的信息
```
{
  "firstname":  "Peter",
  "lastname":   "Smith"
}
{
  "street":   "5 Poland Street",
  "city":     "London",
  "country":  "United Kingdom",
  "postcode": "W1V 3DG"
}
```
2. 这与之前描述的多字符串查询很像，但这存在着巨大的区别。在多字符串查询中，我们为每个字段使用不同的字符串，在本例中，我们想使用单个字符串在多个字段中进行搜索。我们的用户可能想搜索“Peter Smith”这个人，或“Poland Street W1V”这个地址，这些词出现在不同的字段中，所以如果使用`dis_max`或`best_fields`查询去查找单个最佳匹配字段显然是个错误的方式
3. 依次查询每个字段并将每个字段的匹配评分结果相加，听起来真像是`bool`查询
```
{
  "query": {
    "bool": {
      "should": [
        { "match": { "street":    "Poland Street W1V" }},
        { "match": { "city":      "Poland Street W1V" }},
        { "match": { "country":   "Poland Street W1V" }},
        { "match": { "postcode":  "Poland Street W1V" }}
      ]
    }
  }
}
```
4. 为每个字段重复查询字符串会使查询瞬间变得冗长，可以采用`multi_match`查询，将`type`设置成`most_fields`然后告诉Elasticsearch合并所有匹配字段的评分
```
{
  "query": {
    "multi_match": {
      "query":       "Poland Street W1V",
      "type":        "most_fields",
      "fields":      [ "street", "city", "country", "postcode" ]
    }
  }
}
```
5. 用most_fields这种方式搜索也存在某些问题，这些问题并不会马上显现
	1. 它是为多数字段匹配任意词设计的，而不是在所有字段中找到最匹配的；
	2. 它不能使用`operator`或`minimum_should_match`参数来降低次相关结果造成的长尾效应；
	3. 词频对于每个字段是不一样的，而且它们之间的相互影响会导致不好的排序结果

#### 8. 字段中心式查询
1. 以上三个源于`most_fields`的问题都因为它是字段中心式(field-centric)而不是词中心式(term-centric)的：当真正感兴趣的是匹配词的时候，它为我们查找的是最匹配的字段。best_fields类型也是字段中心式的，它也存在类似的问题
2. 首先查看这些问题存在的原因，再想如何解决它们
	1. 问题1：在多个字段中匹配相同的词  
	`most_fields`查询为每个字段生成独立的`match`查询，再用`bool`查询将他们包起来。可以通过`validate-queryAPI`查看。可以发现，两个字段都与poland匹配的文档要比一个字段同时匹配poland与street文档的评分高。
	```
	GET /_validate/query?explain
	{
	  "query": {
		"multi_match": {
		  "query":   "Poland Street W1V",
		  "type":    "most_fields",
		  "fields":  [ "street", "city", "country", "postcode" ]
		}
	  }
	}
	```
	```
	(street:poland   street:street   street:w1v)
	(city:poland     city:street     city:w1v)
	(country:poland  country:street  country:w1v)
	(postcode:poland postcode:street postcode:w1v)
	```
	2. 问题2：剪掉长尾  
	在匹配精度中，我们讨论过使用`and`操作符或设置`minimum_should_match`参数来消除结果中几乎不相关的长尾，或许可以尝试以下方式。换句话说，使用`and`操作符要求所有词都必须存在于相同字段，这显然是不对的！可能就不存在能与这个查询匹配的文档
	```
	{
      "query": {
		"multi_match": {
		  "query":       "Poland Street W1V",
		  "type":        "most_fields",
		  "operator":    "and",
		  "fields":      [ "street", "city", "country", "postcode" ]
		}
	  }
	}
	```
	```
	(+street:poland   +street:street   +street:w1v)
	(+city:poland     +city:street     +city:w1v)
	(+country:poland  +country:street  +country:w1v)
	(+postcode:poland +postcode:street +postcode:w1v)
	```
	3. 问题3：词频(一个词在单个文档的某个字段中出现的频率越高，这个文档的相关度就越高；一个词在所有文档某个字段索引中出现的频率越高，这个词的相关度就越低)  
	在什么是相关中，我们解释过每个词默认使用TF/IDF相似度算法计算相关度评分。当搜索多个字段时，TF/IDF会带来某些令人意外的结果。想想用字段first_name和last_name查询“Peter Smith”的例子，Peter是个平常的名Smith也是平常的姓，这两者都具有较低的IDF值。但当索引中有另外一个人的名字是“Smith Williams”时，Smith作为名来说很不平常，以致它有一个较高的IDF值！下面这个简单的查询可能会在结果中将“Smith Williams”置于“Peter Smith”之上，尽管事实上是第二个人比第一个人更为匹配
	```
	{
	  "query": {
		"multi_match": {
  		  "query":       "Peter Smith",
		  "type":        "most_fields",
		  "fields":      [ "*_name" ]
		}
	  }
	}
	```
3. 存在这些问题仅仅是因为我们在处理着多个字段，如果将所有这些字段组合成单个字段，问题就会消失。可以为person文档添加full_name字段来解决这个问题
```
{
  "first_name":  "Peter",
  "last_name":   "Smith",
  "full_name":   "Peter Smith"
}
```
4. 当查询full_name字段时，具有更多匹配词的文档会比只有一个重复匹配词的文档更重要。`minimum_should_match`和`operator`参数会像期望那样工作。姓和名的逆向文档频率被合并，所以Smith到底是作为姓还是作为名出现，都会变得无关紧要。这么做当然是可行的，但我们并不太喜欢存储冗余数据。取而代之的是Elasticsearch可以提供两个解决方案(一个在索引时，而另一个是在搜索时)

#### 9. 自定义_all字段
1. 在all-field字段中，我们解释过`_all`字段的索引方式是将所有其他字段的值作为一个大字符串索引的。然而这么做并不十分灵活，为了灵活我们可以给人名添加一个自定义`_all`字段，再为地址添加另一个`_all`字段。Elasticsearch在字段映射中为我们提供`copy_to`参数来实现这个功能
```
PUT /my_index
{
  "mappings": {
    "person": {
      "properties": {
        "first_name": {
          "type":     "string",
          "copy_to":  "full_name"
        },
        "last_name": {
          "type":     "string",
          "copy_to":  "full_name"
        },
        "full_name": {
          "type":     "string"
        }
      }
    }
  }
}
```
2. `first_name`和`last_name`字段中的值会被复制到`full_name`字段。有了这个映射，我们可以用`first_name`来查询名，用`last_name`来查询姓，或者直接使用`full_name`查询整个姓名。`first_name`和`last_name`的映射并不影响`full_name`如何被索引，`full_name`将两个字段的内容复制到本地，然后根据`full_name`的映射自行索引
3. `copy_to`设置对`multi-field`无效。如果尝试这样配置映射，Elasticsearch会抛异常。多字段只是以不同方式简单索引“主”字段；它们没有自己的数据源。也就是说没有可供`copy_to`到另一字段的数据源。只要对“主”字段`copy_to`就能轻而易举的达到相同的效果
```
PUT /my_index
{
  "mappings": {
    "person": {
      "properties": {
        "first_name": {
          "type":     "string",
          "copy_to":  "full_name",
          "fields": {
            "raw": {
              "type": "string",
              "index": "not_analyzed"
            }
          }
        },
        "full_name": {
            "type":     "string"
        }
      }
    }
  }
}
```

#### 10. cross-fields跨字段查询
1. 自定义`all`的方式是一个好的解决方案，只需在索引文档前为其设置好映射。不过，Elasticsearch还在搜索时提供了相应的解决方案：使用`cross_fields`类型进行`multi_match`查询。`cross_fields`使用词中心式(term-centric)的查询方式，这与`best_fields`和`most_fields`使用字段中心式(field-centric)的查询方式非常不同，它将所有字段当成一个大字段，并在_每个字段中查找每个词
2. 为了说明字段中心式(field-centric)与词中心式(term-centric)这两种查询方式的不同，先看看以下字段中心式的`most_fields`查询的`explanation`解释
```
GET /_validate/query?explain
{
  "query": {
    "multi_match": {
      "query":       "peter smith",
      "type":        "most_fields",
      "operator":    "and",
      "fields":      [ "first_name", "last_name" ]
    }
  }
}
```
```
(+first_name:peter +first_name:smith)
(+last_name:peter  +last_name:smith)
```
3. 词中心式会使用以下逻辑(词peter和smith都必须出现，但是可以出现在任意字段中)
```
+(first_name:peter last_name:peter)
+(first_name:smith last_name:smith)
```
4. `cross_fields`类型首先分析查询字符串并生成一个词列表，然后它从所有字段中依次搜索每个词。这种不同的搜索方式很自然的解决了字段中心式查询三个问题中的二个。剩下的问题是逆向文档频率不同。幸运的是`cross_fields`类型也能解决这个问题，通过`validate-query`可以看到
```
GET /_validate/query?explain
{
  "query": {
    "multi_match": {
      "query":       "peter smith",
      "type":        "cross_fields",
      "operator":    "and",
      "fields":      [ "first_name", "last_name" ]
    }
  }
}
```
```
+blended("peter", fields: [first_name, last_name])
+blended("smith", fields: [first_name, last_name])
```
5. 换句话说，它会同时在first_name和last_name两个字段中查找smith的IDF，然后用两者的最小值作为两个字段的IDF。结果实际上就是smith会被认为既是个平常的姓，也是平常的名
6. 为了让`cross_fields`查询以最优方式工作，所有的字段都须使用相同的分析器，具有相同分析器的字段会被分组在一起作为混合字段使用
7. 如果包括了不同分析链的字段，它们会以`best_fields`的相同方式被加入到查询结果中。例如：我们将`title`字段加到之前的查询中(假设它们使用的是不同的分析器)，explanation的解释结果如下
```
(+title:peter +title:smith)
(
  +blended("peter", fields: [first_name, last_name])
  +blended("smith", fields: [first_name, last_name])
)
```
8. 采用`cross_fields`查询与自定义`_all`字段相比，其中一个优势就是它可以在搜索时为单个字段提升权重。这对像`first_name`和`last_name`具有相同值的字段并不是必须的，但如果要用`title`和`description`字段搜索图书，可能希望为`title`分配更多的权重，这同样可以使用前面介绍过的`^`符号语法来实现
```
GET /books/_search
{
  "query": {
    "multi_match": {
      "query":       "peter smith",
      "type":        "cross_fields",
      "fields":      [ "title^2", "description" ]
    }
  }
}
```
9. 自定义单字段查询是否能够优于多字段查询，取决于在多字段查询与单字段自定义`_all`之间代价的权衡，即哪种解决方案会带来更大的性能优化就选择哪一种

#### 11. Exact-Value精确值字段
1. 在结束多字段查询这个话题之前，我们最后要讨论的是精确值`not_analyzed`未分析字段。将`not_analyzed`字段与`multi_match`中`analyzed`字段混在一起没有多大用处。原因可以通过查看查询的`explanation`解释得到，设想将`title`字段设置成`not_analyzed`
```
GET /_validate/query?explain
{
  "query": {
    "multi_match": {
      "query":       "peter smith",
      "type":        "cross_fields",
      "fields":      [ "title", "first_name", "last_name" ]
    }
  }
}
```
2. 因为`title`字段是未分析过的，Elasticsearch会将“petersmith”这个完整的字符串作为查询条件来搜索
```
title:peter smith
(
  blended("peter", fields: [first_name, last_name])
  blended("smith", fields: [first_name, last_name])
)
```
3. 显然这个项不在`title`的倒排索引中，所以需要在`multi_match`查询中避免使用`not_analyzed`字段