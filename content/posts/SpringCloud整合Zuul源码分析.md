---
title: SpringCloud 整合 Zuul 源码分析
tags : [
    "Java",
    "Spring Cloud"
]
date : "2020-07-21"

---

今天我们分析下 SpringCloud 是怎么整合 Zuul 的。
<!--more-->

## 回顾

1. Zuul是通过`ZuulServletFilter`或者 `ZuulServlet`接管我们的请求

2. Zuul整个流程如下：

   `ZuulServletFilter(ZuulServlet)` ->  `ZuulRunner` -> `FilterProcessor` -> `ZuulFilter`

## 目标

明确SpringMVC和Zuul框架是怎么配合的

## 引入Zuul的版本信息

```xml
<properties>
	<spring-cloud.version>Hoxton.RELEASE</spring-cloud.version>
</properties>

<dependencyManagement>
	<dependencies>
  	<dependency>
    	<groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-dependencies</artifactId>
      <version>${spring-cloud.version}</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
    
<dependencies>
	<dependency>
  	<groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
   </dependency>
</dependencies>
```



## Zuul功能启用及配置的加载

### Zuul的启用 - @EnableZuulProxy

```java
// 引入断路器功能
@EnableCircuitBreaker
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
// 注入触发Zuul配置类的标记Bean
@Import(ZuulProxyMarkerConfiguration.class)
public @interface EnableZuulProxy {
}
```

### ZuulProxyAutoConfiguration - Zuul自动配置Bean

```java
// 此配置类不会被代理
@Configuration(proxyBeanMethods = false)
// 引入Ribbon相关配置
@Import({ RibbonCommandFactoryConfiguration.RestClientRibbonConfiguration.class,
		RibbonCommandFactoryConfiguration.OkHttpRibbonConfiguration.class,
		RibbonCommandFactoryConfiguration.HttpClientRibbonConfiguration.class,
		HttpClientConfiguration.class })
@ConditionalOnBean(ZuulProxyMarkerConfiguration.Marker.class)
public class ZuulProxyAutoConfiguration extends ZuulServerAutoConfiguration {

	// 省略部分代码。。。

	// 加载pre filters bean
	@Bean
	@ConditionalOnMissingBean(PreDecorationFilter.class)
	public PreDecorationFilter preDecorationFilter(RouteLocator routeLocator,
			ProxyRequestHelper proxyRequestHelper) {
		return new PreDecorationFilter(routeLocator,
				this.server.getServlet().getContextPath(), this.zuulProperties,
				proxyRequestHelper);
	}

	// 加载route filters bean
	@Bean
	@ConditionalOnMissingBean(RibbonRoutingFilter.class)
	public RibbonRoutingFilter ribbonRoutingFilter(ProxyRequestHelper helper,
			RibbonCommandFactory<?> ribbonCommandFactory) {
		RibbonRoutingFilter filter = new RibbonRoutingFilter(helper, ribbonCommandFactory,
				this.requestCustomizers);
		return filter;
	}

  // 加载route filters bean
	@Bean
	@ConditionalOnMissingBean({ SimpleHostRoutingFilter.class,
			CloseableHttpClient.class })
	public SimpleHostRoutingFilter simpleHostRoutingFilter(ProxyRequestHelper helper,
			ZuulProperties zuulProperties,
			ApacheHttpClientConnectionManagerFactory connectionManagerFactory,
			ApacheHttpClientFactory httpClientFactory) {
		return new SimpleHostRoutingFilter(helper, zuulProperties,
				connectionManagerFactory, httpClientFactory);
	}
}
```

### ZuulServerAutoConfiguration - Zuul自动配置Bean

```java
@Configuration(proxyBeanMethods = false)
// 加载zuul的自定义properties配置
@EnableConfigurationProperties({ ZuulProperties.class })
// 加载前提：classpath下有类ZuulServlet和ZuulServletFilter
@ConditionalOnClass({ ZuulServlet.class, ZuulServletFilter.class })
@ConditionalOnBean(ZuulServerMarkerConfiguration.Marker.class)
public class ZuulServerAutoConfiguration {

	// 省略部分代码。。。

  // ZuulController是Controller的一个实现，负责将拦截的请求交给ZuulServlet处理
	@Bean
	public ZuulController zuulController() {
		return new ZuulController();
	}

  // ZuulHandlerMapping负责路由匹配
	@Bean
	public ZuulHandlerMapping zuulHandlerMapping(RouteLocator routes,
			ZuulController zuulController) {
		ZuulHandlerMapping mapping = new ZuulHandlerMapping(routes, zuulController);
		mapping.setErrorController(this.errorController);
		mapping.setCorsConfigurations(getCorsConfigurations());
		return mapping;
	}

  // 默认加载ZuulServlet
	@Bean
	@ConditionalOnMissingBean(name = "zuulServlet")
	@ConditionalOnProperty(name = "zuul.use-filter", havingValue = "false",
			matchIfMissing = true)
	public ServletRegistrationBean zuulServlet() {
		ServletRegistrationBean<ZuulServlet> servlet = new ServletRegistrationBean<>(
				new ZuulServlet(), this.zuulProperties.getServletPattern());
		// The whole point of exposing this servlet is to provide a route that doesn't
		// buffer requests.
		servlet.addInitParameter("buffer-requests", "false");
		return servlet;
	}

  // 当配置zuul.use-filter=true，加载zuulServletFilter, 表示用filter来拦截请求
	@Bean
	@ConditionalOnMissingBean(name = "zuulServletFilter")
	@ConditionalOnProperty(name = "zuul.use-filter", havingValue = "true",
			matchIfMissing = false)
	public FilterRegistrationBean zuulServletFilter() {
		final FilterRegistrationBean<ZuulServletFilter> filterRegistration = new FilterRegistrationBean<>();
		filterRegistration.setUrlPatterns(
				Collections.singleton(this.zuulProperties.getServletPattern()));
		filterRegistration.setFilter(new ZuulServletFilter());
		filterRegistration.setOrder(Ordered.LOWEST_PRECEDENCE);
		// The whole point of exposing this servlet is to provide a route that doesn't
		// buffer requests.
		filterRegistration.addInitParameter("buffer-requests", "false");
		return filterRegistration;
	}

	// 在Zuul各阶段filter处理过程中捕获异常，SendErrorFilter会forward "/error" 
	@Bean
	public SendErrorFilter sendErrorFilter() {
		return new SendErrorFilter();
	}

	@Configuration(proxyBeanMethods = false)
	protected static class ZuulFilterConfiguration {

    // 注入Spring容器中的ZuulFilter类型所有的实现类，包括内置和自定义的Filter，内置的有10个
		@Autowired
		private Map<String, ZuulFilter> filters;

    // 注册ZuulFilter到FilterRegistry中
		@Bean
		public ZuulFilterInitializer zuulFilterInitializer(CounterFactory counterFactory,
				TracerFactory tracerFactory) {
			FilterLoader filterLoader = FilterLoader.getInstance();
			FilterRegistry filterRegistry = FilterRegistry.instance();
			return new ZuulFilterInitializer(this.filters, counterFactory, tracerFactory,
					filterLoader, filterRegistry);
		}
	}
}
```

以上两个类，加载了Zuul的相关配置类：

+ 拦截请求：

  + 和SpringMVC结合的Bean：`ZuulController`、`ZuulHandlerMapping`
  + 通过Web Filter拦截请求Bean：`ZuulServletFilter`

+ Zuul流程需要的Bean：

  + 自带的ZuulFilter，有10个下面会一一介绍

  + 监控相关

  + ZuulFilter的容器：`FilterRegistry`

    

### 默认的ZuulFilters

### Pre Filter

1. `ServletDetectionFilter`  order = -3

   作用：判断请求是否是由`DispatcherServlet` or  `ZuulServlet`传来的，并把判断结果以键值对的形式放在`RequestContext`

2. `Servlet30WrapperFilter` order = -2

   作用：包装request，兼容servlet3.0

3. `FormBodyWrapperFilter` order = -1

   作用：包装表单数据并为下游服务重新编码

4. `DebugFilter` order = 1

   作用：如果debug请求，那么会在`RequestContext`中标记为debug请求和routing

5. `PreDecorationFilter` = 5

   作用：请求路由和zuul路由配置进行匹配，并设置与代理相关的头部信息

### Route Filter

1. `RibbonRoutingFilter` order = 10

   作用：使用Ribbon、Hytrix和可插拔的httpClient发送请求，serviceId、是否重试以及负载均衡策略在相关联的`RequestContext`获取

2. `SimpleHostRoutingFilter`  order = 100

   作用：用HttpClient发送请求到预定的URLs，URLs通过`RequestContext#getRouteHost()`获取

3. `SendForwardFilter` order = 500

   作用：用`RequestDispatcher`forwards请求，转发的地址是`RequestContext`的`FilterConstants#FORWARD_TO_KEY`对应value

### Post Filter

1. `SendResponseFilter` order = 1000

   作用：写 代理的请求得到的响应 到 当前响应

### Error Filter

1. `SendErrorFilter`  order = 0

   作用：如果`RequestContext#getThrowable()` 不为空，默认将请求转发到 /error



## SpringMVC怎么把请求转发给Zuul？

### 从配置类分析

从上述配置可以看下几个重要的配置类源码：

#### `ZuulController`

```java
public class ZuulController extends ServletWrappingController {

	public ZuulController() {
    // 设置Servlet的类型
		setServletClass(ZuulServlet.class);
		setServletName("zuul");
		setSupportedMethods((String[]) null); // Allow all
	}

	@Override
	public ModelAndView handleRequest(HttpServletRequest request,
			HttpServletResponse response) throws Exception {
		try {
			// We don't care about the other features of the base class, just want to
			// handle the request
			return super.handleRequestInternal(request, response);
		}
		finally {
			// @see com.netflix.zuul.context.ContextLifecycleFilter.doFilter
			RequestContext.getCurrentContext().unset();
		}
	}

}
```

#### `ServletWrappingController`

```java
public class ServletWrappingController extends AbstractController implements BeanNameAware, InitializingBean, DisposableBean {
  // 省略代码。。
  
	@Override
		public void afterPropertiesSet() throws Exception {
			if (this.servletClass == null) {
				throw new IllegalArgumentException("'servletClass' is required");
			}
			if (this.servletName == null) {
				this.servletName = this.beanName;
			}
      // 通过反射 初始化servlet
			this.servletInstance = ReflectionUtils.accessibleConstructor(this.servletClass).newInstance();
			this.servletInstance.init(new DelegatingServletConfig());
		}

		// 通过servlet实例处理请求
		@Override
		protected ModelAndView handleRequestInternal(HttpServletRequest request, HttpServletResponse response)
		throws Exception {
			Assert.state(this.servletInstance != null, "No Servlet instance");
			this.servletInstance.service(request, response);
			return null;
		}
}
```

#### `ZuulHandlerMapping`

```java
public class ZuulHandlerMapping extends AbstractUrlHandlerMapping {
  	private final ZuulController zuul;
  	private volatile boolean dirty = true;

  // 根据寻找路由处理器
	@Override
	protected Object lookupHandler(String urlPath, HttpServletRequest request) throws Exception {
		if (this.errorController != null && urlPath.equals(this.errorController.getErrorPath())) {
			return null;
		}
    // 如果属于配置的忽视路由，则返回null
		if (isIgnoredPath(urlPath, this.routeLocator.getIgnoredPaths())) return null;
		RequestContext ctx = RequestContext.getCurrentContext();
		if (ctx.containsKey("forward.to")) {
			return null;
		}
		if (this.dirty) {
      // dirty默认为true，第一次会触发注册处理器到Spring容器中
      // 或者发送zuul路由刷新事件，设置dirty为true，见ZuulRefreshListener
			synchronized (this) {
				if (this.dirty) {
					registerHandlers();
					this.dirty = false;
				}
			}
		}
    // 交给Spring查找路由对应的handler
		return super.lookupHandler(urlPath, request);
	}

  // 注册配置路由对应的处理器
	private void registerHandlers() {
		Collection<Route> routes = this.routeLocator.getRoutes();
		if (routes.isEmpty()) {
			this.logger.warn("No routes found from RouteLocator");
		}
		else {
			for (Route route : routes) {
        // 在Spring容器中注册zuul路由配置对应ZuulController处理器
				registerHandler(route.getFullPath(), this.zuul);
			}
		}
	}

}
```

以上配置类：

+ `ZuulController`：它是`ServletWrappingController`的 子类，将请求给到`ZuulServlet`去处理
+ `ZuulHandlerMapping`：它是`AbstractUrlHandlerMapping`的子类，将请求路由到`ZuulController`处理
+ `ZuulServlet`：由上一篇知道它是Zuul流程的入口之一

> 回顾SpingMVC对于请求的处理流程
>
> 1. 客户端请求交给SpringMVC的`DispatcherServlet`统一处理
> 2. 通过已经注册的`HandlerMapping`, 根据请求路由找到处理器执行链`HandlerExecutionChain`，包括请求各个拦截器`HandlerInterceptor`和请求处理器`handler`
> 3. 找到请求处理器对应的适配器`HandlerAdapter`
> 4. 执行已注册的各拦截器的`preHandle`方法
> 5. 调用处理器处理请求，返回模型数据以及视图`ModelAndView`
> 6. 执行已注册的各拦截器的`postHandle`方法
> 7. 根据给定的`ModelAndView`进行渲染
> 8. 响应客户端

结合SpingMVC对于请求的处理流程可以猜到，当请求给到SpringMVC的`DispatcherServlet`后，如果该路由是需要Zuul拦截的请求，那么会匹配到`ZuulHandlerMapping`，从而找到处理器`ZuulController`，之后在处理的时候，会交给`ZuulServlet`，后面的流程见上一篇文章。

### Debug验证

zuul拦截配置：

```properties
# zuul
# 是否启用ZuulServletFilter
# zuul.use-filter=true
ribbon.ConnectTimeout = 30000
ribbon.ReadTimeout = 30000
ribbon.eureka.enabled = false

management.endpoints.web.exposure.include = *
zuul.routes.test.path = /test/**
zuul.routes.test.stripPrefix = false
test.ribbon.listOfServers = ${service.test}
service.test=http://127.0.0.1:8081/t/test
```

请求：`curl -v http://127.0.0.1:8080/test`

图示过程：

![image-20200724183906920](http://qiniu.5ires.top/uPic/image-20200724183906920.png)

![image-20200724184049891](http://qiniu.5ires.top/uPic/image-20200724184049891.png)

结果显示：猜想是正确的。
大致流程：`DispatcherServlet` -> `ZuulController` -> `ZuulServlet` -> 执行各阶段`ZuulFilters`

![image-20200803183334690](http://qiniu.5ires.top/uPic/ZuulServlet%E6%8E%A5%E7%AE%A1%E6%B5%81%E7%A8%8B%E5%9B%BE.png)

## ZuulServletFilter - 另一种拦截请求流程

### 配置

```java
	// 在类ZuulServerAutoConfiguration中加载

	@Bean
	@ConditionalOnMissingBean(name = "zuulServletFilter")
	@ConditionalOnProperty(name = "zuul.use-filter", havingValue = "true",
			matchIfMissing = false)
	public FilterRegistrationBean zuulServletFilter() {
		final FilterRegistrationBean<ZuulServletFilter> filterRegistration = new FilterRegistrationBean<>();
    // URL匹配规则: /zuul
		filterRegistration.setUrlPatterns(
				Collections.singleton(this.zuulProperties.getServletPattern()));
		filterRegistration.setFilter(new ZuulServletFilter());
		filterRegistration.setOrder(Ordered.LOWEST_PRECEDENCE);
		filterRegistration.addInitParameter("buffer-requests", "false");
		return filterRegistration;
	}
```

`ZuulServletFilter`的URL匹配规则是`/zuul`, 而且如果要是使得`ZuulServletFilter`Bean加载，必须在配置文件中，添加：`zuul.use-filter=true`，如图：

```properties
# 是否启用filter拦截
zuul.use-filter=true
zuul.routes.test.path = /zuul/test/**
zuul.routes.test.stripPrefix = false
test.ribbon.listOfServers = ${service.test}
service.test=http://127.0.0.1:8081
```

### ZuulServletFilter源码

```java
public class ZuulServletFilter extends com.netflix.zuul.filters.ZuulServletFilter {
	@Override
	public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse,
			FilterChain filterChain) throws IOException, ServletException {
		RequestContext context = RequestContext.getCurrentContext();
		context.setZuulEngineRan();
		super.doFilter(servletRequest, servletResponse, filterChain);
	}
}
```

源码很简单，在请求上下文添加了一个标志位`zuulEngineRan`为true。并执行父类`com.netflix.zuul.filters.ZuulServletFilter`的`doFilter`方法，进而进入了Zuul的核心流程当中，后面的流程我们已经熟悉了。

其中要注意下，`com.netflix.zuul.filters.ZuulServletFilter`虽然是Filter，但是并没有在其`doFilter`方法中调用`FilterChain`的`doFilter`方法，我们可以回想下，如果是我们自己写FIlter，一定会调用。之所以`ZuulServletFilte`没有这么做，是因为它要接管请求，并不要Servlet来处理。

![image-20200803183210907](http://qiniu.5ires.top/uPic/ZuulServletFilter.png)

大致流程如图：

![image-20200803183440164](http://qiniu.5ires.top/uPic/ZuulServletFilter%E6%8E%A5%E7%AE%A1%E6%B5%81%E7%A8%8B%E5%9B%BE.png)

## 总结

Zuul和Spring结合并接管请求主要有两种方式：

+ 在Spring容器中通过注册请求处理器`ZuulController`和路由处理器的映射`ZuulHandlerMapping`，做到请求的拦截，并内置了一些ZuulFIlter保证请求的处理。
+ 通过注册`ZuulServletFilter`,使用Filter方式接管请求，注意默认的路径匹配及生效配置

