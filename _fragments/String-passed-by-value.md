---
layout: fragment
title: Java中值传递
tags: [Java]
description: some word here
keywords: java
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

Content here

- 注意，这里的值传递意思是传递的是指向引用所指向对象在堆中地址值，而不是引用自身在堆栈中地址值。
- 下面是String值传递示例：

```java
public class Test {

    private static void change(String str){//这里的引用str与main中定义的str不同，两者引用所在地址不同，只不过现在两个引用所存储的对象地址相同
        //因为String类中的value变量时final的，所以当给str赋值的时候“ccccc”所代表的对象值不能被改变，当在常量池找不到值为“abab”的对象时，
        //就会新创建一个对象，而不是改变“ccccc”这个对象的值。即str存储的对象地址指向新创建的对象“abab”而不是“ccccc”旧对象地址
        //所以对main中“ccccc”对象的地址所存储的值没有任何改变
        str = "abab";
    }
    public static void main(String[] args) {
        String str = "ccccc";
        change(str);
        System.out.println(str);//输出结果为：ccccc
    }
}
```

- String类对象值不能被改变原因

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    /** The value is used for character storage. */
    private final char value[];
    
    ...
}
```

