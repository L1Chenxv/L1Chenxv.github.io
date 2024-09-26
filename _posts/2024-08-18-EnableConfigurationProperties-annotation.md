---
layout: post
title: 如何利用Spring优雅注入依赖值
categories: [Java]
description: EnableConfigurationProperties注解使用方式与作用
keywords: java
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---



## 两者的对比

- @ConfigurationProperties

使用@ConfigurationProperties的时候，把配置类的属性与yml配置文件绑定起来的时候，还需要加上@Component注解才能绑定并注入IOC容器中，若不加上@Component，则会无效。

- @EnableConfigurationProperties

@EnableConfigurationProperties的作用：则是将让使用了 @ConfigurationProperties 注解的配置类生效,将该类注入到 IOC 容器中,交由 IOC 容器进行管理，此时则不用在配置类上加上@Component。



## 实战的Demo

### @ConfigurationProperties

配置类如下：

```less
@Component
@ConfigurationProperties(prefix = "demo")
@Data
public class DemoConfig {
    private String name;
    private Integer age;
}
```

yml配置文件

```yaml
demo:
  name: 张三
  age: 20
```

测试代码

```java
@Component
public class Test implements ApplicationRunner {

    @Autowired
    DemoConfig demoConfig;

    @Override
    public void run(ApplicationArguments args) throws Exception {
        System.out.println(demoConfig);
    }
}
```

### 测试去掉配置类的@Component

错误如下：

![image](https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/java/image.6m3tqge8mn.webp)

在测试代码上加上@EnableConfigurationProperties，参数指定**哪个配置类**，该配置类上**必须**得有@ConfigurationProperties注解



```java
@Component
@EnableConfigurationProperties(DemoConfig.class)
public class demo implements ApplicationRunner {
    @Autowired
    DemoConfig demoConfig;

    @Override
    public void run(ApplicationArguments args) throws Exception {
        System.out.println(demoConfig);
}
```
结果如下：

![image](https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/java/image.wihevmi5i.webp)

依然可以绑定

## @EnableConfigurationProperties出现的原因

### 主要原因

springboot中EnableAutoConfiguration的自动装配

- EnableAutoConfiguration自动装配的作用：即把指定的类构造成对象，并放入spring容器中，使其成为bean对象，作用类似@Bean注解。
- springboot启动的时候，会扫描该项目下所有spring.factories文件。

> 上述所述造成spring.factories文件配置臃肿，是引入@EnableConfigurationProperties的原因

### 举例解释

- springboot默认扫描路径

1、springboot启动，需要扫描路径下的class文件，生成bean的定义BeanDefinition并创建bean对象，默认扫描路径是启动类所在的目录下。

<img src="https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/java/image.6ik7sqq8hc.webp" alt="image" style="zoom:50%;" />

2、如图下面的测试案例： 在启动类所在包目录之外的类即使加了@Component，也无法被扫描到生成bean对象。

<img src="https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/java/image.9kg3tysjds.webp" alt="image" style="zoom:50%;" />

<img src="https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/java/image.26lel7892x.webp" alt="image" style="zoom:50%;" />

3、如果就是要扫描到启动类所在包目录之外的类，使其生成bean对象，可以在启动类上加@ComponentScan注解

<img src="https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/java/image.5xak6fxzca.webp" alt="image" style="zoom:50%;" />

<img src="https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/java/image.wihevrlto.webp" alt="image" style="zoom:50%;" />

### 引入第三方jar包(里面也有bean对象)

1、这里使用本地自己打包的jar包(SDK)作例子 2、准备一个测试的SDK：

![image](https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/java/image.8z6g7o0r8n.webp)

在测试项目中引入该SDK：

![image](https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/java/image.3uurie1bzg.webp)

3、进行测试验证

<img src="https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/java/image.64ds1vmpn3.webp" alt="image" style="zoom:50%;" />

从上述图片可以看出，我们引入SDK之后进行使用，会报找不到该bean对象，奇怪，SDK的TestDemoBean类不是加了@Component注解吗？这就跟第一大点，springboot默认扫描路径有关系，SDK中的TestDemoBean路径是com.example.jardemo.xxxx, 而测试项目中的扫描路径是：com.example.csdn，自然springboot启动无法扫描到。

**4、提问：**

1. 有的人会说：这不简单，在启动类上加上：@ComponentScan("com.example”) 就好了。

2. 确实这样子就可以使用SDK中的bean对象了，但是你想想看，这个SDK刚刚好是com.example开头，与**本测试项目前缀有重复的**，倘若有其他SDK，如zki.example、abc.demo、king.shuai.demo…等等，此时你的@ComponentScan要怎么写呢？

3. 再说了，什么是SDK？是组件，是为了更加方便开发才存在的，<u>**你提供的SDK，还要别人指定要扫描你的包路径，这岂不是很失败**</u>？

   ==解决：使用EnableAutoConfiguration自动装配==

**5、修改SDK，添加配置文件spring.factories**

<img src="https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/java/image.6m3tqgra0k.webp" alt="image" style="zoom:50%;" />

重新打包放入仓库中，**记得重新加载一下引入的jar包，查看如下出现spring.factories文件**：

<img src="https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/java/image.8ad6nni4vs.webp" alt="image" style="zoom:50%;" />

注：有的人可能在打包SDK的时候，**发现resource目录下的文件没有打包进去**，请修改pom.xml文件配置：

<img src="https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/java/image.45hlbjletk.webp" alt="image" style="zoom:50%;" />

6、重新查看测试代码：

<img src="https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/java/image.70a9hc173j.webp" alt="image" style="zoom:50%;" />



1、上述内容主要是在说：提供SDK的时候，为了方便别人可以使用本SDK的bean对象，需要在其resource/META-INF目录下创建spring.factories文件，使用EnableAutoConfiguration 把需要变成bean对象的类变成bean对象。 

2、这样子其他项目再引用该SDK的时候，不用在启动类中编写需要扫描的路径 

> 其实配置类可以使用@ConfigurationProperties+@Component：但是因为如下的原因，不被建议： 

​	（1）我们知道这种方式是spring在启动的时候来扫描该类上有@Component的时候，才会把它变成bean对象，又因为是在SDK(第三方jar包)，因此需要在spring.factories中EnableAutoConfiguration加上该配置类路径，使其可以被spring扫描到 

​	（2）所以存在一个问题，如果配置项很多了，岂不是你这个spring.factories中EnableAutoConfiguration加上该类的**路径巨多，维护也不方便**，如果在**提供方的SDK**中存在一些配置项需要配置，如下：

<img src="https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/java/image.6t71lwxpf5.webp" alt="image" style="zoom:50%;" />

<img src="https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/java/image.2rv27iu0c6.webp" alt="image" style="zoom:50%;" />

<img src="https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/java/image.3k7xp8ujx7.webp" alt="image" style="zoom:50%;" />

因此，该配置类我们可以使用@EnableConfigurationProperties呀，在需要使用该配置类的上面加上这个注解，会自动把该@EnableConfigurationProperties注解指定的类对象，创建bean对象，这样子就**不用再写到 spring.factories上面**了：

<img src="https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/java/image.45lx5lj3b.webp" alt="image" style="zoom:50%;" />

重新打包，试一下

<img src="https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/java/image.4ckt6zcauf.webp" alt="image" style="zoom:50%;" />

## 总结

可以简单地理解： @ConfigurationProperties+@Component = @EnableConfigurationProperties，深究的铁子们可以动手操作一下，加深理解与印象。在工作中，没有固定说使用哪一个。根据个人习惯使用即可。