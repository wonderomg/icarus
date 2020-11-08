---
title: Elasticsearch实践(2)-索引及索引别名alias
date: 2018-5-16
categories: 
  - elasticsearch
tags: 
  - elasticsearch
  - ik中文分词
  - alias
thumbnail: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1529643264225&di=cb242c448c2a3bab04ce1acab4135689&imgtype=0&src=http%3A%2F%2Fjbcdn2.b0.upaiyun.com%2F2017%2F10%2Fc6cf4b2000277c64f55e00cf6d2f294f.png
---

我们都知道es效率如此高主要和索引是分不开的，需要将每一条数据建立索引，创建索引时数据字段也是你插入时的样子，索引中包含了数据的属性字段。而且创建索引也比较耗时（当然了，肯定比插入关系型数据库中更快😁），但是毕竟每一条数据都要建一次，数据量到达千万级亿级时，时间不是很乐观。🤔

💡想像这样一个场景，产品上线一段时间后，由于产品需要，要将某个字段类型改变，比如需要将某个long型字段改成string类型，直接改会类型报错，查阅官方文档可知，es是不支持索引的更新操作的，需要对所有已有数据的所有field进行reindex，这意味着，需要停止服务进行重建索引操作，停服是最不想看到😭，这时如果你当初建了索引别名，你会感谢你当初使用别名的决定，为何要用别名alias？🤷

<!--more-->

Elasticsearch的索引别名alias，类似于数据库的视图。因为索引别名alias是映射到之前已创建的索引上的，对你要创建新的索引毫无影响。在创建好新索引之后，我们只需将这别名alias映射到新创建的别名上即可，这些操作只需要在服务器上通过指令操作即可，完全不影响线上服务，实现无缝切换索引。然后再将老的索引删掉即可，以节省空间。



##  1. 创建一个索引

创建一个索引，名为`goods`：

```shell
$ curl -XPUT 10.10.13.234:9200/goods
```

## 2. 为索引创建mapping

为索引`goods`创建`mapping`，这里制定`content`字段属性使用ik中文分词器，只要制定搜索`content`内容就会查询出分词之后的关联结果：

```shell
$ curl -XPOST 10.10.13.234:9200/goods/fulltext/_mapping -d'
{
    "fulltext": {
             "_all": {
            "analyzer": "ik"
        },
        "properties": {
            "content": {
                "type" : "string",
                "boost" : 8.0,
                "term_vector" : "with_positions_offsets",
                "analyzer" : "ik",
                "include_in_all" : true
            }
        }
    }
}'
```

这里创建的索引`goods`，如果未使用别名，程序中也是用`goods`这个索引名，如果建了新的索引，那么程序也要用新的索引名称，这是不友好的，如果用了索引别名alias就可以解决这个问题，继续看后面的操作。

## 3. 插入数据

插入一些测试数据：

```shell
$ curl -XPOST 10.10.13.234:9200/goods/fulltext/1 -H 'Content-Type:application/json' -d'
{ "content": "进口华盛顿红蛇果 苹果4个装 单果重约180g 新鲜水果" }'

$ curl -XPOST 10.10.13.234:9200/goods/fulltext/2 -H 'Content-Type:application/json' -d'
{ "content": "华盛顿苹果礼盒 8个装（4个红蛇果+4个青苹果）单果重约145g-180g 新鲜水果礼盒" }'

$ curl -XPOST 10.10.13.234:9200/goods/fulltext/4 -H 'Content-Type:application/json' -d'
{ "content": "新疆阿克苏冰糖心 约5kg 单果200-250g（7Fresh 专供）" }'

$ curl -XPOST 10.10.13.234:9200/goods/fulltext/5 -H 'Content-Type:application/json' -d'
{ "content": "果花 珍珠岩 无土栽培基质 颗粒状 保温性能好 园艺用品" }'

#查看goods的mpping，后面需要与索引别名的mapping对比
$ curl -XGET "10.10.13.234:9200/goods/_mapping?pretty" 
```

## 4. 测试分词高亮

测试命中分词高亮：

```shell
$ curl -XPOST 10.10.13.234:9200/goods/fulltext/_search -H 'Content-Type:application/json' -d'
{ "query" : { "match" : { "content" : "进口水果" }},
    "highlight" : {
        "pre_tags" : ["<tag1>", "<tag2>"],
        "post_tags" : ["</tag1>", "</tag2>"],
        "fields" : {
            "content" : {}
        }
    }
}'
```

## 5. 创建索引别名

新建同义词mapping：

(1) 创建一个索引，这个索引的名称最好带上版本号，比如my_index_v1,my_index_v2等;

(2) 创建一个指向本索引的同义词。为的是以后不停服更新mapping。

步骤(1)的索引我们以刚刚创建的`goods`所以为例，创建别名映射到`goods`所以上：

```shell
$ curl -XPOST 10.10.13.234:9200/_aliases -d '  
{  
    "actions": [  
        { "add": {  
            "alias": "my_index",  
            "index": "goods"  
        }}  
    ]  
}'
```

然后查看索引`my_index`的mapping：

```shell
$ curl -XGET "10.10.13.234:9200/my_index/_mapping?pretty" 
```

索引`my_index`与`goods`的`mapping`一致。

## 6. 切换索引

再创建一个索引`goods_v2`，重复步骤1、2，名称换成`goods_v2`

```shell
$ curl -XPUT 10.10.13.234:9200/goods_v2
$ curl -XPOST 10.10.13.234:9200/goods_v2/fulltext/_mapping -d'
{
    "fulltext": {
             "_all": {
            "analyzer": "ik"
        },
        "properties": {
            "content": {
                "type" : "string",
                "boost" : 8.0,
                "term_vector" : "with_positions_offsets",
                "analyzer" : "ik",
                "include_in_all" : true
            }
        }
    }
}'
```

插入测试数：

```shell
$ curl -XPOST 10.10.13.234:9200/goods_v2/fulltext/1 -H 'Content-Type:application/json' -d'
{ "content": "进口华盛顿红蛇果 苹果4个装 单果重约180g 新鲜水果",
"number": 21}'

#查看goods_v2的mpping，后面需要与索引别名的mapping对比
$ curl -XGET "10.10.13.234:9200/goods_v2/_mapping?pretty"
```

**重点是更换索引的操作**，将`my_index`别名切换映射到`goods_v2`上：

```shell
$ curl -XPOST 10.10.13.234:9200/_aliases -d '
{
    "actions": [
        { "remove": {
            "alias": "my_index",
            "index": "goods"
        }},
        { "add": {
            "alias": "my_index",
            "index": "goods_v2"
        }}
    ]
}'

#查看my_index的mpping，与前面goods_v2的mapping对比
$ curl -XGET "10.10.13.234:9200/my_index/_mapping?pretty" 
```

我们可以对比索引`my_index`与`goods_v2`的`mapping`是一致的，索引切换成功。

创建了索引别名alias的话，程序中制定这个`my_index`这个别名即可，后需要创建新索引，只需切换索引映射到新索引上即可，实现不停服更新mapping。

下一篇我们来说下通过**创建spring boot工程，通过es的java api操作es，实现分页查询和中文分词检索**的功能。