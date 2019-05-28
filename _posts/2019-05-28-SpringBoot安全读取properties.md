---
layout:     post
title:      SpringBoot安全读取properties
date:       2019-05-28
author:     胡奚冰
header-img: img/post-bg-coffee.jpeg
catalog: 	 true
tags:
    - Java
    - SpringBoot
---

在SpringBoot项目中，我们经常会将一些参数放在配置文件中（.properties或.yml），然后通过`@value` 注解获取配置的值。

但如果参数字段很多，这种方式就显得不那么方便了：
+ 参数字段在哪里使用，是否必须不清晰，需要全局搜索查看使用的地方；
+ 参数key容易拼写错误；

实际上SpringBoot提供了更加方便的方式：`@ConfigurationProperties` 注解可以将自定义参数导入到实体对象中。

首先我们定义一个bean，并添加注解：

```
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

@Component
@ConfigurationProperties(prefix = "aib")
public class AIBProperties {
    private String msg;

    public String getMsg() {
        return msg;
    }

    public void setMsg(String msg) {
        this.msg = msg;
    }
}
```
添加了@Component和@ConfigurationProperties之后，SpringBoot或自动扫描到这个类，当需要实例化这个对象时，自动将对应的参数`aib.msg`的设置到这个对象中。

在IDEA中会出现下面警告：

![提示.png](https://upload-images.jianshu.io/upload_images/4657803-2d36e876838b8c33.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

是提示你添加一个依赖：

```
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
            <optional>true</optional>
        </dependency>
```
添加玩依赖之后，提示会变成：

![提示2.png](https://upload-images.jianshu.io/upload_images/4657803-57fc3d7559ec208c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这个时候已经可以了，我们reBuild一下项目：

![reBuild.png](https://upload-images.jianshu.io/upload_images/4657803-f35eb220b1fa42dd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

build完成我们可以在target/class/META-INF/包下看到一个.json文件：

![metadata.png](https://upload-images.jianshu.io/upload_images/4657803-7bb030bd58f59756.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

有了这个文件配合IDEA就可以实现提示效果：

![hint.png](https://upload-images.jianshu.io/upload_images/4657803-0e37c3e83fae3fa7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

使用properties只需要注入bean即可：

```
    @Autowired
    AIBProperties properties;

    @GetMapping
    public String test() {
        return properties.getMsg();
    }
```

这样做的好处：
+ 参数在一个bean中同一管理；
+ 直接设置默认值；
+ 参数key提示；
