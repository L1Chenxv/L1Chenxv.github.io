---
layout: wiki
title: Java-api
cate1:
cate2:
description: 记录日常开发中一些常用api
keywords: keyword1, keyword2
type:
link:
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

记录日常开发中一些常用api



## String

**join（）**

```
public static String join(CharSequence delimiter,
                          Iterable<? extends CharSequence> elements)
```

返回一个新`String`的副本组成`CharSequence  elements`与指定的副本一起加入`delimiter` 。 

 > For example 
 >
 > ```java
 > `List<String> strings = new LinkedList<>();    
 > strings.add("Java");
 > strings.add("is");     
 > strings.add("cool");     
 > String message = String.join(" ", strings);     
 > //message returned is: "Java is cool"      
 > Set<String> strings = new LinkedHashSet<>();     
 > strings.add("Java"); 
 > strings.add("is");     
 > strings.add("very"); 
 > strings.add("cool");     
 > String message = String.join("-", strings);     
 > //message returned is: "Java-is-very-cool" `
 > ```



**trim（）**

返回一个字符串，其值为此字符串，并删除任何前导和尾随空格。 



## Character

**isDigit()**

```java
public static boolean isDigit(char ch)
```

确定指定的字符是否是数字。 



## List

**Arrays.asList()**

 ```
public static <T> List<T> asList(T... a)
 ```

返回由指定数组支持的固定大小的列表。（将返回的列表更改为“写入数组”。）该方法作为基于数组和基于集合的API之间的桥梁，与`Collection.toArray()`相[结合](../../java/util/Collection.html#toArray--) 。返回的列表是可序列化的，并实现[`RandomAccess`](../../java/util/RandomAccess.html) 。

此方法还提供了一种方便的方式来创建一个初始化为包含几个元素的固定大小的列表： 
List<String> stooges = Arrays.asList("Larry", "Moe", "Curly"); 