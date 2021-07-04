---
title: SpringBoot-Oauth2.0（一） —— 初识
tags : [
    "java"
]
date : "2020-08-26"
---

最近在搞平台 API 的安全和认证的相关东西，接口安全和认证在生产活动中是非常重要的。目前最流行的就是 Oauth2 的认证方式。接下来就用 SpringBoot 的安全依赖简单实践一下，了解一下 Oauth2 的流程。
<!--more-->

## Oauth2的简单认识

### 是什么？

是一种授权机制，用来授权第三方应用，获取用户数据

### 授权的四种方式

+ 授权码模式（authorization-code）

  此方式安全性最高，授权码通过前端传送，令牌存储在后端。主要适用于有后端的web应用。

+ 隐藏式（implicit）

  此方式无授权码中间步骤，很不安全。主要适用于纯前端应用。

+ 密码式（password）

  此方式需要用户高度信任第三方应用，风险比较大。

+ 客户端凭证式（client credentials）

  这种方式适用于服务器和服务器之间的应用，是关于接口安全的认证。

### 客户端凭证认证（client credentials）

这种方式是是我们今天主要实践的认证方式。如果有同学在平时的开放当中，经常和外部第三方对接，应该很熟悉这种认证方式。我们最常见的就是在微信开发当中，到微信公众号官方服务器认证获取 access token，之后我们请求相关微信接口，需要带上此 access token 才有权限访问。

## SpringBoot Oatuh2 简单实践

我们简单认识 Oauth2 之后，接下来，我们就动手快速实现一下。

### 大致的思路步骤：

1. 相关依赖的引入
2. 配置 
   + 认证服务信息
   + 资源服务信息
3. 验证
   + 获取认证 access token
   + 访问资源

### Pom

要想开启 Oauth2 认证，我们必须引入 SpringBoot 安全依赖 和 Spring 的 Oauth2 的依赖

```xml
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-security</artifactId>
    </dependency>

    <dependency>
      <groupId>org.springframework.security.oauth</groupId>
      <artifactId>spring-security-oauth2</artifactId>
    </dependency>
```

引入安全认证依赖之后，SpringBoot 会对资源服务器上所有的资源默认进行保护。

### Authorization Service Config - 认证服务配置

认证服务的配置，主要包括以下3个方面：

1. 定义 token endpoint 的安全约束

   主要配置：是否允许客户端以 Form 表单的形式的登录、定义密码的加密方式等

2. 定义客户端详细的信息

   客户端的信息包括了：客户端的信息存储方式（内存 or 数据库）以及客户端的认证需要的信息，包括 client_id、client_secret、grant_type、scope

3. 定义授权和 token endpoint 以及令牌服务。

   可以配置授权的 endpoint、token 的存储方式等

认证服务配置如下：（本次实践，本着能省掉的就省掉的配置原则）

```java
@Configuration
@EnableAuthorizationServer
public class MyAuthorizationServerConfigurer extends AuthorizationServerConfigurerAdapter {

    /**
     * 配置安全约束相关配置
     * @param security 定义令牌终结点上的安全约束
     * @throws Exception
     */
    @Override
    public void configure(AuthorizationServerSecurityConfigurer security) throws Exception {
        // 支持client_id、client_secret以form表单的形式登录,参考可见：微信获取access token
        security.allowFormAuthenticationForClients();
    }

    /**
     * 配置客户端详细信息
     * @param clients 定义客户端详细信息服务的配置程序。可以初始化客户端详细信息，也可以只引用现有存储。
     * @throws Exception
     */
    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        clients.inMemory()
            // client_id
            .withClient("gold")
            // 授权方式
            .authorizedGrantTypes("client_credentials")
            // 授权范围
            .scopes("write")
            // client_secret
            .secret("{noop}123456");

    }

    /**
     *
     * @param endpoints 定义授权和令牌端点以及令牌服务。
     * @throws Exception
     */
    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        super.configure(endpoints);
    }
}
```

*注意密码的配置，需要配置加密方式，此实践是没有用任何加密方式的。*

*相关信息见：DelegatingPasswordEncoder or PasswordEncoderFactories*

### Resource Service Config - 资源服务的配置

资源服务的配置，主要包括以下2个方面：

1. 资源服务的安全配置

   可以配置 资源 ID、stateless - 资源是否仅允许基于令牌的验证、token 的存储方式等

2. Http 安全配置

   我们的被保护的 API，就是在这里配置

资源相关配置：

API：

```java
@RestController
@AllArgsConstructor
public class ResController {

    @GetMapping("/res/{id}")
    public String testOauth(@PathVariable String id) {
        return "Get the resource " + id;
    }
}
```



资源服务配置：

```java
@Configuration
@EnableResourceServer
public class MyResourceServerConfigurer extends ResourceServerConfigurerAdapter {

    @Override
    public void configure(ResourceServerSecurityConfigurer resources) throws Exception {
        super.configure(resources);
    }

    @Override
    public void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
            .antMatchers("/res/**").authenticated();
    }
}
```

### 配置完成小结

我们已经完成了 SpringBoot 的 Oauth2 的最基本的配置。

Client 的相关信息：

+ client_id : gold
+ client_secret : 123456
+ scopes : write
+ grant_type :  client_credentials

接下来，我们就来验证一下我们的配置是否生效。

下面的请求是在 postman 中进行的。 base_url 为设置的全局变量，实际为 http://127.0.0.1:8080

### 直接请求

当我们直接请求，应用返回未授权，无法访问：

![image-20200828171302993](http://qiniu.5ires.top/uPic/image-20200828171302993.png)

### 获取 access token

springboot oauth 默认获取 token 的 endpoint 是: /oauth/token

![image-20200828153448422](http://qiniu.5ires.top/uPic/image-20200828153448422.png)

从请求结果可以看到，我们获取了access_token，并且刷新时间是43199秒，即12个小时。

### 请求被保护的资源

![image-20200828153851996](http://qiniu.5ires.top/uPic/image-20200828153851996.png)

结果来看，我们成功用 access token 访问到了被保护的资源。

## 小结

本文简单介绍了 Oauth2 是什么及授权方式，想了解更多 Oauth2 知识的同学，可以到[阮一峰老师博客](http://www.ruanyifeng.com/blog/2019/04/oauth_design.html)去学习一下。之后用 SpringBoot 简单实现了客户端凭证式的认证方式，其配置的主要关键点在于理解**资源服务器配置（ResourceServerConfigurer）**和**认证服务器配置（AuthorizationServerConfigurer）**。同学们，动起来，实现一下吧！

**个人水平有限，欢迎大家指正，一起交流哦~~~**

**demo：https://github.com/goldpumpkin/learn-demo/tree/master/springboot-oauth** 

*Reference:*
1. [spring-security-oauth](https://projects.spring.io/spring-security-oauth/docs/oauth2.html )
2. [理解OAuth 2.0](http://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html )

