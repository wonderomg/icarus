---
title: linux-crontab定时执行任务
date: 2017-7-25
categories: 
   - linux
tags:
   - crontab
thumbnail: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1504806839463&di=ed2ff5d6124b81c51ddfba817b3bc75e&imgtype=0&src=http%3A%2F%2Fimg.25pp.com%2Fuploadfile%2Fapp%2Ficon%2F20160410%2F1460238053403981.jpg
---



　　⏱️最近碰到一个关于`crontab`的问题。

<!--more-->

## **1. 事因**

服务器部署了一个`C++`查询数据库词库的服务，以及`Java`发送目标词的服务，`Java`服务通过`soket`长连接向`C++`服务发送目标单词，然后`C++`服务返回数据库中是否存在的结果。

期间，由于数据库有时增加单词需要重启服务，手写了个定时脚本来重启该`C++`服务。

某一次为了调试，把改定时脚本关了，通过`ps -ef |grep xxx`命令查看服务父子进程情况，发现父进程还是每隔十分钟退出，服务会重启，想了许久不知道哪出了问题。

最后询问运维才想起，之前还用了`linux`自带的`crontab`设置过每隔十分钟重启服务，没有删除`crontab`里面的那条设置。

所以我们就来学习一下`linux`自带的可设置定时指定任务的`crontab`。

## **2. 初识crontab**
`cron`是一个服务进程，`cron`服务提供`crontab`命令来设定`cron`服务的，以下是这个命令的一些参数与说明：

+ `crontab -u  //设定某个用户的cron服务，一般root用户在执行这个命令的时候需要此参数 `
+ `crontab -l  //列出某个用户cron服务的详细内容 `
+ `crontab -r  //删除某个用户的cron服务 `
+ `crontab -e  //编辑某个用户的cron服务 `



## **3. 基本用法**

基本格式 :

```shell
 *   *   *   *   *   command
```

​    **分　时　日　月　周　命令 **

第1列表示分钟1～59 每分钟用*或者 */1表示 ；
第2列表示小时1～23（0表示0点） ；
第3列表示日期1～31 ；
第4列表示月份1～12 ；
第5列标识号星期0～6（0表示星期天） ；
第6列要运行的命令 。

`crontab`的一些使用例子:

`*/10 * * * * (cd /opt/resin/bin; ./resin.sh restart)`

表示每隔10分重启`resin`服务；

`30 1 * * * (cd /opt/resin/bin; ./resin.sh restart)`

表示每天1点30分重启`resin`服务；

```* 23-5/1 * * * (cd /opt/resin/bin; ./resin.sh restart)```

表示每天23点到次日5点之间每隔1小时重启`resin`服务；