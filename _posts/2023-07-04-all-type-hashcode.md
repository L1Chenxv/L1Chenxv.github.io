---

layout: post
title: Java中各种类型的Hashcode
categories: [Java]
description: 总结各种类型的计算方式
keywords: Java

---

Java中每种类型的`hashCode()`计算方式不同，以下是常见类型梳理

**Map**：每个entry的哈希值的总和

```java
public int hashCode() {
    int h = 0;
    Iterator<Entry<K,V>> i = entrySet().iterator();
    while (i.hasNext())
        h += i.next().hashCode();
    return h;
}
```

内部的`Entry<K,V>`计算方式：键的`hashcode` `^` 值的`hashcode`

```java
public final int hashCode() {
    return Objects.hashCode(key) ^ Objects.hashCode(value);
}
```

**List**：之前元素 * 31 + 当前元素

```java
public int hashCode() {
    int hashCode = 1;
    for (E e : this)
        hashCode = 31*hashCode + (e==null ? 0 : e.hashCode());
    return hashCode;
}
```

> List是顺序型结构，内部元素顺序不同会使得两个List的hashcode不同



**String**：和List计算方法差不多

```java
public int hashCode() {
    int h = hash;
    if (h == 0 && value.length > 0) {
        char val[] = value;

        for (int i = 0; i < value.length; i++) {
            h = 31 * h + val[i];
        }
        hash = h;
    }
    return h;
}
```



其余基本类型：直接返回`value`

```java
public int hashCode() {
    return T.hashCode(value); // 基本类型 extends T
}
```

