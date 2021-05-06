## Spring MVC 源码解析

我们都知道, Spring MVC的核心是`DispatcherServlet`。 但是为什么我们会找到这个类, 是如何定位到, 这是SpringMVC的核心处理器? 其内部的处理流程到底是怎样的?

### 1. 离不开Servlet

当我们刚刚接触web编程时, 一定要学习的基础就是Servlet, 这是处理我们web请求的最基本的接口。

```java
public interface Servlet {
    void init(ServletConfig var1) throws ServletException;

    ServletConfig getServletConfig();

    void service(ServletRequest var1, ServletResponse var2) throws ServletException, IOException;

    String getServletInfo();

    void destroy();
}
```

它的方法非常的简单, 启动web容器调用init初始化方法, service方法用于接收, 处理请求, 并且响应结果, 当容器关闭时, 调用destory销毁方法。

>这是最基本的web请求的处理流程了。但是每次处理的请求, 我们都需要在Servlet中根据我们uri的不同, 在service方法中定义对应的业务处理方法, 这是非常麻烦的。而Spring MVC的出现, 就是为了简化uri的处理, 让开发人员更注重于业务代码的编写。

所以我们知道, Servlet就是处理web请求的基础。那这就好办了, 只要实现了Servlet接口, 实现对应的功能满足我们对业务处理的请求即可。

我们可以在spring-webmvc的项目下, 可以找到一个`FrameworkServlet`类, 我们看下这个类图:

![springmvc_frameworkservlet1](/image/springmvc_frameworkservlet1.png)

我们看到, FrameworkServlet的底层接口就是实现了Servlet, 而且还实现了`ApplicationContextAware`, 也就是说`FrameworkServlet`与Spring整合, 获取到当前的BeanFactory。而且, 还可以实现对应的处理web请求的功能。

那么最核心的功能就是`service`方法了。

*FrameworkServlet#service*:

```java
/**
    * Override the parent class implementation in order to intercept PATCH requests.
    */
@Override
protected void service(HttpServletRequest request, HttpServletResponse response)
        throws ServletException, IOException {

    HttpMethod httpMethod = HttpMethod.resolve(request.getMethod());
    if (httpMethod == HttpMethod.PATCH || httpMethod == null) {
        processRequest(request, response);
    }
    else {
        super.service(request, response);
    }
}
```

这段代码非常简单, 它的真正处理逻辑是`processRequest`方法:

```java
/**
* Process this request, publishing an event regardless of the outcome.
* <p>The actual event handling is performed by the abstract
* {@link #doService} template method.
*/
protected final void processRequest(HttpServletRequest request, HttpServletResponse response)
        throws ServletException, IOException {

    ...

    try {
        doService(request, response);
    }
    ...
}

protected abstract void doService(HttpServletRequest request, HttpServletResponse response)
        throws Exception;
```

`processRequest`方法的其他逻辑我们先不管, 但是如果我们debug进来的话, 我们会发现它的核心处理方法就是`doService`, 这是个抽象的方法, 也就是子类需要实现这个方法。而`DispatcherServlet`类正是这个子类, 并且实现了`doService`方法。

*DispatcherServlet#doService*:

```java
/**
* Exposes the DispatcherServlet-specific request attributes and delegates to {@link #doDispatch}
* for the actual dispatching.
*/
@Override
protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {

    // 前面都是对应HttpServletRequest的属性参数处理, 我们暂时不管
    ...

    try {
        doDispatch(request, response);
    }

    ...
}
```

最终我们发现, 执行service方法的正在逻辑, 其实是`DispatcherServlet`中的`doDispatch`方法。

**所以，这也是我们常说的, Spring MVC的底层原理其实还是Servlet, 它离不开Servlet。**

### 2. 深入解析SpringMVC处理流程

我们根据`doDispatch`方法, 基本上就可以知道SpringMVC的处理逻辑。

```java
/**
* Process the actual dispatching to the handler.
* <p>The handler will be obtained by applying the servlet's HandlerMappings in order.
* The HandlerAdapter will be obtained by querying the servlet's installed HandlerAdapters
* to find the first that supports the handler class.
* <p>All HTTP methods are handled by this method. It's up to HandlerAdapters or handlers
* themselves to decide which methods are acceptable.
* @param request current HTTP request
* @param response current HTTP response
* @throws Exception in case of any kind of processing failure
*/
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    HttpServletRequest processedRequest = request;
    HandlerExecutionChain mappedHandler = null;
    boolean multipartRequestParsed = false;

    WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

    ModelAndView mv = null;
    Exception dispatchException = null;

    processedRequest = checkMultipart(request);
    multipartRequestParsed = (processedRequest != request);

    // Determine handler for the current request.
    mappedHandler = getHandler(processedRequest);
    if (mappedHandler == null) {
        noHandlerFound(processedRequest, response);
        return;
    }

    // Determine handler adapter for the current request.
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

    if (!mappedHandler.applyPreHandle(processedRequest, response)) {
        return;
    }

    // Actually invoke the handler.
    mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

    if (asyncManager.isConcurrentHandlingStarted()) {
        return;
    }

    applyDefaultViewName(processedRequest, mv);
    mappedHandler.applyPostHandle(processedRequest, response, mv);

    ....
}
```

上面的代码已经省略了部分代码, 我们只看最核心的部分。

#### 2.1 checkMultipart方法

```java
processedRequest = checkMultipart(request);
```

这个方法, 其实很简单, 就是判断当前请求是否是文件上传请求, 如果是文件上传请求, 就强转成`MultipartHttpServletRequest`返回; 如果不是, 就返回原`HttpServletRequest`。

> 判断这个请求是否是文件上传, 其实也很简单, 只需要判断当前请求的`ContentType`是否以 **"multipart/"**开头即可。

*StandardServletMultipartResolver.java*:

```java
public boolean isMultipart(HttpServletRequest request) {
    return StringUtils.startsWithIgnoreCase(request.getContentType(), "multipart/");
}
```

#### 2.2 getHandler方法

这是SpringMVC中最核心的方法之一。我们从源码中可以知道, 这是根据当前请求, 获取到对应的Handler, 其实也可以理解为我们所说的Controller。

```java
HandlerExecutionChain mappedHandler = getHandler(processedRequest);

/**
* Return the HandlerExecutionChain for this request.
* <p>Tries all handler mappings in order.
* @param request current HTTP request
* @return the HandlerExecutionChain, or {@code null} if no handler could be found
*/
@Nullable
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    if (this.handlerMappings != null) {
        for (HandlerMapping mapping : this.handlerMappings) {
            HandlerExecutionChain handler = mapping.getHandler(request);
            if (handler != null) {
                return handler;
            }
        }
    }
    return null;
}
```

在解析这个方法之前, 我们先思考一个问题?

> 我们在用Spring MVC做一个web项目时, 是如何编写代码的?

1. 先编写一个Controller类, 在此类上加上一个`@Controller`或者`@RestController`注解

2. 在写个方法, 在方法上加上`@RequestMapping(value='/xxx')`注解, value参数就是映射到这个方法的地址;

3. 当我们启动web容器时, 在客户端, 请求对应的url路径, 就可以调用对应的方法。





