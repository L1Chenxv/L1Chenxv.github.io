---
layout: post
title: Springboot利用Spring SPI开发starter
categories: [Java]
description: 
keywords: java,spring
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

## 前言

首先，我们需要明确自定义starter的目标功能，如提供特定领域的服务或集成第三方库。Spring Boot starter 和普通依赖（普通 Jar 包）的主要区别在于 starter 是一组便捷的依赖集合和自动配置的封装，而普通依赖只是单一的功能库或框架。

本文基于Spring注解，介绍常见的starter开发方式，希望对你有所帮助

## 基于@EnableConfigurationProperties开发

> @EnableConfigurationProperties的作用：则是将让使用了 @ConfigurationProperties 注解的配置类生效,将该类注入到 IOC 容器中,交由 IOC 容器进行管理，此时则不用在配置类上加上@Component。

1. 首先创建一个配置类，如下

```java
@ConfigurationProperties(prefix = "demo")
@Data
public class DemoConfig {
    private String name = "default";
    private int age = 0;
}
```

1. 配置文件：

```Properties
demo.name=张三
demo.age=10
```

1. 配置服务类：

```java
@Component
@EnableConfigurationProperties(DemoConfig.class)
public class DemoAutoConfiguration implements ApplicationRunner {

    @Resource
    private DemoConfig demoConfig;

    @Override
    public void run(ApplicationArguments args) throws Exception {
        System.out.println(demoConfig);
    }
}
```

测试：

![image](https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/java/image.4jo3t9h0l9.webp)

可以看到配置服务类可以识别到配置文件中的属性并装填到对应类中

1. 配置spring.factories文件

在`resources/META-INF/spring.factories`文件中指定自动配置类，这样Spring Boot在启动时会自动加载这个配置类。

![image](https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/java/image.7ljzuhiz3m.webp)

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.demo.DemoAutoConfiguration
```

1. 测试自定义Starter

最后，我们在一个新建的项目内引入这个自定义Starter，并编写测试用例来验证它的功能。

首先在`application.properties`中配置属性：

```Properties
demo.name=wangwu
demo.age=20
```

编写测试用例：

```java
public class StarterDemo implements ApplicationRunner {

    @Resource
    private DemoAutoConfiguration demoAutoConfiguration;

    @Override
    public void run(ApplicationArguments args) throws Exception {
        System.out.println(demoAutoConfiguration);
    }
}
```

运行结果：

```java
DemoConfig(name=wangwu, age=20)
```

## Q&A

### 打包出现 BOOT-INF 文件夹问题

spring-boot maven打包，一般pom.xml文件里会加

```xml
 <plugin>
     <groupId>org.springframework.boot</groupId>
     <artifactId>spring-boot-maven-plugin</artifactId>
 </plugin>
```

这样打的jar里会多一个目录BOOT-INF。

**引起问题: 程序包不存在。**

**解决办法: 如果A子模块包依赖了B子模块包，在B子模块的pom文件，加入 configuration.skip = true**

```xml
 <plugin>
     <groupId>org.springframework.boot</groupId>
     <artifactId>spring-boot-maven-plugin</artifactId>
     <configuration>
         <skip>true</skip>
     </configuration>
 </plugin>
```

### 使用@ConfigurationProperties问题

使用@ConfigurationProperties+@Component同样能达到配置的效果，但需要在启动类上加@ComponentScan注解。

但你提供的SDK还需要别人指定扫描你的包路径？这不是很失败吗？

## 总结

自定义starter的步骤可以分为以下几步：

1. 创建配置类
2. 创建配置服务类
3. 配置spring.factories文件
4. 引入到第三方项目中使用
