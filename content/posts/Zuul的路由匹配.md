
---
title: Zuul 的路由匹配
tags : [
    "java"
]
date : "2020-08-19"

---

上一篇，我们已经知道了  Zuul 的配置，其中 Zuul 的路由匹配也是核心配置之一，那 Zuul 是怎么匹配我们的请求，从而让我们的请求进入到一系列的 ZuulFilter 呢？那就和我一起去刨一刨吧。
<!--more-->

## Zuul的路由匹配规则是什么呢？

拿出我的必杀器，DDDDebug 一下：

1. Debug 显示匹配路由是先从 Spring 在`AbstractUrlHandlerMapping`匹配`HandlerExecutionChain`开始

   ![Spring-Match-ZuulController](http://qiniu.5ires.top/uPic/image-20200819230258789.png)

2. 之后进入到`ZuulFilter`的 Pre 类型的FIlter当中的`PreDecorationFilter` ，匹配对应的`ZuulRoute`

3. 首先把请求的路由修理一下，去掉context-path。就像例子当中，请求 url 中`/text/test` 去掉了 `/text`，再接着执行

   ![去掉context-path](http://qiniu.5ires.top/uPic/image-20200819190711091.png)

4. 之后进入到`SimpleRouteLocator`，判断是否属于 Zuul 忽略处理的请求，如果不是，再匹配对应`ZuulRoute`。这里可以发现匹配功能都是由`AntPathMatcher`来负责

   ![SimpleRouteLocator匹配URL](http://qiniu.5ires.top/uPic/image-20200819191434405.png)

那我们发现，不管是 Spring 的匹配 Handler 还是 `PreDecorationFilter` 匹配 `ZuulRoute`，都用到的是`AntPathMatcher`。那我们现在只需要搞明白`AntPathMatcher`匹配规则就好了。Go on!

## ANT Style Pattern

匹配规则如下：

| 符号 | 描述                      |
| ---- | ------------------------- |
| ?    | 匹配一个字符              |
| *    | 匹配0个或者更多的字符     |
| **   | 匹配路径中0个或者更多目录 |

举例：

| 例子                           | 解释                                                         |
| ------------------------------ | ------------------------------------------------------------ |
| `com/t?st.jsp`                 | 可以匹配 com/test.jsp 或者 `com/tast.jsp` 或者 `com/txst.jsp ` 等等 |
| `com/*.jsp`                    | 匹配到 com 目录下所有 .jsp 文件                              |
| `com/**/test.jsp`              | 匹配在 com 路径下，所有的 test.jsp 文件                      |
| `org/springframework/**/*.jsp` | 匹配 org/springframework 路径下所有 .jsp文件                 |
| `org/**/servlet/bla.jsp`       | 可以匹配 org 路径下，后面多层目录且最后一个目录是 servlet/bla.jsp 的路径 |

## 总结

其实，刨下来 Zuul 的路由匹配还挺简单的，主要理解并掌握 Ant 的匹配规则就完事儿了。来动手试一试吧。

Demo地址是：https://github.com/goldpumpkin/learn-demo

*Ref.*
*[stackoverflow-learning-ant-path-style](https://stackoverflow.com/questions/2952196/learning-ant-path-style)*

**我的个人水平有限，欢迎大家指正，欢迎交流~**

