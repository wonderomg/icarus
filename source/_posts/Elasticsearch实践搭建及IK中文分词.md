---
title: Elasticsearch实践(1)-搭建及IK中文分词
date: 2018-5-15
categories: 
  - elasticsearch
tags: 
  - elasticsearch
  - ik中文分词
thumbnail: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1529643264225&di=cb242c448c2a3bab04ce1acab4135689&imgtype=0&src=http%3A%2F%2Fjbcdn2.b0.upaiyun.com%2F2017%2F10%2Fc6cf4b2000277c64f55e00cf6d2f294f.png
---



🔍近期在做商品搜索优化，对比了Solr与Elasticsearch的区别，两者都是基于Lucene实现的封装，但es在数据量越大的情况下实时检索性能优于solr，更适用于实时商品搜索，于是选用了es，下面介绍elasticsearch搜索引擎的搭建及使用，这里使用的是2.2.1的es版本，以及1.8.1的ik分词版本。

<!--more-->

# 安装es及IK中文分词

##  1. 下载安装包

```properties
##我的环境是Redhat6.4，此处使用的是es2.2.1版本
##（1）可以使用指令安装
$ yum install https://download.elasticsearch.org/elasticsearch/release/org/elasticsearch/distribution/rpm/elasticsearch/2.2.1/elasticsearch-2.2.1.rpm
##（2）也可以直接下载文件，然后上传至linux目录下
https://download.elasticsearch.org/elasticsearch/release/org/elasticsearch/distribution/tar/elasticsearch/2.2.1/elasticsearch-2.2.1.tar.gz
```

## 2. 上传安装文件到linux目录并解压

我这里将`elasticsearch-2.2.1.tar.gz`文件放至`/opt/elasticsearch`目录下。

解压：

```shell
$ cd /opt/elasticsearch
$ tar -zxvf elasticsearch-2.2.1.tar.gz
```

**此时切记不能用 `./bin/elasticsearch`启动，因为在root权限下会提示错误**：

```shell
Exception in thread "main" java.lang.RuntimeException: don't run elasticsearch as root.
```

## 3. 新添加一个elasticsearch用户



```sh
#添加elasticsearch用户：
$ useradd elasticsearch
#给用户elasticsearch设置密码，连续输入2次：
$ passwd elasticsearch
#创建一个用户组es:
$ groupadd es
#分配elasticsearch到es组:
$ usermod -G es elasticsearch
#在elasticsearch 根目录下，给定用户权限。-R表示逐级（N层目录） ， * 表示 任何文件:
$ chown -R elasticsearch:es *
#切换到elasticsearch用户:
$ su elasticsearch
```

## 4. 修改配置文件

```shell
$ vi config/elasticsearch.yml
```

修改以下内容：

```properties
#cluster name
cluster.name: my-application
#节点名称
node.name: node-1
path.data: /opt/elasticsearch/elasticsearch-2.2.1/data
path.logs: /opt/elasticsearch/elasticsearch-2.2.1/logs
#绑定IP和端口
network.host: 0.0.0.0
http.port: 9200
```

## 5. 安装head

head是elasticsearch的可视化工具，可以不用安装head，如果不需要安装，跳过此步骤。

需要注意的是，es2.x的版本head可直接安装，es5.x以上的版本需要先安装nodejs。

### 5.1. ES2.x.x版本安装head

在`/opt/elasticsearch/elasticsearch-2.2.1/`目录下新建文件夹：plugins

```shell
$ cd /opt/elasticsearch/elasticsearch-2.2.1/
$ mkdir plugins
$ cd elasticsearch/bin
$ ./plugin install mobz/elasticsearch-head
```

安装成功即可。启动es之后，访问`http://ip:9200/_plugin/head/`即可看到可视化界面。

### 5.2 ES5.x.x版本安装head

ES5.x版本暂时不支持直接安装，但是head作者提供了另一种安装方法。

```shell
$ git clone git://github.com/mobz/elasticsearch-head.git
$ cd elasticsearch-head
$ npm install
$ npm run start
```

然后打开`http://localhost:9100/`访问即可。

如果提示未安装node.js，先安装node.js，

node.js官网：[http://nodejs.cn/download/](http://nodejs.cn/download/)

选择安装包，我这边是64位linux，下载的是[https://nodejs.org/dist/v8.11.3/node-v8.11.3-linux-x64.tar.xz](https://nodejs.org/dist/v8.11.3/node-v8.11.3-linux-x64.tar.xz)

解压（注：如果是虚拟机，不要在共享文件夹中解压，会报`Cannot create symlink to`创建软链的错，请放至linux下解压）：

```shell
$ xz -d node-v8.11.3-linux-x64.tar.xz
$ tar -xvf node-v8.11.3-linux-x64.tar
$ mv node-v8.11.3-linux-x64 nodejs
#确认一下nodejs下bin目录是否有node 和npm文件，如果有执行软连接，如果没有重新下载执行上边步骤；建立软连接，变为全局:
$ ln -s /opt/nodejs/bin/npm /usr/local/bin/
$ ln -s /opt/nodejs/bin/node /usr/local/bin/
#检查是否安装成功
$ node -v
```



## 6. 启动ES

```shell
$ cd /opt/elasticsearch/elasticsearch-2.2.1
$ su elasticsearch
$ ./bin/elasticsearch
```

再访问`http://ip:9200/_plugin/head/`即可。

## 7. 安装IK分词器

在[https://github.com/medcl/elasticsearch-analysis-ik](https://github.com/medcl/elasticsearch-analysis-ik)

此处选择与es版本对应的ik分词版本，我这里下载的是[elasticsearch-analysis-ik-1.8.1.zip](https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v1.8.1/elasticsearch-analysis-ik-1.8.1.zip)，在 `/opt/elasticsearch/elasticsearch-2.2.1/plugins`目录下新建文件夹：`ik`;将`elasticsearch-analysis-ik-1.8.1.zip`的解压文件放入`ik`目录下，并将解压后的`config`下的`ik`文件夹放入 `/opt/elasticsearch/elasticsearch-2.2.1/config`下面;

```shell
$ unzip elasticsearch-analysis-ik-1.8.1.zip
$ cd /opt/elasticsearch/elasticsearch-2.2.1/plugins
$ mkdir ik
$ cp -rf /opt/elasticsearch/elasticsearch-analysis-ik-1.8.1/ ik/
$ mv /opt/elasticsearch/elasticsearch-analysis-ik-1.8.1/config/ik /opt/elasticsearch/elasticsearch-2.2.1/config
$ cd /opt/elasticsearch/elasticsearch-2.2.1/config
```

在`/config/elasticsearch.yml`文件的最后加上一句话：

```properties
index.analysis.analyzer.ik.type : "ik"
```



然后启动es，测试分词效果：

```shell
curl '10.10.13.234:9200/index/_analyze?analyzer=ik&pretty=true' -d '
{
"text":"世界如此之大 进口水果"
}'
```



或者浏览器访问`http://ip:9200/_analyze?analyzer=ik&pretty=true&text=sojson在线工具测试分词`，即可看到中文分词效果。

至此，elasticsearch搭建完成，下一篇我们来说下**索引的创建**。

