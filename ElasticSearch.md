elasticsearch是面向文档的。并且将文档进行索引，生成一个倒排索引。并且能够支持全文检索。全文检索一般是对于文字很多的列进行的。而且elasticsearch还会对结果进行打分。并不像关系型数据库一样，一条数据要么符合查询要么不符合查询，在es中还有一些模糊查询等。并且可以对结果进行某些字段的高亮显示。并且可以进行一些聚合查询。聚合查询类似于关系型数据库的分组查询。并且可以对这些查询进行组合。比如说，先进行一个term查询，然后对结果进行聚合查询。

查询可以分为match查询，term查询。

> match 查询

​	march查询又可以分为match，match_phrase , match_phrase_prefix 。

- match 

  match查询是一种bool类型的查询。会将要查询的短语进行分词。并且默认情况下是省略了一个参数`operator`为`or`。那么，既然要分词，也可以指定分词器。如果没有指定则用mapping时候指定的。如果mapping也没有指定，那么就会用默认的search analyzer。并且也可以设置`lenient`参数。这个参数用来指定在类型错误，并且不能转化的时候是否要报错。还有就是写法。

  ```json
  GET matchtest/people/_search
  {
    "query": {
      "match": {
        "hobbies": "football basketball"   // 此处，可以直接写查询的条件。也可以像下面一样，再写一个大括号
      }
    }
  }
  // 可以写的很简单，也可以像下面这样带上很多的参数。
  GET books/_search
  {
    "query": {
      "match": {
        "name": {
          "analyzer": "ik_max_word",
          "query": "全国计算机",
          "lenient": "true",
          "operator": "or",
          "fuzziness": "auto",
          "prefix_length": 1
        }
      }
    }
  }
  ```

- match_phrase

  match_phrase也会将查询短语进行分词。但是默认要求被匹配的文档要包含分词后的每个词，并且顺序需要一致，还必须连续。这样的条件过于苛刻，那么还可以通过`slop`参数来指定可以允许经过多少次移动，也可以被匹配。

- multi-match

  multi-match可以指定在多个字段中进行匹配。并且有多种执行方式。

> term 查询

### 更新文档

```json
// 更新操作 
1. 会覆盖原来的数据：
PUT blog/_doc/1
{
  "title":"666"
}
2. 部分更新，不会覆盖原来的数据
POST blog/_update/1
{
  "script": {
    "lang": "painless",
    "source":"ctx._source.title=params.title",
    "params": {
      "title":"jieqiyue"
    }
  }
}
// 再来一个更新操作
POST spring_boot/_update/79
{
  "script": {
    "lang": "painless",
    "source":"ctx._source.title=params.title",
    "params": {
      "title":"new_title"
    }
  }
}
3.使用参数带update的更新方式
POST spring_boot/_doc/79/_update
{
  "doc":{
    "title":"thredd"
  }
}
```

以上三种都可以更新文档。

还有一些高级更新，

##### 条件更新

```json
// 条件更新
GET spring_boot/_doc/79/_source

POST spring_boot/_update/79
{
  "script": {
    "lang": "painless",
    "source": "if (ctx._source.age==\"88\"){ctx._source.age=\"123\"}else{ctx.op=\"none\"}"
  }
}
// 根据查询来更新，这个貌似也可以转换为条件更新？
POST spring_boot/_update_by_query
{
  "script": {
    "source": "ctx._source.age=\"8\"",
    "lang": "painless"
  },
  "query": {
    "term": {
      "age":"6"
    }
  }
}
```

### 删除文档

```json
// 删除文档
DELETE spring_boot/_doc/79
```

##### 查询删除

```json
POST blog/_delete_by_query
{
  "query":{
    "term":{
      "title":"666"
    }
  }
}
```

也可以删除某一个索引下的所有文档：

```
POST blog/_delete_by_query
{
  "query":{
    "match_all":{
      
    }
  }
}
```

###  批量操作

首先需要将所有的批量操作写入一个 JSON 文件中，然后通过 POST 请求将该 JSON 文件上传并执行。

例如新建一个名为 aaa.json 的文件，内容如下：![图片](https://mmbiz.qpic.cn/mmbiz_png/GvtDGKK4uYmxjmzP3pwKJILMZdMnJQqu4Tiby3TxMNWemwqZLaicKDgicmduSsc3DuVH5xcnugLoicZAPlzQ9JRq8w/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

aaa.json 文件创建成功后，在该目录下，执行请求命令，如下：

```
curl -XPOST "http://localhost:9200/user/_bulk" -H "content-type:application/json" --data-binary @aaa.json
```

执行完成后，就会创建一个名为 user 的索引，同时向该索引中添加一条记录，再修改该记录，最终结果如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/GvtDGKK4uYmxjmzP3pwKJILMZdMnJQqu2KT3WfxoiaibOIBgqLiam5ic6U8Yyu3Y1hD0t0qTKSbntse5NWQcaWibJMg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 并发控制

> 之前的版本

使用 version 和 version_type 来进行控制。**这个version是针对某一个文档的。**

```json
PUT blog/_doc/2?version=2423&version_type=external_gte
{
  "title":"7adfsdfaaa"
}
```

> 现在的版本控制

```json
PUT blog/_doc/4?if_seq_no=17&if_primary_term=3
{
   "title":"4444aaa"
}
```

`_seq_no` 是对于整个索引来说的，对于索引的增加或者是删除等操作都会将这个增加。但是对于其中的某一个文档来说还是不会增加的。

### elasticsearch 简介及发展历史 

![image-20210504160856119](../../TyporaImages/image-20210504160856119.png)

![image-20210504161018796](../../TyporaImages/image-20210504161018796.png)

### 安装elasticsearch

![image-20210504163534505](../../TyporaImages/image-20210504163534505.png)

![image-20210504164932329](../../TyporaImages/image-20210504164932329.png)

### 基本概念

- 文档 Document

![image-20210504180022042](../../TyporaImages/image-20210504180022042.png)

![image-20210504180233117](../../TyporaImages/image-20210504180233117.png)

![image-20210504180704340](../../TyporaImages/image-20210504180704340.png)

![image-20210504180712983](../../TyporaImages/image-20210504180712983.png)

![image-20210504181554712](../../TyporaImages/image-20210504181554712.png)

![image-20210505230153729](../../TyporaImages/image-20210505230153729.png)

![image-20210505230214750](../../TyporaImages/image-20210505230214750.png)

### 节点，集群，分片，副本

![image-20210507145907147](../../TyporaImages/image-20210507145907147.png)

![image-20210507145921380](../../TyporaImages/image-20210507145921380.png)

![](../../TyporaImages/image-20210507150859486.png)

分片就是对于一个索引的拆分。默认情况下，创建一个索引会创建一个 1 个分片，并且为每一个分片创建一个副本。因为如果单个索引放在一台机器上，性能不够。分片不是越多越好，一般要根据总数据量来进行计算。

建议：（仅参考）

   1、每一个分片数据文件小于30GB

   2、每一个索引中的一个分片对应一个节点

   3、节点数大于等于分片数

### crud

> 索引

一、创建索引

1. 发送put请求，比如说发送 put 请求到 localhost:9200/test，就创建了一个test索引。
2. 通过head插件进行创建。head插件就是浏览器上的一个插件。
3. 使用kibana发送 put test 请求。

注意两点：

- 索引名称不能重复
- 索引名称不能有大写字母

二、更新索引

​	1. 修改分片数量：

```
PUT book/_settings
{
  "number_of_replicas": 2
}
```

三、查看索引

```
GET book/_settings
GET _all/_settings
```

四、删除索引

```
DELETE test
```

删除一个不存在的索引会报错。

五、关闭索引

```
POST book/_close
```

六、索引别名

```
// 添加索引
POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "book",
        "alias": "book_alias"
      }
    }
  ]
}
 // 移除索引
POST /_aliases
{
  "actions": [
    {
      "remove": {
        "index": "book",
        "alias": "book_alias"
      }
    }
  ]
}

GET /book/_alias

// 查看某一个别名对应的索引（book_alias 表示一个别名）：
GET /book_alias/_alias

GET /_alias
```

> 文档

![image-20210507152641060](../../TyporaImages/image-20210507152641060.png)

```
// 写入一条文档
PUT book/_doc/1
{
  "title":"三国演义"
}
```

权限相关：

```
// 关闭索引的写权限:
PUT book/_settings
{
  "blocks.write": true
}
```

一、新增文档

指定id的方式：

```
PUT blog/_doc/1
{
  "title":"6. ElasticSearch 文档基本操作",
  "date":"2020-11-05",
  "content":"微信公众号**江南一点雨**后台回复 **elasticsearch06** 下载本笔记。首先新建一个索引。"
}
```

```
// 添加成功后返回值如下：
{
  "_index" : "blog",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 2,
    "successful" : 2,
    "failed" : 0
  },
  "_seq_no" : 0,
  "_primary_term" : 1
}
```

- _index 表示该文档属于哪个索引。
- _type 表示文档的类型。
- _id 表示文档的 id。
- _version 表示文档的版本（更新文档，版本会自动加 1，针对一个文档的）。
- result 表示执行结果。
- _shards 表示分片信息。
- `_seq_no` 和 `_primary_term` 这两个也是版本控制用的（针对当前 索引）。而_version 针对的是文档。

不指定id的方式：

```
POST blog/_doc
{
  "title":"666",
  "date":"2020-11-05",
  "content":"微信公众号**江南一点雨**后台回复 **elasticsearch06** 下载本笔记。首先新建一个索引。"
}
```

二、获取文档

获取单个文档

```
GET blog/_doc/RuWrl3UByGJWB5WucKtP
```

也可以批量获取文档

```
GET blog/_mget
{
  "ids":["1","RuWrl3UByGJWB5WucKtP"]
}
```

三、更新文档

更新整个文档：

这种方式会将原文档全部覆盖！

```
PUT blog/_doc/RuWrl3UByGJWB5WucKtP
{
  "title":"666"
}
```

更新分文档：

```
POST blog/_update/1
{
  "script": {
    "lang": "painless",
    "source":"ctx._source.title=params.title",
    "params": {
      "title":"666666"
    }
  }
}
```

更新的请求格式：POST {index}/_update/{id}

在脚本中，lang 表示脚本语言，painless 是 es 内置的一种脚本语言。source 表示具体执行的脚本，ctx 是一个上下文对象，通过 ctx 可以访问到 `_source`、`_title` 等。

通过上面可以看出，更新的格式就是最后要加上一个id值来指定某一个具体的文档。但是我们也可以使用查询更新。就是通过给定一个查询条件，然后将查询出来的文档进行更新。

```
POST blog/_update_by_query
{
  "script": {
    "source": "ctx._source.content=\"888\"",
    "lang": "painless"
  },
  "query": {
    "term": {
      "title":"666"
    }
  }
}
```

注意看这里查询更新的格式，最后并没有指定id值。

四、删除文档

删除和update一样，也是有两种。一种是直接指定id删除，一种是根据查询来删除。

```
1. 	DELETE blog/_doc/TuUpmHUByGJWB5WuMasV

2. 	POST blog/_delete_by_query
    {
      "query":{
        "term":{
          "title":"666"
        }
      }
    }
    
  删除一个索引下面的所有文档  
3.	POST blog/_delete_by_query
    {
      "query":{
        "match_all":{

        }
      }
    }
```

也可以通过Bulk API进行批量操作：

https://blog.csdn.net/prestigeding/article/details/84205785

五、其它

https://blog.csdn.net/ZYC88888/article/details/100727714#:~:text=%E4%B8%80%E8%88%AC%E7%9A%84%EF%BC%8Celas,%E7%BE%A4%E4%B8%8D%E5%B9%B3%E8%A1%A1%E7%9A%84%E6%83%85%E5%86%B5%E3%80%82

当新增一个文档的时候，这个文档究竟是存在哪个分片上的呢？

![img](https://img-blog.csdnimg.cn/20190911101509931.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly96eWM4OC5ibG9nLmNzZG4ubmV0,size_16,color_FFFFFF,t_70)

当一个索引被创建以后，一般分片数都不允许更改。原因就是当更改了分片数之后，通过原来的所有的路由（routing）值都会无效，文档就再也找不到了。因为长度变化了，但是hash值没有变。

https://www.elastic.co/guide/cn/elasticsearch/guide/current/distrib-read.html

https://www.elastic.co/guide/cn/elasticsearch/guide/current/distributed-search.html

**在博客中说，文档会被分散在节点中，elasticsearch自己都不知道文档具体在哪个分片上面，所以得要发送给所有的分片。那为什么不能够再根据hash算法算出文档具体在哪个分片上，然后直接去那个分片上面拿数据不就行了？**

取回单个文档确实是先通过hash算法来计算出文档所在分片，然后再去这个分片的所有副本中选择一个来进行查询（负载均衡），然后再将结果返回的。此时可能就是要去副本分片上面查询，但是结果现在还是存放在主分片上面的。所以，可能会返回空。但是主分片中是有值的。

所以说要分清楚对于单个文档做查询还是说是分布式搜索。对于分布式搜索，每一个分片都有可能有满足你搜索条件的结果。

对于***查询接口***来说，默认会搜索所有的分片：

```
GET route_test/_search?routing=key1,key2 
{
  "query": {
    "match": {
      "data": "b"
    }
  }
}
```

### 数据类型

 [www.cnblogs.com/shoufeng/p/10692113.html](https://www.cnblogs.com/shoufeng/p/10692113.html#:~:text=1 核心数据类型 1.... 2 复杂数据类型 2.1  数组类型,4.1  IP类型 4.... 5 参考资料 6 版权声明)

https://zhuanlan.zhihu.com/p/136981393

### 映射参数

- analyzer 与 search_analyzer 参数：

```
PUT blog
{
  "mappings": {
    "properties": {
      "title":{
        "type":"text",
        "analyzer": "ik_smart"
      }
    }
  }
}
```

​	创建索引的时候指定分词器是ik分词器。

​	search_analyzer 是搜索的时候指定分词器的。

- normalizer 

  ```
  // 未标准化之前
  PUT blog
  {
    "mappings": {
      "properties": {
        "author":{
          "type": "keyword"
        }
      }
    }
  }
  // 标准化之后
  PUT blog
  {
    "settings": {
      "analysis": {
        "normalizer":{
          "my_normalizer":{
            "type":"custom",
            "filter":["lowercase"]
          }
        }
      }
    }, 
    "mappings": {
      "properties": {
        "author":{
          "type": "keyword",
          "normalizer":"my_normalizer"
        }
      }
    }
  ```

- boost

  boost 参数可以设置字段的权重。

  boost 有两种使用思路，一种就是在定义 mappings 的时候使用，在指定字段类型时使用；另一种就是在查询时使用。

  实际开发中建议使用后者，前者有问题：如果不重新索引文档，权重无法修改。

  ```
  mapping 中使用 boost（不推荐）：
  PUT blog
  {
    "mappings": {
      "properties": {
        "content":{
          "type": "text",
          "boost": 2
        }
      }
    }
  }
  另一种方式就是在查询的时候，指定 boost
  GET blog/_search
  {
    "query": {
      "match": {
        "content": {
          "query": "你好",
          "boost": 2
        }
      }
    }
  }
  ```

  

- doc_values 和 fielddata

  doc_values 默认是开启的，如果确定某个字段不需要排序或者不需要聚合，那么可以关闭 doc_values。而fielddata默认是关闭的。fielddata是用在text字段上面的，在text第一次做聚合或者是排序的操作的时候。那么一般来说，不会再text字段上面做这种操作。而且这个fielddata是存在内存里面的。

  对于非text字段默认是开启的。那么就能够直接在这个字段上面做排序的操作。但是，如果关闭了doc_values的话，再去用这个字段做排序的话，就会报错。

- enabled

  某一个字段是否需要被建立倒排索引。enabled为false的字段是不能够被搜索到的。

- ignore_above

  text会进行分词。keyword不会进行分词。那么，对于keyword来说，能够被索引到的最大长度就被ignore_above指定。只要超过这个长度，就不能够被搜索到。如果没有指定这个值，那么都能够被索引到。

- index

  指定该字段是否要被索引 。

  1. analyzed:字段被索引，会做分词，可搜索。反过来，如果需要根据某个字段进搜索，index属性就应该设置为analyzed。
  2. not_analyzed：字段值不分词，会被原样写入索引。反过来，如果某些字段需要完全匹配，比如人名、地名，index属性设置为not_analyzed为佳。
  3. no:字段不写入索引，当然也就不能搜索。反过来，有些业务要求某些字段不能被搜索，那么index属性设置为no即可。

- store

  store属性是指定是否为某一列另外存储一份数据。

  https://www.cnblogs.com/sanduzxcvbnm/p/12157453.html

### 搜索

#### term查询

1. term查询类似于“=”。是精确匹配的。比如说你输入一个句子，它也是将这个句子看成是一个整体，不会再对其进行分词。你输入了之后，然后，它会拿着这个整体去elasticsearch中寻找。但是可以给多个整体，就是给一个数组。

```
GET books/_search
{
  "query": {
    "term": {
      "name": "十一五"
    }
  },
  "size": 20,
  "from": 0, 
  "min_score":1.75,
  "_source": ["name","author"],
  "highlight": {
    "fields": {
      "name": {}
    }
  }
}

GET books/_search
{
  "query": {
    "terms": {
      "name": ["程序","设计"]
    }
  }
}


```

那么，就会去查找20条数据。并且会进行过滤。只会得到name，和author字段。

那么，有些时候某一些文档的得分特别低。那么，我们可以设置一个最低分，低于最低分的就不进行返回。

还可以设置高亮返回。



#### 全文查询

```
GET books/_search
{
  "query": {
    "match": {
      "name": {
        "query": "美术计算机",
        "operator": "and"
      }
    }
  }
}
```

在这个查询中，就和上面那个不同。这个查询会将输入的文本进行分词。上面的输入进行分词之后，就得到美术和计算机两个词。然后再去跟term查询一样去查询。默认是or的关系。还要注意一点的就是，上面分词得到的两个词不需要顺序和文档中的顺序相同。而且间隔也无所谓。

```
GET books/_search
{
  "query": {
    "match_phrase": {
      "name": {
        "query": "十一五计算机",
        "slop": 8
      }
    }
  }
}
```

而这个match_phrase的两个词就必须要顺序和文档中词的出现顺序相同。并且还有指定间隔。间隔指的不是词之间的汉字间隔，而是说文档中的句子被分词之后，那些词之间的间隔。

```
{
  "tokens" : [
    {
      "token" : "普通",
      "start_offset" : 0,
      "end_offset" : 2,
      "type" : "CN_WORD",
      "position" : 0
    },
    {
      "token" : "高等教育",
      "start_offset" : 2,
      "end_offset" : 6,
      "type" : "CN_WORD",
      "position" : 1
    },
    {
      "token" : "高等",
      "start_offset" : 2,
      "end_offset" : 4,
      "type" : "CN_WORD",
      "position" : 2
    },
    {
      "token" : "教育",
      "start_offset" : 4,
      "end_offset" : 6,
      "type" : "CN_WORD",
      "position" : 3
    },
    {
      "token" : "十一五",
      "start_offset" : 7,
      "end_offset" : 10,
      "type" : "CN_WORD",
      "position" : 4
    },
    {
      "token" : "十一",
      "start_offset" : 7,
      "end_offset" : 9,
      "type" : "CN_WORD",
      "position" : 5
    },
    {
      "token" : "五",
      "start_offset" : 9,
      "end_offset" : 10,
      "type" : "CN_CHAR",
      "position" : 6
    },
    {
      "token" : "国家级",
      "start_offset" : 11,
      "end_offset" : 14,
      "type" : "CN_WORD",
      "position" : 7
    },
    {
      "token" : "国家",
      "start_offset" : 11,
      "end_offset" : 13,
      "type" : "CN_WORD",
      "position" : 8
    },
    {
      "token" : "级",
      "start_offset" : 13,
      "end_offset" : 14,
      "type" : "CN_CHAR",
      "position" : 9
    },
    {
      "token" : "规划",
      "start_offset" : 14,
      "end_offset" : 16,
      "type" : "CN_WORD",
      "position" : 10
    },
    {
      "token" : "教材",
      "start_offset" : 16,
      "end_offset" : 18,
      "type" : "CN_WORD",
      "position" : 11
    },
    {
      "token" : "大学",
      "start_offset" : 19,
      "end_offset" : 21,
      "type" : "CN_WORD",
      "position" : 12
    },
    {
      "token" : "计算机",
      "start_offset" : 21,
      "end_offset" : 24,
      "type" : "CN_WORD",
      "position" : 13
    },
    {
      "token" : "计算",
      "start_offset" : 21,
      "end_offset" : 23,
      "type" : "CN_WORD",
      "position" : 14
    },
    {
      "token" : "算机",
      "start_offset" : 22,
      "end_offset" : 24,
      "type" : "CN_WORD",
      "position" : 15
    },
    {
      "token" : "基础",
      "start_offset" : 24,
      "end_offset" : 26,
      "type" : "CN_WORD",
      "position" : 16
    }
  ]
}
```

```
GET books/_search
{
  "query": {
    "multi_match": {
      "query": "阳光",
      "fields": ["name^4","info"]
    }
  }
}
```

还可以指定在多个域中去进行查询。如上面的那个查询，就指定在name域和info域中进行查询。并且，name域中的得分乘以4倍。

#### 复合查询

- function_score query

  有时候进行搜索的时候，要进行评分。如果在评分的时候，还要考虑上另外一个子段的值，比如说对于博客搜索并对结果评分，而且最终排名需要把博客投票数给考虑进去。那么，此时就可以用这种查询。

  有几种不同的计算方式，只说两种：

  ```
  PUT blog
  {
    "mappings": {
      "properties": {
        "title":{
          "type": "text",
          "analyzer": "ik_max_word"
        },
        "votes":{
          "type": "integer"
        }
      }
    }
  }
  
  PUT blog/_doc/1
  {
    "title":"Java集合详解",
    "votes":100
  }
  
  PUT blog/_doc/2
  {
    "title":"Java多线程详解，Java锁详解",
    "votes":10
  }
  GET blog/_search
  {
    "query": {
      "function_score": {
        "query": {
          "match": {
            "title": "java"
          }
        },
        "functions": [
          {
            "script_score": {
              "script": {
                "lang": "painless",
                "source": "_score + doc['votes'].value"
              }
            }
          }
        ],
        "boost_mode": "replace"
        ## 加上这句话之后，就代表不乘以原来的分数了
      }
    }
  }
  ## 上面这种就是自定义脚本来进行计算，将原分数加上votes字段的值。得到一个值后并没有直接的就是分数了，还会把这个值给乘以原来的分数。如果不想再去乘以原来的score的话，可以加上boost_mode参数去指定。
  ```

  #### 嵌套查询

  #### 聚合查询

  ##### 指标聚合

  - 最大

  - 最小

  - 平均

  - 求和

  - 基数统计，类似于distinct count(0)

  - 按照字段统计文档数量

    只要文档中有这个字段那么，就会被计数。

  - 百分位统计

    在百分之几的时候，索引中的值是多少

  ##### 桶聚合

  - 按照某一个字段进行分组，分组之后还可以做指标聚合
  - 