---
layout: fragment
title: catch中使用e.printStackTrace的危害
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



### 原因

1. 占用太多内存，造成锁死。要打印字符串输出到控制台上，需要字符串常量池所在的内存块有足够的空间。然而，因为e.printStackTrace() 语句要产生的字符串记录的是堆栈信息，太长太多，内存被填满了！大量线程产出字符串产出到一半，等待有内存被释放，锁死了，导致整个应用挂掉了。
2. 日志交错混合，不易读。printStackTrace()默认使用了System.err输出流进行输出，与System.out是两个不同的输出流，那么在打印时自然就形成了交叉。再就是输出流是有缓冲区的，所以对于什么时候具体输出也形成了随机。

### 建议

使用log日志来替代e.printStackTrace

### 举例

```java
    
public void demo() {
  
    try {

    } catch (Exception e) {
        log.error("demo error", e);
        return null;
    }
    
}

```

### 总结

在高并发场景下，短时间内大量请求访问此接口 -> 代码本身有问题，很多情况下抛异常  -> e.printStackTrace() 来打印异常到控制台 -> 产生错误堆栈字符串到字符串池内存空间 -> 此内存空间一下子被占满了 -> 开始在此内存空间产出字符串的线程还没完全生产完整，就没空间了 ->  大量线程产出字符串产出到一半，等在这儿（等有内存了继续搞啊）-> 相互等待，等内存，锁死了，整个应用挂掉了。