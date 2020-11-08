---
title: zookeeper提示Unable to read additional data from server sessionid 0x
date: 2017-9-15
categories: 
  - zookeeper
tags: 
  - zookeeper
  - 集群
thumbnail: http://static.oschina.net/uploads/img/201308/08171345_l5K3.jpg
---

配置zookeeper集群，一开始配置了两台机器server.1和server.2。

配置参数，在zoo.cfg中指定了整个zookeeper集群的server编号、地址和端口：

> server.1=10.10.16.151:2888:3888
> server.2=10.10.16.234:2888:3888

然后为这两个个节点创建对应的编号文件，在/tmp/zookeeper/data/myid文件中。如下：<!--more-->

在server.1=10.10.16.151机器上执行：

```sh
echo 1 > /tmp/zookeeper/data/myid
```

在server.1=10.10.16.234机器上执行：

```sh
echo 2 > /tmp/zookeeper/data/myid
```

启动server.1测试dubbo服务，使用`zkServer.sh start`启动了server.1，然后使用`zkServer.sh status`查看工作状态，显示
```sh
Error contacting service. It is probably not running.
```
通过`zkCli.sh -server 10.10.16.151:2181`查看服务详细，发现服务提示了下面的错误信息。

启动时提示：

```sh
2017-09-15 14:57:27,139 [myid:] - INFO  [main-SendThread(10.10.16.151:2181):ClientCnxn$SendThread@1035] - Opening socket connection to server 10.10.16.151/10.10.16.151:2181. Will not attempt to authenticate using SASL (java.lang.SecurityException: Ϟ·¨¶¨λµȂ¼Ƥ׃)
2017-09-15 14:57:27,140 [myid:] - INFO  [main-SendThread(10.10.16.151:2181):ClientCnxn$SendThread@877] - Socket connection established to 10.10.16.151/10.10.16.151:2181, initiating session
2017-09-15 14:57:27,143 [myid:] - INFO  [main-SendThread(10.10.16.151:2181):ClientCnxn$SendThread@1161] - Unable to read additional data from server sessionid 0x0, likely server has closed socket, closing socket connection and attempting reconnect
```

后来才搞明白，由于我在zoo.cfg中配置了2台机器，但是只启动了1台，zookeeper就会认为服务处于不可用状态。

**通过zookeeper的选举算法得知，当整个集群超过半数机器宕机，zookeeper会认为集群处于不可用状态。**所以启动2台服务正常。

然后我又增加了1台`server.3=10.10.16.241:2888:3888`机器节点，在3台都启动的情况下，关掉其中1台，服务正常，关掉2台，服务不可用。

所以，zookeeper集群只启动一台无法连接，如果启动机器数为半数及以上就可以连接了。