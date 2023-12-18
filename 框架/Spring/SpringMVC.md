## SpringMVC

### MVC

**Model**：数据模型，提供要展示的数据，因此包含数据和行为，可以认为是领域模型或JavaBean组件（包含数据和行为），不过现在一般都分离开来：Value Object（数据Dao） 和 服务层（行为Service）。也就是模型提供了模型数据查询和模型数据的状态更新等功能，包括数据和业务。

**View**：负责进行模型的展示，一般就是我们见到的用户界面，客户想看到的东西。

**Controller**（调度员）：接收用户请求，委托给模型进行处理（状态改变），处理完毕后把返回的模型数据返回给视图，由视图负责展示。

最常用的MVC：（Model）Bean +（view） Jsp +（Controller） Servlet

### 原理

#### 流程

1.  客户端（浏览器）发送请求，直接请求到   DispatcherServlet 。
2.  DispatcherServlet 根据请求信息调⽤   HandlerMapping ，解析请求对应的   Handler 。
3.  解析到对应的   Handler （也就是我们平常说的   Controller 控制器）后，开始由HandlerAdapter 适配器处理。
4.  HandlerAdapter 会根据   Handler 来调⽤真正的处理器开处理请求，并处理相应的业务逻辑。
5.  处理器处理完业务后，会返回⼀个   ModelAndView 对象， Model 是返回的数据对象， View是个逻辑上的   View 。
6.  ViewResolver 会根据逻辑   View 查找实际的   View 。
7.  DispaterServlet 把返回的   Model 传给   View （视图渲染）。

#### 代码

SpringMVC启动时通过 `org.springframework.web.servlet.DispatcherServlet#onRefresh`方法初始化ViewResolvers、HandlerMappings、HandlerAdapters等。

```java
protected void onRefresh(ApplicationContext context) {
    initStrategies(context);
}

protected void initStrategies(ApplicationContext context) {
    initMultipartResolver(context);
    initLocaleResolver(context);
    initThemeResolver(context);
    initHandlerMappings(context);
    initHandlerAdapters(context);
    initHandlerExceptionResolvers(context);
    initRequestToViewNameTranslator(context);
    initViewResolvers(context);
    initFlashMapManager(context);
}
```

初始化完成之后， 通过 `org.springframework.web.servlet.DispatcherServlet#doService`方法启动，`doService()`方法将具体实现交由 `doDispatch()`方法实现

```java
protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
    ......
    try {
        doDispatch(request, response);
    }
    ......
}

protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    HttpServletRequest processedRequest = request;
    HandlerExecutionChain mappedHandler = null;
    boolean multipartRequestParsed = false;

    WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

    try {
        ModelAndView mv = null;
        Exception dispatchException = null;

        try {
            //检查是否为Multipart请求
            processedRequest = checkMultipart(request);
            multipartRequestParsed = (processedRequest != request);

            // Determine handler for the current request.
            //从前面初始化的handler中获取对应的handler
            mappedHandler = getHandler(processedRequest);
            if (mappedHandler == null) {
                //无法找到对应handler，返回错误
                noHandlerFound(processedRequest, response);
                return;
            }

            // Determine handler adapter for the current request.
            //根据handler获取adapter
            HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

            // Process last-modified header, if supported by the handler.
            String method = request.getMethod();
            boolean isGet = "GET".equals(method);
            if (isGet || "HEAD".equals(method)) {
                long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
                if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
                    return;
                }
            }
			//若响应已被拦截器（HandlerInterceptor）处理，直接返回
            if (!mappedHandler.applyPreHandle(processedRequest, response)) {
                return;
            }

            // Actually invoke the handler.
            //调用adapter返回ModelAndView
            mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

            if (asyncManager.isConcurrentHandlingStarted()) {
                return;
            }
			//根据配置的prefix和suffix获取真实的ViewName
            applyDefaultViewName(processedRequest, mv);
            //执行HandlerInterceptor方法
            mappedHandler.applyPostHandle(processedRequest, response, mv);
        }
        catch (Exception ex) {
            dispatchException = ex;
        }
        catch (Throwable err) {
            // As of 4.3, we're processing Errors thrown from handler methods as well,
            // making them available for @ExceptionHandler methods and other scenarios.
            dispatchException = new NestedServletException("Handler dispatch failed", err);
        }
        //通过ViewResolver渲染ModelAndView，并返回
        processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
    }
    catch (Exception ex) {
        triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
    }
    catch (Throwable err) {
        triggerAfterCompletion(processedRequest, response, mappedHandler,
                               new NestedServletException("Handler processing failed", err));
    }
    finally {
        if (asyncManager.isConcurrentHandlingStarted()) {
            // Instead of postHandle and afterCompletion
            if (mappedHandler != null) {
                mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
            }
        }
        else {
            // Clean up any resources used by a multipart request.
            if (multipartRequestParsed) {
                cleanupMultipart(processedRequest);
            }
        }
    }
}
```

### 拦截器

- 数据独立性：Servlet中的是过滤器，而拦截器是SpringMVC框架独有的，独享request和response
- 拦截器只会拦截访问的控制器方法，如果访问的是jsp/html/css等式不会拦截的

- 拦截器是基于AOP思想的，和AOP实现是一样的，在application.xml中配置

#### 自定义拦截器

自定义拦截器需要实现 `org.springframework.web.servlet.HandlerInterceptor`接口

```java
public interface HandlerInterceptor {
	//在实际的处理程序运行之前
    //此方法返回一个布尔值。您可以使用此方法来中断或继续执行链的处理。当此方法返回时true，处理程序执行链继续。当它返回false时，DispatcherServlet 假定拦截器本身已经处理了请求（例如，渲染了一个适当的视图），并且不会继续执行其他拦截器和执行链中的实际处理程序。
	default boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {

		return true;
	}
	//处理程序运行后
	default void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler,
			@Nullable ModelAndView modelAndView) throws Exception {
	}
	//完整请求完成后
	default void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler,
			@Nullable Exception ex) throws Exception {
	}
}
```

实现 `HandlerInterceptor`:

```java
public class MyInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        //return true：执行下一个拦截器
        System.out.println("===========处理前,这里进行拦截处理=================");
        return true;
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        System.out.println("===========处理后，通常进行日志管理=================");
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        System.out.println("===========清洁中=================");
    }
}
```

配置拦截器：

代码方式

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LocaleChangeInterceptor());
        registry.addInterceptor(new ThemeChangeInterceptor()).addPathPatterns("/**").excludePathPatterns("/admin/**");
        registry.addInterceptor(new SecurityInterceptor()).addPathPatterns("/secure/*");
    }
}
```

xml方式

```xml
<!--拦截器配置-->
<mvc:interceptors>
    <bean class="org.springframework.web.servlet.i18n.LocaleChangeInterceptor"/>
    <mvc:interceptor>
        <mvc:mapping path="/**"/>
        <mvc:exclude-mapping path="/admin/**"/>
        <bean class="org.springframework.web.servlet.theme.ThemeChangeInterceptor"/>
    </mvc:interceptor>
    <mvc:interceptor>
        <mvc:mapping path="/secure/*"/>
        <bean class="org.example.SecurityInterceptor"/>
    </mvc:interceptor>
</mvc:interceptors>
```

