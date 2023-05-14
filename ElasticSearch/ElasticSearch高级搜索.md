# 结构化搜索

## term filter

constant_score : 将所有的document 相关度设置为1, 其中包含query 或 filter语句, TF排序算法的相关度问题, 只获取数据.

term 在 filter/query 语法中 对搜索文本不进行分词, 直接进入倒排索引中进行匹配,  输入什么就进行匹配什么. 对于 时间 bool 数字1 这些不需要进行分词的最好用.

```
GET /index/type/_search
{
    "query" : {
        "constant_score" : {
            "filter" : {
                "term" : {
                    "userID":1
                }
            }
        }
    }
}
```


当 查看分词时
```
GET /index/_analyze
{
    "field" : "articleId",
    "text" : "XHDK-A-1293-#fJ3"
}
```
XHDK-A-1293-#fJ3 已经被分为 XHDK A 1293 #fJ3 等 倒排索引了 原本的 articleId 已经没有了 term是不对搜索文本分词的 也就是 原本整个字符串,  所以是无法匹配到

所以 当不需要索引的时候就不要建立分词器
重建索引

DELETE /index

PUT /index
{
    "mapping":{
        "article": {
            "properties" : {
                articleID : {
                    "type" : "string",
                    "index" : "non_analyzed"
                }
            }
        }
    }
}

(1). term query 根据exact value 进行搜索, 数字, boolean, date 天然支持
(2). text需要建索引时指定为 not_analyzed 才能用term query
(3). 相当于SQL中当个where条件


## _filter 执行原理和bitset机制与caching机制


eg :

| word | doc1 | doc2 | doc3 |
| :---: | :---: | :---: | :---: |
| 2017-01-01 | * | * |  |
| 2017-02-02 |  | * | * |
| 2017-03-03 | * | * | * |

1. 当使用filter过滤2017-02-02时, 找到的document list 是doc2 和 doc3
2. 使用找到的document list 构建一个bitset, 一个二进制数组, 数组每个元素都是0 或 1, 用来标识一个doc对一个filter条件是否匹配, 如果匹配标识为1 , 不匹配标识为0, eg: (在filter过滤时, doc2 和 doc3 包含那么数组就类似 [0,1,1])
3. 尽可能用简单的数据结构实现复杂的功能, 以节省内存空间提高性能
4. 一次search 请求中可以发出多个filter条件, 每个filter条件都会对应一个bitset, 遍历每一个filter条件对应的bitset, 从最稀疏的bitset开始遍历 eg [0, 0, 1] 就比 [0, 1, 1] 稀疏, 就会过滤掉尽可能多的数据 , 找到匹配所有filter条件的doc
5. 当每一个指定的filter的bitset会缓存在内存中, 当下次拥有同样条件的请求过来时, 就不用重新扫描倒排索引, 反复生成bitset, 可以大幅提高filter的性能.
6. caching机制, 就是当某个filter超过了一定次数, 次数不固定, 就会自动缓存这个filter所对应的bitset,
7. filter 对于小型的segment获取到的结果, 可以不缓存, segment记录数 < 1000 或者segment大小 < index 总大小的 %3. 因为segment数据量很小的时候, 就算全局扫描也不会效率低, 小型的segment会在指定时间合并为大segment, 所以价值很低.
8. filter 相比于query 在于会进行bitset caching 缓存,  如果document 有新增或修改, 那么cached bitset 会被自动更新, 缓存自动更新.
9. 如果以后有相同的filter条件,会直接来使用对应的cache bitset.

## 基于bool组合多个filter条件进行搜索数据

如 : 发帖日期为 2017-01-01, 或 帖子id 为 XHDK-A-123的帖子, 同时要求帖子日期不为2017-01-02

```
GET /forum/article/_search
{
    "query" : {
        "constant_socre" : {
            "filter" : {
                "bool" : {
                    "should" : [
                        {"term" : {"postDate" : "2017-01-01"}},
                        {"term" : {"articelID" : "XHDK-A-123"}}
                    ], 
                    "must_not" : {
                        "term" : {"postDate" : "2017-01-02"}
                    }
                }
            }
        }
    }
}
```

bool 可以进行组合嵌套 里面可组合条件为 must, should, must_not, filter;  
`must` 表示必须匹配;
 `should`  表示匹配其中任意一个即可;
 `must_not` 表示必须不匹配;
 `filter` 表示数据过滤;


## 使用terms 搜索多个值以及搜索结果优化

```
GET /index/type/_search
{
    "query" : {
        "constant_score" : {
            "filter": {
                "terms" : {
                    articleID: [
                        "XHDK-A-123",
                        "XHDK-A-223"
                    ]
                }
            }
        }
    }
}
```
使用terms 进行多个值搜索

如果数据是数组模式,多个值, 那么对应必须指定值的数量 tag_cnt 数量 和 名称 tag
eg: 必须满足条件为 条件个数只有一条, 而且必须tag名称为 java
```
{
    "query" : {
        "constant_score" : {
            "filter" : {
                "bool" : {
                    "must" : [
                        {
                            "term" : {
                                "tag_cnt":1
                            }
                        },
                        {
                            "terms" : {
                                "tag" : ["java"]
                            }
                        }
                    ]
                }
            }
        }
    }
}
```

## 使用range_filter 来进行范围过滤

view_cnt字段大于30小于60
```
GET /index/type/_serach
{
     "query" : {
         "constant_score" : {
             "filter" : {
                 "range" : {
                     "view_cnt" : {
                         "gt" :30,
                         "lt" : 60
                     }
                 }
             }
         }
     }
}
```

日期的范围查询 当前时间往前推一个月 就是 now-1m 简写
```
GET index/type/_search
{
    "query" : {
        "constant_score" : {
            "filter" : {
                "range" : {
                    "postDate" : {
                        "gt" : "now-1m";
                    }
                }
            }
        }
    }
}

 或另一种日期选择模式 gt : "2017-03-10||-30d"
```


## 深度搜索技术 手动控制全文索引的精准度
在text文档中搜索包含java或elasticsearch的blog
通过空格进行分词搜索
```
GET /index/type/_search
{
    "query" : {
        "match" : {
            "title" : "java elsticsearch"
        }
    }
}
```
包含java并且包含elasticsearch的blog
使用and关键词, 所有关键词都匹配的那么就用and.

```
GET /index/type/_search
{
    "query" : {
        "match" : {
            "title" :  {
                "query" : "java elasticsearch",
                "operator" : "and"
            }
        }
    }
}
```

包含关键字java, elasticSearch, spark, hadoop 4个关键字中至少三个才行
```
GET /index/type/_search
{
    "query" : {
        "match" : {
            "title" : {
                "query" : "java elasticsearch spark hadoop",
                "minimum_should_match" : "75%"
            }
        }
    }
}
```

使用bool 组合多个值搜索,  title必须包含 java 必须不包含spark 可以包含hadoop 或 elasticsearch
```
GET /index/type/_search
{
    "query" : {
        "bool" : {
            "must" : {
                "match" : {"title" : "java"}
            },
            "must_not" : {
                "match" : {"title" : "spark"}
            },
            "should" : [
                {"match" : {"title" : "hadoop"}},
                {"match" : {"title" : "elasticsearch"}}
            ]
        }
    }
}
```

精准到至少包含了其中3个关键字
默认情况下只要包含一个就可以返回就是true, should可以精准控制, 到3个或指定个数
```
GET /index/type/_search
{
    "query" : {
        "bool" : {
            "should" : [
                {"match" : {"title" : "java"}},
                {"match" : {"title": "elasticsearch"}},
                {"match" : {"title" : "hadoop"}},
                {"match" : {"title" : "spark"}},
            ],
            "minimum_should_match" : 3
        }
    }
}
```

ElasticSearch 在底层进行转换

将普通的match 转换为term + should
{
    "match" : {"title" : "java elasticsearch"}
}
es 底层就会将操作转换为 这样should 和 term 的组合
{
    "bool" : {
        "should" : [
            {"term" : {"title" : "java"}},
            {"term" : {"title" : "elasticsearch"}}
        ]
    }
}

 and 和 match 转换为 termux + must
{
    "match" : {
        "title" : {
            "query" : "java elasticsearch",
            "operator" : "and"
        }
    }
}
es 底层转换为 must + termux 的组合
{
    "bool" : {
        "must" : [
            {"term" : {"title" : "java"}},
            {"term" : {"title" : "elasticsearvh"}}
        ]
    }
}

minimum_should_match自动转换
{
    "match" : {
        "title" : {
            "query" : "java elasticsearch hadoop spark"
        }
    }
}


## 基于boos的细粒度搜索条件权重控制

搜索条件的权重, boost, 可以将某个搜索条件的权重加大, 此时, 当匹配这个搜索条件和匹配另一个搜索条件的document, 计算relevance score时, 匹配权重更大的document 会优先被返回.
默认情况下所有boost权重都为1
```
GET /index/type/_search
{
    "query" : {
        "bool" : {
            "must" : {
                "match" : {
                    "title" : "java"
                }
            },
            "should" : [
                {
                    "match" : {
                        "title" : {
                            "query":"hadoop",
                            "boost":3
                        }
                    }
                },
                {
                    "match" : {
                        "title":{
                            "query" : "elasticsearch",
                            "boost":2
                        }
                    }
                }
            ],
        }
    }
}
```

## 多shard场景下 relevance score 不准确问题大揭秘

如果一个index 有多个shard的话, 可能搜索会不准确.
每个请求会发送到所有的primy shard上

在某一个shard中, 有很多document, 包含title中有java这个关键字, 如果说有10个document包含java关键字.

当某一个搜索title包含java的请求, 发送到这个shard的时候, 应该会这个计算, relevance score, TF/IDF 算法

(1). 在一个document的title中, java出现了几次
(2). 在所有的document的title中, java 出现了几次
(3). 这个document的title长度

shard 中只是一部分document默认, 就在shard local本地计算

在另一个shard中, 只有1个document title 包含java, 此时计算shard local IDF，就会分数很高, 相关度很高.

所以 在只1个doucment 包含java的的shard local IDF 相关度分数就会很高.

结果相关度排序, 相关度较高的doc排在了教后的位置, 相关度较低的,确排在了前面.

解决方案 : 

(1). 生产环境下, 数量很大, 尽可能实现均匀分配, 在负载均衡时, 如果title都包含java, 那么会在每个shard上平均分配, 数据分配较为均匀.
(2). 测试环境下, 将索引primary shard设置为1个, number_of_shards=1, index settings 如果只有一个,就不会存在 shard 数据分配不均.


## dis_max 实现 best field 策略进行多字段搜索

添加content字段
```
POST /forum/article/_bulk
{ "update": { "_id": "1"} }
{ "doc" : {"content" : "i like to write best elasticsearch article"} }
{ "update": { "_id": "2"} }
{ "doc" : {"content" : "i think java is the best programming language"} }
{ "update": { "_id": "3"} }
{ "doc" : {"content" : "i am only an elasticsearch beginner"} }
{ "update": { "_id": "4"} }
{ "doc" : {"content" : "elasticsearch and hadoop are all very good solution, i am a beginner"} }
{ "update": { "_id": "5"} }
{ "doc" : {"content" : "spark is best big data solution based on scala ,an programming language similar to java"} }
```

搜索title 或 content中包含java或solution的帖子

```
GET /forum/article/_search
{
    "query": {
        "bool": {
            "should": [
                { "match": { "title": "java solution" }},
                { "match": { "content":  "java solution" }}
            ]
        }
    }
}
```

结果分析
希望的结果是doc 5 确获得了doc2,  doc4 在前面.

计算document的relevance score, 每个query的分数, 乘以 matched query数量, 除以总query 数量
` { "match": { "title": "java solution" }}` 针对doc4 是有一个分数的.
`{ "match": { "content": "java solution" }}` 针对doc4 也是有一个分数
所以是两个分数加起来 
eg : 
1.1 + 1.2 = 2.3
matched query 数量 = 2
总query 数量 = 2 
2.3 * 2/ 2 = 2.3

doc 5 的分数.
` { "match": { "title": "java solution" }}` 针对doc5 是没有分数的.  
`{ "match": { "content": "java solution" }}` 针对doc5 有一个分数  
只有一个query 是有分数的 eg : 2.3  
match query 数量= 1  
总的query 数量为2  
2.3 * 1 / 2 = 1.15  
所以总的得分 doc5 的 1.15 分 < doc4 的 2.3 分  

最佳字段 策略, best fields策略，dis_max
best fields策略，就是说，搜索到的结果，应该是某一个field中匹配到了尽可能多的关键词，被排在前面；而不是尽可能多的field匹配到了少数的关键词，排在了前面

dis_max语法，直接取多个query中，分数最高的那一个query的分数即可

{ "match": { "title": "java solution" }}，针对doc4，是有一个分数的，1.1
{ "match": { "content":  "java solution" }}，针对doc4，也是有一个分数的，1.2
取最大分数，1.2

{ "match": { "title": "java solution" }}，针对doc5，是没有分数的
{ "match": { "content":  "java solution" }}，针对doc5，是有一个分数的，2.3
取最大分数，2.3

然后doc4的分数 = 1.2 < doc5的分数 = 2.3，所以doc5就可以排在更前面的地方，符合我们的需要

GET /forum/article/_search
{
    "query": {
        "dis_max": {
            "queries": [
                { "match": { "title": "java solution" }},
                { "match": { "content":  "java solution" }}
            ]
        }
    }
}

## 基于 tie_break 参数优化 dis_max 搜索效果

1、搜索title或content中包含java beginner的帖子

GET /forum/article/_search
{
    "query": {
        "dis_max": {
            "queries": [
                { "match": { "title": "java beginner" }},
                { "match": { "content":  "java beginner" }}
            ]
        }
    }
}

可能在实际场景中出现的一个情况是这样的：

（1）某个帖子，doc1，title中包含java，content不包含java beginner任何一个关键词  
（2）某个帖子，doc2，content中包含beginner，title中不包含任何一个关键词  
（3）某个帖子，doc3，title中包含java，content中包含beginner  
（4）最终搜索，可能出来的结果是，doc1和doc2排在doc3的前面，而不是我们期望的doc3排在最前面  

dis_max，只是取分数最高的那个query的分数而已。

2、dis_max只取某一个query最大的分数，完全不考虑其他query的分数

3、使用tie_breaker将其他query的分数也考虑进去

tie_breaker参数的意义，在于说，将其他query的分数，乘以tie_breaker，然后综合与最高分数的那个query的分数，综合在一起进行计算
除了取最高分以外，还会考虑其他的query的分数
tie_breaker的值，在0~1之间，是个小数，就ok

GET /forum/article/_search
{
    "query": {
        "dis_max": {
            "queries": [
                { "match": { "title": "java beginner" }},
                { "match": { "body":  "java beginner" }}
            ],
            "tie_breaker": 0.3
        }
    }
}

##  基于multi_match 语法实现 dis_max+tie_breaker
```
GET /forum/article/_search
{
  "query": {
    "multi_match": {
        "query":                "java solution",
        "type":                 "best_fields", 
        "fields":               [ "title^2", "content" ],
        "tie_breaker":          0.3,
        "minimum_should_match": "50%" 
    }
  } 
}
```

```
GET /forum/article/_search
{
  "query": {
    "dis_max": {
      "queries":  [
        {
          "match": {
            "title": {
              "query": "java beginner",
              "minimum_should_match": "50%",
	      "boost": 2
            }
          }
        },
        {
          "match": {
            "body": {
              "query": "java beginner",
              "minimum_should_match": "30%"
            }
          }
        }
      ],
      "tie_breaker": 0.3
    }
  } 
}
```

minimum_should_match，主要用于去长尾，long tail.
长尾，比如你搜索5个关键词，但是很多结果是只匹配1个关键词的，其实跟你想要的结果相差甚远，这些结果就是长尾
minimum_should_match，控制搜索结果的精准度，只有匹配一定数量的关键词的数据，才能返回

## multi_match + most fiels 策略进行 multi-field 搜索

从best-fields 换成 most-fields 策略
best-fields 策略, 主要将field 匹配尽可能多的关键词 doc 优先返回回来.
most-fields 策略, 主要是说尽可能返回更多的field匹配到某个关键词的doc, 优先返回回来.
```
POST /forum/_mapping/article
{
  "properties": {
      "sub_title": { 
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
```

```
POST /forum/article/_bulk
{ "update": { "_id": "1"} }
{ "doc" : {"sub_title" : "learning more courses"} }
{ "update": { "_id": "2"} }
{ "doc" : {"sub_title" : "learned a lot of course"} }
{ "update": { "_id": "3"} }
{ "doc" : {"sub_title" : "we have a lot of fun"} }
{ "update": { "_id": "4"} }
{ "doc" : {"sub_title" : "both of them are good"} }
{ "update": { "_id": "5"} }
{ "doc" : {"sub_title" : "haha, hello world"} }
```

```
GET /forum/article/_search
{
  "query": {
    "match": {
      "sub_title": "learning courses"
    }
  }
}

{
  "took": 3,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 2,
    "max_score": 1.219939,
    "hits": [
      {
        "_index": "forum",
        "_type": "article",
        "_id": "2",
        "_score": 1.219939,
        "_source": {
          "articleID": "KDKE-B-9947-#kL5",
          "userID": 1,
          "hidden": false,
          "postDate": "2017-01-02",
          "tag": [
            "java"
          ],
          "tag_cnt": 1,
          "view_cnt": 50,
          "title": "this is java blog",
          "content": "i think java is the best programming language",
          "sub_title": "learned a lot of course"
        }
      },
      {
        "_index": "forum",
        "_type": "article",
        "_id": "1",
        "_score": 0.5063205,
        "_source": {
          "articleID": "XHDK-A-1293-#fJ3",
          "userID": 1,
          "hidden": false,
          "postDate": "2017-01-01",
          "tag": [
            "java",
            "hadoop"
          ],
          "tag_cnt": 2,
          "view_cnt": 30,
          "title": "this is java and elasticsearch blog",
          "content": "i like to write best elasticsearch article",
          "sub_title": "learning more courses"
        }
      }
    ]
  }
}
```


sub_title用的是enligsh analyzer，所以还原了单词

为什么，因为如果我们用的是类似于english analyzer这种分词器的话，就会将单词还原为其最基本的形态，stemmer
learning --> learn
learned --> learn
courses --> course

sub_titile: learning coureses --> learn course

{ "doc" : {"sub_title" : "learned a lot of course"} }，就排在了{ "doc" : {"sub_title" : "learning more courses"} }的前面

```
GET /forum/article/_search
{
   "query": {
        "match": {
            "sub_title": "learning courses"
        }
    }
}
```

```
GET /forum/article/_search
{
   "query": {
        "multi_match": {
            "query":  "learning courses",
            "type":   "most_fields", 
            "fields": [ "sub_title", "sub_title.std" ]
        }
    }
}
```

```
{
  "took": 2,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 2,
    "max_score": 1.219939,
    "hits": [
      {
        "_index": "forum",
        "_type": "article",
        "_id": "2",
        "_score": 1.219939,
        "_source": {
          "articleID": "KDKE-B-9947-#kL5",
          "userID": 1,
          "hidden": false,
          "postDate": "2017-01-02",
          "tag": [
            "java"
          ],
          "tag_cnt": 1,
          "view_cnt": 50,
          "title": "this is java blog",
          "content": "i think java is the best programming language",
          "sub_title": "learned a lot of course"
        }
      },
      {
        "_index": "forum",
        "_type": "article",
        "_id": "1",
        "_score": 1.012641,
        "_source": {
          "articleID": "XHDK-A-1293-#fJ3",
          "userID": 1,
          "hidden": false,
          "postDate": "2017-01-01",
          "tag": [
            "java",
            "hadoop"
          ],
          "tag_cnt": 2,
          "view_cnt": 30,
          "title": "this is java and elasticsearch blog",
          "content": "i like to write best elasticsearch article",
          "sub_title": "learning more courses"
        }
      }
    ]
  }
}
```

你问我，具体的分数怎么算出来的，很难说，因为这个东西很复杂， 还不只是TF/IDF算法。因为不同的query，不同的语法，都有不同的计算score的细节。

与best_fields的区别

（1）best_fields，是对多个field进行搜索，挑选某个field匹配度最高的那个分数，同时在多个query最高分相同的情况下，在一定程度上考虑其他query的分数。简单来说，你对多个field进行搜索，就想搜索到某一个field尽可能包含更多关键字的数据

优点：通过best_fields策略，以及综合考虑其他field，还有minimum_should_match支持，可以尽可能精准地将匹配的结果推送到最前面
缺点：除了那些精准匹配的结果，其他差不多大的结果，排序结果不是太均匀，没有什么区分度了

实际的例子：百度之类的搜索引擎，最匹配的到最前面，但是其他的就没什么区分度了

（2）most_fields，综合多个field一起进行搜索，尽可能多地让所有field的query参与到总分数的计算中来，此时就会是个大杂烩，出现类似best_fields案例最开始的那个结果，结果不一定精准，某一个document的一个field包含更多的关键字，但是因为其他document有更多field匹配到了，所以排在了前面；所以需要建立类似sub_title.std这样的field，尽可能让某一个field精准匹配query string，贡献更高的分数，将更精准匹配的数据排到前面

优点：将尽可能匹配更多field的结果推送到最前面，整个排序结果是比较均匀的
缺点：可能那些精准匹配的结果，无法推送到最前面

实际的例子：wiki，明显的most_fields策略，搜索结果比较均匀，但是的确要翻好几页才能找到最匹配的结果




## 使用 most_fields策略进行cross-fields search 弊端大揭秘

cross-fields 搜索, 一个唯一的标识, 跨了多个field. 
eg: 一个人 姓名 跨first_name, last_name, country, province, city中.
跨多个field搜索一个标识, 搜索一个人名, 或者一个地址, 就是cross-field.

如果需要实现, 可能用most_fields 比较合适, 因为best_fields 是优先搜索单个field最匹配的结果, cross-fields本身就不是一个field问题了.

```
POST /forum/article/_bulk
{ "update": { "_id": "1"} }
{ "doc" : {"author_first_name" : "Peter", "author_last_name" : "Smith"} }
{ "update": { "_id": "2"} }
{ "doc" : {"author_first_name" : "Smith", "author_last_name" : "Williams"} }
{ "update": { "_id": "3"} }
{ "doc" : {"author_first_name" : "Jack", "author_last_name" : "Ma"} }
{ "update": { "_id": "4"} }
{ "doc" : {"author_first_name" : "Robbin", "author_last_name" : "Li"} }
{ "update": { "_id": "5"} }
{ "doc" : {"author_first_name" : "Tonny", "author_last_name" : "Peter Smith"} }
```

```
GET /forum/article/_search
{
  "query": {
    "multi_match": {
      "query":       "Peter Smith",
      "type":        "most_fields",
      "fields":      [ "author_first_name", "author_last_name" ]
    }
  }
}
```

Peter Smith，匹配author_first_name，匹配到了Smith，这时候它的分数很高，为什么啊？？？
因为IDF分数高，IDF分数要高，那么这个匹配到的term（Smith），在所有doc中的出现频率要低，author_first_name field中，Smith就出现过1次
Peter Smith这个人，doc 1，Smith在author_last_name中，但是author_last_name出现了两次Smith，所以导致doc 1的IDF分数较低


```
{
  "took": 2,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 3,
    "max_score": 0.6931472,
    "hits": [
      {
        "_index": "forum",
        "_type": "article",
        "_id": "2",
        "_score": 0.6931472,
        "_source": {
          "articleID": "KDKE-B-9947-#kL5",
          "userID": 1,
          "hidden": false,
          "postDate": "2017-01-02",
          "tag": [
            "java"
          ],
          "tag_cnt": 1,
          "view_cnt": 50,
          "title": "this is java blog",
          "content": "i think java is the best programming language",
          "sub_title": "learned a lot of course",
          "author_first_name": "Smith",
          "author_last_name": "Williams"
        }
      },
      {
        "_index": "forum",
        "_type": "article",
        "_id": "1",
        "_score": 0.5753642,
        "_source": {
          "articleID": "XHDK-A-1293-#fJ3",
          "userID": 1,
          "hidden": false,
          "postDate": "2017-01-01",
          "tag": [
            "java",
            "hadoop"
          ],
          "tag_cnt": 2,
          "view_cnt": 30,
          "title": "this is java and elasticsearch blog",
          "content": "i like to write best elasticsearch article",
          "sub_title": "learning more courses",
          "author_first_name": "Peter",
          "author_last_name": "Smith"
        }
      },
      {
        "_index": "forum",
        "_type": "article",
        "_id": "5",
        "_score": 0.51623213,
        "_source": {
          "articleID": "DHJK-B-1395-#Ky5",
          "userID": 3,
          "hidden": false,
          "postDate": "2017-03-01",
          "tag": [
            "elasticsearch"
          ],
          "tag_cnt": 1,
          "view_cnt": 10,
          "title": "this is spark blog",
          "content": "spark is best big data solution based on scala ,an programming language similar to java",
          "sub_title": "haha, hello world",
          "author_first_name": "Tonny",
          "author_last_name": "Peter Smith"
        }
      }
    ]
  }
}

```
问题1：只是找到尽可能多的field匹配的doc，而不是某个field完全匹配的doc

问题2：most_fields，没办法用minimum_should_match去掉长尾数据，就是匹配的特别少的结果

问题3：TF/IDF算法，比如Peter Smith和Smith Williams，搜索Peter Smith的时候，由于first_name中很少有Smith的，所以query在所有document中的频率很低，得到的分数很高，可能Smith Williams反而会排在Peter Smith前面


## 使用copy_to 定制组合field 解决cross-fields搜索弊端

用most_fields策略, 去实现cross-fileds搜索
(1). 使用copy_to, 将多个field组合成一个field
当有多个field, 有多个field以后, 就很尴尬, 我们只要想办法将一个表示跨在多个field的情况, eg : 本来人名 包裹 frist_name, last_name, 现在合并为一个full_name, 就可以解决多个filed问题
```
PUT /forum/_mapping/article
{
  "properties": {
      "new_author_first_name": {
          "type":     "string",
          "copy_to":  "new_author_full_name" 
      },
      "new_author_last_name": {
          "type":     "string",
          "copy_to":  "new_author_full_name" 
      },
      "new_author_full_name": {
          "type":     "string"
      }
  }
}
```
用这个copy_to语法后, 就可以将多个字段拷贝到一个字段中, 并建立倒排索引.
```
POST /forum/article/_bulk
{ "update": { "_id": "1"} }
{ "doc" : {"new_author_first_name" : "Peter", "new_author_last_name" : "Smith"} }		--> Peter Smith
{ "update": { "_id": "2"} }	
{ "doc" : {"new_author_first_name" : "Smith", "new_author_last_name" : "Williams"} }		--> Smith Williams
{ "update": { "_id": "3"} }
{ "doc" : {"new_author_first_name" : "Jack", "new_author_last_name" : "Ma"} }			--> Jack Ma
{ "update": { "_id": "4"} }
{ "doc" : {"new_author_first_name" : "Robbin", "new_author_last_name" : "Li"} }			--> Robbin Li
{ "update": { "_id": "5"} }
{ "doc" : {"new_author_first_name" : "Tonny", "new_author_last_name" : "Peter Smith"} }		--> Tonny Peter Smith
```

搜索获取
```
GET /forum/article/_search
{
  "query": {
    "match": {
      "new_author_full_name":       "Peter Smith"
    }
  }
}
```

问题1：只是找到尽可能多的field匹配的doc，而不是某个field完全匹配的doc   
解决: 最匹配的document被最先返回

问题2：most_fields，没办法用minimum_should_match去掉长尾数据，就是匹配的特别少的结果
解决 : 可以使用minimum_should_match去掉长尾数据

问题3：TF/IDF算法，比如Peter Smith和Smith Williams，搜索Peter Smith的时候，由于first_name中很少有Smith的，所以query在所有document中的频率很低，得到的分数很高，可能Smith Williams反而会排在Peter Smith前面 
解决 : Smith和Peter在一个field了，所以在所有document中出现的次数是均匀的，不会有极端的偏差


## 使用copy_to 定制组合field解决cross-field搜索弊端

```
GET /forum/article/_search
{
  "query": {
    "multi_match": {
      "query": "Peter Smith",
      "type": "cross_fields", 
      "operator": "and",
      "fields": ["author_first_name", "author_last_name"]
    }
  }
}
```
multi_match 直接指定 type 为 cross_fields

问题1：只是找到尽可能多的field匹配的doc，而不是某个field完全匹配的doc   
解决 : 要求每个term都必须在任何一个field中出现

要求Peter必须在author_first_name或author_last_name中出现  
要求Smith必须在author_first_name或author_last_name中出现

Peter Smith可能是横跨在多个field中的，所以必须要求每个term都在某个field中出现，组合起来才能组成我们想要的标识，完整的人名

原来most_fiels，可能像Smith Williams也可能会出现，因为most_fields要求只是任何一个field匹配了就可以，匹配的field越多，分数越高

问题2：most_fields，没办法用minimum_should_match去掉长尾数据，就是匹配的特别少的结果  
解决 : 既然每个term都要求出现，长尾肯定被去除掉了

java hadoop spark --> 这3个term都必须在任何一个field出现了

比如有的document，只有一个field中包含一个java，那就被干掉了，作为长尾就没了

问题3：TF/IDF算法，比如Peter Smith和Smith Williams，搜索Peter Smith的时候，由于first_name中很少有Smith的，所以query在所有document中的频率很低，得到的分数很高，可能Smith Williams反而会排在Peter Smith前面 --> 计算IDF的时候，将每个query在每个field中的IDF都取出来，取最小值，就不会出现极端情况下的极大值了

Peter Smith
Peter
Smith

Smith，在author_first_name这个field中，在所有doc的这个Field中，出现的频率很低，导致IDF分数很高；Smith在所有doc的author_last_name field中的频率算出一个IDF分数，因为一般来说last_name中的Smith频率都较高，所以IDF分数是正常的，不会太高；然后对于Smith来说，会取两个IDF分数中，较小的那个分数。就不会出现IDF分过高的情况。

## phrase matching 搜索技术使用

java is my favourite programming language, and I also think spark is a very good big data system.  
java spark are very related, because scala is spark's programming language and scala is also based on jvm like java.

match query 搜索 java spark
```
{
    "match" : {
        "content" : "java spark"
    }
}
```

match query，只能搜索到包含java和spark的document，但是不知道java和spark是不是离的很近

包含java或包含spark，或包含java和spark的doc，都会被返回回来。我们其实并不知道哪个doc，java和spark距离的比较近。如果我们就是希望搜索java spark，中间不能插入任何其他的字符，那这个时候match去做全文检索，能搞定我们的需求吗？答案是，搞不定。

如果我们要尽量让java和spark离的很近的document优先返回，要给它一个更高的relevance score，这就涉及到了proximity match，近似匹配

1、java spark，就靠在一起，中间不能插入任何其他字符，就要搜索出来这种doc

2、java spark，但是要求，java和spark两个单词靠的越近，doc的分数越高，排名越靠前

要实现上述两个需求，用match做全文检索，是搞不定的，必须得用proximity match，近似匹配

phrase match，proximity match：短语匹配，近似匹配

这一讲，要学习的是phrase match，就是仅仅搜索出java和spark靠在一起的那些doc，比如有个doc，是java use'd spark，不行。必须是比如java spark are very good friends，是可以搜索出来的。

phrase match，就是要去将多个term作为一个短语，一起去搜索，只有包含这个短语的doc才会作为结果返回。不像是match，java spark，java的doc也会返回，spark的doc也会返回。

match_phrase
```
GET /forum/article/_search
{
  "query": {
    "match": {
      "content": "java spark"
    }
  }
}
```

单单包含了java的doc页返回了, 不是我们想要的结果.

```
POST /forum/article/5/_update
{
  "doc": {
    "content": "spark is best big data solution based on scala ,an programming language similar to java spark"
  }
}
```

将一个doc的document设置为恰巧包含java spark 这个短语

match_phrase 语法

```
GET /forum/article/_search
{
    "query": {
        "match_phrase": {
            "content": "java spark"
        }
    }
}
```
成功了，只有包含java spark这个短语的doc才返回了，只包含java的doc不会返回

3、term position

hello world, java spark		doc1
hi, spark java				doc2

hello 		doc1(0)		
wolrd		doc1(1)
java		doc1(2) doc2(2)
spark		doc1(3) doc2(1)

了解什么是分词后的position

```
GET _analyze
{
  "text": "hello world, java spark",
  "analyzer": "standard"
}
```

4、match_phrase的基本原理

索引中的position，match_phrase

hello world, java spark		doc1
hi, spark java				doc2

hello 		doc1(0)		
wolrd		doc1(1)
java		doc1(2) doc2(2)
spark		doc1(3) doc2(1)

java spark --> match phrase

java spark --> java和spark

java --> doc1(2) doc2(2)
spark --> doc1(3) doc2(1)

要找到每个term都在的一个共有的那些doc，就是要求一个doc，必须包含每个term，才能拿出来继续计算

doc1 --> java和spark --> spark position恰巧比java大1 --> java的position是2，spark的position是3，恰好满足条件

doc1符合条件

doc2 --> java和spark --> java position是2，spark position是1，spark position比java position小1，而不是大1 --> 光是position就不满足，那么doc2不匹配

## 基于slop参数实现近似匹配以及原理解析和相关实验

```
GET /forum/article/_search
{
    "query": {
        "match_phrase": {
            "title": {
                "query": "java spark",
                "slop":  1
            }
        }
    }
}
```
什么是slop  
query string，搜索文本，中的几个term，要经过几次移动才能与一个document匹配，这个移动的次数，就是slop

实际举例，一个query string经过几次移动之后可以匹配到一个document，然后设置slop

hello world, java is very good, spark is also very good.

java spark，match phrase，搜不到


如果我们指定了slop，那么就允许java spark进行移动，来尝试与doc进行匹配

| java	| is | very | good | spark | is |
| :----: | :--: | :--: | :--: | :---: | :----: |
| java | spark |  |  |  |  |  |
| java |   --->   | spark  |  |  |  |
| java |   |  ---->  |  spark  |   |   |
| java |  |   |  ----> |  spark |


这里的slop，就是3，因为java spark这个短语，spark移动了3次，就可以跟一个doc匹配上了

slop的含义，不仅仅是说一个query string terms移动几次，跟一个doc匹配上。一个query string terms，最多可以移动几次去尝试跟一个doc匹配上

slop，设置的是3，那么就ok

```
GET /forum/article/_search
{
    "query": {
        "match_phrase": {
            "title": {
                "query": "java spark",
                "slop":  3
            }
        }
    }
}
```
就可以把刚才那个doc匹配上，那个doc会作为结果返回

但是如果slop设置的是2，那么java spark，spark最多只能移动2次，此时跟doc是匹配不上的，那个doc是不会作为结果返回的

做实验，验证slop的含义

```
GET /forum/article/_search
{
  "query": {
    "match_phrase": {
      "content": {
        "query": "spark data",
        "slop": 3
      }
    }
  }
}
```

slop搜索下，关键词离的越近，relevance score就会越高，做实验说明。。。
```
{
  "took": 4,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 3,
    "max_score": 1.3728157,
    "hits": [
      {
        "_index": "forum",
        "_type": "article",
        "_id": "2",
        "_score": 1.3728157,
        "_source": {
          "articleID": "KDKE-B-9947-#kL5",
          "userID": 1,
          "hidden": false,
          "postDate": "2017-01-02",
          "tag": [
            "java"
          ],
          "tag_cnt": 1,
          "view_cnt": 50,
          "title": "this is java blog",
          "content": "i think java is the best programming language",
          "sub_title": "learned a lot of course",
          "author_first_name": "Smith",
          "author_last_name": "Williams",
          "new_author_last_name": "Williams",
          "new_author_first_name": "Smith"
        }
      },
      {
        "_index": "forum",
        "_type": "article",
        "_id": "5",
        "_score": 0.5753642,
        "_source": {
          "articleID": "DHJK-B-1395-#Ky5",
          "userID": 3,
          "hidden": false,
          "postDate": "2017-03-01",
          "tag": [
            "elasticsearch"
          ],
          "tag_cnt": 1,
          "view_cnt": 10,
          "title": "this is spark blog",
          "content": "spark is best big data solution based on scala ,an programming language similar to java spark",
          "sub_title": "haha, hello world",
          "author_first_name": "Tonny",
          "author_last_name": "Peter Smith",
          "new_author_last_name": "Peter Smith",
          "new_author_first_name": "Tonny"
        }
      },
      {
        "_index": "forum",
        "_type": "article",
        "_id": "1",
        "_score": 0.28582606,
        "_source": {
          "articleID": "XHDK-A-1293-#fJ3",
          "userID": 1,
          "hidden": false,
          "postDate": "2017-01-01",
          "tag": [
            "java",
            "hadoop"
          ],
          "tag_cnt": 2,
          "view_cnt": 30,
          "title": "this is java and elasticsearch blog",
          "content": "i like to write best elasticsearch article",
          "sub_title": "learning more courses",
          "author_first_name": "Peter",
          "author_last_name": "Smith",
          "new_author_last_name": "Smith",
          "new_author_first_name": "Peter"
        }
      }
    ]
  }
}

GET /forum/article/_search
{
  "query": {
    "match_phrase": {
      "content": {
        "query": "java best",
        "slop": 15
      }
    }
  }
}

{
  "took": 3,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 2,
    "max_score": 0.65380025,
    "hits": [
      {
        "_index": "forum",
        "_type": "article",
        "_id": "2",
        "_score": 0.65380025,
        "_source": {
          "articleID": "KDKE-B-9947-#kL5",
          "userID": 1,
          "hidden": false,
          "postDate": "2017-01-02",
          "tag": [
            "java"
          ],
          "tag_cnt": 1,
          "view_cnt": 50,
          "title": "this is java blog",
          "content": "i think java is the best programming language",
          "sub_title": "learned a lot of course",
          "author_first_name": "Smith",
          "author_last_name": "Williams",
          "new_author_last_name": "Williams",
          "new_author_first_name": "Smith"
        }
      },
      {
        "_index": "forum",
        "_type": "article",
        "_id": "5",
        "_score": 0.07111243,
        "_source": {
          "articleID": "DHJK-B-1395-#Ky5",
          "userID": 3,
          "hidden": false,
          "postDate": "2017-03-01",
          "tag": [
            "elasticsearch"
          ],
          "tag_cnt": 1,
          "view_cnt": 10,
          "title": "this is spark blog",
          "content": "spark is best big data solution based on scala ,an programming language similar to java spark",
          "sub_title": "haha, hello world",
          "author_first_name": "Tonny",
          "author_last_name": "Peter Smith",
          "new_author_last_name": "Peter Smith",
          "new_author_first_name": "Tonny"
        }
      }
    ]
  }
}
```

其实，加了slop的phrase match，就是proximity match，近似匹配

1、java spark，短语，doc，phrase match  
2、java spark，可以有一定的距离，但是靠的越近，越先搜索出来，proximity match


## 混合使用match和近似匹配实现召回率与精准度的平衡

召回率

比如你搜索一个java spark，总共有100个doc，能返回多少个doc作为结果，就是召回率，recall.

精准度

比如你搜索一个java spark，能不能尽可能让包含java spark，或者是java和spark离的很近的doc，排在最前面，precision

直接用match_phrase短语搜索，会导致必须所有term都在doc field中出现，而且距离在slop限定范围内，才能匹配上

match phrase，proximity match，要求doc必须包含所有的term，才能作为结果返回；如果某一个doc可能就是有某个term没有包含，那么就无法作为结果返回

java spark --> hello world java --> 就不能返回了
java spark --> hello world, java spark --> 才可以返回

近似匹配的时候，召回率比较低，精准度太高了

但是有时可能我们希望的是匹配到几个term中的部分，就可以作为结果出来，这样可以提高召回率。同时我们也希望用上match_phrase根据距离提升分数的功能，让几个term距离越近分数就越高，优先返回

就是优先满足召回率，意思，java spark，包含java的也返回，包含spark的也返回，包含java和spark的也返回；同时兼顾精准度，就是包含java和spark，同时java和spark离的越近的doc排在最前面

此时可以用bool组合match query和match_phrase query一起，来实现上述效果

```
GET /forum/article/_search
{
  "query": {
    "bool": {
      "must": {
        "match": { 
          "title": {
            "query":                "java spark" --> java或spark或java spark，java和spark靠前，但是没法区分java和spark的距离，也许java和spark靠的很近，但是没法排在最前面
          }
        }
      },
      "should": {
        "match_phrase": { --> 在slop以内，如果java spark能匹配上一个doc，那么就会对doc贡献自己的relevance score，如果java和spark靠的越近，那么就分数越高
          "title": {
            "query": "java spark",
            "slop":  50
          }
        }
      }
    }
  }
}
```

请求
```
GET /forum/article/_search 
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "content": "java spark"
          }
        }
      ]
    }
  }
}
```
返回
```
{
  "took": 5,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 2,
    "max_score": 0.68640786,
    "hits": [
      {
        "_index": "forum",
        "_type": "article",
        "_id": "2",
        "_score": 0.68640786,
        "_source": {
          "articleID": "KDKE-B-9947-#kL5",
          "userID": 1,
          "hidden": false,
          "postDate": "2017-01-02",
          "tag": [
            "java"
          ],
          "tag_cnt": 1,
          "view_cnt": 50,
          "title": "this is java blog",
          "content": "i think java is the best programming language",
          "sub_title": "learned a lot of course",
          "author_first_name": "Smith",
          "author_last_name": "Williams",
          "new_author_last_name": "Williams",
          "new_author_first_name": "Smith",
          "followers": [
            "Tom",
            "Jack"
          ]
        }
      },
      {
        "_index": "forum",
        "_type": "article",
        "_id": "5",
        "_score": 0.68324494,
        "_source": {
          "articleID": "DHJK-B-1395-#Ky5",
          "userID": 3,
          "hidden": false,
          "postDate": "2017-03-01",
          "tag": [
            "elasticsearch"
          ],
          "tag_cnt": 1,
          "view_cnt": 10,
          "title": "this is spark blog",
          "content": "spark is best big data solution based on scala ,an programming language similar to java spark",
          "sub_title": "haha, hello world",
          "author_first_name": "Tonny",
          "author_last_name": "Peter Smith",
          "new_author_last_name": "Peter Smith",
          "new_author_first_name": "Tonny",
          "followers": [
            "Jack",
            "Robbin Li"
          ]
        }
      }
    ]
  }
}
```

请求
```
GET /forum/article/_search 
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "content": "java spark"
          }
        }
      ],
      "should": [
        {
          "match_phrase": {
            "content": {
              "query": "java spark",
              "slop": 50
            }
          }
        }
      ]
    }
  }
}
```

响应
```
{
  "took": 5,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 2,
    "max_score": 1.258609,
    "hits": [
      {
        "_index": "forum",
        "_type": "article",
        "_id": "5",
        "_score": 1.258609,
        "_source": {
          "articleID": "DHJK-B-1395-#Ky5",
          "userID": 3,
          "hidden": false,
          "postDate": "2017-03-01",
          "tag": [
            "elasticsearch"
          ],
          "tag_cnt": 1,
          "view_cnt": 10,
          "title": "this is spark blog",
          "content": "spark is best big data solution based on scala ,an programming language similar to java spark",
          "sub_title": "haha, hello world",
          "author_first_name": "Tonny",
          "author_last_name": "Peter Smith",
          "new_author_last_name": "Peter Smith",
          "new_author_first_name": "Tonny",
          "followers": [
            "Jack",
            "Robbin Li"
          ]
        }
      },
      {
        "_index": "forum",
        "_type": "article",
        "_id": "2",
        "_score": 0.68640786,
        "_source": {
          "articleID": "KDKE-B-9947-#kL5",
          "userID": 1,
          "hidden": false,
          "postDate": "2017-01-02",
          "tag": [
            "java"
          ],
          "tag_cnt": 1,
          "view_cnt": 50,
          "title": "this is java blog",
          "content": "i think java is the best programming language",
          "sub_title": "learned a lot of course",
          "author_first_name": "Smith",
          "author_last_name": "Williams",
          "new_author_last_name": "Williams",
          "new_author_first_name": "Smith",
          "followers": [
            "Tom",
            "Jack"
          ]
        }
      }
    ]
  }
}
```

## 使用rescoring机制优化近似匹配搜索的性能

match和phrase match(proximity match)区别

match : 只要简单的匹配到了一个term, 就可以理解将term对应的doc作为结果返回, 扫描倒排索引, 扫描到了就正确.
phrase match : 首先扫描到所有的term的doc list; 找到包含所有term的doc list; 然后对每个doc都计算每个term的position, 是否符合指定范围; slop 需要进行复杂的运算, 来判断是否通过slop移动, 匹配一个doc.

match query的性能比phrase match和proximity match (有slop) 要高很多.  因为后两者都要对计算position的距离.

match query比phrase match的性能要高10倍, 比proximity match的性能要高20倍.

es的性能都是在毫秒级, 所以 phrase match和proximity match的性能 也是在几十和几百毫秒以内.

优化proximity match的性能, 一般就是要减少要进行proximity match搜索的document数量. 主要思路就是, 用match query 先过滤出需要的数据, 然后再用proximity match 来根据term 距离提高doc分数. 同时proximity match 只针对每个shard的分数排名前n个doc起作用，来重新调整它们的分数，这个过程称之为rescoring，重计分。因为一般用户会分页查询，只会看到前几页的数据，所以不需要对所有结果进行proximity match操作.

可以使用match + proximity match 同时实现召回率和精准度

默认情况下，match也许匹配了1000个doc，proximity match全都需要对每个doc进行一遍运算，判断能否slop移动匹配上，然后去贡献自己的分数
但是很多情况下，match出来也许1000个doc，其实用户大部分情况下是分页查询的，所以可能最多只会看前几页，比如一页是10条，最多也许就看5页，就是50条
proximity match只要对前50个doc进行slop移动去匹配，去贡献自己的分数即可，不需要对全部1000个doc都去进行计算和贡献分数

rescore：重打分

match：1000个doc，其实这时候每个doc都有一个分数了; proximity match，前50个doc，进行rescore，重打分，即可; 让前50个doc，term举例越近的，排在越前面

```
GET /forum/article/_search 
{
  "query": {
    "match": {
      "content": "java spark"
    }
  },
  "rescore": {
    "window_size": 50,
    "query": {
      "rescore_query": {
        "match_phrase": {
          "content": {
            "query": "java spark",
            "slop": 50
          }
        }
      }
    }
  }
}
```

## 使用实战前缀搜索、通配符搜索、正则搜索技术.

前缀搜索
C3D0-KD345
C3K5-DFG65
C4I8-UI365

根据字符串前缀搜索 C3  类似于 like C3% 的SQL 语法

```
PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "title": {
          "type": "keyword"
        }
      }
    }
  }
}
```

```
GET my_index/my_type/_search
{
  "query": {
    "prefix": {
      "title": {
        "value": "C3"
      }
    }
  }
}
```

前缀搜索的原理

prefix query不计算relevance score，与prefix filter唯一的区别就是，filter会cache bitset

扫描整个倒排索引，举例说明

前缀越短，要处理的doc越多，性能越差，尽可能用长前缀搜索

前缀搜索，它是怎么执行的？性能为什么差呢？

match

C3-D0-KD345  
C3-K5-DFG65  
C4-I8-UI365  

全文检索

每个字符串都需要被分词
c3 --> 扫描倒排索引 --> 一旦扫描到c3，就可以停了，因为带c3的就2个doc，已经找到了 --> 没有必要继续去搜索其他的term了

match性能往往是很高的

不分词

C3-D0-KD345
C3-K5-DFG65
C4-I8-UI365

c3 --> 先扫描到了C3-D0-KD345，很棒，找到了一个前缀带c3的字符串 --> 还是要继续搜索的，因为后面还有一个C3-K5-DFG65，也许还有其他很多的前缀带c3的字符串 --> 你扫描到了一个前缀匹配的term，不能停，必须继续搜索 --> 直到扫描完整个的倒排索引，才能结束

因为实际场景中，可能有些场景是全文检索解决不了的

C3D0-KD345
C3K5-DFG65
C4I8-UI365

c3 --> match --> 扫描整个倒排索引，能找到吗  
c3 --> 只能用prefix  
prefix性能很差  

通配符搜索

跟前缀搜索类似，功能更加强大

C3D0-KD345
C3K5-DFG65
C4I8-UI365

5字符-D任意个字符5

5?-*5：通配符去表达更加复杂的模糊搜索的语义

```
GET my_index/my_type/_search
{
  "query": {
    "wildcard": {
      "title": {
        "value": "C?K*5"
      }
    }
  }
}
```

?：任意字符
*：0个或任意多个字符

性能一样差，必须扫描整个倒排索引，才ok

正则搜索
```
GET /my_index/my_type/_search 
{
  "query": {
    "regexp": {
      "title": "C[0-9].+"
    }
  }
}
```
C[0-9].+

[0-9]：指定范围内的数字
[a-z]：指定范围内的字母
.：一个字符
+：前面的正则表达式可以出现一次或多次

wildcard和regexp，与prefix原理一致，都会扫描整个索引，性能很差

对于这些高级的搜索语法, 性能太差, 能不用就不用.


## 使用match_phrase_prefix 实现 search-time 搜索推荐

 搜索推荐 search as you type
 
 hello w --> 搜索

hello world  
hello we  
hello win  
hello wind  
hello dog  
hello cat  


hello w -->

hello world  
hello we  
hello win  
hello wind  

搜索推荐的功能

百度 --> elas -> elasticsearch --> elastcsearch 权威指南

```
GET /my_index/my_type/_search 
{
  "query": {
    "match_phrase_prefix": {
      "title": "hello d"
    }
  }
}
```

原理跟match_phrase类似，唯一的区别，就是把最后一个term作为前缀去搜索

hello就是去进行match，搜索对应的doc
w，会作为前缀，去扫描整个倒排索引，找到所有w开头的doc
然后找到所有doc中，即包含hello，又包含w开头的字符的doc
根据你的slop去计算，看在slop范围内，能不能让hello w，正好跟doc中的hello和w开头的单词的position相匹配

也可以指定slop，但是只有最后一个term会作为前缀

max_expansions：指定prefix最多匹配多少个term，超过这个数量就不继续匹配了，限定性能

默认情况下，前缀要扫描所有的倒排索引中的term，去查找w打头的单词，但是这样性能太差。可以用max_expansions限定，w前缀最多匹配多少个term，就不再继续搜索倒排索引了。

尽量不用, 性能消耗极大.


## 通过ngram 分词机制实现index-time搜索推荐

quick 5种长度下的ngram

ngram length=1，q u i c k
ngram length=2，qu ui ic ck
ngram length=3，qui uic ick
ngram length=4，quic uick
ngram length=5，quick

quick，anchor首字母后进行ngram

q
qu
qui
quic
quick

使用edge ngram将每个单词都进行进一步的分词切分，用切分后的ngram来实现前缀搜索推荐功能

hello world
hello we

h
he
hel
hell
hello		doc1,doc2

w			doc1,doc2
wo
wor
worl
world
e			doc2

helloworld

min ngram = 1
max ngram = 3

h
he
hel

hello w

hello --> hello，doc1
w --> w，doc1

doc1，hello和w，而且position也匹配，所以，ok，doc1返回，hello world

搜索的时候，不用再根据一个前缀，然后扫描整个倒排索引了; 简单的拿前缀去倒排索引中匹配即可，如果匹配上了，那么就好了; match，全文检索

实验一下ngram
```
PUT /my_index
{
    "settings": {
        "analysis": {
            "filter": {
                "autocomplete_filter": { 
                    "type":     "edge_ngram",
                    "min_gram": 1,
                    "max_gram": 20
                }
            },
            "analyzer": {
                "autocomplete": {
                    "type":      "custom",
                    "tokenizer": "standard",
                    "filter": [
                        "lowercase",
                        "autocomplete_filter" 
                    ]
                }
            }
        }
    }
}
```

```
GET /my_index/_analyze
{
  "analyzer": "autocomplete",
  "text": "quick brown"
}
```

```
PUT /my_index/_mapping/my_type
{
  "properties": {
      "title": {
          "type":     "string",
          "analyzer": "autocomplete",
          "search_analyzer": "standard"
      }
  }
}
```

hello world

h
he
hel
hell
hello		

w			
wo
wor
worl
world

hello w

h
he
hel
hell
hello	

w

hello w --> hello --> w

```
GET /my_index/my_type/_search 
{
  "query": {
    "match_phrase": {
      "title": "hello w"
    }
  }
}
```

如果用match，只有hello的也会出来，全文检索，只是分数比较低
推荐使用match_phrase，要求每个term都有，而且position刚好靠着1位，符合我们的期望的


## 揭秘 TF & IDF 算法以及向量空间模型算法

1  boolean model

类似and这种逻辑操作符, 先过滤出包含指定term的doc

query "hello world" --> 过滤出 (包含) ---> hello / world / hello & world  
bool ---> must / must not /should --> 过滤出(包含/不包含/ 可能包含)   
doc --> 不打分数 --> 正或反 true or false --> 为了减少后续需要计算的doc数量--> 提升性能


2 TF/IDF

单个term在doc中的分数
query: hello world --> doc.content  
doc1: java is my favourite programming language, hello world !!!  
doc2: hello java, you are very good, oh hello world!!!  

hello对doc1的评分
TF: term frequency 

找到hello在doc1中出现了几次，1次，会根据出现的次数给个分数
一个term在一个doc中, 出现的次数越多, 那么最后给的相关度评分就会越高.

IDF : inversed document frequency

找到hello 在所有doc中出现的次数.
一个term在一个doc中，出现的次数越多，那么最后给的相关度评分就会越低

length norm
hello 搜索的那个field的长度, field长度越长, 给的相关度评分越低, field长度越短,给的相关度评分越高.

最后会将hello 这个term, 对doc1的分数, 综合 TF IDF length norm 计算一个综合性的分数 ---> vector space model

vector space model

多个term对一个doc的总分数
hello world --> es 会根据 hello world 在所有doc中

hello world --> es 会根据 hello world 在所有doc 中的评分情况, 并计算出一个query vector, query 向量

hello 这个 term 给的基于所有doc的一个评分就是2.
world 这个term 给的基于所有doc的一个评分就是5.

[2, 5] query vector
doc vector, ３个 doc, 一个包含了1个term,　一个包含了另一个term, 一个包含了２个term

3个doc

doc1, 包含了hello
doc2, 包含了world
doc3, 包含了 hello, world

会给每一个 doc, 拿每个term计算出一个分数来, hello 有一个分数, world 有一个分数, 再拿所有term的分数, 组成一个doc vector

画在一个图中，取每个doc vector对query vector的弧度，给出每个doc对多个term的总分数

每个doc vector计算出对query vector的弧度，最后基于这个弧度给出一个doc相对于query中多个term的总分数
弧度越大，分数月底; 弧度越小，分数越高

如果是多个term，那么就是线性代数来计算，无法用图表示


## 揭示lucene的相关度分数算法

相关度算法模型
boolean model 、TF/IDF 、vector space model

深入TF/IDF 算法, lucene中, 底层, 到底进行TF/IDF算法计算的一个完整的公式

boolean model

query : hello world
```
"match" : {
    "title" : "hello world"
}

"bool" : {
    "should" : [
      {
        "match" : {
            "title" : "hello"
        }
      },
      {
        "match" : {
          "title" : "world"
        }
      }
    ]
}
```
普通multivalue搜索，转换为bool搜索，boolean model

1. lucene 中 practical scoring function

practical scoring function 计算query对一个doc的分数 的公式, 该函数会使用一个公式计算.


score(q,d)  =  
            queryNorm(q)  
          · coord(q,d)    
          · ∑ (           
                tf(t in d)   
              · idf(t)2      
              · t.getBoost() 
              · norm(t,d)    
            ) (t in q) score(q, d) = queryNorm(q) 

score(q,d) score(q,d) is the relevance score of document d for query q.

公式最终结果,就是一个query (叫 q), 对一个doc(叫做d) 的最终的总评分.

queryNorm(q) is the query normalization factor (new).

queryNorm, 是用来让一个doc的分数处于一个合理的区间内, 不要太离谱. 

coord(q,d) is the coordination factor (new).

对于更加匹配的doc, 进行一些分数上的成倍的奖励.

the sum of the weights for each term t the query q for document d.

∑：求和的符号

∑ (t in q)：query中每个term，query = hello world，query中的term就包含了hello和world

query中每个term对doc的分数，进行求和，多个term对一个doc的分数，组成一个vector space，然后计算.

tf(t in d) is thetf(t in d) is the term frequency for term t in document d.

计算每一个term 对doc的分数的时候, 就是TF/IDF 算法

idf(t) is the inverse document frequency for term t.

t.getBoost() is the boost that has been applied to the query (new).

norm(t,d) is the field-length norm, combined with the index-time field-level boost, if any. (new).

2. query normalization factor

queryNorm = 1 /  √sumOfSquaredWeights

sumOfSquaredWeights = 所有term的IDF分数之和，开一个平方根，然后做一个平方根分之1
主要是为了将分数进行规范化 --> 开平方根，首先数据就变小了 --> 然后还用1去除以这个平方根，分数就会很小 --> 1.几 / 零点几
分数就不会出现几万，几十万，那样的离谱的分数


3. query coodination

奖励那些匹配更多字符的doc更多的分数

Document 1 with hello → score: 1.5
Document 2 with hello world → score: 3.0
Document 3 with hello world java → score: 4.5

Document 1 with hello → score: 1.5 * 1 / 3 = 0.5
Document 2 with hello world → score: 3.0 * 2 / 3 = 2.0
Document 3 with hello world java → score: 4.5 * 3 / 3 = 4.5

把计算出来的总分数 * 匹配上的term数量 / 总的term数量，让匹配不同term/query数量的doc，分数之间拉开差距



## 四种常见相关度分数优化方法

es 相关度分数算法 TF/IDF vector model, boolean model;

实际的公式 query norm, query coordination, boost

对相关度评分进行调节和优化的常见的4种方法

1. query-time boost

```
GET /forum/article/_search
{
  "query": {
    "bool": {
      "should": [
        {
          "match": {
            "title": {
              "query": "java spark",
              "boost": 2
            }
          }
        },
        {
          "match": {
            "content": "java spark"
          }
        }
      ]
    }
  }
}
```

通过boost设置优先级

2. 重构查询结构

重构查询结果, 在es新版本中, 影响会越来越小, 可以不用.

```
GET /forum/article/_search 
{
  "query": {
    "bool": {
      "should": [
        {
          "match": {
            "content": "java"
          }
        },
        {
          "match": {
            "content": "spark"
          }
        },
        {
          "bool": {
            "should": [
              {
                "match": {
                  "content": "solution"
                }
              },
              {
                "match": {
                  "content": "beginner"
                }
              }
            ]
          }
        }
      ]
    }
  }
}
```

3. negative boost

搜索包含java，不包含spark的doc，但是这样子很死板
搜索包含java，尽量不包含spark的doc，如果包含了spark，不会说排除掉这个doc，而是说将这个doc的分数降低
包含了negative term的doc，分数乘以negative boost，分数降低

```
GET /forum/article/_search 
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "content": "java"
          }
        }
      ],
      "must_not": [
        {
          "match": {
            "content": "spark"
          }
        }
      ]
    }
  }
}
```

通过negative boost 进行设置

```
GET /forum/article/_search 
{
  "query": {
    "boosting": {
      "positive": {
        "match": {
          "content": "java"
        }
      },
      "negative": {
        "match": {
          "content": "spark"
        }
      },
      "negative_boost": 0.2
    }
  }
}
```

negative 的 doc, 会乘以negative_boost, 降低分数

4. constant_score

如果不需要相关度评分, 那么使用constant_score加fileter, 所有的doc分数就是1, 没有评分概念.

```
GET /forum/article/_search 
{
  "query": {
    "bool": {
      "should": [
        {
          "constant_score": {
            "query": {
              "match": {
                "title": "java"
              }
            }
          }
        },
        {
          "constant_score": {
            "query": {
              "match": {
                "title": "spark"
              }
            }
          }
        }
      ]
    }
  }
}
```

## 使用function_score自定义相关度分数算法

自定义一个function_score函数, 自己将某个field的值, 跟es内置算出来的分数进行运算, 然后由自己指定的field来进行分数的增强.

给所有的帖子增加follower数量
```
POST /forum/article/_bulk
{ "update": { "_id": "1"} }
{ "doc" : {"follower_num" : 5} }
{ "update": { "_id": "2"} }
{ "doc" : {"follower_num" : 10} }
{ "update": { "_id": "3"} }
{ "doc" : {"follower_num" : 25} }
{ "update": { "_id": "4"} }
{ "doc" : {"follower_num" : 3} }
{ "update": { "_id": "5"} }
{ "doc" : {"follower_num" : 60} }
```

对帖子搜索到的分数, 跟follower_num进行运算, 由follower_num 在一定成都商增强帖子的分数.
看帖子的人越多, 那么帖子的分数越高.

```
GET /forum/article/_search
{
  "query": {
    "function_score": {
      "query": {
        "multi_match": {
          "query": "java spark",
          "fields": ["tile", "content"]
        }
      },
      "field_value_factor": {
        "field": "follower_num",
        "modifier": "log1p",
        "factor": 0.5
      },
      "boost_mode": "sum",
      "max_boost": 2
    }
  }
}
```

如果只有field，那么会将每个doc的分数都乘以follower_num，如果有的doc follower是0，那么分数就会变为0，效果很不好。因此一般会加个log1p函数，公式会变为，new_score = old_score * log(1 + number_of_votes)，分数这样计算会比较合理

再增加一个factor, 可以进一步影响分数, new_score = old_score * log(1 + factor * number_of_votes)

boost_mode : 可以决定分数和指定字段的值如何计算, multiply, sum, min, max, replace.

max_boost : 限制计算出的分数不能超过max_boost指定的值.

## 错误拼写时fuzzy模糊搜索技术

当搜索的时候, 可能会出现问错拼写错误的情况

doc1 : hello world  
doc2 : hello java

eg ：搜索 hallo world

fuzzy搜索技术 : 自动将拼写错误的搜索文本, 进行纠正, 纠正以后尝试匹配索引中的数据.

批量添加数据
```
POST /my_index/my_type/_bulk
{ "index": { "_id": 1 }}
{ "text": "Surprise me!"}
{ "index": { "_id": 2 }}
{ "text": "That was surprising."}
{ "index": { "_id": 3 }}
{ "text": "I wasn't surprised."}
```
请求
```
GET /my_index/my_type/_search 
{
  "query": {
    "fuzzy": {
      "text": {
        "value": "surprize",
        "fuzziness": 2
      }
    }
  }
}
```

surprize  --> 拼写错误 --> surprise --> s --> z  
surprize  --> surprised --> z --> s, 末尾加d, 纠正２次, 也可以匹配上, 在fuziness指定的２格范围内  
surprize  --> surprising --> z --> s, 去掉e, ing, 3此, 总共要５次, 才可以匹配上, 始终纠正不了

fuzzy搜索以后, 会自动尝试你的搜索文本进行纠错, 然后去跟文本进行匹配.

fuzziness 搜索文本最多可以纠正几个字母跟数据进行匹配, 如果不设置, 默认就是2.

eg : 
```
GET /my_index/my_type/_search 
{
  "query": {
    "match": {
      "text": {
        "query": "SURPIZE ME",
        "fuzziness": "AUTO",
        "operator": "and"
      }
    }
  }
}
```

## IK中文分词核心

1、ik分词器示例

中国人很喜欢吃油条  
standard：中 国 人 很 喜 欢 吃 油 条  
ik：中国人 很 喜欢 吃 油条  


2、ik分词器基础知识

两种analyzer，你根据自己的需要自己选吧，但是一般是选用ik_max_word

ik_max_word: 会将文本做最细粒度的拆分，比如会将“中华人民共和国国歌”拆分为“中华人民共和国,中华人民,中华,华人,人民共和国,人民,人,民,共和国,共和,和,国国,国歌”，会穷尽各种可能的组合；

ik_smart: 会做最粗粒度的拆分，比如会将“中华人民共和国国歌”拆分为“中华人民共和国,国歌”。

中文分词器使用
添加索引
```
PUT /my_index 
{
  "mappings": {
    "my_type": {
      "properties": {
        "text": {
          "type": "text",
          "analyzer": "ik_max_word"
        }
      }
    }
  }
}
```

添加数据
```
POST /my_index/my_type/_bulk
{ "index": { "_id": "1"} }
{ "text": "男子偷上万元发红包求交女友 被抓获时仍然单身" }
{ "index": { "_id": "2"} }
{ "text": "16岁少女为结婚“变”22岁 7年后想离婚被法院拒绝" }
{ "index": { "_id": "3"} }
{ "text": "深圳女孩骑车逆行撞奔驰 遭索赔被吓哭(图)" }
{ "index": { "_id": "4"} }
{ "text": "女人对护肤品比对男票好？网友神怼" }
{ "index": { "_id": "5"} }
{ "text": "为什么国内的街道招牌用的都是红黄配？" }
```

```
GET /my_index/_analyze
{
  "text": "男子偷上万元发红包求交女友 被抓获时仍然单身",
  "analyzer": "ik_max_word"
}
```

## ik分词器配置和自定义词库

1、ik配置文件

ik配置文件地址：es/plugins/ik/config目录

IKAnalyzer.cfg.xml：用来配置自定义词库
main.dic：ik原生内置的中文词库，总共有27万多条，只要是这些单词，都会被分在一起
quantifier.dic：放了一些单位相关的词
suffix.dic：放了一些后缀
surname.dic：中国的姓氏
stopword.dic：英文停用词

ik原生最重要的两个配置文件

main.dic：包含了原生的中文词语，会按照这个里面的词语去分词  
stopword.dic：包含了英文的停用词

停用词，stopword

a the and at but

一般，像停用词，会在分词的时候，直接被干掉，不会建立在倒排索引中

2、自定义词库  
(1) 自己建立的词库 : 每年都要涌出一些流行词, 网红, 等等 一般不会出现ik的原生词典里
需要自己补充最新的词语到ik

IKAnalyzer.cfg.xml：ext_dict，custom/mydict.dic

补充自己的词语，然后需要重启es，才能生效

(2) 自己建立停用词库：比如了，的，啥，么，我们可能并不想去建立索引，让人家搜索

custom/ext_stopword.dic，已经有了常用的中文停用词，可以补充自己的停用词，然后重启es

## 修改ik分词器源码来基于mysql热更新词库

热更新词库

每次都是在es的扩展词典中, 手动添加新词语, 麻烦.  
(1). 每次添加完毕, 需要重启es才能生效.  
(2). es是分布式的,可能有数百个节点, 无法一个节点一个节点上去修改.

es 不停机, 直接在外部的某个地方添加新的词语, es中立即热加载到这些新词语.

热更新方案  
(1). 修改ik分词器源码, 然后手动支持从mysql, 每个一段时间,自动加载.  


## 聚合数据分析 _bucket与 metric 两个核心概念讲解

两个核心概念: bucket 好 metric
bucket : 一个数据分组

| city | name |
| :---: | :----: |
| 北京 | 小李 |
| 北京 | 小王 |
| 上海 | 小张 |
| 上海 | 小丽 |
| 上海 | 小陈 |

基于city划分buckets

划分出来两个bucket, 一个是北京bucket, 一个是上海bucket

北京bucket : 包含了2个人, 小李, 小王.  
上海bucket : 包含了3个人, 小张, 小丽, 小陈.

按照某个字段进行bucket划分, 那个字段的值相同的那些数据, 就会被划分到一个bucket中.

mysql, 聚合, 首先第一步是分组, 对每个组内的数据进行聚合分析, 分组, 就是我们的bucket

metric : 对一个数据分组执行的统计

当我们有了一堆bucket之后，就可以对每个bucket中的数据进行聚合分词了，比如说计算一个bucket内所有数据的数量，或者计算一个bucket内所有数据的平均值，最大值，最小值

metric，就是对一个bucket执行的某种聚合分析的操作，比如说求平均值，求最大值，求最小值

select count(*) from access_log group by user_id

bucket：group by user_id --> 那些user_id相同的数据，就会被划分到一个bucket中

metric：count(*)，对每个user_id bucket中所有的数据，计算一个数量

## 聚合数据分析统计案例

案例:
以一个家电卖场中的电视销售数据为背景, 来对各个品牌, 各种颜色的电视销量和销售额, 进行各种角度的分析.

```
PUT /tvs
{
	"mappings": {
		"sales": {
			"properties": {
				"price": {
					"type": "long"
				},
				"color": {
					"type": "keyword"
				},
				"brand": {
					"type": "keyword"
				},
				"sold_date": {
					"type": "date"
				}
			}
		}
	}
}
```
keyword 是不分词的.

```
POST /tvs/sales/_bulk
{ "index": {}}
{ "price" : 1000, "color" : "红色", "brand" : "长虹", "sold_date" : "2016-10-28" }
{ "index": {}}
{ "price" : 2000, "color" : "红色", "brand" : "长虹", "sold_date" : "2016-11-05" }
{ "index": {}}
{ "price" : 3000, "color" : "绿色", "brand" : "小米", "sold_date" : "2016-05-18" }
{ "index": {}}
{ "price" : 1500, "color" : "蓝色", "brand" : "TCL", "sold_date" : "2016-07-02" }
{ "index": {}}
{ "price" : 1200, "color" : "绿色", "brand" : "TCL", "sold_date" : "2016-08-19" }
{ "index": {}}
{ "price" : 2000, "color" : "红色", "brand" : "长虹", "sold_date" : "2016-11-05" }
{ "index": {}}
{ "price" : 8000, "color" : "红色", "brand" : "三星", "sold_date" : "2017-01-01" }
{ "index": {}}
{ "price" : 2500, "color" : "蓝色", "brand" : "小米", "sold_date" : "2017-02-12" }
```

统计哪种颜色电视销量最高..
```
GET /tvs/sales/_search
{
    "size" : 0,
    "aggs" : { 
        "popular_colors" : { 
            "terms" : { 
              "field" : "color"
            }
        }
    }
}
```

size : 只获取聚合结果, 而不要执行聚合的原始数据  
aggs: 固定语法, 要对一份数据执行分组聚合操作  
popular_colors: 就是对每个aggs, 都要起一个名字, 这个名字是随机的, 你随便取什么都ok.  
terms : 根据字段的值进行分组.  
field : 根据指定的字段的值进行分组.  

```
{
  "took": 61,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 8,
    "max_score": 0,
    "hits": []
  },
  "aggregations": {
    "popular_color": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": [
        {
          "key": "红色",
          "doc_count": 4
        },
        {
          "key": "绿色",
          "doc_count": 2
        },
        {
          "key": "蓝色",
          "doc_count": 2
        }
      ]
    }
  }
}
```

hits.hits : 我们指定了size是0, 所以hits.hits就是空的, 否则会把执行聚合的那些原始数据给你返回回来.  
aggregations：聚合结果  
popular_color：我们指定的某个聚合的名称  
buckets：根据我们指定的field划分出的buckets  
key：每个bucket对应的那个值  
doc_count：这个bucket分组内，有多少个数据  
数量，其实就是这种颜色的销量  

每种颜色对应的bucket中的数据的  
默认的排序规则：按照doc_count降序排序


案例2 :
统计每种颜色电视平均价格

统计价格 平均数
```
GET /tvs/sales/_search
{
   "size" : 0,
   "aggs": {
      "colors": {
         "terms": {
            "field": "color"
         },
         "aggs": { 
            "avg_price": { 
               "avg": {
                  "field": "price" 
               }
            }
         }
      }
   }
}
```

按照color去分bucket, 可以拿到color bucket中的数量, 这个仅仅只是一个bucket操作, doc_count其实只是es的bucket操作默认执行的一个内置metric.

这一讲,就是除了bucket操作, 分组, 还要对每个bucket执行一个metric聚合统计操作

在一个aggs执行的bucket操作(terms), 平级的json结构下, 再加一个aggs, 这个第二个aggs内部, 同样取个名字, 执行一个metric操作, avg,  对之前的每个bucket中的数据的指定field, price field,  求一个平均值.

```
"aggs": { 
   "avg_price": { 
      "avg": {
         "field": "price" 
      }
   }
}
```

就是一个metric，就是一个对一个bucket分组操作之后，对每个bucket都要执行的一个metric

第一个metric，avg，求指定字段的平均值

```
{
  "took": 28,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 8,
    "max_score": 0,
    "hits": []
  },
  "aggregations": {
    "group_by_color": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": [
        {
          "key": "红色",
          "doc_count": 4,
          "avg_price": {
            "value": 3250
          }
        },
        {
          "key": "绿色",
          "doc_count": 2,
          "avg_price": {
            "value": 2100
          }
        },
        {
          "key": "蓝色",
          "doc_count": 2,
          "avg_price": {
            "value": 2000
          }
        }
      ]
    }
  }
}
```

buckets 除了key和doc_count
avg_price : 我们自己取的metric aggs的名字
value：我们的metric计算的结果，每个bucket中的数据的price字段求平均值后的结果

类似sql语句 : select avg(price) from tvs.sales group by color


案例3:

聚合数据分析_bucket嵌套实现颜色 + 生产商的多层下钻分析

从颜色到品牌进行下钻分析，每种颜色的平均价格，以及找到每种颜色每个品牌的平均价格

我们可以进行多层次的下钻

比如说，现在红色的电视有4台，同时这4台电视中，有3台是属于长虹的，1台是属于小米的

红色电视中的3台长虹的平均价格是多少？
红色电视中的1台小米的平均价格是多少？

下钻的意思是，已经分了一个组了，比如说颜色的分组，然后还要继续对这个分组内的数据，再分组，比如一个颜色内，还可以分成多个不同的品牌的组，最后对每个最小粒度的分组执行聚合分析操作，这就叫做下钻分析

es，下钻分析，就要对bucket进行多层嵌套，多次分组

按照多个维度（颜色+品牌）多层下钻分析，而且学会了每个下钻维度（颜色，颜色+品牌），都可以对每个维度分别执行一次metric聚合操作

下钻分析数据请求
```
GET /tvs/sales/_search 
{
  "size": 0,
  "aggs": {
    "group_by_color": {
      "terms": {
        "field": "color"
      },
      "aggs": {
        "color_avg_price": {
          "avg": {
            "field": "price"
          }
        },
        "group_by_brand": {
          "terms": {
            "field": "brand"
          },
          "aggs": {
            "brand_avg_price": {
              "avg": {
                "field": "price"
              }
            }
          }
        }
      }
    }
  }
}
```

响应返回
```
{
  "took": 8,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 8,
    "max_score": 0,
    "hits": []
  },
  "aggregations": {
    "group_by_color": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": [
        {
          "key": "红色",
          "doc_count": 4,
          "color_avg_price": {
            "value": 3250
          },
          "group_by_brand": {
            "doc_count_error_upper_bound": 0,
            "sum_other_doc_count": 0,
            "buckets": [
              {
                "key": "长虹",
                "doc_count": 3,
                "brand_avg_price": {
                  "value": 1666.6666666666667
                }
              },
              {
                "key": "三星",
                "doc_count": 1,
                "brand_avg_price": {
                  "value": 8000
                }
              }
            ]
          }
        },
        {
          "key": "绿色",
          "doc_count": 2,
          "color_avg_price": {
            "value": 2100
          },
          "group_by_brand": {
            "doc_count_error_upper_bound": 0,
            "sum_other_doc_count": 0,
            "buckets": [
              {
                "key": "TCL",
                "doc_count": 1,
                "brand_avg_price": {
                  "value": 1200
                }
              },
              {
                "key": "小米",
                "doc_count": 1,
                "brand_avg_price": {
                  "value": 3000
                }
              }
            ]
          }
        },
        {
          "key": "蓝色",
          "doc_count": 2,
          "color_avg_price": {
            "value": 2000
          },
          "group_by_brand": {
            "doc_count_error_upper_bound": 0,
            "sum_other_doc_count": 0,
            "buckets": [
              {
                "key": "TCL",
                "doc_count": 1,
                "brand_avg_price": {
                  "value": 1500
                }
              },
              {
                "key": "小米",
                "doc_count": 1,
                "brand_avg_price": {
                  "value": 2500
                }
              }
            ]
          }
        }
      ]
    }
  }
}
```

## 聚合数据分析 更多metrics: 统计每种颜色电视最大最小价格

更多 metric

count, avg

count : bucket, terms 自动就会有一个doc_count, 就相当于count
avg : avg aggs 求平均值
max : 求一个bucket内, 指定field值最大的那个数据
min : 求一个bucket内, 指定field值最小的那个数据
sum :  求一个bucket内, 指定field值的总和

一般来说, 90%的常见数据分析的操作, metric, 无非就是count, avg, max, min, sum

```
GET /tvs/sales/_search
{
   "size" : 0,
   "aggs": {
      "colors": {
         "terms": {
            "field": "color"
         },
         "aggs": {
            "avg_price": { "avg": { "field": "price" } },
            "min_price" : { "min": { "field": "price"} }, 
            "max_price" : { "max": { "field": "price"} },
            "sum_price" : { "sum": { "field": "price" } } 
         }
      }
   }
}
```

求总和, 就可以拿到一个颜色下的所有电视的销售总额

```
{
  "took": 16,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 8,
    "max_score": 0,
    "hits": []
  },
  "aggregations": {
    "group_by_color": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": [
        {
          "key": "红色",
          "doc_count": 4,
          "max_price": {
            "value": 8000
          },
          "min_price": {
            "value": 1000
          },
          "avg_price": {
            "value": 3250
          },
          "sum_price": {
            "value": 13000
          }
        },
        {
          "key": "绿色",
          "doc_count": 2,
          "max_price": {
            "value": 3000
          },
          "min_price": {
            "value":
          }, 1200
          "avg_price": {
            "value": 2100
          },
          "sum_price": {
            "value": 4200
          }
        },
        {
          "key": "蓝色",
          "doc_count": 2,
          "max_price": {
            "value": 2500
          },
          "min_price": {
            "value": 1500
          },
          "avg_price": {
            "value": 2000
          },
          "sum_price": {
            "value": 4000
          }
        }
      ]
    }
  }
}
```

## 聚合数据分析 实战hitogram 按价格区间

histogram：类似于terms，也是进行bucket分组操作，接收一个field，按照这个field的值的各个范围区间，进行bucket分组操作

eg :
```
"histogram":{ 
  "field": "price",
  "interval": 2000
},
```

interval : 2000 划分范围, 0~2000, 2000~4000, 4000~6000, 6000~8000, 8000~10000, buckets

去根据price的值, 比如2500, 看落在哪个区间内, 如 2000~4000, 此时将会将这条数据放入2000~4000对应的哪个bucket中.

bucket有了之后，一样的，去对每个bucket执行avg，count，sum，max，min，等各种metric操作，聚合分析

请求
```
GET /tvs/sales/_search
{
   "size" : 0,
   "aggs":{
      "price":{
         "histogram":{ 
            "field": "price",
            "interval": 2000
         },
         "aggs":{
            "revenue": {
               "sum": { 
                 "field" : "price"
               }
             }
         }
      }
   }
}
```

响应
```
{
  "took": 13,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 8,
    "max_score": 0,
    "hits": []
  },
  "aggregations": {
    "group_by_price": {
      "buckets": [
        {
          "key": 0,
          "doc_count": 3,
          "sum_price": {
            "value": 3700
          }
        },
        {
          "key": 2000,
          "doc_count": 4,
          "sum_price": {
            "value": 9500
          }
        },
        {
          "key": 4000,
          "doc_count": 0,
          "sum_price": {
            "value": 0
          }
        },
        {
          "key": 6000,
          "doc_count: {
            "value":": 0,
          "sum_price" 0
          }
        },
        {
          "key": 8000,
          "doc_count": 1,
          "sum_price": {
            "value": 8000
          }
        }
      ]
    }
  }
}
```

## 聚合数据分析 实战date hitogram 之统计每月电视销量

bucket 分组操作, histogram, 按照某个值的interval, 划分一个一个的bucket

date histogram, 按照我们指定的某个date类型的日期field, 以及日期interval, 按照一定的日期间隔, 去划分bucket

date interval = 1m,

2017-01-01~2017-01-31, 就是一个bucket
2017-02-01~2017-02-28, 就是一个bucket

然后会去扫描每个数据的date field, 判断date 落在哪个bucket中, 就将其放入那个bucket

2017-01-05, 就将其放入2017-01-01～2017-01-31, 就是一个bucket

min_doc_count : 即使某个日期interval, 2017-01-01~2017-01-31中, 一条数据都没有, 那么区间也要返回的, 不然默认是会过滤掉这个区间extended_bounds, min, max, 划分bucket的时候, 会限定在这个起始日期,和截止日期内

请求
```
GET /tvs/sales/_search
{
   "size" : 0,
   "aggs": {
      "sales": {
         "date_histogram": {
            "field": "sold_date",
            "interval": "month", 
            "format": "yyyy-MM-dd",
            "min_doc_count" : 0, 
            "extended_bounds" : { 
                "min" : "2016-01-01",
                "max" : "2017-12-31"
            }
         }
      }
   }
}
```

响应
```
{
  "took": 16,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 8,
    "max_score": 0,
    "hits": []
  },
  "aggregations": {
    "group_by_sold_date": {
      "buckets": [
        {
          "key_as_string": "2016-01-01",
          "key": 1451606400000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-02-01",
          "key": 1454284800000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-03-01",
          "key": 1456790400000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-04-01",
          "key": 1459468800000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-05-01",
          "key": 1462060800000,
          "doc_count": 1
        },
        {
          "key_as_string": "2016-06-01",
          "key": 1464739200000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-07-01",
          "key": 1467331200000,
          "doc_count": 1
        },
        {
          "key_as_strin
          "key_as_string": "2016-09-01",
          "key": 1472688000000,
          "doc_count": 0
        },g": "2016-08-01",
          "key": 1470009600000,
          "doc_count": 1
        },
        {
        {
          "key_as_string": "2016-10-01",
          "key": 1475280000000,
          "doc_count": 1
        },
        {
          "key_as_string": "2016-11-01",
          "key": 1477958400000,
          "doc_count": 2
        },
        {
          "key_as_string": "2016-12-01",
          "key": 1480550400000,
          "doc_count": 0
        },
        {
          "key_as_string": "2017-01-01",
          "key": 1483228800000,
          "doc_count": 1
        },
        {
          "key_as_string": "2017-02-01",
          "key": 1485907200000,
          "doc_count": 1
        }
      ]
    }
  }
}
```

## 深入聚合数据分析 _下钻分析之统计每个季度每个品牌的销售额.

添加时间和下钻分析数据

请求
```
GET /tvs/sales/_search 
{
  "size": 0,
  "aggs": {
    "group_by_sold_date": {
      "date_histogram": {
        "field": "sold_date",
        "interval": "quarter",
        "format": "yyyy-MM-dd",
        "min_doc_count": 0,
        "extended_bounds": {
          "min": "2016-01-01",
          "max": "2017-12-31"
        }
      },
      "aggs": {
        "group_by_brand": {
          "terms": {
            "field": "brand"
          },
          "aggs": {
            "sum_price": {
              "sum": {
                "field": "price"
              }
            }
          }
        },
        "total_sum_price": {
          "sum": {
            "field": "price"
          }
        }
      }
    }
  }
}
```

获取响应
```
{
  "took": 10,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 8,
    "max_score": 0,
    "hits": []
  },
  "aggregations": {
    "group_by_sold_date": {
      "buckets": [
        {
          "key_as_string": "2016-01-01",
          "key": 1451606400000,
          "doc_count": 0,
          "total_sum_price": {
            "value": 0
          },
          "group_by_brand": {
            "doc_count_error_upper_bound": 0,
            "sum_other_doc_count": 0,
            "buckets": []
          }
        },
        {
          "key_as_string": "2016-04-01",
          "key": 1459468800000,
          "doc_count": 1,
          "total_sum_price": {
            "value": 3000
          },
          "group_by_brand": {
            "doc_count_error_upper_bound": 0,
            "sum_other_doc_count": 0,
            "buckets": [
              {
                "key": "小米",
                "doc_count": 1,
                "sum_price": {
                  "value": 3000
                }
              }
            ]
          }
        },
        {
          "key_as_string": "2016-07-01",
          "key": 1467331200000,
          "doc_count": 2,
          "total_sum_price": {
            "value": 2700
          },
          "group_by_brand": {
            "doc_count_error_upper_bound": 0,
            "sum_other_doc_count": 0,
            "buckets": [
              {
                "key": "TCL",
                "doc_count": 2,
                "sum_price": {
                  "value": 2700
                }
              }
            ]
          }
        },
        {
          "key_as_string": "2016-10-01",
          "key": 1475280000000,
          "doc_count": 3,
          "total_sum_price": {
            "value": 5000
          },
          "group_by_brand": {
            "doc_count_error_upper_bound": 0,
            "sum_other_doc_count": 0,
            "buckets": [
              {
                "key": "长虹",
                "doc_count": 3,
                "sum_price": {
                  "value": 5000
                }
              }
            ]
          }
        },
        {
          "key_as_string": "2017-01-01",
          "key": 1483228800000,
          "doc_count": 2,
          "total_sum_price": {
            "value": 10500
          },
          "group_by_brand": {
            "doc_count_error_upper_bound": 0,
            "sum_other_doc_count": 0,
            "buckets": [
              {
                "key": "三星",
                "doc_count": 1,
                "sum_price": {
                  "value": 8000
                }
              },
              {
                "key": "小米",
                "doc_count": 1,
                "sum_price": {
                  "value": 2500
                }
              }
            ]
          }
        },
        {
          "key_as_string": "2017-04-01",
          "key": 1491004800000,
          "doc_count": 0,
          "total_sum_price": {
            "value": 0
          },
          "group_by_brand": {
            "doc_count_error_upper_bound": 0,
            "sum_other_doc_count": 0,
            "buckets": []
          }
        },
        {
          "key_as_string": "2017-07-01",
          "key": 1498867200000,
          "doc_count": 0,
          "total_sum_price": {
            "value": 0
          },
          "group_by_brand": {
            "doc_count_error_upper_bound": 0,
            "sum_other_doc_count": 0,
            "buckets": []
          }
        },
        {
          "key_as_string": "2017-10-01",
          "key": 1506816000000,
          "doc_count": 0,
          "total_sum_price": {
            "value": 0
          },
          "group_by_brand": {
            "doc_count_error_upper_bound": 0,
            "sum_other_doc_count": 0,
            "buckets": []
          }
        }
      ]
    }
  }
}
```

## 聚合数据分析, 搜索 + 聚合 : 统计指定品牌下每个颜色的销量

组合聚合使用

sql 语句 : select count(*) from tvs.sales where brand like "%长%" group by price.

es aggregation, scope, 任何聚合, 都必须在搜索出来的结果数据中之行, 搜索结果, 就是聚合分析操作的scope

请求
```
GET /tvs/sales/_search 
{
  "size": 0,
  "query": {
    "term": {
      "brand": {
        "value": "小米"
      }
    }
  },
  "aggs": {
    "group_by_color": {
      "terms": {
        "field": "color"
      }
    }
  }
}
```

聚合分析响应, 对搜索结果进行聚合
```
{
  "took": 5,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 2,
    "max_score": 0,
    "hits": []
  },
  "aggregations": {
    "group_by_color": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": [
        {
          "key": "绿色",
          "doc_count": 1
        },
        {
          "key": "蓝色",
          "doc_count": 1
        }
      ]
    }
  }
}
```

## 聚合数据分析 _global bucket 单个品牌与所有品牌销量对比

aggregation, scope, 一个聚合操作, 必须在query的搜索结果范围内执行

出来两个结果, 一个结果, 是基于query搜索结果来聚合的; 一个结果, 是对所有的数据执行聚合的.

```
GET /tvs/sales/_search 
{
  "size": 0, 
  "query": {
    "term": {
      "brand": {
        "value": "长虹"
      }
    }
  },
  "aggs": {
    "single_brand_avg_price": {
      "avg": {
        "field": "price"
      }
    },
    "all": {
      "global": {},
      "aggs": {
        "all_brand_avg_price": {
          "avg": {
            "field": "price"
          }
        }
      }
    }
  }
}
```

global：就是global bucket，就是将所有数据纳入聚合的scope，而不管之前的query

```
{
  "took": 4,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 3,
    "max_score": 0,
    "hits": []
  },
  "aggregations": {
    "all": {
      "doc_count": 8,
      "all_brand_avg_price": {
        "value": 2650
      }
    },
    "single_brand_avg_price": {
      "value": 1666.6666666666667
    }
  }
}
```
single_brand_avg_price：就是针对query搜索结果，执行的，拿到的，就是长虹品牌的平均价格
all.all_brand_avg_price：拿到所有品牌的平均价格

## 聚合数据分析 _过滤 + 聚合 : 统计价格大于1200的电视平均价格.

搜索 + 聚合
过滤 + 聚合

```
GET /tvs/sales/_search 
{
  "size": 0,
  "query": {
    "constant_score": {
      "filter": {
        "range": {
          "price": {
            "gte": 1200
          }
        }
      }
    }
  },
  "aggs": {
    "avg_price": {
      "avg": {
        "field": "price"
      }
    }
  }
}
```

```
{
  "took": 41,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 7,
    "max_score": 0,
    "hits": []
  },
  "aggregations": {
    "avg_price": {
      "value": 2885.714285714286
    }
  }
}
```

## 聚合数据分析 _bucket filter : 统计品牌最近一个月平均价格.

统计价格
```
GET /tvs/sales/_search 
{
  "size": 0,
  "query": {
    "term": {
      "brand": {
        "value": "长虹"
      }
    }
  },
  "aggs": {
    "recent_150d": {
      "filter": {
        "range": {
          "sold_date": {
            "gte": "now-150d"
          }
        }
      },
      "aggs": {
        "recent_150d_avg_price": {
          "avg": {
            "field": "price"
          }
        }
      }
    },
    "recent_140d": {
      "filter": {
        "range": {
          "sold_date": {
            "gte": "now-140d"
          }
        }
      },
      "aggs": {
        "recent_140d_avg_price": {
          "avg": {
            "field": "price"
          }
        }
      }
    },
    "recent_130d": {
      "filter": {
        "range": {
          "sold_date": {
            "gte": "now-130d"
          }
        }
      },
      "aggs": {
        "recent_130d_avg_price": {
          "avg": {
            "field": "price"
          }
        }
      }
    }
  }
}
```

aggs.filter, 针对聚合去做的.  
如果放query 里面的filter, 是全局的, 会对所有的数据有影响  
但是, 如果, 比如说, 你要统计, 长虹电视, 最近一个月的平均值; 最近3个月的平均值; 最近6个月的平均值.


## 聚合数据分析 _排序 : 按每种颜色的平均销售额降序排序

一般排序都是按每个bucket的doc_count降序来排的.

请求
```
GET /tvs/sales/_search 
{
  "size": 0,
  "aggs": {
    "group_by_color": {
      "terms": {
        "field": "color"
      },
      "aggs": {
        "avg_price": {
          "avg": {
            "field": "price"
          }
        }
      }
    }
  }
}
```

响应
```
{
  "took": 2,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 8,
    "max_score": 0,
    "hits": []
  },
  "aggregations": {
    "group_by_color": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": [
        {
          "key": "红色",
          "doc_count": 4,
          "avg_price": {
            "value": 3250
          }
        },
        {
          "key": "绿色",
          "doc_count": 2,
          "avg_price": {
            "value": 2100
          }
        },
        {
          "key": "蓝色",
          "doc_count": 2,
          "avg_price": {
            "value": 2000
          }
        }
      ]
    }
  }
}
```

但是假如说, 我们现在统计出来每个颜色的电视销售额, 需要按照销售额降序排序.

```
GET /tvs/sales/_search 
{
  "size": 0,
  "aggs": {
    "group_by_color": {
      "terms": {
        "field": "color",
        "order": {
          "avg_price": "asc"
        }
      },
      "aggs": {
        "avg_price": {
          "avg": {
            "field": "price"
          }
        }
      }
    }
  }
}
```

## 聚合数据分析 颜色 + 品牌 下钻分析时按最深层metric进行排序

按照下钻分析时最深证metric进行排序

请求
```
GET /tvs/sales/_search 
{
  "size": 0,
  "aggs": {
    "group_by_color": {
      "terms": {
        "field": "color"
      },
      "aggs": {
        "group_by_brand": {
          "terms": {
            "field": "brand",
            "order": {
              "avg_price": "desc"
            }
          },
          "aggs": {
            "avg_price": {
              "avg": {
                "field": "price"
              }
            }
          }
        }
      }
    }
  }
}
```

响应
```
{
  "took": 4,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 8,
    "max_score": 0,
    "hits": []
  },
  "aggregations": {
    "group_by_color": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": [
        {
          "key": "红色",
          "doc_count": 4,
          "group_by_brand": {
            "doc_count_error_upper_bound": 0,
            "sum_other_doc_count": 0,
            "buckets": [
              {
                "key": "三星",
                "doc_count": 1,
                "avg_price": {
                  "value": 8000
                }
              },
              {
                "key": "长虹",
                "doc_count": 3,
                "avg_price": {
                  "value": 1666.6666666666667
                }
              }
            ]
          }
        },
        {
          "key": "绿色",
          "doc_count": 2,
          "group_by_brand": {
            "doc_count_error_upper_bound": 0,
            "sum_other_doc_count": 0,
            "buckets": [
              {
                "key": "小米",
                "doc_count": 1,
                "avg_price": {
                  "value": 3000
                }
              },
              {
                "key": "TCL",
                "doc_count": 1,
                "avg_price": {
                  "value": 1200
                }
              }
            ]
          }
        },
        {
          "key": "蓝色",
          "doc_count": 2,
          "group_by_brand": {
            "doc_count_error_upper_bound": 0,
            "sum_other_doc_count": 0,
            "buckets": [
              {
                "key": "小米",
                "doc_count": 1,
                "avg_price": {
                  "value": 2500
                }
              },
              {
                "key": "TCL",
                "doc_count": 1,
                "avg_price": {
                  "value": 1500
                }
              }
            ]
          }
        }
      ]
    }
  }
}
```

## 聚合数据分析 cardinality 去重以及统计每月销售电视品牌数量

* ES, 去重, cardinality metric, 对每个bucket中指定field进行去重, 取去重后的count, 类似count(discint)

去重请求
```
GET /tvs/sales/_search
{
  "size" : 0,
  "aggs" : {
      "months" : {
        "date_histogram": {
          "field": "sold_date",
          "interval": "month"
        },
        "aggs": {
          "distinct_colors" : {
              "cardinality" : {
                "field" : "brand"
              }
          }
        }
      }
  }
}
```

响应
```
{
  "took": 70,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 8,
    "max_score": 0,
    "hits": []
  },
  "aggregations": {
    "group_by_sold_date": {
      "buckets": [
        {
          "key_as_string": "2016-05-01T00:00:00.000Z",
          "key": 1462060800000,
          "doc_count": 1,
          "distinct_brand_cnt": {
            "value": 1
          }
        },
        {
          "key_as_string": "2016-06-01T00:00:00.000Z",
          "key": 1464739200000,
          "doc_count": 0,
          "distinct_brand_cnt": {
            "value": 0
          }
        },
        {
          "key_as_string": "2016-07-01T00:00:00.000Z",
          "key": 1467331200000,
          "doc_count": 1,
          "distinct_brand_cnt": {
            "value": 1
          }
        },
        {
          "key_as_string": "2016-08-01T00:00:00.000Z",
          "key": 1470009600000,
          "doc_count": 1,
          "distinct_brand_cnt": {
            "value": 1
          }
        },
        {
          "key_as_string": "2016-09-01T00:00:00.000Z",
          "key": 1472688000000,
          "doc_count": 0,
          "distinct_brand_cnt": {
            "value": 0
          }
        },
        {
          "key_as_string": "2016-10-01T00:00:00.000Z",
          "key": 1475280000000,
          "doc_count": 1,
          "distinct_brand_cnt": {
            "value": 1
          }
        },
        {
          "key_as_string": "2016-11-01T00:00:00.000Z",
          "key": 1477958400000,
          "doc_count": 2,
          "distinct_brand_cnt": {
            "value": 1
          }
        },
        {
          "key_as_string": "2016-12-01T00:00:00.000Z",
          "key": 1480550400000,
          "doc_count": 0,
          "distinct_brand_cnt": {
            "value": 0
          }
        },
        {
          "key_as_string": "2017-01-01T00:00:00.000Z",
          "key": 1483228800000,
          "doc_count": 1,
          "distinct_brand_cnt": {
            "value": 1
          }
        },
        {
          "key_as_string": "2017-02-01T00:00:00.000Z",
          "key": 1485907200000,
          "doc_count": 1,
          "distinct_brand_cnt": {
            "value": 1
          }
        }
      ]
    }
  }
}
```

## 聚合数据分析 cardinality算法之优化内存开销以及HLL算法

* cardinality，count(distinct)，5%的错误率，性能在100ms左右

1. precision_threshold优化准确率和内存开销

```
GET /tvs/sales/_search
{
    "size" : 0,
    "aggs" : {
        "distinct_brand" : {
            "cardinality" : {
              "field" : "brand",
              "precision_threshold" : 100 
            }
        }
    }
}
```

* brand去重, 如果brand的unique value, 在100个以内, 小米, 长虹, 三星, TCL, HTL...
* 在多少个unique value以内, cardinality, 几乎保证100%准确
* cardinality算法, 会占用precision_threshold * 8 byte 内存消耗, 100 * 8 = 800个字节
* 占用内存很小... 而且unique value如果的确在值以内, 那么可以确保100%准确
* 100, 数百万的unique value, 错误率在5%以内
* precision_threshold, 值设置的越大, 占用内存越大, 1000 * 8 = 8000 / 1000 = 8KB, 可以确保更多unique value的场景下, 100%的准确

field，去重，count，这时候，unique value，10000，precision_threshold=10000，10000 * 8 = 80000个byte，80KB

2、HyperLogLog++ (HLL)算法性能优化

cardinality底层算法：HLL算法，HLL算法的性能

会对所有的uqniue value取hash值，通过hash值近似去求distcint count，误差

默认情况下，发送一个cardinality请求的时候，会动态地对所有的field value，取hash值; 将取hash值的操作，前移到建立索引的时候

建立索引
```
PUT /tvs/
{
  "mappings": {
    "sales": {
      "properties": {
        "brand": {
          "type": "text",
          "fields": {
            "hash": {
              "type": "murmur3" 
            }
          }
        }
      }
    }
  }
}
```
查询获取
```
GET /tvs/sales/_search
{
    "size" : 0,
    "aggs" : {
        "distinct_brand" : {
            "cardinality" : {
              "field" : "brand.hash",
              "precision_threshold" : 100 
            }
        }
    }
}
```

## 聚合数据分析 percentiles 百分比算法以及网站访问延时统计

案例 : 
比如有一个网站, 记录下每次请求的访问耗时, 需要统计tp50, tp90, tp99

tp50 : 50%的请求的耗时最长在多长时间.
tp90 : 90%的请求的耗时最长在多长时间
tp99 : 99%的请求的耗时最长在多长时间

添加索引
```
PUT /website
{
    "mappings": {
        "logs": {
            "properties": {
                "latency": {
                    "type": "long"
                },
                "province": {
                    "type": "keyword"
                },
                "timestamp": {
                    "type": "date"
                }
            }
        }
    }
}
```

批量插入数据
```
POST /website/logs/_bulk
{ "index": {}}
{ "latency" : 105, "province" : "江苏", "timestamp" : "2016-10-28" }
{ "index": {}}
{ "latency" : 83, "province" : "江苏", "timestamp" : "2016-10-29" }
{ "index": {}}
{ "latency" : 92, "province" : "江苏", "timestamp" : "2016-10-29" }
{ "index": {}}
{ "latency" : 112, "province" : "江苏", "timestamp" : "2016-10-28" }
{ "index": {}}
{ "latency" : 68, "province" : "江苏", "timestamp" : "2016-10-28" }
{ "index": {}}
{ "latency" : 76, "province" : "江苏", "timestamp" : "2016-10-29" }
{ "index": {}}
{ "latency" : 101, "province" : "新疆", "timestamp" : "2016-10-28" }
{ "index": {}}
{ "latency" : 275, "province" : "新疆", "timestamp" : "2016-10-29" }
{ "index": {}}
{ "latency" : 166, "province" : "新疆", "timestamp" : "2016-10-29" }
{ "index": {}}
{ "latency" : 654, "province" : "新疆", "timestamp" : "2016-10-28" }
{ "index": {}}
{ "latency" : 389, "province" : "新疆", "timestamp" : "2016-10-28" }
{ "index": {}}
{ "latency" : 302, "province" : "新疆", "timestamp" : "2016-10-29" }
```

通过pencentiles进行数据百分比筛选
```
GET /website/logs/_search 
{
  "size": 0,
  "aggs": {
    "latency_percentiles": {
      "percentiles": {
        "field": "latency",
        "percents": [
          50,
          95,
          99
        ]
      }
    },
    "latency_avg": {
      "avg": {
        "field": "latency"
      }
    }
  }
}
```

响应
```
{
  "took": 31,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 12,
    "max_score": 0,
    "hits": []
  },
  "aggregations": {
    "latency_avg": {
      "value": 201.91666666666666
    },
    "latency_percentiles": {
      "values": {
        "50.0": 108.5,
        "95.0": 508.24999999999983,
        "99.0": 624.8500000000001
      }
    }
  }
}
```

获取50%请求,数值的最大值
```
GET /website/logs/_search 
{
  "size": 0,
  "aggs": {
    "group_by_province": {
      "terms": {
        "field": "province"
      },
      "aggs": {
        "latency_percentiles": {
          "percentiles": {
            "field": "latency",
            "percents": [
              50,
              95,
              99
            ]
          }
        },
        "latency_avg": {
          "avg": {
            "field": "latency"
          }
        }
      }
    }
  }
}
```

响应
```
{
  "took": 33,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 12,
    "max_score": 0,
    "hits": []
  },
  "aggregations": {
    "group_by_province": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": [
        {
          "key": "新疆",
          "doc_count": 6,
          "latency_avg": {
            "value": 314.5
          },
          "latency_percentiles": {
            "values": {
              "50.0": 288.5,
              "95.0": 587.75,
              "99.0": 640.75
            }
          }
        },
        {
          "key": "江苏",
          "doc_count": 6,
          "latency_avg": {
            "value": 89.33333333333333
          },
          "latency_percentiles": {
            "values": {
              "50.0": 87.5,
              "95.0": 110.25,
              "99.0": 111.65
            }
          }
        }
      ]
    }
  }
}
```

聚合数据分析 percentiles rank 以及网站访问时延SLA统计

SLA : 就是我们提供的服务的标准

网站提供的访问延时的SLA, 确保所有的请求100%, 都必须在200ms以内, 大公司内, 一般都是要求100%在200ms以内

如果超过1s, 则需要升级到A级故障, 代表网站的访问性能和用户体验会下降.

所要达到的条件:
在200ms以内的, 有百分之多少, 在1000毫秒以内的有百分之多少, percentile ranks metric

percentile ranks, 其实pencentile还要常用.

按照品牌分组, 计算, 电视机, 售价在1000占比, 2000占比, 3000占比

请求
```
GET /website/logs/_search 
{
  "size": 0,
  "aggs": {
    "group_by_province": {
      "terms": {
        "field": "province"
      },
      "aggs": {
        "latency_percentile_ranks": {
          "percentile_ranks": {
            "field": "latency",
            "values": [
              200,
              1000
            ]
          }
        }
      }
    }
  }
}
```

响应
```
{
  "took": 38,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 12,
    "max_score": 0,
    "hits": []
  },
  "aggregations": {
    "group_by_province": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": [
        {
          "key": "新疆",
          "doc_count": 6,
          "latency_percentile_ranks": {
            "values": {
              "200.0": 29.40613026819923,
              "1000.0": 100
            }
          }
        },
        {
          "key": "江苏",
          "doc_count": 6,
          "latency_percentile_ranks": {
            "values": {
              "200.0": 100,
              "1000.0": 100
            }
          }
        }
      ]
    }
  }
}
```

percentile的优化

TDigest算法，用很多节点来执行百分比的计算，近似估计，有误差，节点越多，越精准

compression

限制节点数量最多 compression * 20 = 2000个node去计算

默认100

越大，占用内存越多，越精准，性能越差

一个节点占用32字节，100 * 20 * 32 = 64KB

如果你想要percentile算法越精准，compression可以设置的越大





## 聚合算法原理

1.  讲解易并行聚合算法 : max

* 有些聚合分析的算法, 是很容易就可以并行的, 比如说max  
* 有些聚合分析的算法, 是不好并行的, 比如说, count(distinct), 并不是说, 在每个node上, 直接就出一些distinct value, 就可以的, 因为数据可能会很多
* es会采取近似聚合的方式, 就是采用在每个node上进行近估计的方式, 得到最终的结论, cuont(distcint), 100万, 1050万/95万 --> 5%左右的错误率, 近似估计后的结果, 不完全准确, 但是速度会很快, 一般会达到完全精准的算法的性能的数十倍

2. 三角选择原则

* 精准+实时+大数据 --> 选择2个  
  (1). 精准+实时: 没有大数据，数据量很小，那么一般就是单击跑，随便你则么玩儿就可以  
  (2). 精准+大数据：hadoop，批处理，非实时，可以处理海量数据，保证精准，可能会跑几个小时  
  (3). 大数据+实时：es，不精准，近似估计，可能会有百分之几的错误率  

3. 近似聚合算法

* 如果采取近似估计的算法 : 延时在100ms左右, 0.5%错误
* 如果采取100%精准的算法 : 延时一般在5s~几十s, 甚至几十分钟, 几小时,  0%错误






