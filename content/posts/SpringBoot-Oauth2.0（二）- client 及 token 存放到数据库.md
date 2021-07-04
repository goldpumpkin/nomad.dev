---
title: SpringBoot-Oauth2.0（二）—— client 及 token 存放到数据库
tags : [
    "java"
]
date : "2020-08-31"
---

[上一篇](https://juejin.im/post/6865971696017833997)我们已经用最简单的方式，搭建了一个授权方式是 client_credentials 的 Oauth2 的流程。那么现在，在此基础上，我们就再往前迈一步，我们把 client 信息和 token 存储到数据库当中，方便我们管理。并且密码需要保证安全，那么就需要加密。目标很明确，那我们开始吧！
<!--more-->

## client&token存储到数据库

### 步骤：

1. 数据库准备：只创建我们需要用到的表
2. 添加数据库相关依赖：使用 mysql 数据库
3. Oauth 存储配置的设置
4. 验证

### 数据库准备

在本地数据库，创建两张表：

+ 一张表存储 client 相关信息
+ 另一张表存储 token

```sql
# client 相关信息
create table oauth_client_details
(
    client_id               VARCHAR(256) PRIMARY KEY comment '必填，Oauth2 client_id',
    resource_ids            VARCHAR(256) comment '可选，资源id集合，多个资源用英文逗号隔开',
    client_secret           VARCHAR(256) comment '必填，Oauth2 client_secret',
    scope                   VARCHAR(256) comment '必填，Oauth2 权限范围，比如 read，write等可自定义',
    authorized_grant_types  VARCHAR(256) comment '必填，Oauth2 授权类型，支持类型：authorization_code,password,refresh_token,implicit,client_credentials，多个用英文逗号隔开',
    web_server_redirect_uri VARCHAR(256) comment '可选，客户端的重定向URI,当grant_type为authorization_code或implicit时,此字段是需要的',
    authorities             VARCHAR(256) comment '可选，指定客户端所拥有的Spring Security的权限值',
    access_token_validity   INTEGER comment '可选，access_token的有效时间值(单位:秒)，不填写框架(类refreshTokenValiditySeconds)默认12小时',
    refresh_token_validity  INTEGER comment '可选，refresh_token的有效时间值(单位:秒)，不填写框架(类refreshTokenValiditySeconds)默认30天',
    additional_information  VARCHAR(4096) comment '预留字段，格式必须是json',
    autoapprove             VARCHAR(256) comment '该字段适用于grant_type="authorization_code"的情况下，用户是否自动approve操作'
);

# token 存储
create table oauth_access_token
(
    token_id          VARCHAR(256) comment 'MD5加密后存储的access_token',
    token             BLOB comment 'access_token序列化的二进制数据格式',
    authentication_id VARCHAR(256) PRIMARY KEY comment '主键，其值是根据当前的username(如果有),client_id与scope通过MD5加密生成的,具体实现参见DefaultAuthenticationKeyGenerator',
    user_name         VARCHAR(256),
    client_id         VARCHAR(256),
    authentication    BLOB comment '将OAuth2Authentication对象序列化后的二进制数据',
    refresh_token     VARCHAR(256) comment 'refresh_token的MD5加密后的数据'
);
```

之后，我们需要添加自定义 client 信息。本次演示添加的 client 信息如下（依旧本着尽量能不配置就不配置的原则）：

```sql
INSERT INTO oauth_client_details (client_id, resource_ids, client_secret, scope, authorized_grant_types, web_server_redirect_uri, authorities, access_token_validity, refresh_token_validity, additional_information, autoapprove) VALUES ('gold', 'res', '{noop}123456', 'write', 'client_credentials', null, null, null, null, null, null);
```

> 说明
>
> 1. 在官方给出的源码当中，是有对应的 [schema.sql](https://github.com/spring-projects/spring-security-oauth/blob/master/spring-security-oauth2/src/test/resources/schema.sql) 文件，这里只创建涉及我们示例中需要的表
> 2. 对两张表的操作，我们可以去看这两个类：JdbcClientDetailsService & JdbcTokenStore
> 3. 此外本示例，添加了 resource_ids ，注意在配置 `ResourceServerSecurityConfigurer` 中对应

### Pom

```xml
    <dependency>
      <groupId>mysql</groupId>
      <artifactId>mysql-connector-java</artifactId>
    </dependency>

    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-jdbc</artifactId>
    </dependency>
```

### 认证服务器代码修改

```java
@Configuration
@EnableAuthorizationServer
public class MyAuthorizationServerConfigurer extends AuthorizationServerConfigurerAdapter {

    private final DataSource dataSource;

    public MyAuthorizationServerConfigurer(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        clients.jdbc(dataSource);
    }

    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        endpoints.tokenStore(new JdbcTokenStore(dataSource));
    }
}
```

上述代码只显示改动的部分，修改 client 的配置和 tokenStore 的配置，都进行添加数据源即可

### 请求 token

下面的请求是在 postman 中进行的。 base_url 为设置的全局变量，实际为 http://127.0.0.1:8080

![获取token](http://qiniu.5ires.top/uPic/image-20200901102558730.png)

### 获取资源

![请求资源](http://qiniu.5ires.top/uPic/image-20200901102640629.png)

### 观察数据库

![token存储](http://qiniu.5ires.top/uPic/image-20200901102741912.png)

由于 token 在数据库是存储的是二进制形式，但是我们通过 client_id 数据，可以看出是我们刚刚请求的 client。

到此为止我们已经是实现了，把 client 及 token 信息存储到数据库了，这样更便于我们对 client 及 token 数据的管理。但是数据库存储明文密码是不安全，那么接下来，我们对 client_secret 进行加密。

## client_secret 加密

### 配置 passwordEncoder

SpringBoot Oauth 本身支持的加密算法有很多种，详细信息可以看类 `PasswordEncoderFactories` ，包括我们最常用的 MD5、SHA 等，我们使用 [bcrypt 加密算法](https://zh.wikipedia.org/wiki/Bcrypt)， 那么直接配置支持全部算法的 passwordEncoder ，即 `DelegatingPasswordEncoder` 。

```java
@Override
public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        	  		clients.jdbc(dataSource).passwordEncoder(PasswordEncoderFactories.createDelegatingPasswordEncoder());

}
```

### 加密 client_secret 

既然我们这次使用 bcrypt 加密， 直接可以找到 `BCryptPasswordEncoder` ，通过它可以给我们的密码进行加密，并把密码存储于数据库当中。

```java
@NoArgsConstructor(access = AccessLevel.PRIVATE)
public class PasswordEncodeUtil {

    private static final BCryptPasswordEncoder bcryptEncoder = new BCryptPasswordEncoder();

    public static String bcryptEncode(String password) {
        return bcryptEncoder.encode(password);
    }

    public static String genOauthEncodePwd(String password) {
        return "{bcrypt}" + bcryptEncode(password);
    }


    public static void main(String[] args) {
        String oriPwd = "123456";
        System.out.println(genOauthEncodePwd(oriPwd));

    }
}
```

原密码：123456
加密后密码：{bcrypt}$2a$10$NPxtsEUMmBGTlzVXlT.scubSCXNEDlBAq2r2t7iQFB/.RaNBlh0nO

![client_sercret 加密](http://qiniu.5ires.top/uPic/image-20200901113958428.png)

*注意：加密密码的前缀 大括号 "{xxx}"，是指定算法名称。因为框架支持多种算法，那么必须需要带有算法前缀。*

### 请求获取 token

下面的请求是在 postman 中进行的。 base_url 为设置的全局变量，实际为 http://127.0.0.1:8080

![请求token](http://qiniu.5ires.top/uPic/image-20200901114224122.png)

### 请求资源

![请求资源](http://qiniu.5ires.top/uPic/image-20200901114259361.png)

## 小结

本文主要是以 client_credentials 的授权方式，把 client 和 token 信息存储在数据库当中，来方便我们管理。同时，为了保证密码的安全，我们把 client_secret 用 bcrypt 算法进行了加密操作，并存储到数据库当中。有了上一篇基础，这篇整体挺简单的吧，那同学们动起来，实现一下吧！

**个人水平有限，欢迎大家指正，一起交流哦~~~**

**demo：https://github.com/goldpumpkin/learn-demo/tree/master/springboot-oauth** 

*Reference:*

1. [spring-oauth-server 数据库表说明](http://andaily.com/spring-oauth-server/db_table_description.html)