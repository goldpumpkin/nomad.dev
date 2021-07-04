---
title: SpringBoot-Oauth2.0（三） —— Token 的存储方案
tags : [
    "java"
]
date : "2020-09-01"
---

上一篇，我们把 Token 放到了关系型数据库当中存储，如果你的系统对认证接口响应时间要求很高，那么在关系型数据库中，查询 Token，一定会是一个瓶颈。那么怎么办呢？如果仅从存储 Token 方面考虑，有什么可以替代关系型数据的存储呢？
<!--more-->

## Token 存储的分析

第一，一般关系型数据库中的数据会存放于磁盘当中的，时间主要消耗于 IO 操作。那我们把 Token 放到内存中就可以解决 IO 问题，顺便也减少了对数据库的网络请求，而在 SpringBoot Oauth 框架中默认就是存储就是在内存当中的。

第二，第一种方案是有缺陷的，现在大多是应用都是分布式架构，把 Token 存放于一台实例的内存，是非常不合理的。这时候需要一个性能很高的中间介质来替代关系型数据库，[Redis](https://redis.io/topics/introduction) 就是一个很好的选择。一方面是因为 Redis 是基于内存操作的，性能非常出色；另一方面，Redis 可以设置过期时间， 正好符合 Token 定时过期的特性。

第三，还有没有其他方案呢？我们从另一个角度想想，我们为什么要存储 Token 呢？ 因为 Token 是系统发放的，是允许客户端访问系统的一种授权凭证，当客户端携带 Token 请求资源的时候，系统是需要验证 Token 是合法授权的，才允许客户端可以访问相关资源。那么我们是不是也可以这样理解，只要系统能够验证授权的 Token ，不存起来也是可以的。其实有一种 Token 可以做到不存储， token 本身就带有授权信息，系统只需要在内存中用对应的算法就可以验证 Token 是不是合法授权的 ，这种方式就是用 [JWT](https://en.wikipedia.org/wiki/JSON_Web_Token) 。

> **JWT是什么？**
>
> JSON Web Token(JWT)，是一种认证解决方案，客户端和服务器用 JWT 规定格式的 Token 进行身份认证交互。JWT 的格式分为三段，每段之间用「.」做间隔，并且每段包含了不同的信息，当然作用也不同。分别是：
>
> **Header - 头部**
> 	数据格式：JSON 数据经过 Base64URL 编码
> 	信息：指定了加密类型及 Type 为 JWT
>
> **Payload - 负载**
> 	数据格式：JSON 数据经过 Base64URL 编码
> 	信息：Client 的授权信息等
>
> **Signature - 签名**
> 	说明：前两部分和约定好的秘钥经过指定加密算法生成，也叫[信息验证码 MAC ](https://zh.wikipedia.org/wiki/%E8%A8%8A%E6%81%AF%E9%91%91%E5%88%A5%E7%A2%BC)，防止数据篡改	

以上我们提出了3种方案，来替代数据库存储 Token 。其中，第一种在第一篇文章中已经实践过了。那接下来，我们分别实践一下另外两种方案。

## Token 存储到 Redis 

### Pom 

```xml
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
```

### 配置

```yaml
spring:
  redis:
    host: {redis host}
    password: {redis password}
```

### 请求 Token

![redis-请求token](http://qiniu.5ires.top/uPic/image-20200903180205938.png)

### Redis 中的 Token 数据

![redis-存储token](http://qiniu.5ires.top/uPic/image-20200903180425965.png)

可以看出数据库新增了4个 Key ，那他们的 Value 是什么呢？直接看数据，都是二进制数据，看不出来。那我们回到源码去找答案。找到类 OAuth2Authentication 的 storeAccessToken 方法，可以看出除了 Key 为 auth:token 的 Value 是OAuth2Authentication 实例的序列化二进制数据外，其他 Key 的 Value 都是 Token 对应的二进制数据。

那现在 Token 已经存到了我们预期的 Redis 当中了，最后再请求下资源，看是否可以通过，完整的验证下。

### 请求资源

![redis-请求资源](http://qiniu.5ires.top/uPic/image-20200903182015962.png)

毫不意外和惊喜的接受预期的结果吧。

现在我们已经实现了把 Token 存储到 Redis 当中，其实和存储到数据库中的做法很像，更换 TokenStore 就好了。那我们继续实践下 JWT 方案吧。

## JWT

### Pom

```xml
    <dependency>
      <groupId>org.springframework.security</groupId>
      <artifactId>spring-security-jwt</artifactId>
    </dependency>
```

### 配置

上面已经简单介绍过 JWT ，其中和配置相关我们要注意的是，我们需要约定一个秘钥并且指定 JWT 对应的算法。JWT 默认的算法是 HMACSHA256 ，在框架找到对应的验证器 `MacSigner` 

```java
/**
* 配置jwt相关
* 省略了一部分代码
**/
@Configuration
@EnableAuthorizationServer
public class MyAuthorizationServerConfigurer extends AuthorizationServerConfigurerAdapter {

		// 指定加密秘钥
    @Value("${jwt.key:GoLdJwtKey}")
    private String tokenSecretKey;

    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
				// 指定 token 转化器
        JwtAccessTokenConverter jwtAccessTokenConverter = new JwtAccessTokenConverter();
      	// 设置加签秘钥
        jwtAccessTokenConverter.setSigningKey(tokenSecretKey);
      	// 设置信息验证码校验器
        jwtAccessTokenConverter.setVerifier(new MacSigner(tokenSecretKey));
        TokenStore tokenStore = new JwtTokenStore(jwtAccessTokenConverter);

        endpoints.accessTokenConverter(jwtAccessTokenConverter);
        endpoints.tokenStore(tokenStore);
    }
}
```

### 获取 Token 

![jwt-获取token](http://qiniu.5ires.top/uPic/image-20200904114623764.png)

这个 Token 也太长了吧，完整 Token 如下：

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOlsicmVzIl0sInNjb3BlIjpbIndyaXRlIl0sImV4cCI6MTU5OTIzNDM1OSwianRpIjoiYjQ2NmVkNDEtNWI1Ni00NDc2LWE4ZjctYjEwYjQ0MTFhNTViIiwiY2xpZW50X2lkIjoiZ29sZCJ9.P-510ioyW4mfjS_UFlCREqnCail2GfMHFx4Mc2Jjf4Q
```

来一起看下这个 Token，确实有三段，前两段可以直接用 Base64URL 解码。那我们直接到 [JWT 官网](https://jwt.io/)解码一下：

![jwt-decode](http://qiniu.5ires.top/uPic/image-20200904115142148.png)

信息如图所示，看到了客户端的相关信息，这也是我们想要的 Token 本身已经承载了 Client 的相关授权信息。接下来继续完成我们的验证，请求一下资源看结果。

### 请求资源

![jwt-请求资源](http://qiniu.5ires.top/uPic/image-20200904115634873.png)

很顺利，我们请求资源成功了。

## 总结

今天我们的目的就是寻找替代数据库存储 Token 的方案，分析之后找出了3种方案，并分别进行了实践。如果你的应用是单机，那么 Token 直接用内存就可以，很方便。如果你的应用是分布式的，那么关系型数据库是一种选择，如果对性能要求很高，那就上 Redis 吧。不过 JWT 方案性能也很高，还不要存储，只是暴露了一些授权信息，你可以把 Token 生效时间控制一下，因为它颁发后就无法在服务器侧失效，生产用它也没有太大问题。具体情况具体分析后，再选择合适的方式存储 Token 吧。

**个人水平有限，欢迎大家指正，欢迎关注微信公众号「小黄的笔记」一起交流哦~~~**
![小黄的笔记](http://qiniu.5ires.top/uPic/1598968637527.jpg)

**demo：https://github.com/goldpumpkin/learn-demo/tree/master/springboot-oauth** 