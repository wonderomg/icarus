---
title: mongodb学习笔记--C++操作mongodb
date: 2016-11-10
categories: 
  - C++
  - MongoDB
tags: 
  - mongodb
  - C++
thumbnail: http://bizhi.zhuoku.com/2015/04/29/zhuoku/ZHUOKU005.jpg
---

　　在学习mongodb过程当中，必须学习的就是用C++（Java、PHP、C#等）操作mongodb，这里讲述C++操作mongodb，在官方提供的mongo-cxx-driver驱动中有相关的操作例子，可以结合例子学习，目录是``mongo-cxx-driver-legacy-1.0.0-rc0\src\mongo\client\examples``，这里针对几个重要的点讲述。

<!-- more -->

##  **1. C++连接mongodb**

**（1）<font color=#0000FF>有无密码都适用</font>**
```C++
mongo::client::GlobalInstance instance;
if (!instance.initialized()) {
	std::cout << "failed to initialize the client driver: " << instance.status() << std::endl;
    return EXIT_FAILURE;
}

std::string uri = "mongodb://username:password@127.0.0.1:27017";
std::string errmsg;

ConnectionString cs = ConnectionString::parse(uri, errmsg);

if (!cs.isValid()) {
	std::cout << "Error parsing connection string " << uri << ": " << errmsg << std::endl;
    return EXIT_FAILURE;
}

boost::scoped_ptr<DBClientBase> conn(cs.connect(errmsg));
if (!conn) {
	cout << "couldn't connect : " << errmsg << endl;
	return EXIT_FAILURE;
}
```
　　<font color=#0000FF>这种方式有密码和无密码的情况都适用，``"username"``为登录名，``"password"``为登录密码；无密码时去掉``"username:password@"``即可连接，这样连接方式比较常用。</font>

**（2）常规连接操作（不带密码连接）**
```C++
mongo::DBClientConnection conn; 
mongo::Status status = mongo::client::initialize();
if (!status.isOK()) {
	MongoException m(-1, "failed to initialize the client driver: " + status.toString());
	return -1;
}
string url = "localhost:27017";
if (!conn.connect(url, errmsg)) {
	MongoException m(0, "couldn’t connect : " + errmsg);
	return -1;
}
```
通过这种方式可建立与mongodb的连接，但是是未带权限的连接。

**（3）带密码连接**
```C++
conn.auth( "admin" , "username" , "password" , errmsg );
```
　　其中``"admin"``为验证用户的所在数据库，一般在``"test.system.users"``或``"admin.system.users"``表中，所以填库名``"test"``或``"admin"``，``"username"``为登录名，``"password"``为登录密码。
或者下面的auth方式也可以：

```C++
conn.auth(BSON(
	"user" << "root" << "db" << "admin" <<
	"pwd" << "password" << "mechanism" << "DEFAULT"
));
```
根据mongodb的设置更改传递给函数``conn.auth``的参数值。

##  **2. C++查询数据**
```C++
const char* ns = "test.first";
conn->findOne(ns, BSONObj()); //查询一条数据
conn->findOne(ns, BSONObjBuilder().append("name", "joe").obj()); //条件查询
```

##  **3. C++插入数据**
```C++
const char* ns = "test.first";
conn->insert(ns, BSON("name" << "joe" 
                      << "pwd" << "123456" 
                      << "age" << 20));
```

##  **4. C++删除数据**
```C++
const char* ns = "test.first";
conn->remove(ns, BSONObj()); //删除所有
conn->remove(ns, BSONObjBuilder().append("name", "joe").obj()); //删除指定文档
```

##  **5. C++更新数据**

**(1) 在原数据基础上增加属性**
```C++
const char* ns = "test.first";
BSONObj res = conn->findOne(ns, BSONObjBuilder().append("name", "joe").obj());
BSONObj after = BSONObjBuilder().appendElements(res).append("name2", "h").obj();
conn->update(ns, BSONObjBuilder().append("name", "jeo2").obj(), after);	//在原数据上增加属性
```
这种update方式只是在原数据基础上增加属性，并不会改变指定属性值。
**(2) 更改指定的某个属性值**
**<font color=red>想要update指定属性的值，需要用到"$set"指令</font>**，如下代码所示：
```C++
BSONObj res = conn->findOne(ns, BSONObjBuilder().append("name", "joe").obj());
//更改指定的某个属性值
conn->update("test.first", res, BSON("$set" << BSON("age" << 11)));
```
<font color=#0000FF>主要想强调下update更改指定的某个属性值的方式，比较常用。</font>

##  **6. C++操作例子**
(1) simple_client_test.cpp
```C++
/*
#if defined(_WIN32)
#include <winsock2.h>
#include <windows.h>
#endif*/
//g++ simple_client_test.cpp -pthread -lmongoclient -lboost_thread -lboost_system -lboost_regex -o simple_client_test
#include "mongo/client/dbclient.h"  // the mongo c++ driver

#include <iostream>

#ifndef verify
#define verify(x) MONGO_verify(x)
#endif

using namespace std;
using namespace mongo;
//using mongo::BSONObj;
//using mongo::BSONObjBuilder;

int main(int argc, char* argv[]) {
    if (argc > 2) {
        std::cout << "usage: " << argv[0] << " [MONGODB_URI]" << std::endl;
        return EXIT_FAILURE;
    }
    
    mongo::client::GlobalInstance instance;
    if (!instance.initialized()) {
        std::cout << "failed to initialize the client driver: " << instance.status() << std::endl;
        return EXIT_FAILURE;
    }

    std::string uri = argc == 2 ? argv[1] : "mongodb://127.0.0.1:27017";
    std::string errmsg;

    ConnectionString cs = ConnectionString::parse(uri, errmsg);

    if (!cs.isValid()) {
        std::cout << "Error parsing connection string " << uri << ": " << errmsg << std::endl;
        return EXIT_FAILURE;
    }

    boost::scoped_ptr<DBClientBase> conn(cs.connect(errmsg));
    if (!conn) {
        cout << "couldn't connect : " << errmsg << endl;
        return EXIT_FAILURE;
    }

    const char* ns = "test.first";

    conn->dropCollection(ns);
    
    //clean up old data from any previous tesets
    conn->remove(ns, BSONObj());
    verify(conn->findOne(ns, BSONObj()).isEmpty()); //verify逻辑表达式，判断是否还存在test.first，是否clenn up成功
    
    //test insert
    cout << "(1) test insert----" << endl;
    conn->insert(ns, 
                 BSON("name" << "joe" 
                      << "pwd" << "123456" 
                      << "age" << 20));
    verify(!conn->findOne(ns, BSONObj()).isEmpty());     //判断是否添加成功
    cout << "insert data : " << conn->findOne(ns, BSONObj()) << endl;
    cout << "insert success!" << endl;

    // test remove
    cout << "(2) test remove----" << endl;
    conn->remove(ns, BSONObj());
    verify(conn->findOne(ns, BSONObj()).isEmpty());
    cout << "remove success!" << endl;
    
    // insert, findOne testing
    conn->insert(ns, 
                 BSON("name" << "joe" 
                      << "pwd" << "234567" 
                      << "age" << 21));
    {
        BSONObj res = conn->findOne(ns, BSONObj());
        verify(strstr(res.getStringField("name"), "joe"));
        verify(!strstr(res.getStringField("name2"), "joe"));
        verify(21 == res.getIntField("age"));
    }
    cout << "insert data : " << conn->findOne(ns, BSONObj()) <<endl;

    // test update
    {
        BSONObj res = conn->findOne(ns, BSONObjBuilder().append("name", "joe").obj());
        verify(!strstr(res.getStringField("name2"), "jeo"));
        
        BSONObj after = BSONObjBuilder().appendElements(res).append("name2", "w").obj();
        cout << "(3) test update name joe add name2 name3----" << endl;
        //upsert type1 update method
        conn->update(ns, BSONObjBuilder().append("name", "joe").obj(), after);
        //res = conn->findOne(ns, BSONObjBuilder().append("name2", "w").obj());
        verify(!strstr(res.getStringField("name2"), "joe"));
        verify(conn->findOne(ns, BSONObjBuilder().append("name", "joe2").obj()).isEmpty());
        //cout << " update1 data: " << conn->findOne(ns, BSONObj()) << endl;
        cout << " update1 data : " << conn->findOne(ns, Query("{name2:'w'}")) << endl;
        
        //upsert type2 update method 更改指定的某个属性值 
        const string TEST_NS = "test.first";
        conn->update("test.first", res, BSON("$set" << BSON("age" << 11)));
        cout << " update2 data : " << conn->findOne(ns, BSONObj()) << endl;
        res = conn->findOne(ns, BSONObjBuilder().append("name", "joe").obj());

        //upsert type3 update method
        try
        {
            after = BSONObjBuilder().appendElements(res).append("name3", "h").obj();
            conn->update(ns, BSONObjBuilder().append("name", "joe").obj(), after);
        }
        catch (OperationException&)
        {
            cout << " update error: " << conn->getLastErrorDetailed().toString() << endl;
        }
        verify(!conn->findOne(ns, BSONObjBuilder().append("name", "joe").obj()).isEmpty());
        //cout << " update3 data: " << conn->findOne(ns, BSONObj()) << endl;
        cout << " update3 data : " << conn->findOne(ns, Query("{name3:'h'}")) << endl;

        cout << "(4) test query-----" << "\n query data : " 
            << conn->findOne(ns, BSONObjBuilder().append("name", "joe").obj()) << endl;
        cout << " Query data : " << conn->findOne(ns, Query("{name:'joe'}")) << endl;
    }

    return 0;
}

```
(2) makefile文件
```
CC = g++
TEST = simple_client_test

$(TEST):
	$(CC) $@.cpp -pthread -lmongoclient -lboost_thread -lboost_system -lboost_regex -o $@
	./$(TEST)
	rm -f $(TEST)
run:
	./$(TEST)

clean:
	rm -f $(TEST)
```