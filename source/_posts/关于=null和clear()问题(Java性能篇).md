---
title: 关于=null和clear()问题（Java性能优化）
date: 2016-04-19
categories: 
  - Java
tags: 
  - Java
  - 性能优化
thumbnail: http://pic1.win4000.com/wallpaper/a/54535d25dfe79.jpg
---

　　以ArrayList为例，根据情况来看吧，**ArrayList内部维护的是一个数组**。<!--more-->
## **1. list = null**
>那么你**list = null; ** 就是**释放这个数组对象，当然里面所引用的对象也就释放了**。


## **2. list.clear()**

>如果**list.clear();** 看看源代码就知道了，是**把list里面对象遍历赋值为null**，意思就是**释放list里面所有对象**。

　　这样就很清楚了，如果你这个list还需要使用，那么你可以使用clear去释放掉list里面所有的数据，GC在回收的时候就有可能回收掉原list里面的这些数据了。如果你都不需要了，那么你就用=null这样GC就有可能把list以及里面关联的一些数据也都回收了。

## **3. 咱们来看下clear()源码**

`clear()`源码解析：
If we take an `ArrayList` as an example, the `clear()` method does this:
```Java
public void clear() { 
    modCount++;  
    
    // Let gc do its work 
    for (int i = 0; i < size; i++)  
        elementData[i] = null;  
        
    size = 0;  
}
```
　　Basically, if elements of a List are not referenced anywhere else in the code there is really no need (or at least you do not gain anything) to call `clear()`. You also do not need to assign `null` to the List because it will be garbage collected as soon as it falls out of scope. 
**译文：** 
　　基本上，如果List中的元素没有被引用到代码中的其他地方，那么确实没有必要（或至少你没有获得任何东西）来调用`clear()`。 您也不需要为List指定`null`，因为它会在超出范围的情况下被垃圾回收。