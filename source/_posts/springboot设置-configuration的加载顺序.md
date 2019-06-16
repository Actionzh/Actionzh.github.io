---
title: springboot设置@configuration的加载顺序
date: 2019-06-16 12:41:21
tags:
---
#### 1、简介

Spring Boot会检查你发布的jar中是否存在`META-INF/spring.factories`文件，该文件中以`EnableAutoConfiguration`为key的属性应该列出你的配置类：

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.mycorp.libx.autoconfigure.LibXAutoConfiguration,\
com.mycorp.libx.autoconfigure.LibXWebAutoConfiguration

```

你可以使用[`@AutoConfigureAfter`]或[`@AutoConfigureBefore`]注解为配置类指定特定的顺序。例如，如果你提供web-specific配置，你的类就需要应用在`WebMvcAutoConfiguration`后面。

你也可以使用`@AutoconfigureOrder`注解为那些相互不知道存在的自动配置类提供排序，该注解语义跟常规的`@Order`注解相同，但专为自动配置类提供顺序。

> 注：自动配置类只能通过这种方式加载，确保它们定义在一个特殊的package中，特别是不能成为组件扫描的目标。

#### 2、例子

例如下面的项目，想要保证加载`BananaConf.java`后再加载`AppleConf.java`
![微信截图_20190616124419](springboot%E8%AE%BE%E7%BD%AE-configuration%E7%9A%84%E5%8A%A0%E8%BD%BD%E9%A1%BA%E5%BA%8F/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20190616124419.png)

1）添加文件`spring.factories`（若没有`META-INF`文件夹，则需要先添加）
2）在`spring.factories`中加入

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.demo.conf.AppleConf,\
com.example.demo.conf.BananaConf

```

3）`AppleConf`中添加 `@AutoConfigureAfter(BananaConf.class)`

```
@Configuration
@AutoConfigureAfter(BananaConf.class)
public class AppleConf
{
    xxx
}

```

`BananaConf`正常配置即可

```
@Configuration
public class BananaConf
{
    xxx
}

```

注意：在spring.factories注册了的配置类，可以不用再该配置类写@Configuration
