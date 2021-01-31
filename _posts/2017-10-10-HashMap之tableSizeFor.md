---             
title:      HashMap之tableSizeFor
date:       2017-10-10    
key: 2017101001   
author:     ChayCao    
catalog: true 
tags:  Java                           
---


看HashMap的源码时，发现了里面好多很不错的算法。
tableSizeFor的功能（不考虑大于最大容量的情况）是返回大于输入参数且最近的2的整数次幂的数。比如10，则返回16。
该算法源码如下：
```java
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```
 详解如下：

先来分析有关n位操作部分：先来假设n的二进制为01xxx...xxx。接着

对n右移1位：001xx...xxx，再位或：011xx...xxx

对n右移2为：00011...xxx，再位或：01111...xxx

此时前面已经有四个1了，再右移4位且位或可得8个1

同理，有8个1，右移8位肯定会让后八位也为1。

综上可得，该算法让最高位的1后面的位全变为1。

最后再让结果n+1，即得到了2的整数次幂的值了。

由于int是32位，所以>>>16便能满足。

![举个栗子.jpg](http://upload-images.jianshu.io/upload_images/2489662-446566a23b9be33f.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
现在回来看看第一条语句：
```java
int n = cap - 1;
```
让cap-1再赋值给n的目的是另找到的目标值大于或等于原值。例如二进制1000，十进制数值为8。如果不对它减1而直接操作，将得到答案10000，即16。显然不是结果。减1后二进制为111，再进行操作则会得到原来的数值1000，即8。


HashMap里的MAXIMUM_CAPACITY是2^30^。我结合tableSizeFor()的实现，猜测设置原因如下：
int的正数最大可达2^31^-1，而没办法取到2^31^。所以容量也无法达到2^31^。又需要让容量满足2的幂次。所以设置为2^30^。