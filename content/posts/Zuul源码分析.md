---
title: Zuul 的源码分析
tags : [
    "java"
]
date : "2020-07-21"

---

Zuul 的源码读过吗？让我们一起看看吧
<!--more-->

## 目标

明确Zuul的执行流程和重要类的分析

## Zuul过滤器的生命周期

![preview](http://qiniu.5ires.top/uPic/view.png)

## 源码分析

### zuul怎么拦截我们的请求？

`ZuulServletFilter` - 继承 Filter | `ZuulServlet` - 继承 HttpServlet
可以通过这两个类，让Zuul接管请求。由于他们的逻辑基本一致，下面用`ZuulServletFilter`来分析

```java
/**
* Zuul核心处理类，拦截请求
**/
public class ZuulServletFilter implements Filter {

    private ZuulRunner zuulRunner;

		// 省略...

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        try {
         		// 初始化requests和responses到RequestContext中，详见ZuulRunner#init
            init((HttpServletRequest) servletRequest, (HttpServletResponse) servletResponse);
            try {
              	// 执行 filterType=pre 的过滤器
                preRouting();
            } catch (ZuulException e) {
                // 执行 filterType=error 的过滤器
                error(e);
                postRouting();
                return;
            }
            
            // Only forward onto to the chain if a zuul response is not being sent
            if (!RequestContext.getCurrentContext().sendZuulResponse()) {
                filterChain.doFilter(servletRequest, servletResponse);
                return;
            }
            
            try {
	              // 执行 filterType=route 的过滤器
                routing();
            } catch (ZuulException e) {
                error(e);
                postRouting();
                return;
            }
            try {
              	// 执行 filterType=post 的过滤器
                postRouting();
            } catch (ZuulException e) {
                error(e);
                return;
            }
        } catch (Throwable e) {
            error(new ZuulException(e, 500, "UNCAUGHT_EXCEPTION_FROM_FILTER_" + e.getClass().getName()));
        } finally {
          	// 清空线程变量
            RequestContext.getCurrentContext().unset();
        }
    }
}
```

以上方法核心步骤：

+ 初始化请求上下文`RequestContext`

+ 执行 pre、route、post过滤器，如果有错，执行error过滤器

  

`ZuulRunner` - 初始化`RequestContext`中的requests和responses并转发Filter相关方法到`FilterProcessor`

```java
public class ZuulRunner {

    private boolean bufferRequests;

    // 省略...

    /**
     * 初始化RequestContext：生成请求和响应wapper保存
     * RequestContext：继承了ConcurrentHashMap，是一个Map容器，主要存放请求、响应供ZuulFilters使用。
     */
    public void init(HttpServletRequest servletRequest, HttpServletResponse servletResponse) {

        RequestContext ctx = RequestContext.getCurrentContext();
        if (bufferRequests) {
            ctx.setRequest(new HttpServletRequestWrapper(servletRequest));
        } else {
            ctx.setRequest(servletRequest);
        }

        ctx.setResponse(new HttpServletResponseWrapper(servletResponse));
    }

    /**
     * executes "post" filterType  ZuulFilters
     *
     * @throws ZuulException
     */
    public void postRoute() throws ZuulException {
        FilterProcessor.getInstance().postRoute();
    }

   // 省略route() preRoute() error() 方法
}	
```

以上看出Zuul是通过`ZuulServletFilter`以filter的方式（或者以`ZuulServlet`以servlet的方式）拦截或者承接我们的请求，并在`doFilter`方法（`service`方法）中处理各种类型的ZuulFilters，并通过`ZuulRunner`转发到`FilterProcessor`中找到对应的filter并执行相关逻辑。整个大致流程比较简单清晰，类似于设计模式中的门面模式。

​	其中，`RequestContext`是存在`ThreadLocal`当中，可以注意到当Zuul处理完毕之后，会清空线程变量`RequestContext`,以防止内存泄露。



### `FilterProcessor`怎么找到相应ZuulFilters并执行呢？

`FilterProcessor` - 执行filters的核心类

```java
/**
* 执行对应阶段ZuulFilters
* sType：即为filterType，例如"post"、"pre"、"route"、"error"
* 
*/
public Object runFilters(String sType) throws Throwable {
        if (RequestContext.getCurrentContext().debugRouting()) {
            Debug.addRoutingDebug("Invoking {" + sType + "} type filters");
        }
        boolean bResult = false;
			  // 获取已经注册了的ZuulFilters，根本是从FilterRegistry中获取。并且list是已经排好序的，
  			// 根据给定的filterOrder
        List<ZuulFilter> list = FilterLoader.getInstance().getFiltersByType(sType);
        if (list != null) {
            for (int i = 0; i < list.size(); i++) {
                ZuulFilter zuulFilter = list.get(i);
                // 执行ZuulFilter逻辑并
                Object result = processZuulFilter(zuulFilter);
                if (result != null && result instanceof Boolean) {
                    bResult |= ((Boolean) result);
                }
            }
        }
        return bResult;
    }

/**
* 执行ZuulFilter，并把执行情况组合成ZuulFilterResult并返回
*/
public Object processZuulFilter(ZuulFilter filter) throws ZuulException {
  // 省略部分代码...
  // 具体执行在 ZuulFilter#runFilter
  ZuulFilterResult result = filter.runFilter();
  ExecutionStatus s = result.getStatus();

  switch (s) {
    case FAILED:
      t = result.getException();
      ctx.addFilterExecutionSummary(filterName, ExecutionStatus.FAILED.name(), execTime);
      break;
    case SUCCESS:
      o = result.getResult();
      ctx.addFilterExecutionSummary(filterName, ExecutionStatus.SUCCESS.name(), execTime);
    default:
      break;
  }
            
  if (t != null) throw t;

  // 统计每个filter的每次执行情况
  usageNotifier.notify(filter, s);
  return o;
}

```

以上方法核心步骤：

+ 按序执行各阶段ZuulFilters

+ 记录ZuulFilter执行结果

+ 统计执行情况

  

`ZuulFilter` - 最基本的Filter抽象类，自定义的Filter是继承此Filter，`FilterProcessor`执行Filter最终会转发到此类的`runFilter`方法

```java
 public ZuulFilterResult runFilter() {
   // 执行结果以及执行成功与否情况包装成ZuulFilterResult返回
   ZuulFilterResult zr = new ZuulFilterResult();
   // 此filter是已被archaius禁用 「archaius是netflix开源的动态属性配置框架」
   if (!isFilterDisabled()) {
     // 执行自定filter的shouldFilter方法判断是否执行此filter
     if (shouldFilter()) {
       Tracer t = TracerFactory.instance().startMicroTracer("ZUUL::" + this.getClass().getSimpleName());
       try {
         Object res = run();
         zr = new ZuulFilterResult(res, ExecutionStatus.SUCCESS);
       } catch (Throwable e) {
         t.setName("ZUUL::" + this.getClass().getSimpleName() + " failed");
         zr = new ZuulFilterResult(ExecutionStatus.FAILED);
         zr.setException(e);
       } finally {
         t.stopAndLog();
       }
     } else {
       zr = new ZuulFilterResult(ExecutionStatus.SKIPPED);
     }
   }
   return zr;
 }
```

Zuul把ZuulFilters存储在类`FilterLoader`属性名为`hashFiltersByType`的`ConcurrentHashMap`中，key为filterType(eg: pre、route、post、error或者自定义)

那么问题来了，这些存在于`FilterLoader`的ZuulFilter是怎么加载进来的呢？



### ZuulFIlter的加载

通过层层搜索，找到类`FilterFileManager` ，在此类初始化的时候，会到指定路径下读取指定文件。同时在初始时，会创建守护线程来定时扫描加载文件。

```java
public class FilterFileManager {
  // 省略代码...

  public static void init(int pollingIntervalSeconds, String... directories) throws Exception, IllegalAccessException, InstantiationException {
    if (INSTANCE == null) INSTANCE = new FilterFileManager();
    INSTANCE.aDirectories = directories;
    // 守护线程的轮询间隔时间
    INSTANCE.pollingIntervalSeconds = pollingIntervalSeconds;
    // 读取并处理文件
    INSTANCE.manageFiles();
    // 开启文件扫描的守护线程
    INSTANCE.startPoller();
  }
  
  void manageFiles() throws Exception, IllegalAccessException, InstantiationException {
        // 读取文件
				List<File> aFiles = getFiles();
    		// 通过FilterLoader来处理文件
        processGroovyFiles(aFiles);
    }
  
  // 扫描指定目录下的指定类型文件
  List<File> getFiles() {
    List<File> list = new ArrayList<File>();
    for (String sDirectory : aDirectories) {
      if (sDirectory != null) {
        File directory = getDirectory(sDirectory);
        // Zuul有自带类`GroovyFileFilter`是扫描 .groovy 文件.
        File[] aFiles = directory.listFiles(FILENAME_FILTER);
        if (aFiles != null) {
          list.addAll(Arrays.asList(aFiles));
        }
      }
    }
    return list;
  }
  
  // 开启守护线程进行轮询
  void startPoller() {
    poller = new Thread("GroovyFilterFileManagerPoller") {
      public void run() {
        while (bRunning) {
          try {
            sleep(pollingIntervalSeconds * 1000);
            manageFiles();
          } catch (Exception e) {
            e.printStackTrace();
          }
        }
      }
    };
    poller.setDaemon(true);
    poller.start();
  }
  
}
```

```java
public class FilterLoader {
  
  // 处理文件
  public boolean putFilter(File file) throws Exception {
    String sName = file.getAbsolutePath() + file.getName();
    // 判断如果文件被修改过，则删除对应已经注册的filter
    if (filterClassLastModified.get(sName) != null && (file.lastModified() != filterClassLastModified.get(sName))) {
      LOG.debug("reloading filter " + sName);
      filterRegistry.remove(sName);
    }
    
    ZuulFilter filter = filterRegistry.get(sName);
    if (filter == null) {
      // 编译文件 - zuul自带GroovyCompiler编译groovy编写的文件
      Class clazz = COMPILER.compile(file);
      // 如果不是抽象类即ZuulFilter，则进行实例化并放入内存
      if (!Modifier.isAbstract(clazz.getModifiers())) {
        filter = (ZuulFilter) FILTER_FACTORY.newInstance(clazz);
        List<ZuulFilter> list = hashFiltersByType.get(filter.filterType());
        if (list != null) {
          hashFiltersByType.remove(filter.filterType()); //rebuild this list
        }
        filterRegistry.put(file.getAbsolutePath() + file.getName(), filter);
        filterClassLastModified.put(sName, file.lastModified());
        return true;
      }
    }
    
    return false;
  }	
}
```

以上两个类核心步骤 即为 `FilterFileManager`初始化过程

+ 扫描指定目录下的groovy文件，通过`FilterLoader`编译文件，并加载ZuulFilter
+ 开启守护进程，轮询文件，动态加载ZuulFilter



## 总结

+  Zuul的源码执行路径：

  ![image-20200724141526966](http://qiniu.5ires.top/uPic/image-20200724141526966.png)

+ ZuulFilter的加载方式：是通过扫描`.groovy`文件来加载，并支持动态加载，具体可以看官方示例zuul-simple-webapp

+ Zuul的整个流程，是基于servlet或filter方式在service或doFilter方法中衔接请求，并运用类似[门面模式](https://zh.wikipedia.org/wiki/%E5%A4%96%E8%A7%80%E6%A8%A1%E5%BC%8F)编写

  

​      至此，Zuulfilter的加载以及各类型Filter的执行都在源码中找到了。zuul-core的代码还是很容易能看懂。下一篇，会分析SpringCloud怎么整合Zuul。



