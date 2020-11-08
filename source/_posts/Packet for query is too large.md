---
title: Packet for query is too large (84 > -1).
date: 2018-3-15
categories: 
  - ibatis
  - mysql
  - resin
tags: 
  - zookeeper
  - mysql
  - ibatis
  - resin
thumbnail: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1522683004066&di=e2500c85d9aa9d7c8b6c1329e31d50b0&imgtype=0&src=http%3A%2F%2Fmedia.fashiongroup.com%2Ffashionmag%2Fnewsletters%2Fimages%2F20110928%2Fefu_12386fac-2e69-4d75-83b6-486dc6c0600e.jpg
---

ibatis工程使用resin配置mysql出现`Packet for query is too large (84 > -1).`错误。

windows下的resin配置连接mysql，常用的安全的做法是将数据库信息配置到conf目录下的resin.xml文件中。<!--more-->

因为resin连接mysql不是必须的，所以resin本身没有提供mysql-connector的jar包，需要自己加到resin目录下的lib里面，我加了个`mysql-connector-java-5.1.45-bin.jar`进去，然而在运行工程执行数据库查询操作的时候，却出现了问题，报如下错误：

```
org.springframework.dao.TransientDataAccessResourceException: SqlMapClient operation; SQL [];   
--- The error occurred while applying a parameter map.  
--- Check the T_USER_INFO.selectByMap-InlineParameterMap.  
--- Check the statement (query failed).  
--- Cause: com.mysql.jdbc.PacketTooBigException: Packet for query is too large (84 > -1). You can change this value on the server by setting the max_allowed_packet' variable.; nested exception is com.ibatis.common.jdbc.exception.NestedSQLException:   
--- The error occurred while applying a parameter map.  
--- Check the T_USER_INFO.selectByMap-InlineParameterMap.  
--- Check the statement (query failed).  
--- Cause: com.mysql.jdbc.PacketTooBigException: Packet for query is too large (84 > -1). You can change this value on the server by setting the max_allowed_packet' variable.
```

只进行查询却出现`Packet for query is too large`，而且查询的条件只有几个字母，网上搜了很多解答，都是说需要把mysql的`max_allowed_packet`参数调大，但是我通过`show VARIABLES like '%max_allowed_packet%';`查询的结果也很大，如果是参数过小，应该是提示大于数据库中设置的packet的值，而不应该报-1。

最后发现有个帖子说是`mysql-connector-java`jar包的版本问题，版本不能太高，随后我将`mysql-connector-java-5.1.45-bin.jar`换成`mysql-connector-java-5.1.26-bin.jar`，不再出现packet错误，问题解决。

可能是因为我**用的是`ibatis2.0`版本的缘故，不支持太高的mysql连接jar版本**，换成`mysql-connector-java-5.1.40`以下版本即可解决`Packet for query is too large (84 > -1)`问题。