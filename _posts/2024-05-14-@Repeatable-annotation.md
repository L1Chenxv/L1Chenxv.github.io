---
layout: post
title: java8注解@Repeatable使用技巧
categories: [Java]
description: @Repeatable
keywords: java
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---



## 前言

`@Repeatable`是java8为了解决同一个注解不能重复在同一类/方法/属性上使用的问题。

## 应用场景

举一个比较贴近开发的例子，在spring/springboot我们引入资源文件可以使用注解@PropertySource

```java
@PropertySource("classpath:sso.properties")
public class Application {
}
```

那要引入多个资源文件怎么办，没事，我把PropertySource中的value设置成String[]数组就行了

```java
public @interface PropertySource {
      ...
      String[] value();
}
```

那么就能这么用

```java
@PropertySource({"classpath:sso.properties","classpath:dubbo.properties","classpath:systemInfo.properties"})
public class Application {
}
```

就spring引入配置文件来讲，肯定是没事问题的。但是如果注解中2个值存在依赖关系，那么这样就不行了。比如下面这个

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
//@Repeatable(Validates.class)
public @interface Validate {

    /**
     * 业务类型
     * @return
     */
    String bizCode();

    /**
     * 订单类型
     * @return
     */
    int orderType();

}
```

上面的@validate注解，bizcode和orderType是一对一的关系，我希望可以添加如下的注解

```java
@Validate(bizCode = "fruit",orderType = 1)
@Validate(bizCode = "fruit",orderType = 2)
@Validate(bizCode = "vegetable",orderType = 2)
public class BizLogic2 {
}
```

很抱歉在java8之前，这种方式不行，不过你可以这么做，新建一个如下的注解

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
public @interface Validates {

    Validate[] value();

}
```

> 注意这个聚合注解默认约束value来存储子注解

然后对应代码修改如下

```java
@Validates(value = {
        @Validate(bizCode = "fruit",orderType = 1)
        @Validate(bizCode = "fruit",orderType = 2)
        @Validate(bizCode = "vegetable",orderType = 2)
})
public class BizLogic2 {
}
```

在java8的@Repeatable出来之后，我们在不改动@Validates的情况下，对@Validate进行修改，增加`@Repeatable(Validates.class)`

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Repeatable(Validates.class)
public @interface Validate {

    /**
     * 业务类型
     * @return
     */
    String bizCode();

    /**
     * 订单类型
     * @return
     */
    int orderType();

}
```

那么就可以在类上使用多个@Validate注解了。

回过头来看下@PropertySource和@PropertySources也是如此

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Repeatable(PropertySources.class)
public @interface PropertySource 

/**
 * Container annotation that aggregates several {@link PropertySource} annotations.
 *
 * <p>Can be used natively, declaring several nested {@link PropertySource} annotations.
 * Can also be used in conjunction with Java 8's support for <em>repeatable annotations</em>,
 * where {@link PropertySource} can simply be declared several times on the same
 * {@linkplain ElementType#TYPE type}, implicitly generating this container annotation.
 *
 * @author Phillip Webb
 * @since 4.0
 * @see PropertySource
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface PropertySources
```

从上面的注释中可以看到，spring在4.0支持了java8这个特性

## 原理

**那么@Repeatable的原理是啥？**

**语法糖而已！**

我们反编译下面代码

```java
/*@Validates(value = {*/
        @Validate(bizCode = "fruit",orderType = 1)
        @Validate(bizCode = "fruit",orderType = 2)
        @Validate(bizCode = "vegetable",orderType = 2)
/*})*/
public class BizLogic2 {
}
```

发现反编译变成了

![9919411-27b2c5eb066741aa](https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/java/9919411-27b2c5eb066741aa.5c0ukuljn4.webp)

## 问题来了

那么再反编译下面代码试试

```java
/*@Validates(value = {*/
        @Validate(bizCode = "fruit",orderType = 1)
       // @Validate(bizCode = "fruit",orderType = 2)
        //@Validate(bizCode = "vegetable",orderType = 2)
/*})*/
public class BizLogic2 {
}
```

生成

![9919411-096b5877129e3ec7](https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/java/9919411-096b5877129e3ec7.45hjc9avyk.webp)

那这样就存在问题了，我以为加了@Repeatable之后@Validate都会生成被语法糖@Validates包裹。没想到它居然这么智能，只有一个@Validate注解的时候居然不转换。

所以使用多个@Validate的时候就会留坑，你需要兼容1个或多个的场景。

## 推荐方式

直接使用@Validates读取注解代码如下

```java
Validates validates = AnnotationUtils.getAnnotation(BizLogic2.class,Validates.class);
        Arrays.stream(validates.value()).forEach(a->{
            System.out.println(a.bizCode());
        });
```

带兼容的逻辑如下

```java
Validate validate = AnnotationUtils.getAnnotation(BizLogic.class,Validate.class);
        if(Objects.nonNull(validate)){
            System.out.println(validate.bizCode());
        }
        Validates validates = AnnotationUtils.getAnnotation(BizLogic2.class,Validates.class);
        Arrays.stream(validates.value()).forEach(a->{
            System.out.println(a.bizCode());
        });
```

如果你是自己写业务的，我觉得第一种方式更加方便。
但是如果你是开发中间件的，那么必须兼容，你永远不知道你的傻逼用户会干吗，哈哈。

> 还有一点需要注意，我的@Validate是被@Component元注解，当多个@Validate语法糖转换成@Validates之后，由于@Validates上没@Component，导致我的bean加载不到spring容器中去