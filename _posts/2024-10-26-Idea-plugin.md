---
layout: post
title: idea 插件开发demo
categories: [java]
description: 演示一下基本步骤
keywords: java,idea
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

### 前言

本文使用的 **idea 2024.2** 版本进行插件入门开发，首先要说明的是 idea 2022.2 版本及以后的 idea，对插件开发进行了一定程度的变动：

1、创建项目时不再支持 maven 选项

2、必须是 jdk21 及以后版本

> **DE and** **Java** **Versions**
>
> Java version must be set depending on the target platform version.
>
> 2024.2+: Java 21
>
> 2022.2+: Java 17 (blog post)
>
> 2020.3+: Java 11 (blog post)

3、默认创建的项目是基于 kotlin 的

4、idea 默认没有安装 “Plugin DevKit” 插件，需要自己安装

基于以上相关内容，本文创建一个 HelloWorld 级别的 idea 插件。

### 入门教程

#### 一、安装开发插件

<img src="https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/java/image.39l6mxm1sf.webp" alt="image" style="zoom:70%;" />

#### 二、新建 Project

<img src="https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/java/image.7sn7pwslnz.webp" alt="image" style="zoom:67%;" />

因为 idea 默认基于 gradle 构建项目，所有工程中不再有 pom.xml 文件。虽然 idea 默认使用 kotlin，但是我想目前阶段大部分开发者应该还是习惯 java 模式，所以需要做一些修改，去除 kotlin 的相关配置。

1、去除 build.gradle.kts 和 settings.gradle.kts 文件的 .kts 后缀，修改 src/main/kotlin 中的 kotlin 为 java，修改后的截图如下所示。

<img src="https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/java/image.8ojp5d3fq2.webp" alt="image" style="zoom:50%;" />

2、build.gradle.kts 文件介绍和修改

这个文件定义了IDEA插件构建时依赖的环境，以及最终支持在哪些环境下面运行插件，因为我们不使用 kotlin，所以我需要进行一些修改。

修改后的内容如下：

```Properties
plugins {
    id("java")
    id("org.jetbrains.intellij") version "1.17.4"
}

group = "com.example"
version = "1.0-SNAPSHOT"

repositories {
    maven { url 'https://maven.aliyun.com/repository/central/'}
    maven { url 'https://maven.aliyun.com/repository/public/' }
    maven { url 'https://maven.aliyun.com/repository/google/' }
    maven { url 'https://maven.aliyun.com/repository/jcenter/'}
    maven { url 'https://maven.aliyun.com/repository/gradle-plugin'}
    //    mavenCentral()
}

// Configure Gradle IntelliJ Plugin
// Read more: https://plugins.jetbrains.com/docs/intellij/tools-gradle-intellij-plugin.html
intellij {
    version = "2024.2"
    type = "IU" // Target IDE Platform（IC=社区版；IU=企业版）
    // 沙箱目录位置，用于保存IDEA的设置，默认在build文件下面，防止clean，放在用户目录下
    sandboxDir = System.getProperty("user.home") + "/idea-plugins-sandbox"
    // 依赖的插件，例如某插件需要基于com.intellij.database插件至上是用，这里就需要添加database插件依赖
    plugins = [/* Plugin Dependencies */]
    updateSinceUntilBuild = false
}
tasks.withType(JavaCompile.class).configureEach(task -> {
    task.getOptions().setEncoding("UTF-8");
    sourceCompatibility = "17";
    targetCompatibility = "17";
});
tasks.patchPluginXml {
    // 注意这个版本号不能高于上面intellij的version，否则runIde会报错
    sinceBuild.set("231")
    // 注意这个版本号必须高于上面intellij的version，否则runIde会报错
    untilBuild.set("243.*")
}
tasks.signPlugin {
    certificateChain.set(System.getenv("CERTIFICATE_CHAIN"))
    privateKey.set(System.getenv("PRIVATE_KEY"))
    password.set(System.getenv("PRIVATE_KEY_PASSWORD"))
}
tasks.publishPlugin {
    token.set(System.getenv("PUBLISH_TOKEN"))
}
```

修改完成后，点击 Gradle 小图标，确保最终控制台输出 BUILD SUCCESSFUL，通过构建后，右侧的 Gradle 面板就会出现各项 Task 内容，如下图所示：

<img src="https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/java/image.4uaxmelqhq.webp" alt="image" style="zoom:50%;" />

![image](https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/java/image.4uaxmelw50.webp)

如果提示Java版本过低 需要手动修改gradle配置

<img src="https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/java/image.3yeg6ycd2d.webp" alt="image" style="zoom:67%;" />

3、配置 IntelliJ Platform Plugin SDK

IntelliJ Platform Plugin SDK 就是开发 IntelliJ 平台插件的 SDK，是基于 JDK 之上运行的，类似于开发 Android 应用需要 Android SDK。

导航到 File | Project Structure，选择对话框左侧栏 Platform Settings 下的 SDKs 进行添加操作，如下图所示：

![image](https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/java/image.175dyvqcs0.webp)

此时会自动识别到idea安装目录，选择插件的jdk版本

![image](https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/java/image.9rjeg901nh.webp)

配置项目的SDK

![image](https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/java/image.92q4w8clye.webp)

#### 三、创建 Action

现在创建一个点击可以弹出 HelloWorld 提示框的一个Action（动作）

![image](https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/java/image.8vmx0sqjyo.webp)

![image](https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/java/image.51e5hu8oz2.webp)

![image](https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/java/image.1vyniwedqa.webp)

##### **`plugin.xml`****文件介绍**

这个文件可以理解为我们插件的元数据文件，用于定义我们的插件名、开发人员、插件依赖以及插件包含的内容等信息，具体可以看下面的介绍

```XML
<!-- Plugin Configuration File. Read more: https://plugins.jetbrains.com/docs/intellij/plugin-configuration-file.html -->
<idea-plugin>
    <!-- 插件id，不可重复，必须唯一。插件的升级后续也是依赖插件id来进行识别的 -->
    <id>com.example.idea-plugin-helloworld</id>

    <!--  插件名称 -->
    <name>Idea-plugin-helloworld</name>

    <!-- 插件开发人员，这里写一下开发者的个人信息. -->
    <vendor email="support@yourcompany.com" url="https://www.yourcompany.com">YourCompany</vendor>

    <!--  插件描述，这里一般是写插件的功能介绍啥的 -->
    <description><![CDATA[
    Enter short description for your plugin here.<br>
    <em>most HTML tags may be used</em>
  ]]></description>

    <!--  插件依赖，这里我们默认引用idea自带的依赖即可  -->
    <depends>com.intellij.modules.platform</depends>

    <!-- 定义拓展点，比较少用到，一般是用于你去拓展其他人插件功能拓展点，或者是你的插件扩展了 IntelliJ 平台核心功能才会配置到这里 -->
    <extensions defaultExtensionNs="com.intellij">
        <actions>
            <action id="com.example.ideapluginhelloworld.ShowHelloAction"
                    class="com.example.ideapluginhelloworld.ShowHelloAction" text="show~">
                <add-to-group group-id="HelpMenu" anchor="after" relative-to-action="About"/>
            </action>
        </actions>
    </extensions>
</idea-plugin>
```

##### 编写动作代码

接下来在`actionPerformed`方法中编写动作代码，这里写一段代码使之弹出一个对话框

```Java
public class ShowHelloAction extends AnAction {

    @Override
    public void actionPerformed(AnActionEvent e) {
        String message = "你好，秀一下";
        Messages.showInfoMessage(e.getProject(), message, "标题");
    }
} 
```

#### 四、测试 Action

编写好代码后，我们在右侧 Gracle 中运行或者 Debug运行，稍等一会后就会打开一个新的 idea（这个运行的 idea 对应的版本就是 build.gradle 配置文件中配置的 version 所对应的版本） ，然后我们在这个新打开的 idea 中操作我们刚刚创建的 Action 即可验证效果。

![image](https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/java/image.7ax61btndn.webp)

![image](https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/java/image.1aozwlk4wp.webp)

![image](https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/java/image.231vec0tpd.webp)

#### 五、打包插件

插件需要传播和发布，都需要打包，打包后会获得一个 zip 包，你可以分享给其他人安装，或者到 idea 插件市场里上传发布。

执行 buildPlugin 进行打包，观察控制台输出打包成功后，文件会生成到工程的 build/distributions 目录中，如下图所示：

<img src="https://cdn.statically.io/gh/L1Chenxv/picx-images-hosting@master/java/image.86tngs3l3i.webp" alt="image" style="zoom:50%;" />
<img src="https://poizon.feishu.cn/space/api/box/stream/download/asynccode/?code=N2M5ZWUyNjQ2OWEzMmM5NjQ3YTUwNmJkM2ZhMTI1MTFfS2RCWmtUb09UMXRYb2ZPT1owNlNaQWFaRDFxaThQcVRfVG9rZW46UExWaWJ5Nms0b0NrS3N4MXVud2N0dEZtbkZmXzE3Mjk5NTM3OTY6MTcyOTk1NzM5Nl9WNA" alt="img" style="zoom:50%;" />

#### 六、插件校验

因为IDEA版本很多，如果你的插件希望可以被多个 idea 版本兼容的话，那么在你分享给朋友或者发布到 idea 插件市场之前，建议先走一次校验流程。这个校验流程会把所有版本的 idea 自动走一次你的插件（只是校验是否能否正常在各个版本中安装运行）。由于这个版本会校验 idea 版本的兼容性，所以这里的耗时相对来说会比较长，因为它会自动下载各个版本的 idea 去挨个测试，每个版本的 idea 都要大几百上 G 的大小，这里你需要特别注意你的磁盘空间。
