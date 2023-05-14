# ElasticSearch 原理

#### 倒排索引结构和索引不可变原因
倒排索引是适合用于进行搜索的

倒排索引结构  
(1). 包含这个关键词的document list  
(2). 包含这个关键词的所有document的数量 `IDF (inverse document frequency)`  
(3). 这个关键词在每个document中出现的次数  `TF (term frequency)`
(4). 这个关键词在每个document中出现的次序  
(5). 每个document的长度  
(6). 包含这个关键词的所有document的平均长度

绝大部分搜索就是进行结构的相关度评分

倒排索引数据不可变原因及优点
(1). 所有的修改操作都是先标记删除,后新增一个新数据,因为索引数据不可变, 所以不需要锁的问题, 提升并发效果.
(2). 数据不变, 一直保存在os cache中, 只要cache内存足够
(3). filter cache 一直驻留在内存中, 因为数据不变
(4). 可以压缩, 节省cpu和io开销

倒排索引数据不可变的缺点
（1). 如果倒排索引mapping结构变化那么就必须重新构建



#### doc Value 正排索引

搜索的时候需要倒排索引, 排序的时候, 需要依靠正排索引, 看到每个document的每个field, 然后进行排序, 所谓正排索引, 其实就是doc value
在建立索引的时候, 一方面会建立倒排索引, 以提供搜索, 一方面建立正排索引, 以供聚合, 过滤等操作使用.
doc value 是保存在磁盘上的, 如果内存足够则会保存在内存中, 提高性能.


#### _all metadata的原理和作用

语法: GET /index/type/_search?q=value
直接通过q直接搜索所有的field 只要包含value关键字 就可以搜索到
es中_all元数据, 在建立索引的时候,我们插入一条document,它里面包含了多个field, es会将多个filed的值全部用字符串拼接成一个长字符串, 作为_all field的值建立索引
所以在q不指定field的请求下默认搜索的是_all field的值


### mapping实质

mapping 就是index的type的元数据, 每个type都有一个自己的mapping, 决定了数据类型, 建立倒排索引的行为, 
(1) 如果不输入mapping 那么数据进入时就会进行dynamic mapping 自动建立index, 创建type, 以及对应的mapping, mapping中包含了每个field对应的数据类型, 以及如何分词等设置
(2) mapping中就自动定义了每个field的数据类型
(3) 不同的数据类型 (如 text 和 date) 有的是exact value 有的是 full text , full text 会进行分词和 normalization 再去 倒排索引中去搜索去搜索
(4) 可以用dynamic mapping 让其自动建立mapping 包括自动数据类型.
(5) 可以手动创建doc的mapping 对各自的field 进行设置包括数据类型和索引行为和分词器

mapping 核心的数据类型
string, byte, short, integer, long, float, double, boolean, date

dynamic mapping 数据对应
true or false  -> boolean
123 -> long
123.45 -> double
2017-01-01 -> date
"hello world" -> string

mapping 复杂数据类型
multivalue field
"{"tags":["tag1","tag2"]}"
建立索引时和string 是一样的, 数据类型不能混

empty field
null, [], [null]
这些就是empty field

object field
这里json所包含的对象 address 就是一个object类型
{
    "address" : {
        "country" : "china",
        "province" : "guangzhou",
        "city" : "guangzhou",
    },
    "name":"java",
    "join_date":"2019-10-19"
}
在底层显示中就会
{
    "name" : "java",
    "join_date" : "2019-10-10",
    "address.country" : "china",
    "address.province" : "guangzhou",
    "address.city" : "guangzhou"
}




#### 搜索索引 相关度评分 TF&IDF算法
relevance score 算法, 简单来说, 就是计算出, 一个索引的文本, 与搜索文本之间的匹配度
ElasticSearch 使用的是 term frequency/ inverse document frequency 算法, 简称TF/IDF算法

Term frequency : 搜索文本中的各个词条在field文本中出现了多少次,出现次数越多, 就越相关
Inverse document frequency : 搜索文本中各个词条在`整个索引`的所有文档中出现多少次, 出现的次数越多, 就越不相关
Field-length norm field长度, filed越长, 相关度越弱`(反之field长度越短, 相关度越强)`
使用执行计划查看评分执行过程  
```
GET /_search?explain
{
    "query" : {"match" : {"name" : "java"}}
}
```



#### document 写入原理
数据 --(document会被先写入缓存中)--> 内存buffer缓冲
  --(当触发某条件或隔段时间就会进行commit操作, 将数据提交到lucene中的segment中)--> index segment
--(index segment 会将数据写入os cache缓存中)--> os cache --(由fsync强制刷新os cache 到disk中, 并清空内存buffer缓冲)--> os disk

优化写入流程实现NRT近实时
当数据写入 os cache中时, 并被打开供搜索的过程, 叫做refresh, 默认是间隔1秒refresh一次, 也就是每隔1秒就会将buffer 数据写入到一个新的index segment file, 先写入os cache 中, 所以, es是近实时的, 数据写入到可以被搜索, 默认是1秒

POST /my_index/_refresh 可以手动refresh

如果对于某个索引数据时效性要求较低 那么可以修改refresh_interval的时间间隔
```
PUT /my_index
{
    "settings" : {
        "refresh_interval" : "30s"
    }
}
```


优化写入流程实现durability实现可靠存储(translog, flush)

```
          /-->内存buffer缓冲--每隔1s-->indexsegmentfile--(当segmentfile进入oscache后就被打开供search使用)-->oscache(并清空buffer)
         /
数据 ----/
        \
         \
          \-->translog日志文件(并commit)--(随着时间的推移translogs数据大到一定程度flush操作触发)-->将全部数据写入到一个segmentfile,刷入oscache 提供search
          -(第一个commit到磁盘上,标明有哪些index segment)->commit point -(最后所有filesystemcache中所有index segment file缓存被fsync刷新到磁盘上)->清空现有translog,并新建  
```

fsync会清空translog,就是flush，默认每隔30分钟,flush一次,或者当translog过大的时候也会flush

translog, 每隔5秒会fsync到磁盘上,在一次增删改请求后，当fsync在primary shard和replica shard都成功后,那次增删改才会成功.



优化写入流程实现海量磁盘文件合并(segment merge, optimize)

内存buffer中每隔1s会形成一个新的segment file, 文件过多,每次search要搜索所有的segment,很耗时
默认会在后台执行segment merge操作, 在merge的时候, 被标记为delete的document也会被彻底物理删除
segment merge执行流程
(1).选择一些有相似大小的segment merge成一个大的segment
(2).将新的segment flush到磁盘上
(3).写一个新的commit point包括新的segment,并排除就的那些segment
(5).将旧的segment删除



#### type 底层数据结构解密
type 是一个index中用来区分类似的数据, 类似的数据, 但可能有不同的fields, 而不同属性来控制索引建立, 分词器
field的 value 在底层的lucene 中建立索引的时候, 全部是 opaque bytes类 不同分类型的.
lucene是没有type的概念的, 在document中, 实际上将type作为document的field来存储, 即type es通过 type的 过滤和筛选
一个index中的多个type, 实际上放在一起存储的, 因此一个index下, 不能有多个type重名.



# ElasticSearch 小操作


#### ElasticSearch 乐观锁并发控制
1.ElasticSearch 内部基于_version进行乐观锁并发控制
(1)._version元数据
    第一次创建document的时候, 它的_version内部版本号就是1,以后每次对这个document执行修改或者删除操作都会对_version进行+1
    传递version进行锁控制


#### ElasticSearch PartialUpdate

POST index/type/id/_update
{
    "doc":{
        "field" : "value"
    }
}

// 所要修改的field传递即可,不需要全量数据.
// 内部执行几乎一样的全量替换但是在ES内部执行
// 好处是减少并发冲突的情况


#### paging 和 deep_paging

GET /_search?size=1
获取的数据中进行数据个数为1
GET /_search?size=1&from=2
获取数据从第第2条数据开始

Notice: mounting
deepPaging性能问题,以及原理.
deepPaging 深度搜索,比如总共有10000条数据,每页是10条数据
你的请求可能会发送到一个不包含index的shard的node上,这个node就是一个coordinate node 那么这个coordinate node 就会将搜索请求转发到index的三个shard上面node
我只拿出10000-10010条看起来好像只拿出了10条数据其实是拿出了10010条数据, 每个shard都会返回10010条数据,3个shards总共返回30030条数据给coordinate node 然后在这写数据中进行排序(按_score), 然后渠道排位后,会取到第1000页的10条,这10条数据, 然后就我们要的数据10000-10010条数据
这就是deepPaging


#### ElasticSearch multi-index 和 mutil-type
multi-index 和 mutil-type 搜索模式
告诉你如何一次性搜索多个index或多个type下的数据.

/_search 所有索引 所有type的数据全部获取
/index/_search 获取index索引下全部的type数据
/index1,index2/_search 获取指定index1和index2下的全部数据
/test*/_search 获取通配符下的全部数据
/index/type/_search 获取一个index type下的全部数据
/index/type1,type2/_search 获取一个
/index1,index2/type1,type2/_search 获取指定index1,index2下的
/_all/type1,type2/_search 所有index下 指定type的数据


#### ElasticSearch 批量查询

1. 批量查询的好处
    减少100次查询的网络开销

GET /_mget
```json
{
    "docs":[
    {
        "_index":"index",
        "_type":"type",
        "_id":1
    },
    {
        "_index":"test_index",
        "_type":"test_type",
        "_id":11
    }
    ]
}
```

获取上面两个doc数据

如果查询document是1个index下不同type种的话
GET /test_index/_mget
```json
{
    "docs":[
        {
            "_type":"test_type",
            "_id" : 1
        },
        {
            "_type": "test_type",
            "_id" : 2
        }
    ]
}
```
对比多次查询批量查询很有价值


#### ElasticSearch GroovyScript 修改

POST index/type/1/_update
{
    "script":"ctx._source.num+=1"
}
POST index/type/1/_update
{
    "script" : "ctx._source.num+=1",
        "upsert" : {
            "num" : 0,
            "tags" : []
        }
}
如果document存在就执行 script方法
如果document不存在就执行 upsert方法

#### ElasticSearch Document 
1.document的全量替换
(1).语法与创建文档是一样的,如果document id不存在,那么就是创建,如果document id 已经存在, 那么就是全量替换操作, 替换document的json串内容.
(2).document是不可变的,如果要修改document的内容,第一种方式就是全量替换,直接对document重新建立索引,替换所有内容.
(3).es将老去的数据标记为delete,然后新增我们给定的一个document,当我们创建越来越多document时,es会在适当的时机后台自动删除标记为delete的document

2.document的强制创建
(1).创建文档与全局替换的语法是一致的,有时候我们只是想新建文档,不想替换文档,PUT .../?op_type=create, PUT .../_create

3.document的删除
(1).不会理解为物理删除,只会标记为deleted,当数据越来越多的时候,在后台自动删除.

#### bulk api批量修改
批量修改
每个操作要有两个json串, 语法如下:
{"action":{"metadata"}}
{"data"}

eg: 创建docuemnt,放bulk里
{"index":{"_index":"test_index", "test_type":"_id", "_id":"1"}}
{"test_field1":"test1", "test_field2":"test2"}

POST /_bulk
bulk 对json语法有严格要求 不能换行, 同一个json串和一个json串之间必须要一个换行
有哪些类型的操作可以执行
(1).delete: 删除一个文档, 只需要1个json串就可以了
{"delete":{"_index":"test_index", "_type":"test_Type","_id":"3"}}
(2).create: PUT /index/type/id/_create 强制创建
{"craete":{"-index":"test_index", "_type":"test_type", "_id":"12"}}
{"test_field":"test12"}
(3).index: 不同的put操作,可以是创建文档,也可以是全量替换文档
{"index":{"_index":"test_index", "_type":"test_type","_id":"2"}}
{"test_field":"replace_test2"}
(4).update:执行partial update操作
{"update":{"_index":"test_index", "_type":"test_type", "_id":"1", "_retry_on_conflict":"3"}}
{"doc":{"test_field2":"bulk test1"}}


#### bulk api 的奇特json格式解密
{"action":...}
{"data"}

1. bulk中每个操作都比可能要转发到不同的node的shard去执行
2. 如果采取比较良好的json数组格式
允许任意的换行,整个可读性非常棒,读起来非常爽,es拿到这种标准格式的json串后,要按照下述流程去进行处理.
(1).将JSON数据解析为JSONArray对象,这个时候整个数据,就会在内存中出现一份一模一样的拷贝,一份json文本,一份JSONArray对象.
(2).解析json数组里的每个json,对每个请求中的document进行路由
(3).为路由到同一个shard上的多个请求,创建一个请求数组
(4).将这个请求数组序列化
(5).将序列化后的数据发送到对应的节点上去

3. 耗费更多的内存, 更多的jvm gc开销
eg:bulk_size 几千条, 大小都在10M,发布到节点100个,就是1G,复制为2G+更多的内存占用量

内存占用导致jvm gc 频繁耗时更多.

4. 奇特格式的好处
(1).不用再转换为json对象不会出现相同的数据拷贝,直接按照换行符,切割.
(2).对每两个一组的json,读取meta,进行document路由
(3).直接对应的json发送到node上去

不用拷贝数据拷贝 不会浪费内存 也不会提高gc消耗



#### form+size分页

我们通过_search进行数据获取后,通过size和from参数来确定选取数据的位置和大小
eg : GET /_search?size=10&from=10
这种分页方式只适合少量数据，因为随from增大，查询的时间就会越大，而且数据量越大，查询的效率指数下降

* 三个shards node上存储了30000条数据  
    1. 请求size=10 from=1000 发送到 coordinate node`(协调节点)`上, coordinate node 不处理将请求转发给所在shards node 上.
    2. 三个shards node 每一个都获取 10010条数据 并返回给 coordinate node.
    3. coordinate node 获取到30030条数,并按_score`(相关度分数)`进行排序.
    4. 最后获取的第1000页的10条数据并返回.

优点：from+size在数据量不大的情况下，效率比较高
缺点：数据量非常大的情况下，from+size分页会把全部记录加载到内存中，不但运行速递特别慢，es出现内存大量消耗,cpu大量消费,deepPaging的深度分页操作



#### scroll深分页

一次性获取大量的数据比如10万条数据, 那么性能会很差, 此时一般会采取用scroll滚动查询的方式.
使用scroll 滚动搜索, 可以先搜索一批数据, 然后搜索下一批, 直到搜索出全部的数据.
scroll搜索会在第一次搜索时, 保存一个视图快照, 之后只会基于该旧的视图快照提供数据搜索.
基于_doc 进行排序的方式, 性能较高
每次发送scroll请求, 我们还需要指定一个scroll参数,每次搜索请求只要在这个时间窗口内完成就可以了
 ```
GET my_index/my_type/_search?scroll=1m
{
    "query" : {
        "match_all" : {}
    },
    "sort" : ["_doc"],
    "size" : 3
}
```
返回一个包含scroll_id 的json

下次请求时 携带scroll_id 并且会附带上次的条件
```
GET /_search/scroll
{
    "scroll": "1m",
    "scroll_id" : "..."
}
```


#### 基于scroll + bulk + 索引别名实现零停机重建索引
一个field的设置是不能被修改的, 如果要修改一个field, 那么应该重新按照新的mapping, 建立一个index, 然后将数据批量查询出来, 重新用bulk api 写入index中
批量查询的时候, 建议采用scroll api, 并且采用多线程并发方式来reindex数据, 每次scroll就查询指定日期的一段数据, 交给一个线程即可.

PUT /my_index/my_type/1
{
  "title": "2017-01-01"
}

当dynamic mapping插入数据并将title自动映射为date格式, 但我们希望是string类型, 当插入string类型时就会报错
此时想修改title类型 是无法做到的, 唯一修改的办法就是reindex, 也就是重新建立一个索引, 将旧索引查询出来再重新导入索引

所以给应用起一个别名, good_index 这里应用使用的good_index就是原my_index
PUT /my_index/_alias/good_index

(1). 新建一个Index, 调整其title的类型为正确的string, 并命名为new添加新索引
```
PUT /my_index_new
{
    "mapping" : {
        "my_type" : {
            "properties": {
                "title" : {
                    "type" : "text"
                }
            }
        }
    }
}
```
(2). 建立完新索引后使用scroll api 将数据批量查询出
GET /my_index/_search?scroll=1m
{
    "query" : {
        "match_all" : {}
    }
    "sort" : ["_doc"],
    "size" : 1000
}

(3). 用bulk api 将scroll 查出来一批一批的数据批量重新写入新索引my_index_new
POST /_bulk
{"index": {"_index" : "my_index_new", "_type" : "test_type", "_id" : "2"}}
{"title" : "2019-09-09"}

(4). 依次遍历全部旧索引数据

(5). 将别名good_index切换到my_index_new上面, 这时应用就会使用到新索引中的全部数据.
修改别名指向索引
POST /_aliases
{
    "actions" : [
        {"remove" : {"index" : "my_index", "alias" : "goods_index"}},
        {"add" : {"index" : "my_index_new", "alias":"goods_index"}}
    ]
}
(6). 最后直接通过good_index别名可以直接获取全部数据


### query_string search, query DSL  和 _all metadata的原理

#### query_string search的语法

语法: GET /_all/type/_search?q+=field:value
根据query_string 在指定的字段field 搜索指定的关键词value  
+号表示必须包含 - 表示必须不包含


#### query DSL 语法 search 语法
语法: GET my_index/my_type/_search
```json
{
    "query": {
        "bool" : {
            "must" : {"match" : {"name" : "java"}},
            "should": [
                {"match" : {"hired" : true}},
                {"bool" :  {
                  "must" : {"match" :  {"personality" :  "good"}},
                  "must_not" : {"match" :  {"rude" :  true}}
                }}
            ],
            "minimum_should_match" : 1
        },
        "filter" : {
          "range" : {
            "age" : {
              "gte" : 30
            }
          }
        }
    }
}
```

bool : 组合多个拆分条件 `(bool 是一个复合的过滤器，可以接受多个子过滤器)`  
must : 必须匹配的条件  
should : 可以匹配也可以不匹配`(should不必包hired 或 子项)[如果包含用于提升score分数排名]`
must_not :  不能够匹配
minimum_should_match : 至少匹配到一条

filter : 根据搜索条件过滤出需要的数据而已, 不计算任何相关度分数, 对相关度没有任何影响
query : 会计算每个document相对于搜索条件的相关度, 并按照相关度排序

> 如果只是按条件进行搜索, 按匹配的相关度数据先返回 使用query
> 如果你只是根据一些筛选出一部分数据, 不关注其排序, 那么filter

> filter 不需要计算相关度分数, 不需要按照相关度分数进行排序, 同时还有内置的自动cache常用的filter数据
> query 需要计算相关度分数 按照相关度进行排序无法进行cache



#### exact value
精准匹配
exact value 搜索的时候 必须数据完全一致 才能匹配
全文索引
full text 
1. 缩写 cn china 等缩写
2. 格式换换 like likes 等 复数模式
3. China china 大小写转换
4. like love 同义词搜索1

### 分词器内部组成
切分词语, normalization (提升recall召回率[搜索的时候增加能够搜索到的结果数量])
character filter 在一段文本进行分词之前, 先进行预处理 (eg:过滤html标签等)
tokenizer 分词 : 将词语分为单词
token filter : lowercase(大小写转换), stop word (去除停用词)
一个分词器: 要将一段代码进行处理, 最后处理好的结果才会去建立倒排索引
内置分词器有 (系统默认分词器)standard analyzer[大小写转换去掉符号等] simple analyzer[简单转换] whitespace analyzer[不会根据特殊符号拆分] language analyzer[根据特定语言的分词器]



#### query_string 分词

query_string 默认情况下, es会使用对应的field建立倒排索引时相同的分词器去进行分词, 分词和 normalization
只有这样才能实现正确的倒排搜索

q=value 使用query_string 建立倒排索引时 建立为 full text索引
q=filed : value 使用exact value  建立倒排索引时, 建立为exact value索引


####  search api 的基本语法
1. search api语法
 指定分页索引
GET test_index,test_index2/test_type/_search
{
    "form" : 0,
    "size" : 10
}
took: 整个搜索请求话费了多少毫秒
_shards: 请求打到了 多个少 _shards.total 上 多少个成功了 _shards.successful, 多少个失败了 _shards.failed
hits.total 本次搜索,返回了几条结果
hits.max_score: 本次搜索的所有结果中,最大相关度分数是多少,每一条document对于search的相关度,越相关,_score分数越大,排位越靠前
hits.hits 默认查询前10条数据,完整数据,_score降序排序
timeout: 默认情况下是没有timeout,latency平衡completeness 手动指定timeout, timeout查询执行机制
如果搜索特别慢, 每个shard花好几分钟才能查询所有的数据,那么你的搜索请求也会等待好几分钟
指定每个shard在timeout范围内检索到的部分数据返回给client，而不是等待全部数据.
eg: 本该在10分钟内获取2000条数据,但是在10ms获取20条数据.


#### field 索引两次排序

当string filed 进行分词排序后, 结果往往是不准确的, 因为分词后是多个单词 
通常方案为将string filed 建立两次索引, 一个分词, 用来进行搜索, 一个不分词, 进行排序





#### query phase 

1. query phase  
(1). 请求发送到一个coordinate node 构建一个priority queue,  长度以paging操作from和size为准, 默认为10.  
(2). 请求转发到对应的primary shard(replica shard)上, 接收到请求的每个shard, 都会构建一个form + size 大小的本地priority queue.  
(3). 每个shard将自己的数据返回给coordinate node 进行merge成 from + size 大小的priority queue, 全局排序后的queue, 放到自己的queue中.  
(4).  最后 coordinate queue 就可以将自己的 priority queue中取出第from页数据的from+10条数据了.

2. replica shard 增加搜索吞吐量
每次请求都要发送到所有的shard的replica/primary上去, 如果每个shard都有多个replica, 那么同时并发过来的搜索请求可以请求replica上面, replica 越多所承载的请求也就会越多

3. fetch phase  
(1). coordinate node 构建完 priority queue 之后, 获取到是一堆doc id等信息
(2). 发送mget api去各个shard上批量一次获取自己想要的数据
(3). 每个shard 获取到对应document后, 返回给coordinate node.
(4). 最后coordinate 拿到所有document 返回给client


#### 参数和bouncing result 问题解决

bouncing results 问题, 两个document 排序field值相同, 不同的shard上, 可能排序不同, 每次请求轮询到不同的shard商, 每次页面上看到的搜索结果都不一样, 这就是bouncing result 也就是跳跃结果.
方案 : 将 preference 设置为一个字符串 eg : user_id 让每一个user进行搜索的时候都会沿着同一个replica shard 去指定，就不会出现bouncing result了.
1. preference 决定了哪些shard会用来执行搜索操作
2. timeout 在指定的时间内获取数据直接返回, 避免时间过长无响应
3. routing document 文档路由, _id 路由, 对应相同的routing到同一个shard上去


#### 定制分词器
默认为standard 分词器

standard tokenizer : 以单词边界进行分词
standard token filter : 什么都不做
lowercase token filter : 将所有的字母转换为小写
stop token filter (默认被禁用) : 移除停用词, 比如a the it 等等.

修改分词器设置
eg : 启用english停用词token filter  
```
PUT /my_index
{
    "settings" : {
        "analysis" : {
            "analyzer" : {
                "es_std" : {
                    "type" : "standard",
                    "stopwords" : "_english_"
                }
            }
        }
    }
}
```
测试使用自定义停用词

GET my_index/_analyze
{
    "analyzer" : "es_std",
    "text" : "a dog is in house"
}

返回分割语句
```json
{
    "tokens": [
        {
            "token": "dog",
            "start_offset": 2,
            "end_offset": 5,
            "type": "<ALPHANUM>",
            "position": 1
        },
        {
            "token": "house",
            "start_offset": 18,
            "end_offset": 23,
            "type": "<ALPHANUM>",
            "position": 5
        }
    ]
}
```

#### 定制化分词器

PUT /my_index
```json
{
    "settings" : {
        "analysis" : {
            "char_filter" : {
                "&_to_and" : {
                    "type" : "mapping",
                    "mappings" : ["&=> and "]
                }},
            "filter" : {
              "my_stopwords": {
                "type" : "stop",
                "stopwords" : ["the", "a"]
              }},
            "analyzer" : {
              "my_analyzer" : {
                "type" : "custom",
                "char_filter" : ["html_strip", "&_to_and"],
                "tokenizer" : "standard",
                "filter" : ["lowercase", "my_stopwords"]
              }
            }
        }
    }
}
```
分别是字符过滤转换 和自定义停用词

测试自定义分词器

```
GET /my_index/_analyze
{
  "text":"Tom &Jerry is friends <a> HAHA",
  "analyzer": "my_analyzer"
}
```

返回自定义分词后的数据
```json
{
    "tokens": [
        {
            "token": "tom",
            "start_offset": 0,
            "end_offset": 3,
            "type": "<ALPHANUM>",
            "position": 0
        },
        {
            "token": "andjerry",
            "start_offset": 4,
            "end_offset": 10,
            "type": "<ALPHANUM>",
            "position": 1
        },
        {
            "token": "is",
            "start_offset": 11,
            "end_offset": 13,
            "type": "<ALPHANUM>",
            "position": 2
        },
        {
            "token": "friends",
            "start_offset": 14,
            "end_offset": 21,
            "type": "<ALPHANUM>",
            "position": 3
        },
        {
            "token": "haha",
            "start_offset": 26,
            "end_offset": 30,
            "type": "<ALPHANUM>",
            "position": 4
        }
    ]
}
```



####  mapping root object 解析
1. root object
就是某个type对应mapping json, 包括 properties, metadata (_id, _source, _type), settings(analyzer), 其他settings (eg: include_in_all)

2. properties
type, index, analyzer

3. _source
(1). 查询的时候可以拿到完整的document, 不需要先拿document id, 再发送一次请求拿 document
(2). partial update 基于 _source实现
(3). reindex时, 直接基于_source实现, 不需要查询数据再查询,再修改
(4). 可以基于_source定制返回field
(5). debug query 更容易, 因为可以看到_source

如果不需要_source可以禁用
```
PUT /my_index
{
    "mappings" : {
        "my_type" : {
            "_source" : {
                "enable" : false
            }
        }
    }
}
```

4. _all
将所有field打包拼接在一起, 一个_all field 建立索引, 没有指定field进行搜索时, 就是_all field搜索.

也可以在field级别设置include_in_all field, 设置是否将field的值, 包含在_all field中.

PUT /my_index/my_type/_mapping
{
    "my_type" : {
        "include_in_all" : false
    }
}

5. 标识性metadata
_index, _type, _id


#### 自定义 dynamic mapping策略
1. 定制dynamic策略
true : 遇到陌生字段, 就进行dynamic mapping
false : 遇到陌生字段, 就忽略
strict : 遇到陌生字段, 就报错

自定义dynamic mapping
```
PUT my_index
{
    "mapping" : {
        "my_type": {
            "dynamic" : "strict",
            "properties" : {
                "title" : {
                    "type" : "text"
                },
                "address" : {
                    "type" : "object",
                    "dynamic" : "true"
                }
            }
        }
    }
}
```

请求
```json
{
  "title" : "my article",
  "content" : "this is my article",
  "address" : {
    "province":"guangdong",
    "city":"guangzhou"
  }
}
```

错误提示
```json
{
    "error": {
        "root_cause": [
            {
                "type": "strict_dynamic_mapping_exception",
                "reason": "mapping set to strict, dynamic introduction of [content] within [my_type] is not allowed"
            }
        ],
        "type": "strict_dynamic_mapping_exception",
        "reason": "mapping set to strict, dynamic introduction of [content] within [my_type] is not allowed"
    },
    "status": 400
}
```
如果content field 删除 就会成功

2. 定制dynamic mapping 策略
(1). date_detection
默认会按照一定格式识别date, 比如yyyy-MM-dd, 如果传递 2017-01-01的值, 就会被dynamic mapping成date.

(2). 定制自己的dynamic mapping template

```
PUT /my_index
{
    "mappings" : {
        "my_type" : {
            "dynamic_templates" : [
                {"es" : {
                    "match" : "*_es",
                    "match_mapping_type" : "string",
                    "mapping" : {
                        "type" : "string",
                        "analyzer" : "spanish"
                    }
                }},
                {"en" :  {
                  "match" : "*",
                  "match_mapping_type" : "string",
                  "mapping" : {
                    "type" : "string",
                    "analyzer" : "english"
                  }
                }}
            ]
        }
    }
}
```

(3). 定制自己default mapping template (index level)

PUT /my_index
{
    "mappings" : {
        "_default" : {
            "_all" : {"enabled" : false}
        },
        "blog" : {
            "_all" : {"enabled" : true}
        }
    }
}


