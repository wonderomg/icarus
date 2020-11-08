---
title: C++中sprintf函数的用法
date: 2015-07-21
categories: 
  - C++
tags: 
  - sprintf
thumbnail: http://pic1.win4000.com/wallpaper/5/56eb942607643.jpg
---

## **1.常用方式**

　　 ``sprintf``函数的功能与``printf``函数的功能基本一样，只是它把结果输出到指定的字符串中了，看个例子就明白了：

<!--more-->

例：将``"test,1,2"``写入数组s中

``` c++
#include<stdio.h>
int main(int argc, char *avgv[])
{
    char s[40];
    sprintf(s,"%s%d%c","test",1,'2');
    /*第一个参数就是指向要写入的那个字符串的指针，剩下的就和printf()一样了
    你可以比较一下，这是向屏幕输入*/
    printf("%s%d%c","test",1,'2');
    return 0;
}
```
编译：
>g++ sprinftest.cpp -o sprinftest && ./sprinftest

输出结果：
>sprintftest12
>sprintftest12

## **2.若"%s"等输出符在字符串中**

例：补全字符串str的缺省内容
``` c++
#include <iostream>
#include <stdio.h>
#include <cstring>

int main(int argc, char *avgv[])
{
    char str[] = "hel%co wo%sd! sp%stf test%d";
    char buf[strlen(str)];
    sprintf(buf, str, 'l', "rl", "rin", 1);
    std::cout << "str = "<< buf << "\nlen = " <<  strlen(buf) << std::endl;
    return 0;
}
```
编译：
>g++ sprinftest.cpp -o sprinftest && ./sprinftest

输出结果：
>str = hello world! sprintf test1
>len = 27

这种形式也可以将多个字符值或字符串值赋值到字符串str中，有多少个输出符就后面就加多少个参数。

