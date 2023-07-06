---
layout: post
title: SpringBoot最多可以处理多少请求？
categories: [Java]
description: 由一次逛csdn引发的一次spring学习。ps：还是得去深入学习下web
keywords: java, Spring
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

SpringBoot默认的内嵌容器是Tomcat，与其说SpringBoot可以处理多少请求，到不如说Tomcat可以处理多少请求。

## 正文
SpringBoot默认的内嵌容器是Tomcat，与其说SpringBoot可以处理多少请求，到不如说Tomcat可以处理多少请求。
和处理请求数量有关的参数如下：
![](https://cdn.jsdelivr.net/gh/L1Chenxv/picx-images-hosting@master/java/canshu.58wc1npie2w0.webp)

- **server.tomcat.threads.min-spare**：最少的工作线程数，默认大小是10。该参数相当于长期工，如果并发请求的数量达不到10，就会依次使用这几个线程去处理请求。
- **server.tomcat.threads.max**：最多的工作线程数，默认大小是200。参照最大线程数，这里有一点不同，tomacat7使用的是无界队列，所以不会像线程池那样等待队列满启动额外线程，这里会直接启动线程处理，等到达最大线程书后再回放队列。
- **server.tomcat.max-connections**：最大连接数，默认大小是8192。表示Tomcat可以处理的最大请求数量，超过8192的请求就会被放入到等待队列。
- **server.tomcat.accept-count**：等待队列的长度，默认大小是100。

测试
```yaml
server:
 tomcat:
   threads:
     # 最少线程数
     min-spare: 10
     # 最多线程数
     max: 15
   # 最大连接数
   max-connections: 30
   # 最大等待数
   accept-count: 10

```
开一百个线程模拟 观察一下测试结果：

![](https://cdn.jsdelivr.net/gh/L1Chenxv/picx-images-hosting@master/java/jieguo.2278vl86j9ds.webp)

从结果中可以看出，由于设置的 **max-connections+accept-count** 的和是40，所以有60个请求会被丢弃，这和我们的预期是相符的。由于最大线程是15，也就是有25个请求会先等待，等前15个处理完了再处理15个，最后在处理10个，也就是将40个请求分成了15,15,10这样三批进行处理。
### 结论：
如果并发请求数量低于**server.tomcat.threads.max**，则会被立即处理，超过的部分会先进行等待，如果数量超过max-connections与accept-count之和，则多余的部分则会被直接丢弃。
## 加餐
#### Q : 
#### 既然只按批次处理threads.max数量的线程，其他线程都要等待，max-connections & accept-count 两者有什么区别？
**A:**
处理一个请求是需要耗费资源的，CPU、内存等等，当现有连接数超出最大连接数之后，代表当前程序的性能达到了顶峰（默认的顶峰），没有更多的资源去处理请求了，所以会将后面的请求直接阻塞，不再处理新的请求，而是等前面的请求处理完了之后，再从等待队列里面拿请求处理。

就好比你去足浴店，里面只有十张床十个技师，现在已经满员了，那么后面来的客人，就只能在大厅拿号排队，等前面的客人走了之后，技师才能为这些排队的客人提供服务。

但Tomcat里面除开最大线程数之外，还有一个最大连接数的概念，那为什么有了最大线程数还有个这玩意儿呢？因为在Tomcat8以下的版本中，采用的是BIO模型，即一个请求需要分配一个线程去处理，**之前的版本中，最大线程数就等于最大连接数。**

而到了Tomcat8及以上的版本中，默认使用了NIO模型处理请求，这意味着一个线程可以同时处理多个请求，线程与请求之间的模型为一对多，因此理论上，每条线程同时处理的请求数，最多为（最大连接数/最大线程数），即8192/200=40个左右（这个是平均值，具体要根据实际情况来决定）。

换到前面的例子里面，之前的足浴店会为每个客人分配一个技师，这是BIO模型，但后面发现这样效率太低了，所以设立了一个新规矩，一个技师同时可以接待多个客人，即NIO模型。

## 参考资料
 [SpringBoot可以同时处理多少请求？](https://juejin.cn/post/7203648441721126972?utm_source=gold_browser_extension)

