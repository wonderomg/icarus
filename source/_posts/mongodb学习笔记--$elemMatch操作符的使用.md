---
title: mongodb学习笔记--$elemMatch操作符的使用
date: 2017-02-21
categories: 
  - mongodb
tags: 
  - elemMatch条件操作符
thumbnail: http://www.bigdata-research.org/uploadfile/2014/1113/20141113053033429.jpg?1437154270828
---

　　mongodb提供了很多操作符以便更加方便快捷地操作数据，下面我们来认识一下查询操作符``$elemMatch``，``$elemMatch``投影操作符将限制查询返回的数组字段的内容只包含匹配``$elemMatch``条件的数组元素。

<!-- more -->

> 注意：
>
> * 数组中元素是内嵌文档。
> * 如果多个元素匹配``$elemMatch``条件，操作符返回数组中第一个匹配条件的元素。

## **1.首先创建一个简单文档**

```javascript
db.test.insert({"id":1, "members":[{"name":"BuleRiver1", "age":27, "gender":"M"}, {"name":"BuleRiver2", "age":23, "gender":"F"}, {"name":"BuleRiver3", "age":21, "gender":"M"}]});
```

## **2.使用多种方式尝试查询**

(1) 使用``db.test.find({"members":{"name":"BuleRiver1"}});``进行查询：

```javascript
db.test.find({"members":{"name":"BuleRiver1"}});
```

查询的结果是空集。

(2) 只有完全匹配一个的时候才能获取到结果，因此：

```javascript
db.test.find({"members":{"name":"BuleRiver1", "age":27, "gender":"M"}});
```

可以得到结果。

(3) 如果把键值进行颠倒,也得不到结果：

```javascript
db.test.find({"members":{"age":27, "name":"BuleRiver1", "gender":"M"}});
```

得到的结果是空集。

(4) 我们这样查询：

```javascript
db.test.find({"members.name":"BuleRiver1"});
```

是可以查询出结果的。

(5) 如果需要两个属性：

```javascript
db.test.find({"members.name":"BuleRiver1", "members.age":27});
```

也可以查询出结果。

(6) 我们再进行破坏性尝试：

```javascript
db.test.find({"members.name":"BuleRiver1", "members.age":23});
```

也可以查询出结果。

不过我们应该注意到：``BuleRiver1``是数组中第一个元素的键值，而23是数组中第二个元素的键值，这样也可以查询出结果。

## **3.使用$elemMatch操作符查询**

　　对于我们的一些应用来说，以上结果显然不是我们想要的结果。所以我们应该使用``$elemMatch``操作符:

(1)``$elemMatch+同一个元素中的键值组合

```javascript
db.test.find({"members":{"$elemMatch":{"name":"BuleRiver1", "age":27}}});
```

可以查询出结果；

(2)``$elemMatch``+不同元素的键值组合

```javascript
db.test.find({"members":{"$elemMatch":{"name":"BuleRiver1", "age":23}}});
```

查询不出结果。

因此，(1)展示的嵌套查询正是我们想要的查询方式。