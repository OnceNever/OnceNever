---
title: 记一次Spring boot本地启动报IllegalAccessError异常原因
date: 2023/01/05 16:05:22
categories:
- [问题记录]
tags:
- 热部署
- 双亲委派
- devtools
---

### 序言

最近项目接入公司一个 `SSO` 统一身份认证组件，这里简单说明一下：

组件提供的自动配置类启动时如果 `UserCenterActivated` 这个`Bean`不存在的话会使用组件默认的用户激活组件，然后我是在项目中实现了自己的一个用户激活类并继承了`UserCenterActivated`抽象类。

![image-20240105162049473](https://image.seeyourface.cn/2024/01/image-20240105162049473.png)

项目启动时奇怪的出现了以下错误：

```tex
class xxx.sso.support.$Proxy130 cannot access its superinterface xxx.sso.support.AppRegistryInfo
```

并且该情况只在本地出现，打包部署到服务器并未出现问题。

所以我判断肯能与本地的`maven`插件有关，因为有些`maven`插件只有在开发中生效。

最终定位到是由于`spring-boot-devtools`这个插件导致的。



### 热部署插件

#### 介绍

`spring-boot-devtools`插件支持在`spring-boot`开发中对项目修改的代码进行快速部署，而无需重启`JVM`的一款工具。

使用`spring-boot-devtools`，可以实现指定目录（默认为`classpath`路径）下的文件进行更改后，项目自动重启，更改后的代码自动生效。



#### 原理

根据官方的定义：`spring-boot-devtools`使用了两个类加载器`ClassLoader`，一个`ClassLoader`加载不会发生更改的类（**第三方jar包**）一般是应用类加载器，我们暂且叫他 `base ClassLoader`，另一个`ClassLoader`（`restart ClassLoader`）加载会更改的类（**自定义的类**）。

默认情况下，`IDE` 中的任何打开项目都使用`restart`类加载器加载，而普通的`jar` 文件则使用`base`类加载器加载。如果使用 `mvn spring-boot:run` 或 `gradle bootRun`，情况也是如此: 包含 `@SpringBootApplication` 的项目使用`restart`类加载器加载，其他所有内容使用`base`类加载器加载。

![image-20240105173751969](https://image.seeyourface.cn/2024/01/image-20240105173751969.png)

这就导致我项目中的代码都是由`restart ClassLoader`加载器加载的，而我引入的组件`jar`包是由`app ClassLoader`加载的，这就可能会导致一些类加载问题，特别是在多模块项目中。

`SSO`组件在第三方`jar`包中，这将导致`DevTools`的重新启动类加载器出现问题。代理对象是由`restart`类加载器加载的，但被代理对象本身是由应用类加载器加载，因此会导致`IllegalAccessError`。

#### 解决方案

- 大力出奇迹，删掉或者注释掉`spring-boot-devtools`插件。
- 在`META-INF/spring-devtools.properties`中设置`restart.include.projectcommon=被代理对象所在的jar`，保证代理对象和被代理对象使用同一个类加载器。



### 参考

[^1]: https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#using.devtools.diagnosing-classloading-issues

