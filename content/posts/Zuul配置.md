
---
title: Zuul 的配置
tags : [
    "java"
]
date : "2020-08-06"

---

这一篇主要介绍Zuul的配置。为了大家快速上手，主要通过示例来演示如何进行Zuul配置。
<!--more-->

## 说明

配置可以分为几个部分：

[Zuul路由配置](https://docs.spring.io/spring-cloud-netflix/docs/2.2.4.RELEASE/reference/html/#netflix-zuul-starter)：

+ Zuul自己的配置对应的类是`ZuulProperties`，配置的前缀是`zuul`

+ 自定义的路由配置在代码中对应的是`ZuulProperties`的属性：`Map<String, ZuulRoute> routes = new LinkedHashMap<>()`
+ routes是一个Map，key代表route名称，value是路由规则配置的详细信息

[Ribbon配置信息](https://github.com/Netflix/ribbon/wiki/Getting-Started)：主要提供负载均衡、重试、超时等配置。配置类是[CommonClientConfigKey](http://netflix.github.io/ribbon/ribbon-core-javadoc/com/netflix/client/config/CommonClientConfigKey.html)，其中要注意配置key是大小写敏感的！

[Hytrix配置信息](https://github.com/Netflix/Hystrix/wiki/Configuration#coreSize)：主要提供熔断策略、超时等配置

## 路由配置分为两种

+ 转发到物理地址
+ 转发到服务

## 转发到物理路径的配置

### 配置说明

```yaml
zuul:
  host:
  	# 链接超时时间
    connect-timeout-millis: 2000
    # socket超时时间
    socket-timeout-millis: 10000
  # routes 即为 Map<String, ZuulRoute> routes = new LinkedHashMap<>()
  routes:
    # route名称 - 是ZuulRoute属性中的id，也是Map中的key
    test:
      # 匹配路径模板
      path: /test/**
      # 请求转发到的服务id 和 url属性互斥，两个只能配置一个
      # 请求转发到的物理路径
      url: http://127.0.0.1:9000 
      # 转发前是否跳过前缀,默认为true，前缀指的是：匹配路径的path前缀 or zuul.prefix定义
      stripPrefix: true
      # 敏感Header， 转发时会去掉敏感的头部信息，以下是默认的配置。
      sensitiveHeaders: Cookie,Set-Cookie,Authorization
```

### Example - 1 - 最简洁配置

```yaml
zuul:
  routes:
    test:
      path: /test/**
      url: http://127.0.0.1:9000 
```

Request URL：http://127.0.0.1:8001/test

Zuul Forward URL：http://127.0.0.1:9000/

### Example - 2 - 试验其他配置(sensitiveHeaders & stripPrefix)

```yaml
zuul:
  routes:
    test:
      path: /test/**
      url: http://127.0.0.1:9000 
      stripPrefix: false
      sensitiveHeaders: Test-Sensitive-Header
```

Request:

+ URL: http://127.0.0.1:8001/test
+ Header: 
  + Test-Sensitive-Header : ThisIsSensitiveHeader
  + Test-nomal-Header: ThisIsNomalHeader

Zuul Forward：

+ URL：http://127.0.0.1:9000/test
+ Header:
  + Test-Sensitive-Header - 是敏感Header不转发
  + Test-nomal-Header - 进行转发

![Example2-route-matched](http://qiniu.5ires.top/uPic/image-20200816225825634.png)

![Example2-请求情况](http://qiniu.5ires.top/uPic/image-20200816230622007.png)

## 转发到Service的配置

### 配置说明

```yaml
# 转发到某个的服务 - 这个服务可以是服务发现的服务，也可以配置物理地址
zuul:
  host:
    connect-timeout-millis: 2000
    socket-timeout-millis: 10000
  routes:
    test-service:
      path: /test/**
      # 服务名称: 如果使用服务治理框架那么填写服务名称，否则自定义服务名称
      serviceId: test-service
      # 是否开启重试，具体重试设置是在Ribbon的配置中设置
      retryable: true
      ...
      
# hystrix配置：提供熔断配置
hystrix:
  command:
    # 服务名称：default or 自己的服务名称
    default:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 3000
            ...

ribbon:
	# 是否使用 Eureka
  eureka:
    enabled: false
  # 对实例重试的次数(不包括每个实例的首次的请求)
  MaxAutoRetries: 1
  # 同一个服务 重试其他实例数量
  # eg: listOfServers: s1, s2, s3  当此配置为1，第一次访问s1, 请求重试只会在s1、s2中进行
  MaxAutoRetriesNextServer: 1
  # 是否所有操作都允许重试。默认值为false
  OkToRetryOnAllOperations: true
  ...

# 服务的相关配置
test-service:
  ribbon:
    # 在提供的server列表中获取服务实例
    NIWSServerListClassName: com.netflix.loadbalancer.ConfigurationBasedServerList
    listOfServers: 127.0.0.1:9000, 127.0.0.1:9001
    ...
```

基本配置是和前面提到的转发到物理地址的配置是相同的，不同的是这里增加了Ribbon和Hytrix的相关配置。

匹配路由映射到服务，服务是由服务治理框架提供，如Eureka、Consul。之后通过Ribbon，进行负载均衡，Hytrix负责防护熔断。

所以说Zuul除了路由配置是自身提供的，其余的配置是由Ribbon和Hytrix提供的。

### Example - 3 - Ribbon的简单配置

```yaml
zuul:
  routes:
    test-service:
      path: /test/**
      serviceId: test-service
      stripPrefix: false

test-service:
  ribbon:
    listOfServers: 127.0.0.1:9000
```

Request URL：http://127.0.0.1:8001/test
Zuul Forward URL：http://127.0.0.1:9000/test

### Example - 4 - 重试

```yaml
zuul:
  routes:
    test-service:
      path: /test/**
      serviceId: test-service
      stripPrefix: false
      # 开启重试
      retryable: true
      
# 具体重试规则在ribbon中配置
ribbon:
  eureka:
    enabled: false
  MaxAutoRetries: 1
  MaxAutoRetriesNextServer: 1
  
test-service:
  ribbon:
    listOfServers: 127.0.0.1:9000, 127.0.0.1:9001, 127.0.0.1:9002
```

Request URL：http://127.0.0.1:8001/test

第一步：

+ 转发到：http://127.0.0.1:9000/test
+ 返回：code: 500

![Example4-第一次请求到端口9000](http://qiniu.5ires.top/uPic/image-20200817190925927.png)

第二步：

+ 重试：http://127.0.0.1:9000/test
+ 返回：code: 500

![Example4-重试第一次端口号9001](http://qiniu.5ires.top/uPic/image-20200817191248495.png)

第三步和第四步：

+ 重试：http://127.0.0.1:9001/test
+ 返回：因为9001端口没有启动服务，返回没有连接

![Example4-重试转发至服务的另一个实例 端口号9001](http://qiniu.5ires.top/uPic/image-20200817191521361.png)

至此，本次请求，一共重试了3次。

测试代码：https://github.com/goldpumpkin/learn-demo

## 总结

Zuul的配置，包括了自己的路由配置、Ribbon配置以及Hytrix配置。其中路由配置分为两种，一种是把请求转发到物理地址，另一种是把请求转发到微服务。要想掌握和配置好Zuul，那么必须学习Ribbon和Hytrix，这样才能很好的驾驭Zuul。

**我的个人水平有限，欢迎大家指正，欢迎交流~**

