---

layout: fragment
title: Metaspace内存不足导致FGC问题排查
tags: [Java]
description: 
keywords: OOM
---

实习期间测试环境接口调用超时，服务异常重启，进行排查

### bug

发现K8S的pod经常重启，通过监控发现应用频繁进行Full GC

根据GC日志初步判断由于Metaspace元空间不足，出现内存溢出，导致jvm频繁触发full GC。

找运维拉了下dump文件，没有发现重复创建对象，内存泄漏情况。猜测可能是元空间的内存参数设置太小。

### 解决

调整MaxMetaspaceSize从256M到512M，重启服务，上线观察优化前后元空间增长，的确是参数太小导致的。



### 总结

要根据代码引入类的多少，确定合理的JVM参数。



