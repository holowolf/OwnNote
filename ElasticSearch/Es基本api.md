
将Json文档插入twitter的索引中,名为tweet的id为1的类型下

// notice 这里为了理解
// index/type/1 相当与 mysql database/table/1
// 某个数据库的某张表id为1的数据添加

```
PUT twitter/tweet/1
{
    "user" : "kimchy",
    "post_date" : "2009-11-15T14:12:12",
    "message" : "trying out Elasticsearch"
}
```


返回数据
```json
{
    "_index": "twitter",
    "_type": "tweet",
    "_id": "1",
    "_version": 1,
    "result": "updated",
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    },
    "created":true 
}
```

图片如

![alt text](file/addIndex.png)

- 关于_shards报头提供索引操作的过程信息:
    - total  
    指示应在其上执行索引操作的分片副本(主分片和副本分片)的数量.
    - successful  
    指示索引操作成功的分片副本数.
    - failed
    在副本分片上索引操作失败的情况下包含与复制相关的错误的数组.
    
索引成功的情况下successful至少为1.



通过Post请求时如果没有指定id 那么讲自动设置id

```
POST twitter/tweet/
{
    "user" : "kimchy",
    "post_date" : "2009-11-15T14:12:12",
    "message" : "trying out Elasticsearch"
}
```

```json
{
    "_index": "twitter",
    "_type": "tweet",
    "_id": "AWpJhtEH8m3slNOBQz9w",
    "_version": 1,
    "result": "created",
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    },
    "created": true
}
```

![alt text](file/autoIndex.png)


路由routing值是一个任意字符串,
它默认是_id但也可以自定义。这个主切片的数量得到一个余数(remainder),
余数的范围永远是0到routing字符串通过哈希函数生成一个数字,
然后除以number_of_primary_shards - 1,这个数字就是特定文档所在的分片。
```
POST twitter/tweet?routing=kimchy
{
    "user" : "kimchy",
    "post_date" : "2009-11-15T14:12:12",
    "message" : "trying out Elasticsearch"
}
```


```json
{
    "_index": "twitter",
    "_type": "tweet",
    "_id": "AWpJmYjA8m3slNOBQ0ck",
    "_version": 1,
    "result": "created",
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    },
    "created": true
}
```

![alt text](file/routingIndex.png)






#GET API


get API允许根据其id从索引中获取类型化的JSON文档。
从名为twitter的索引中获取一个JSON文档，该索引名为tweet，
id值为1：

```
GET twitter/tweet/1
```



```json
{
    "_index": "twitter",
    "_type": "tweet",
    "_id": "1",
    "_version": 2,
    "found": true,
    "_source": {
        "user": "kimchy",
        "post_date": "2009-11-15T14:12:12",
        "message": "trying out Elasticsearch"
    }
}
```

上述结果包括_index，_type，_id和_version 我们希望检索，包括实际文档的_source 文档，如果可以发现（如由指示found 字段中响应）

![alt text](file/getIndex.png)


```
HEAD twitter/tweet/1
```

```
200 - OK
```

HEAD例如以下内容检查文档是否存在

![alt text](file/Isexists.png)

get操作允许指定将通过传递stored_fields参数返回的一组存储字段。如果未存储请求的字段，则将忽略它们。例如，考虑以下映射

添加映射
```
PUT twitters
{
   "mappings": {
      "tweet": {
         "properties": {
            "counter": {
               "type": "integer",
               "store": false
            },
            "tags": {
               "type": "keyword",
               "store": true
            }
         }
      }
   }
}
```

添加文档

```
PUT twitters/tweet/1
{
    "counter" : 1,
    "tags" : ["red"]
}
```

检索它

```
GET twitters/tweet/1?stored_fields=tags,counter
```

```json
{
    "_index": "twitters",
    "_type": "tweet",
    "_id": "1",
    "_version": 1,
    "found": true,
    "fields": {
        "tags": [
            "red"
        ]
    }
}
```
![alt text](file/getMapping.png)


```
PUT twitters/tweet/2?routing=user1
{
    "counter" : 1,
    "tags" : ["white"]
}
```

```
GET twitters/tweet/2?routing=user1&stored_fields=tags,counter
```

```json
{
    "_index": "twitters",
    "_type": "tweet",
    "_id": "2",
    "_version": 1,
    "_routing": "user1",
    "found": true,
    "fields": {
        "tags": [
            "white"
        ]
    }
}
```

![alt text](file/getRouting.png)


#DELETE API



```
DELETE /twitter/tweet/1
```



```json
{
    "found": true,
    "_index": "twitter",
    "_type": "tweet",
    "_id": "1",
    "_version": 4,
    "result": "deleted",
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    }
}
```

![alt text](file/deleteApi.png)


#UPDATE

更新API允许基于提供的脚本更新文档。该操作从索引获取文档（与分片并置），运行脚本（使用可选的脚本语言和参数），并对结果进行索引（也允许删除或忽略操作）。
它使用版本控制来确保在“get”和“reindex”期间没有发生更新。

注意，此操作仍然意味着文档的完全重新索引，它只是删除了一些网络往返，并减少了get和索引之间版本冲突的可能性。_source需要启用该字段才能使此功能正常工作。


```
PUT test/type1/1
{
    "counter" : 1,
    "tags" : ["red"]
}
```


```
POST test/type1/1/_update
{
    "script" : {
        "source": "ctx._source.counter += params.count",
        "lang": "painless",
        "params" : {
            "count" : 4
        }
    }
}
```


```
POST test/type1/1/_update
{
    "script" : {
        "source": "ctx._source.tags.add(params.tag)",
        "lang": "painless",
        "params" : {
            "tag" : "blue"
        }
    }
}
```

修改count 和 tag后

```json
{
    "_index": "test",
    "_type": "type1",
    "_id": "1",
    "_version": 3,
    "found": true,
    "_source": {
        "counter": 5,
        "tags": [
            "red",
            "blue"
        ]
    }
}
```
![alt text](file/updateTag.png)

添加字段
```
POST test/type1/1/_update
{
    "script" : "ctx._source.new_field = 'value_of_new_field'"
}
```


删除字段
```
POST test/type1/1/_update
{
    "script" : "ctx._source.remove('new_field')"
}
```