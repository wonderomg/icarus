---
title: Elasticsearch实践(3)-api分页查询
date: 2018-5-17
categories: 
  - elasticsearch
tags: 
  - elasticsearch
  - ik中文分词
  - 分页查询
thumbnail: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1529643264225&di=cb242c448c2a3bab04ce1acab4135689&imgtype=0&src=http%3A%2F%2Fjbcdn2.b0.upaiyun.com%2F2017%2F10%2Fc6cf4b2000277c64f55e00cf6d2f294f.png
---

elasticsearch系列：

(1)[Elasticsearch实践(1)-搭建及IK中文分词](https://wonderomg.github.io/2018/05/15/Elasticsearch%E5%AE%9E%E8%B7%B5%E6%90%AD%E5%BB%BA%E5%8F%8AIK%E4%B8%AD%E6%96%87%E5%88%86%E8%AF%8D/)

(2)[Elasticsearch实践(2)-索引及索引别名alias](https://wonderomg.github.io/2018/05/16/Elasticsearch%E5%AE%9E%E8%B7%B5%E7%B4%A2%E5%BC%95%E5%8F%8A%E7%B4%A2%E5%BC%95%E5%88%AB%E5%90%8Dalias/)

(3)[Elasticsearch实践(3)-api分页查询](https://wonderomg.github.io/2018/05/17/Elasticsearch%E5%AE%9E%E8%B7%B5api%E5%88%86%E9%A1%B5%E6%9F%A5%E8%AF%A2/)

es分页有两种，from size浅分页和scroll深分页，这里对这两种分页都做了实现，使用的是es的java api。from size类似于mysql的limit分页，from偏移，默认为0，size为返回的结果数量，默认为10。在数据量不大的情况下一般会使用from size，数据量大的时候效率比较低，而且很费内存，每次会把from*size条记录全部加载到内存中，对结果返回前进行全局排序，然后丢弃掉范围外的结果，重复这样的操作会导致内存占用过大而使es挂掉，并且受数据条数限制，10000条，需修改索引限制。🤔

<!--more-->

scroll分页通俗来说就是滚屏翻页，设置每页查询数量之后，每次查询会返回一个scroll_id，即当前文档的位置，下次查询再传这个scroll_id给es返回下一页的数据以及一个新的scroll_id，类似于按书页码顺序翻书和游标，遗憾的是不支持跳页。🤔适用于不需要跳页持续批量拉取结果、对所有数据分页或一次性查询大量数据的场景，这种记录文档位置的查询方式效率非常高，也就是常说的倒排索引。下面使用java api实现这两种分页效果。

## 1. 使用from size查询

使用 from and size 的深度分页，比如说 `?size=10&from=10000` 是非常低效的，
因为 100,000 排序的结果必须从每个分片上取出并重新排序最后返回 10 条。这个过程需要对每个请求页重复。

```shell
curl -XPOST 10.10.13.234:9200/goods/fulltext/_search -H 'Content-Type:application/json' -d'
{ "from" : 0, "size" : 10,
  "query" : { "match" : { "content" : "进口水果" }}}'
```

## 2. 使用scroll查询

使用scroll查询，需要设置`scroll_id`的失效时间。

```shell
curl '10.10.13.234:9200/index/skuId/_search?scroll=1m'  -d '
{
  "query" : { "match" : { "name" : "进口水果" }}
}'
```

会返回`scroll_id`，根据这个`scroll_id`进行下一次查询

```
curl -XGET 10.10.13.234:9200/jd_mall_v2/_search?pretty&scroll=2m -d {"query":{"match_all":{}}}
curl -XGET '10.10.13.234:9200/jd_mall_v2/_search?pretty&scroll=2m&search_type=scan' -d '{"size":3,"query":{"match_all":{}}}'
curl –XGET '10.10.13.234:9200/_search/scroll?scroll=2m&pretty
&scroll_id=c2Nhbjs1OzcyNTY6N1UtOEx3MmhSQXk2SjFnamw4bk9OUTs3MjYwOjdVLThMdzJoUkF5NkoxZ2psOG5PTlE7NzI1Nzo3VS04THcyaFJBeTZKMWdqbDhuT05ROzcyNTg6N1UtOEx3MmhSQXk2SjFnamw4bk9OUTs3MjU5OjdVLThMdzJoUkF5NkoxZ2psOG5PTlE7MTt0b3RhbF9oaXRzOjUxNTM3MDs='

```



## 1. 使用es的java api操作

java api可查阅https://es.quanke.name/，

创建springboot工程，可以在[http://start.spring.io/](http://start.spring.io/)

下载springboot工程模板，也可在github:[https://github.com/wonderomg/elasticsearch-visual](https://github.com/wonderomg/elasticsearch-visual)下载本工程。

(1)导入es的maven依赖，pom.xml内容：

```xml
<!-- Elasticsearch核心依赖包 -->
<!-- https://mvnrepository.com/artifact/org.elasticsearch/elasticsearch -->
<dependency>
  <groupId>org.elasticsearch</groupId>
  <artifactId>elasticsearch</artifactId>
  <version>2.2.1</version>
</dependency>
		
```

### 1.1  判断索引是否存在

检验索引是否创建成功，是否存在，通过prepareExists查询指定索引；

```java
/**
     * 1.判断索引是否存在
     *
     * @param client
     * @param index
     * @return
     */
    public boolean isIndexExists(Client client, String index) {
        IndicesAdminClient indicesAdminClient = client.admin().indices();
        IndicesExistsResponse response = indicesAdminClient.prepareExists(index).get();
        return response.isExists();
        /* 另一种方式
        IndicesExistsRequest indicesExistsRequest = new IndicesExistsRequest(index);
        IndicesExistsResponse response = client.admin().indices().exists(indicesExistsRequest).actionGet();*/
    }
```

### 1.2 判断类型是否存在 

类型存在的检查与索引的检查类似，只不过类型只挂在索引下，所以需要先指定索引名，然后再查询类型是否存在；

```java
/**
     * 2.判断类型是否存在
     *
     * @param client
     * @param index
     * @param type
     * @return
     */
    public boolean isTypeExists(Client client, String index, String type) {
        if (!isIndexExists(client, index)) {
            return false;
        }
        IndicesAdminClient indicesAdminClient = client.admin().indices();
        TypesExistsResponse response = indicesAdminClient.prepareTypesExists(index).setTypes(type).get();
        return response.isExists();
    }
```

### 1.3 创建复杂索引

```java
/**
     * 3.创建复杂索引，即有映射
     *
     * @param indices
     * @param type
     * @param clusterName
     * @param host
     * @param port
     * @return
     */
    public boolean createYxMapping(String indices, String type, String clusterName, String host, int port) {
        try {
            Settings settings = Settings.settingsBuilder()
                    .put("cluster.name", clusterName)
                    .put("client.transport.sniff", true)
                    .build();
            Client client = TransportClient.builder().settings(settings).build()
                    .addTransportAddress(new InetSocketTransportAddress(InetAddress.getByName(host), port));
            client.admin().indices().prepareCreate(indices).execute().actionGet();
            new XContentFactory();
            XContentBuilder builder = XContentFactory.jsonBuilder()
                    .startObject()
                    .startObject(type)
                    .startObject("properties")
                    .startObject("name").field("type", "string").field("store", "yes").field("analyzer", "ik").field("index", "analyzed").endObject()
                    .startObject("num").field("type", "long").endObject()
                    .startObject("sex").field("type", "string").field("index", "not_analyzed").endObject()
                    .endObject()
                    .endObject()
                    .endObject();
            PutMappingRequest mapping = Requests.putMappingRequest(indices).type(type).source(builder);

            client.admin().indices().putMapping(mapping).actionGet();
            client.close();
            return true;
        } catch (Exception e) {
            logger.error("createMapping error ", e);
        }
        return false;
    }
```

### 1.4 创建空索引

创建一个空索引，无映射mapping，当我们设计好mapping的相关设置再映射即可，空索引创建如下：

```java
/**
 * 4.创建空索引  默认setting 无mapping
 * @param client
 * @param index
 * @return
 */
public boolean createSimpleIndex(Client client, String index){
    IndicesAdminClient indicesAdminClient = client.admin().indices();
    CreateIndexResponse response = indicesAdminClient.prepareCreate(index).get();
    return response.isAcknowledged();
}
```

### 1.5 为空索引映射mapping

可以为空索引创建或修改索引的映射，需要确保索引存在，否则会报错，使用如下：

```java
/**
 * 5.设置映射
 * @param client
 * @param index
 * @param type
 * @return
 */
public boolean putIndexMapping(Client client, String index, String type){
    // mapping
    XContentBuilder mappingBuilder;
    try {
        mappingBuilder = XContentFactory.jsonBuilder()
                .startObject()
                .startObject(type)
                .startObject("properties")
                .startObject("name").field("type", "string").field("store", "yes").field("analyzer", "ik").field("index", "analyzed").endObject()
                 .startObject("num").field("type", "long").endObject()
                 .startObject("sex").field("type", "string").field("index", "not_analyzed").endObject()
                 .endObject()
                 .endObject()
                 .endObject();
    } catch (Exception e) {
        logger.error("--------- createIndex 创建 mapping 失败：", e);
        return false;
    }
    IndicesAdminClient indicesAdminClient = client.admin().indices();
    PutMappingResponse response = indicesAdminClient.preparePutMapping(index).setType(type).setSource(mappingBuilder).get();
    return response.isAcknowledged();
}
```



### 1.6 删除索引

删除索引api：

```java
/**
     * 6.删除索引
     *
     * @param client
     * @param index
     */
    public boolean deleteIndex(Client client, String index) {
        IndicesAdminClient indicesAdminClient = client.admin().indices();
        DeleteIndexResponse response = indicesAdminClient.prepareDelete(index).execute().actionGet();
        return response.isAcknowledged();
    }
```

### 1.7 关闭索引

如果我们不想直接删除索引，而是仅仅短暂停止某个索引的使用，那么我们可以关闭索引的这个功能，只是短暂使用之，后续可能有恢复的需求，关闭使用如下：

```java
/**
 * 7.关闭索引
 * @param client
 * @param index
 * @return
 */
public boolean closeIndex(Client client, String index){
    IndicesAdminClient indicesAdminClient = client.admin().indices();
    CloseIndexResponse response = indicesAdminClient.prepareClose(index).get();
    return response.isAcknowledged();
}

```

### 1.8 打开索引

既然有关闭索引，那么肯定也有重新打开索引的操作，使用如下：

```java
/**
 * 8.开启索引
 * @param client
 * @param index
 * @return
 */
public boolean openIndex(Client client, String index){
    IndicesAdminClient indicesAdminClient = client.admin().indices();
    OpenIndexResponse response = indicesAdminClient.prepareOpen(index).get();
    return response.isAcknowledged();
}
```



### 1.9 判断别名是否存在

上一篇我们学习了索引别名的重要，所以api也不能落下，使用如下：

```java
/**
     * 9.判断别名是否存在
     *
     * @param
     * @return
     */
    public boolean isAliasExist(Client client, String... aliases) {
        IndicesAdminClient indicesAdminClient = client.admin().indices();
        AliasesExistResponse response = indicesAdminClient.prepareAliasesExist(aliases).get();
        return response.isExists();
    }
```

### 1.10 添加别名 

别名是映射到实际索引上，所以需要传入实际索引名以及要设置的别名alias，使用如下：

```java
/**
     * 10.添加别名
     *
     * @param
     * @return
     */
    public boolean addAlias(TransportClient client, String index, String alias) {
        IndicesAdminClient indicesAdminClient = client.admin().indices();
        IndicesAliasesResponse response = indicesAdminClient.prepareAliases().addAlias(index, alias).get();
        return response.isAcknowledged();
    }
```

### 1.11 移除别名

当然，添加的别名也可以删除，方便设置新的别名和切换索引别名，使用如下：

```java
/**
     * 11.移除别名
     *
     * @param
     * @return
     */
    public boolean removeAlias(TransportClient client, String index, String alias) {
        IndicesAdminClient indicesAdminClient = client.admin().indices();
        IndicesAliasesResponse response = indicesAdminClient.prepareAliases().removeAlias(index, alias).get();
        return response.isAcknowledged();
    }
```

### 1.12 删除一个别名后再添加一个别名

这里实际上将删除与添加别名放在一起操作，所以需要传就别名和新别名，使用如下：

```java
/**
     * 12.删除一个别名后再添加一个
     *
     * @param
     * @return
     */
    @Override
    public boolean removeAndCreateAlias(TransportClient client, String indexOld, String indexNew, String alias) {
        IndicesAdminClient indicesAdminClient = client.admin().indices();
        IndicesAliasesResponse response = indicesAdminClient.prepareAliases().removeAlias(indexOld, alias)
                .addAlias(indexNew, alias).get();
        return response.isAcknowledged();
    }
```





## 2. 创建测试基类连接es

创建test类测试连接、插入、查询数据是否正常。

这里可以模拟一批数据插入es，后面的分页操作对大量数据才看得出效果，我这里有的40万数据加入到es中，方便后面的分页测试。

ElasticsearchTest.java:

```java
import com.alibaba.fastjson.JSON;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.sun.controller.DateUtils;
import org.elasticsearch.action.admin.indices.analyze.AnalyzeAction;
import org.elasticsearch.action.admin.indices.analyze.AnalyzeRequestBuilder;
import org.elasticsearch.action.admin.indices.analyze.AnalyzeResponse;
import org.elasticsearch.action.admin.indices.mapping.put.PutMappingRequest;
import org.elasticsearch.action.delete.DeleteResponse;
import org.elasticsearch.action.get.GetResponse;
import org.elasticsearch.action.index.IndexResponse;
import org.elasticsearch.action.search.SearchRequestBuilder;
import org.elasticsearch.action.search.SearchResponse;
import org.elasticsearch.action.search.SearchType;
import org.elasticsearch.action.update.UpdateResponse;
import org.elasticsearch.client.Client;
import org.elasticsearch.client.Requests;
import org.elasticsearch.client.transport.TransportClient;
import org.elasticsearch.common.settings.Settings;
import org.elasticsearch.common.transport.InetSocketTransportAddress;
import org.elasticsearch.common.unit.TimeValue;
import org.elasticsearch.common.xcontent.XContentBuilder;
import org.elasticsearch.common.xcontent.XContentFactory;
import org.elasticsearch.index.query.*;
import org.elasticsearch.search.SearchHits;
import org.junit.Before;
import org.junit.Test;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.BufferedReader;
import java.io.File;
import java.io.FileInputStream;
import java.io.InputStreamReader;
import java.net.InetAddress;
import java.net.UnknownHostException;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;

import static org.elasticsearch.common.xcontent.XContentFactory.jsonBuilder;

/**
 * 类功能描述:Elasticsearch的基本测试
 *
 * @author liu
 * @since 2018/5/24
 */
public class ElasticsearchTest {

    private Logger logger = LoggerFactory.getLogger(ElasticsearchTest.class);

    public final static String HOST = "10.10.13.234";

    /** http请求的端口是9200，客户端是9300 */
    public final static int PORT = 9300;

    Client client;
    /** 索引库名 */
    private static String index = "my_index";
    /** 类型名称 */
    private static String type = "skuId";
    private static String clusterName = "my-application";

    /**
     * 创建mapping，注意：每次只能创建一次
     * 创建mapping(field("indexAnalyzer","ik")该字段分词IK索引 ；field("searchAnalyzer","ik")该字段分词ik查询；具体分词插件请看IK分词插件说明)
     * 创建mapping(field("analyzer","ik")该字段分词IK索引 ；field("index","analyzed")该字段分词ik查询；具体分词插件请看IK分词插件说明)2.x版
     * @param indices 索引名称；
     * @param mappingType 类型
     * @throws Exception
     */
    @Test
    public void createMapping()throws Exception {
        String indices = "jd_mall_v2";
        String mappingType = "goods";

        Settings settings = Settings.settingsBuilder()
                .put("cluster.name", clusterName)
                .put("client.transport.sniff", true)
                .build();

        Client client = TransportClient.builder().settings(settings).build()
                .addTransportAddress(new InetSocketTransportAddress(InetAddress.getByName(HOST), PORT));

        client.admin().indices().prepareCreate(indices).execute().actionGet();
        new XContentFactory();
        XContentBuilder builder=XContentFactory.jsonBuilder()
                .startObject()
                .startObject(mappingType)
                .startObject("properties")
                .startObject("name").field("type", "string").field("store", "yes").field("analyzer","ik").field("index","analyzed").endObject()
                .startObject("price").field("type", "long").endObject()
                .endObject()
                .endObject()
                .endObject();
        PutMappingRequest mapping = Requests.putMappingRequest(indices).type(mappingType).source(builder);

        client.admin().indices().putMapping(mapping).actionGet();
        client.close();
    }



    @Before
    public void before()
    {

        /**
         * 1:通过 setting对象来指定集群配置信息
         * //指定集群名称
         * //启动嗅探功能
         */
        Settings settings = Settings.settingsBuilder()
                .put("cluster.name", clusterName)
                .put("client.transport.sniff", true)
                .build();

        /**
         * 2：创建客户端
         * 通过setting来创建，若不指定则默认链接的集群名为elasticsearch
         * 链接使用tcp协议即9300
         */
        try {
            client = TransportClient.builder().settings(settings).build()
                    .addTransportAddress(new InetSocketTransportAddress(InetAddress.getByName(HOST), PORT));
            System.out.println("Elasticsearch connect info:" + client.toString());
        } catch (UnknownHostException e) {
            e.printStackTrace();
        }
        /*transportClient = new TransportClient(setting);
        TransportAddress transportAddress = new InetSocketTransportAddress("192.168.79.131", 9300);
        transportClient.addTransportAddresses(transportAddress);*/

        /**
         * 3：查看集群信息
         * 注意我的集群结构是：
         *   131的elasticsearch.yml中指定为主节点不能存储数据，
         *   128的elasticsearch.yml中指定不为主节点只能存储数据。
         * 所有控制台只打印了192.168.79.128,只能获取数据节点
         *
         */
        /*ImmutableList<DiscoveryNode> connectedNodes = client.connectedNodes();
        for(DiscoveryNode node : connectedNodes)
        {
            System.out.println(node.getHostAddress());
        }*/

    }

    /**
     * 测试Elasticsearch客户端连接
     * @Title: test1
     * @author sunt
     * @date 2017年11月22日
     * @return void
     * @throws UnknownHostException
     */
    @SuppressWarnings("resource")
    @Test
    public void test1() throws UnknownHostException {
        //创建客户端
        /*TransportClient client = new PreBuiltTransportClient(Settings.EMPTY).addTransportAddresses(
                new InetSocketTransportAddress(InetAddress.getByName(HOST),PORT));*/
        TransportClient client = TransportClient.builder().build().addTransportAddress(new InetSocketTransportAddress(InetAddress.getByName(HOST), PORT));

        logger.debug("Elasticsearch connect info:" + client.toString());
        System.out.println("Elasticsearch connect info:" + client.toString());

        //关闭客户端
        client.close();
    }

    /**
     * 查
     *
     * @throws Exception
     */
    @Test
    public void testGet() throws Exception {

        String skuId = "3724941";
        String skuName = "华盛顿苹果礼盒 8个装（4个红蛇果+4个青苹果）单果重约145g-180g 新鲜水果礼盒";
        String skuPrice = "4990";

        //获取_id为1的类型
        /*GetResponse response1 = client.prepareGet(index, "person", "9").get();
        response1.getSourceAsMap();//获取值转换成map
        System.out.println("查询一条数据:" + JSON.toJSON(response1.getSourceAsMap()));*/

        //获取分词
        List<String> listAnalysis = getIkAnalyzeSearchTerms("进口水果");

        //List<Map<String, Object>> resultList6 = testMultiQueryStringQuery(index, type, "name", listAnalysis);
        List<Map<String, Object>> resultList3 = testQueryStringQuery(index, type, "name", "进口水果");
        //List<Map<String, Object>> resultList4 = getDataByWildcard(index, type, "name", "*水果*");
        //List<Map<String, Object>> resultList5 = getDataByMatch(index, type, "name", "进口 水果");
        //List<Map<String, Object>> resultList = getQueryDataBySingleField(index, type, "name", "进口水果");
        //List<Map<String, Object>> resultList2 = testSearchByPrefix(index, "name", "进口水果");
        //System.out.println();


    }

    /**
     * 增查
     *
     * @throws Exception
     */
    @Test
    public void testCreate() throws Exception {
        String[] skuIds = {"2246904", "2138027", "3724941", "5664705", "3711635"};
        String[] skuNames = {"果花 珍珠岩 无土栽培基质 颗粒状 保温性能好 园艺用品",
                            "进口华盛顿红蛇果 苹果4个装 单果重约180g 新鲜水果",
                            "华盛顿苹果礼盒 8个装（4个红蛇果+4个青苹果）单果重约145g-180g 新鲜水果礼盒",
                            "新疆阿克苏冰糖心 约5kg 单果200-250g（7Fresh 专供）",
                            "江西名牌 杨氏精品脐橙 5kg装 铂金果 水果橙子礼盒 2种包装随机发货"};
        String[] skuPrices = {"500", "3800", "4990", "8500", "5880"};

        for (int i=0; i<skuIds.length; i++) {
            String skuId = skuIds[i];
            String skuName = skuNames[i];
            String skuPrice = skuPrices[i];

            String jsonStr = "{" +
                    "\"skuId\":\""+skuId+"\"," +
                    "\"skuName\":\""+skuName+"\"," +
                    "\"skuPrice\":\""+skuPrice+"\"" +
                    "}";


            //参数说明： 索引，类型 ，_id；//setSource可以传以上map string  byte[] 几种方式
            IndexResponse response = client.prepareIndex(index, type, skuId)
                    .setSource(jsonStr,XContentType.JSON)
                    .get();
            boolean created = response.isCreated();
            System.out.println("创建一条记录:" + created);

            //获取_id为1的类型
            GetResponse response1 = client.prepareGet(index, type, skuId).get();
            response1.getSourceAsMap();//获取值转换成map
            System.out.println("查询一条数据:" + JSON.toJSON(response1.getSourceAsMap()));
        }
    }

    /**
     * 删查
     *
     * @throws Exception
     */
    @Test
    public void testDelete() throws Exception {
        String skuId = "3724941";
        String skuName = "华盛顿苹果礼盒 8个装（4个红蛇果+4个青苹果）单果重约145g-180g 新鲜水果礼盒";
        String skuPrice = "4990";
        String skuImage = "jfs/t16450/249/461724533/219792/7f204d7a/5a321ecbN4526f7d3.jpg";

        //删除_id为1的类型
        DeleteResponse response2 = client.prepareDelete(index, type, skuId).get();
        System.out.println("删除一条数据：" + response2.isFound());

        //获取_id为1的类型
        GetResponse response1 = client.prepareGet(index, type, skuId).get();
        response1.getSourceAsMap();//获取值转换成map
        System.out.println("查询一条数据:" + JSON.toJSON(response1.getSourceAsMap()));
    }

    /**
     * 改查
     *
     * @throws Exception
     */
    @Test
    public void testUpdate() throws Exception {
        String skuId = "3724941";
        String skuName = "华盛顿苹果礼盒 8个装（4个红蛇果+4个青苹果）单果重约145g-180g 新鲜水果礼盒";
        String skuPrice = "4990";
        String skuName2 = "华圣 陕西精品红富士苹果 12个装 果径75mm 约2.1kg 新鲜水果";
        //更新
        UpdateResponse updateResponse = client.prepareUpdate(index, type, skuId).setDoc(jsonBuilder()
                .startObject()
                .field("name", skuName2)
                .endObject())
                .get();
        System.out.println("更新一条数据:" + updateResponse.isCreated());

        //获取_id为1的类型
        GetResponse response1 = client.prepareGet(index, type, skuId).get();
        response1.getSourceAsMap();//获取值转换成map
        System.out.println("查询一条数据:" + JSON.toJSON(response1.getSourceAsMap()));
    }

    /**
     * 将对象通过jackson.databind转换成byte[]
     * 注意一下date类型需要格式化处理  默认是 时间戳
     *
     * @return
     */
    public byte[] convertByteArray(Object obj) {
        // create once, reuse
        ObjectMapper mapper = new ObjectMapper();
        try {
            byte[] json = mapper.writeValueAsBytes(obj);
            return json;
        } catch (JsonProcessingException e) {
            e.printStackTrace();
        }
        return null;
    }

    /**
     * 将对象通过JSONtoString转换成JSON字符串
     * 使用fastjson 格式化注解  在属性上加入 @JSONField(format="yyyy-MM-dd HH:mm:ss")
     *
     * @return
     */
    public String jsonStr(Object obj) {
        return JSON.toJSONString(obj);
    }

    /*****************************/
    /**
            * 根据文档名、字段名、字段值查询某一条记录的详细信息；query查询
     * @param type  文档名，相当于oracle中的表名，例如：ql_xz；
            * @param key 字段名，例如：bdcqzh
     * @param value  字段值，如：“”
            * @return  List
     * @author Lixin
     */
    public List getQueryDataBySingleField(String index, String type, String key, String value){
        //TransportClient client = getClient();
        QueryBuilder qb = QueryBuilders.termQuery(key, value);
        SearchResponse response = client.prepareSearch(index).setTypes(type)
                .setQuery(qb)
                .setFrom(0).setSize(10000).setExplain(true)
                .execute()
                .actionGet();
        return responseToList(client,response);
    }


    /**
     * 多条件  文档名、字段名、字段值，查询某一条记录的详细信息
     * @param type 文档名，相当于oracle中的表名，例如：ql_xz
     * @param map 字段名：字段值 的map
     * @return List
     * @author Lixin
     */
    public List getBoolDataByMuchField(String index, String type, Map<String,String> map){
        //TransportClient client = getClient();
        BoolQueryBuilder boolQueryBuilder = QueryBuilders.boolQuery();
        for (String in : map.keySet()) {
            //map.keySet()返回的是所有key的值
            String str = map.get(in);//得到每个key多对用value的值
            boolQueryBuilder.must(QueryBuilders.termQuery(in,str));
        }
        SearchResponse response = client.prepareSearch(index).setTypes(type)
                .setQuery(boolQueryBuilder)
                .setFrom(0).setSize(10000).setExplain(true)
                .execute()
                .actionGet();
        return responseToList(client,response);
    }


    /**
     * 单条件 通配符查询
     * @param type 文档名，相当于oracle中的表名，例如：ql_xz
     * @param key 字段名，例如：bdcqzh
     * @param value 字段名模糊值：如 *123* ;?123*;?123?;*123?;
     * @return List
     * @author Lixin
     */
    public List getDataByWildcard(String index, String type, String key, String value){
        //TransportClient client = getClient();
        WildcardQueryBuilder wBuilder = QueryBuilders.wildcardQuery(key, value);
        SearchResponse response = client.prepareSearch(index).setTypes(type)
                .setQuery(wBuilder)
                .setFrom(0).setSize(10000).setExplain(true)
                .execute()
                .actionGet();
        return responseToList(client,response);
    }

    /**
     * 多条件 通配符查询
     * @param type  type 文档名，相当于oracle中的表名，例如：ql_xz
     * @param map   包含key:value 模糊值键值对
     * @return List
     * @author Lixin
     */
    public List getDataByMuchWildcard(String index, String type, Map<String,String> map){
        //TransportClient client = getClient();
        BoolQueryBuilder boolQueryBuilder = QueryBuilders.boolQuery();
        for (String in : map.keySet()) {
            //map.keySet()返回的是所有key的值
            String str = map.get(in);//得到每个key多对用value的值
            boolQueryBuilder.must(QueryBuilders.wildcardQuery(in,str));
        }
        SearchResponse response = client.prepareSearch(index).setTypes(type)
                .setQuery(boolQueryBuilder)
                .setFrom(0).setSize(10000).setExplain(true)
                .execute()
                .actionGet();

        return responseToList(client,response);
    }

    /**
     * 单条件 模糊拆分查询
     * @param type 文档名，相当于oracle中的表名，例如：ql_xz
     * @param key 字段名，例如：bdcqzh
     * @param value 字段名模糊值：如 *123* ;?123*;?123?;*123?;
     * @return List
     * @author Lixin
     */
    public List getDataByMatch(String index, String type, String key, String value){
        //TransportClient client = getClient();
        MatchQueryBuilder mBuilder = QueryBuilders.matchQuery(key, value);
        SearchResponse response = client.prepareSearch(index).setTypes(type)
                .setQuery(mBuilder)
                .setFrom(0).setSize(10000).setExplain(true)
                .execute()
                .actionGet();
        return responseToList(client,response);
    }
    /**
     * 单条件 中文分词模糊查询
     * @param type 文档名，相当于oracle中的表名，例如：ql_xz
     * @param key 字段名，例如：bdcqzh
     * @param value 字段名模糊值：如 *123* ;?123*;?123?;*123?;
     * @return List
     * @author Lixin
     */
    public List getDataByPrefix(String index, String type, String key, String value){
        //TransportClient client = getClient();
        PrefixQueryBuilder pBuilder = QueryBuilders.prefixQuery(key, value);
        SearchResponse response = client.prepareSearch(index).setTypes(type)
                .setQuery(pBuilder)
                .setFrom(0).setSize(10000).setExplain(true)
                .execute()
                .actionGet();
        return responseToList(client,response);
    }

    /**
     * 中文分词的操作
     * 1.查询以"中"开头的数据，有两条
     * 2.查询以“中国”开头的数据，有0条
     * 3.查询包含“烂”的数据，有1条
     * 4.查询包含“烂摊子”的数据，有0条
     * 分词：
     *      为什么我们搜索China is the greatest country~
     *                 中文：中国最牛逼
     *
     *                 ×××
     *                      中华
     *                      人民
     *                      共和国
     *                      中华人民
     *                      人民共和国
     *                      华人
     *                      共和
     *      特殊的中文分词法：
     *          庖丁解牛
     *          IK分词法
     *          搜狗分词法
     */
    public List testSearchByPrefix(String index, String key, String value) {
        // 在prepareSearch()的参数为索引库列表，意为要从哪些索引库中进行查询
        SearchResponse response = client.prepareSearch(index)
                .setSearchType(SearchType.DEFAULT)  // 设置查询类型，有QUERY_AND_FETCH  QUERY_THEN_FETCH  DFS_QUERY_AND_FETCH  DFS_QUERY_THEN_FETCH
                //.setSearchType(SearchType.DEFAULT)  // 设置查询类型，有QUERY_AND_FETCH  QUERY_THEN_FETCH  DFS_QUERY_AND_FETCH  DFS_QUERY_THEN_FETCH
                //.setQuery(QueryBuilders.prefixQuery("content", "烂摊子"))// 设置相应的query，用于检索，termQuery的参数说明：name是doc中的具体的field，value就是要找的具体的值
//                .setQuery(QueryBuilders.regexpQuery("content", ".*烂摊子.*"))
                .setQuery(QueryBuilders.prefixQuery(key, value))
                .get();
        return responseToList(client,response);
    }

    public List testFuzzyQuery(String index, String type, String key, String value){
        //TransportClient client = getClient();
        FuzzyQueryBuilder fBuilder = QueryBuilders.fuzzyQuery(key, value);
        SearchResponse response = client.prepareSearch(index).setTypes(type)
                .setQuery(fBuilder)
                .setFrom(0).setSize(10000).setExplain(true)
                .execute()
                .actionGet();
        return responseToList(client,response);
    }
    public List testQueryStringQuery(String index, String type, String key, String value){
        //TransportClient client = getClient();
        QueryBuilder fBuilder = QueryBuilders.queryStringQuery(value).field(key);
        SearchResponse response = client.prepareSearch(index).setTypes(type)
                .setQuery(fBuilder)
                .setFrom(0).setSize(10000).setExplain(true)
                .execute()
                .actionGet();
        return responseToList(client,response);
    }
    public List testMultiQueryStringQuery(String index, String type, String key, List<String> listAnalysis){
        //TransportClient client = getClient();
        BoolQueryBuilder boolQueryBuilder = QueryBuilders.boolQuery();
        for (String term : listAnalysis) {
            boolQueryBuilder.should(QueryBuilders.queryStringQuery(term).field(key));
            //这里可以用must 或者 should 视情况而定
        }
        SearchResponse response = client.prepareSearch(index).setTypes(type)
                .setQuery(boolQueryBuilder)
                .setFrom(0).setSize(10000).setExplain(true)
                .execute()
                .actionGet();
        return responseToList(client,response);
    }


    /**
     * 将查询后获得的response转成list
     * @param client
     * @param response
     * @return
     */
    public List responseToList(Client client, SearchResponse response){
        SearchHits hits = response.getHits();
        List<Map<String, Object>> list = new ArrayList<Map<String,Object>>();
        System.out.println("------------------------");
        for (int i = 0; i < hits.getHits().length; i++) {
            Map<String, Object> map = hits.getAt(i).getSource();
            System.out.println(map.get("name"));
            list.add(map);
        }
        System.out.println("------------------------");
        //client.close();
        return list;
    }



    /**
     * 调用 ES 获取 IK 分词后结果
     *
     * @param searchContent
     * @return
     */
    private List<String> getIkAnalyzeSearchTerms(String searchContent) {
        //调用 IK 分词分词
        AnalyzeRequestBuilder ikRequest = new AnalyzeRequestBuilder(client,
                AnalyzeAction.INSTANCE, index, searchContent);
        //ikRequest.setAnalyzer("ik");
        ikRequest.setTokenizer("ik");
        List<AnalyzeResponse.AnalyzeToken> ikTokenList = ikRequest.execute().actionGet().getTokens();

        //循环赋值
        List<String> searchTermList = new ArrayList<>();
        //ikTokenList.forEach(ikToken -> { searchTermList.add(ikToken.getTerm()); });
        for (AnalyzeResponse.AnalyzeToken ikToken: ikTokenList) {
            System.out.println(ikToken.getTerm());
            searchTermList.add(ikToken.getTerm());
        }

        return searchTermList;
    }

    /**
     * 然后就是创建两个查询过程了 ，下面是from-size分页的执行代码：
     */
    @Test
    public void testFromSize() {
        System.out.println("from size 模式启动！");
        long begin = System.currentTimeMillis();
        long count = client.prepareCount(index).setTypes(type).execute()
                .actionGet().getCount();
        //QueryBuilder qBuilder = QueryBuilders.queryStringQuery(value).field(key);
        SearchRequestBuilder requestBuilder = client.prepareSearch(index).setTypes(type)
                .setQuery(QueryBuilders.queryStringQuery("进口水果").field("name"));
        for (int i=0,sum=0; sum<count; i++) {
            SearchResponse response = requestBuilder.setFrom(i).setSize(3).execute().actionGet();
            sum += response.getHits().hits().length;
            responseToList(client,response);
            System.out.println("总量"+count+", 已经查到"+sum);
        }
        System.out.println("elapsed<"+(System.currentTimeMillis()-begin)+">ms.");
    }
    /**
     * 下面是scroll分页的执行代码，注意啊！scroll里面的size是相对于每个分片来说的，
     * 所以实际返回的数量是：分片的数量*size
     */
    @Test
    public void testScroll() {
        //String index, String type,String key,String value
        System.out.println("scroll 模式启动！");
        long begin = System.currentTimeMillis();
        //QueryBuilder qBuilder = QueryBuilders.queryStringQuery(value).field(key);
        //获取Client对象,设置索引名称,搜索类型(SearchType.SCAN),搜索数量,发送请求
        SearchResponse scrollResponse = client.prepareSearch(index)
                .setTypes(type)
                .setQuery(QueryBuilders.queryStringQuery("进口水果").field("name"))
                .setSearchType(SearchType.SCAN).setSize(3).setScroll(TimeValue.timeValueMinutes(1))
                .execute().actionGet();
        //注意:首次搜索并不包含数据，第一次不返回数据，获取总数量
        long count = scrollResponse.getHits().getTotalHits();
        for (int i=0,sum=0; sum<count; i++) {
            scrollResponse = client.prepareSearchScroll(scrollResponse.getScrollId())
                    .setScroll(TimeValue.timeValueMinutes(8)).execute().actionGet();
            sum += scrollResponse.getHits().hits().length;
            responseToList(client,scrollResponse);
            System.out.println("总量"+count+", 已经查到"+sum);
        }
        System.out.println("elapsed<"+(System.currentTimeMillis()-begin)+">ms.");
    }
}
```



## 3. 分页代码

创建TestController类，在上述测试都无问题之后，再进行下面的分页操作。

TestController.java：

```java
package com.sun.controller;

import com.alibaba.fastjson.JSONObject;
import org.elasticsearch.action.search.SearchRequestBuilder;
import org.elasticsearch.action.search.SearchScrollRequestBuilder;
import org.elasticsearch.search.SearchHit;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.elasticsearch.action.admin.indices.analyze.AnalyzeAction;
import org.elasticsearch.action.admin.indices.analyze.AnalyzeRequestBuilder;
import org.elasticsearch.action.admin.indices.analyze.AnalyzeResponse;
import org.elasticsearch.action.search.SearchResponse;
import org.elasticsearch.action.search.SearchType;
import org.elasticsearch.client.Client;
import org.elasticsearch.client.transport.TransportClient;
import org.elasticsearch.common.settings.Settings;
import org.elasticsearch.common.transport.InetSocketTransportAddress;
import org.elasticsearch.common.unit.TimeValue;
import org.elasticsearch.index.query.BoolQueryBuilder;
import org.elasticsearch.index.query.QueryBuilder;
import org.elasticsearch.index.query.QueryBuilders;
import org.elasticsearch.search.SearchHits;
import org.elasticsearch.search.sort.SortOrder;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.ResponseBody;
import org.springframework.web.bind.annotation.RestController;

import java.net.InetAddress;
import java.net.UnknownHostException;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.concurrent.atomic.AtomicLong;

/**
 * 类功能描述:
 *
 * @author liu
 * @since 2018/2/22
 */
@RestController
public class TestController {
    private Logger logger = LoggerFactory.getLogger(this.getClass());
    public final static String HOST = "10.10.13.234";

    /** http请求的端口是9200，客户端是9300 */
    public final static int PORT = 9300;

    /** 索引库名 */
    private static String index = "my_index";
    /** 类型名称 */
    private static String type = "skuId";
    private static String clusterName = "my-application";

    private static final String template = "Hello, %s!";
    private final AtomicLong counter = new AtomicLong();

    @ResponseBody
    @RequestMapping("/greeting")
    public Greeting greeting(@RequestParam(value="name", defaultValue="World") String name) {
        return new Greeting(counter.incrementAndGet(),
                String.format(template, name));
    }

    public long getTotalRows(String keyword, TransportClient client) {
        long totalRows;
        BoolQueryBuilder boolQueryBuilder = QueryBuilders.boolQuery();
        boolQueryBuilder.must(QueryBuilders.termQuery("isShow", 1))
                .must(QueryBuilders.termQuery("status", 1))
                .must(QueryBuilders.termQuery("deleteStatus", 1))
                .must(QueryBuilders.termQuery("salesStatus", 1));
        if (keyword == null || keyword.isEmpty()) {
            totalRows = client.prepareCount(index).setTypes(type)
                    .execute()
                    .actionGet().getCount();
        } else {
            QueryBuilder fBuilder = QueryBuilders.queryStringQuery(keyword).field("name");
            boolQueryBuilder.must(fBuilder);
            totalRows = client.prepareCount(index).setTypes(type)
                    .setQuery(boolQueryBuilder)
                    .execute()
                    .actionGet().getCount();
        }
        return totalRows;
    }

    @ResponseBody
    @RequestMapping("/searchFromSize")
    public Object testSearchByFromSize(@RequestParam(required = false) Integer pageIndex, @RequestParam(required = false) Integer pageSize,
                               @RequestParam(required = false) String keyword, @RequestParam(required = false) String priceSort){
        logger.info("op<testSearchByFromSize> pageIndex<"+pageIndex+"> pageSize<"+pageSize+"> " +
                "keyword<"+keyword+"> priceSort<"+priceSort+">");
        JSONObject result = new JSONObject();
        JSONObject goodsResult = new JSONObject();
        long begin = System.currentTimeMillis();
        TransportClient client = getClient();
        long totalRows = getTotalRows(keyword, client);
        getIkAnalyzeSearchTerms(client, keyword);
        int from = (pageIndex - 1) * pageSize;
        List<Map<String, Object>> goodsExportList = testQueryStringQueryFromSize(client, index, type, "name",
                keyword, from, pageSize, priceSort);
        goodsResult.put( "goods", goodsExportList );
        goodsResult.put( "pageIndex", pageIndex );
        goodsResult.put( "pageSize", pageSize );
        goodsResult.put( "totalRows", totalRows );

        result.put( "success", true );
        result.put( "resultMessage", "" );
        result.put( "resultCode", "0000" );
        result.put( "result", goodsResult );
        System.out.println("---op<searchFromSize> elapsed<"+(System.currentTimeMillis()-begin)+">ms");
        return result;
    }

    public String scrollIdTemp = "";

    @ResponseBody
    @RequestMapping("/searchScroll")
    public Object testSearchByScroll(@RequestParam(required = false) String scrollId, @RequestParam(required = false) Integer pageSize,
                               @RequestParam(required = false) String keyword, @RequestParam(required = false) String priceSort){
        logger.info("op<testSearchByScroll> scrollId<"+scrollId+"> pageSize<"+pageSize+"> " +
                "keyword<"+keyword+"> priceSort<"+priceSort+">");
        JSONObject result = new JSONObject();
        JSONObject goodsResult = new JSONObject();
        long begin = System.currentTimeMillis();
        scrollIdTemp = scrollId;
        TransportClient client = getClient();
        long totalRows = getTotalRows(keyword, client);
        getIkAnalyzeSearchTerms(client, keyword);
        List<Map<String, Object>> goodsExportList = testQueryStringQueryScroll(client, index, type, "name",
                keyword, scrollIdTemp, pageSize, priceSort);
        client.close();

        goodsResult.put( "goods", goodsExportList );
        goodsResult.put( "scrollId", scrollIdTemp );
        goodsResult.put( "pageSize", pageSize );
        goodsResult.put( "totalRows", totalRows );

        result.put( "success", true );
        result.put( "resultMessage", "" );
        result.put( "resultCode", "0000" );
        result.put( "result", goodsResult );
        System.out.println("---op<searchScroll> elapsed<"+(System.currentTimeMillis()-begin)+">ms");
        return result;
    }

    /**
     * 然后就是创建两个查询过程了 ，下面是from-size分页的执行代码：
     */
    public List testQueryStringQueryFromSize(TransportClient client, String index, String type, String key,
                                     String value, Integer pageIndex, Integer pageSize ,String priceSort){
        SearchResponse response = null;
        List resultList = null;
        try {
            //TransportClient client = getClient();
            BoolQueryBuilder boolQueryBuilder = QueryBuilders.boolQuery();
            if (value == null || value.isEmpty()) {
                boolQueryBuilder.must(QueryBuilders.termQuery("isShow", 1))
                        .must(QueryBuilders.termQuery("status", 1))
                        .must(QueryBuilders.termQuery("deleteStatus", 1))
                        .must(QueryBuilders.termQuery("salesStatus", 1));
            } else {
                QueryBuilder fBuilder = QueryBuilders.queryStringQuery(value).field(key);
                boolQueryBuilder.must(QueryBuilders.termQuery("isShow", 1))
                        .must(QueryBuilders.termQuery("status", 1))
                        .must(QueryBuilders.termQuery("deleteStatus", 1))
                        .must(QueryBuilders.termQuery("salesStatus", 1))
                        .must(fBuilder);
            }

            if (priceSort == null || priceSort.isEmpty()) {
                response = client.prepareSearch(index).setTypes(type)
                        .setQuery(boolQueryBuilder)
                        .setFrom(pageIndex).setSize(pageSize).setExplain(true)
                        .execute()
                        .actionGet();
            } else {
                if (("desc").equals(priceSort)) {
                    response = client.prepareSearch(index).setTypes(type)
                            .setQuery(boolQueryBuilder)
                            .setFrom(pageIndex).setSize(pageSize).setExplain(true)
                            .addSort("price", SortOrder.DESC)
                            .execute()
                            .actionGet();
                } else {
                    response = client.prepareSearch(index).setTypes(type)
                            .setQuery(boolQueryBuilder)
                            .setFrom(pageIndex).setSize(pageSize).setExplain(true)
                            .addSort("price", SortOrder.ASC)
                            .execute()
                            .actionGet();
                }
            }
            resultList = responseToList(response);
        } catch (Exception e) {
            e.printStackTrace();
        }

        return resultList;
    }
    /**
     * 下面是scroll分页的执行代码，注意啊！scroll里面的size是相对于每个分片来说的，
     * 所以实际返回的数量是：分片的数量*size
     *
     * Scroll-Scan 方式与普通 scroll 有几点不同：
     * 1.Scroll-Scan 结果没有排序，按 index 顺序返回，没有排序，可以提高取数据性能。
     * 2.初始化时只返回 _scroll_id，没有具体的 hits 结果。
     * 3.size 控制的是每个分片的返回的数据量而不是整个请求返回的数据量。
     */
    public List testQueryStringQueryScroll(TransportClient client, String index, String type, String key,
                                     String value, String scrollId, Integer pageSize ,String priceSort){
        if (pageSize != null) {
            //默认有5个分片，
            pageSize = pageSize/5;
        }
        SearchResponse scrollResponse = null;
        List resultList = null;
        try {
            //TransportClient client = getClient();
            BoolQueryBuilder boolQueryBuilder = QueryBuilders.boolQuery();
            if (value == null || value.isEmpty()) {
                boolQueryBuilder.must(QueryBuilders.termQuery("isShow", 1))
                        .must(QueryBuilders.termQuery("status", 1))
                        .must(QueryBuilders.termQuery("deleteStatus", 1))
                        .must(QueryBuilders.termQuery("salesStatus", 1));
            } else {
                QueryBuilder fBuilder = QueryBuilders.queryStringQuery(value).field(key);
                boolQueryBuilder.must(QueryBuilders.termQuery("isShow", 1))
                        .must(QueryBuilders.termQuery("status", 1))
                        .must(QueryBuilders.termQuery("deleteStatus", 1))
                        .must(QueryBuilders.termQuery("salesStatus", 1))
                        .must(fBuilder);
            }

            if (scrollId == null || scrollId.isEmpty()) {
                if (priceSort == null || priceSort.isEmpty()) {
                    scrollResponse = client.prepareSearch(index).setTypes(type)
                            .setQuery(boolQueryBuilder)
                            .setSearchType(SearchType.SCAN).setSize(pageSize).setScroll(TimeValue.timeValueMinutes(5))
                            .execute()
                            .actionGet();
                } else {
                    if (("desc").equals(priceSort)) {
                        scrollResponse = client.prepareSearch(index).setTypes(type)
                                .setQuery(boolQueryBuilder)
                                .setSearchType(SearchType.SCAN).setSize(pageSize).setScroll(TimeValue.timeValueMinutes(5))
                                .addSort("price", SortOrder.DESC)
                                .execute()
                                .actionGet();
                    } else {
                        scrollResponse = client.prepareSearch(index).setTypes(type)
                                .setQuery(boolQueryBuilder)
                                .setSearchType(SearchType.SCAN).setSize(pageSize).setScroll(TimeValue.timeValueMinutes(5))
                                .addSort("price", SortOrder.ASC)
                                .execute()
                                .actionGet();
                    }
                }
                //注意:首次搜索并不包含数据，第一次不返回数据，所以先查一次获取scrollId，在进行第二次scroll查询
                scrollId = scrollResponse.getScrollId();
            }

            scrollResponse = client.prepareSearchScroll(scrollId)
                    .setScroll(TimeValue.timeValueMinutes(5))
                    .execute()
                    .actionGet();
            
            //注意:首次搜索并不包含数据，第一次不返回数据，获取总数量
            long count = scrollResponse.getHits().getTotalHits();
            System.out.println("scrollResponse.getHits().getHits().length:"+scrollResponse.getHits().hits().length);
            System.out.println("scroll count:"+count);

            //获取本次查询的scrollId
            scrollIdTemp = scrollResponse.getScrollId();
            System.out.println("--------- searchByScroll scrollID:"+scrollIdTemp);
            logger.info("--------- searchByScroll scrollID {}", scrollIdTemp);

            // 搜索结果
            SearchHit[] searchHits = scrollResponse.getHits().hits();
            for (SearchHit searchHit : searchHits) {
                String source = searchHit.getSource().toString();
                logger.info("--------- searchByScroll source {}", source);
            }
            resultList = responseToList(scrollResponse);
        } catch (Exception e) {
            e.printStackTrace();
        }


        return resultList;
    }

    /**
     * 将查询后获得的response转成list
     * @param response
     * @return
     */
    public List responseToList(SearchResponse response){
        SearchHits hits = response.getHits();
        List<Map<String, Object>> list = new ArrayList<>();
        System.out.println("hits.hits().length:"+hits.hits().length);
        System.out.println("------------------------");
        for (int i = 0; i < hits.hits().length; i++) {
            Map<String, Object> map = hits.getAt(i).getSource();
            Map<String, Object> resultMap = new HashMap<>(16);
            System.out.println(map.get("name"));
            resultMap.put("skuId", map.get("skuId"));
            resultMap.put("name", map.get("name"));
            resultMap.put("image", map.get("primaryImage"));
            resultMap.put("price", map.get("luckyPrice"));
            list.add(resultMap);
        }
        System.out.println("------------------------");
        return list;
    }

    public TransportClient getClient() {
        /**
         * 1:通过 setting对象来指定集群配置信息
         * //指定集群名称
         * //启动嗅探功能
         */
        Settings settings = Settings.settingsBuilder()
                .put("cluster.name", clusterName)
                .put("client.transport.sniff", true)
                .build();

        /**
         * 2：创建客户端
         * 通过setting来创建，若不指定则默认链接的集群名为elasticsearch
         * 链接使用tcp协议即9300
         */
        TransportClient client = null;
        try {
            client = TransportClient.builder().settings(settings).build()
                    .addTransportAddress(new InetSocketTransportAddress(InetAddress.getByName(HOST), PORT));
            System.out.println("Elasticsearch connect info:" + client.toString());
        } catch (UnknownHostException e) {
            e.printStackTrace();
        }
        /*transportClient = new TransportClient(setting);
        TransportAddress transportAddress = new InetSocketTransportAddress("192.168.79.131", 9300);
        transportClient.addTransportAddresses(transportAddress);*/

        /**
         * 3：查看集群信息
         * 注意我的集群结构是：
         *   131的elasticsearch.yml中指定为主节点不能存储数据，
         *   128的elasticsearch.yml中指定不为主节点只能存储数据。
         * 所有控制台只打印了192.168.79.128,只能获取数据节点
         *
         */
        /*ImmutableList<DiscoveryNode> connectedNodes = client.connectedNodes();
        for(DiscoveryNode node : connectedNodes)
        {
            System.out.println(node.getHostAddress());
        }*/
        return client;
    }

    /**
     * 调用 ES 获取 IK 分词后结果
     *
     * @param searchContent
     * @return
     */
    private List<String> getIkAnalyzeSearchTerms(Client client, String searchContent) {
        //调用 IK 分词分词
        AnalyzeRequestBuilder ikRequest = new AnalyzeRequestBuilder(client,
                AnalyzeAction.INSTANCE, index, searchContent);
        //ikRequest.setAnalyzer("ik");
        ikRequest.setTokenizer("ik");
        List<AnalyzeResponse.AnalyzeToken> ikTokenList = ikRequest.execute().actionGet().getTokens();

        //循环赋值
        List<String> searchTermList = new ArrayList<>();
        //ikTokenList.forEach(ikToken -> { searchTermList.add(ikToken.getTerm()); });
        for (AnalyzeResponse.AnalyzeToken ikToken: ikTokenList) {
            System.out.println(ikToken.getTerm());
            searchTermList.add(ikToken.getTerm());
        }

        return searchTermList;
    }
}
```



## 4. 前端数据

为上述的分页的数据可视化，加入前端展示页，可按中文关键词搜索分页等操作。

index.html，以及js操作部分，data.js，由于篇幅问题，下载可移步至github:

https://github.com/wonderomg/elasticsearch-visual下载详细内容，

前后端页编写完之后，启动springboot工程，访问`http://localhost:8080/`该工程端口。

可以看到操作 页面，类似下图：

![search](https://raw.githubusercontent.com/wonderomg/elasticsearch-visual/master/search.png)

from size和scroll都可通过该页面进行测试。

## 5. 数据限制问题

测试from size时发现一个问题，当数据超过10000条会报错，提示如下：

```shell
Caused by: QueryPhaseExecutionException[Result window is too large, 
from + size must be less than or equal to: [10000] but was [10001]. 
See the scroll api for a more efficient way to request large data sets. 
This limit can be set by changing the [index.max_result_window] index level parameter.]
```

原因是es索引被限制了可查前10000条数据，参数是`max_result_window`，需要我们手动修改这个限制。

(1)查询index.max_result_window

```shell
curl -XGET "10.10.13.234:9200/jd_mall_v2/_settings?preserve_existing=true"
```

(2)更改index.max_result_window

```shell
curl -XPUT "10.10.13.234:9200/jd_mall_v2/_settings?preserve_existing=true" -d '{ "index" : { "max_result_window" : 100000000}}'
```

设置max_result_window的时候需要先关闭索引，不然会报错`Can't update non dynamic settings[[index.preserve_existing]] for open indices`的错误，可以通过指令关闭，也可以通过head可视化界面关闭和开启，也可以通过上述步骤1.7和1.8的api关闭和开启。

## 6. 排序

分页的时候我们可以还有排序的场景，java api的话增加代码：

```java
searchRequestBuilder.addSort("publish_time", SortOrder.DESC);
```

按照某个字段排序的话，hit.getScore()将会失效
如果是scroll分页无法进行排序。