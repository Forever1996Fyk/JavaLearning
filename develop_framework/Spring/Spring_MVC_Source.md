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

### 2. Spring MVC初始化Controller流程

我们先思考一个问题。我们在用Spring MVC做一个web项目时, 是如何编写代码的?

> 1. 先编写一个Controller类, 在此类上加上一个`@Controller`或者`@RestController`注解
> 2. 在写个方法, 在方法上加上`@RequestMapping(value='/xxx')`注解, value参数就是映射到这个方法的地址;
> 3. 当我们启动web容器时, 在客户端, 请求对应的url路径, 就可以调用对应的方法。


我们应该很直接的想到一个方法, **Map**。

我们可以定义一个map, 这个map的key就是这个uri, value我们就可以用这个类或者方法的信息。

当请求过来时, 我们解析这个请求获取到uri, 然后根据uri获取对应的方法, 在通过反射, 调用这个方法。

这也是Spring MVC的做法。

**Spring MVC在项目启动的时候, 会扫描项目包下所有的Controller类, 获取到所有带有`@RequestMapping`注解的value也就是对应的uri的值, 然后在获取到对应的类和方法信息, 最后把uri作为key, 类和方法作为value存入map中。**

> **至于SpringMVC初始化的时机, 我们可以找到类`AbstractHandlerMethodMapping`, 它实现了`InitializingBean`的接口, 这个是在Spring Bean在初始化完成后, 会调用所有实现这个接口的方法`afterPropertiesSet()`, 所以我们可以在这个类中, 找到初始化的具体实现**

当然了, 这里只是简化了Spring MVC的步骤, 实际的实现SpringMVC封装了一层`HandlerMethod`类, 把对应的方法类的信息, 都放在`HandlerMethod`中。源码如下:

`AbstractHandlerMethodMapping.java`:

```java
public abstract class AbstractHandlerMethodMapping<T> extends AbstractHandlerMapping implements InitializingBean {
	@Override
	public void afterPropertiesSet() {
		initHandlerMethods();
	}

	protected void initHandlerMethods() {
		for (String beanName : getCandidateBeanNames()) {
			if (!beanName.startsWith(SCOPED_TARGET_NAME_PREFIX)) {
				processCandidateBean(beanName);
			}
		}
		handlerMethodsInitialized(getHandlerMethods());
	}

    public Map<T, HandlerMethod> getHandlerMethods() {
		this.mappingRegistry.acquireReadLock();
		try {
			return Collections.unmodifiableMap(this.mappingRegistry.getMappings());
		}
		finally {
			this.mappingRegistry.releaseReadLock();
		}
	}

    class MappingRegistry {

		private final Map<T, MappingRegistration<T>> registry = new HashMap<>();

		private final Map<T, HandlerMethod> mappingLookup = new LinkedHashMap<>();

        public Map<T, HandlerMethod> getMappings() {
			return this.mappingLookup;
		}

        ...
    }

    @Override
	protected HandlerMethod getHandlerInternal(HttpServletRequest request) throws Exception {
		String lookupPath = getUrlPathHelper().getLookupPathForRequest(request);
		request.setAttribute(LOOKUP_PATH, lookupPath);
		this.mappingRegistry.acquireReadLock();
		try {
			HandlerMethod handlerMethod = lookupHandlerMethod(lookupPath, request);
			return (handlerMethod != null ? handlerMethod.createWithResolvedBean() : null);
		}
		finally {
			this.mappingRegistry.releaseReadLock();
		}
	}
}
```

### 3. 声明Controller类


我们再提一点小知识。声明一个Controller类有哪些方式。（现在应该没有人用第二/三种方式吧）

1. 使用注解@Controller 和 请求路径@RequestMapping

2. 实现 Controller 接口 并将该类交给spring容器管理beanName为请求路径

3. 实现 HttpRequestHandler 接口并将该类交给spring容器管理beanName为请求路径

而我们的第二种和第三种的方式基本没有用了，因为会出现类爆炸，就像原始的servlet一样每一个方法都需要写一个类。

这两种方式是通过beanName为路径来实例化对象并执行通过该对象来执行里面的方法的。源码中这两种map的填充方式是在bean的生命周期中通过实现beanFactory的applyBeanPostProcessorsBeforeInitialization方法来填充的。

**正是因为有不同的声明Controller类, 所以SpringMVC使用适配器, 来适配不同的Handler类型。**

### 4. 深入解析SpringMVC处理流程

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


#### 4.1 checkMultipart方法

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

#### 4.2 getHandler方法

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

根据我们上面分析SpringMVC的初始化Controller流程, 再来看getHandler方法, 就知道原理了。我们不看源码, 应该都能猜到其中流程。

> 大致流程 先解析`HttpServletRequest`, 获取到uri, 在根据uri在Map中, 获取到handler。当然了, SpringMVC对这个流程更复杂, 也更严谨。

SpringMVC封装了一个`HandlerExecutionChain`对象

```java
public class HandlerExecutionChain {

    ...

	public HandlerExecutionChain(Object handler, @Nullable HandlerInterceptor... interceptors) {
		if (handler instanceof HandlerExecutionChain) {
			HandlerExecutionChain originalChain = (HandlerExecutionChain) handler;
			this.handler = originalChain.getHandler();
			this.interceptorList = new ArrayList<>();
			CollectionUtils.mergeArrayIntoCollection(originalChain.getInterceptors(), this.interceptorList);
			CollectionUtils.mergeArrayIntoCollection(interceptors, this.interceptorList);
		}
		else {
			this.handler = handler;
			this.interceptors = interceptors;
		}
	}

    ...
}
```

这个类是非常重要的, 它的构造时机, 就是在`getHandler`的时候封装的。具体实现就是在`HandlerMapping`接口的实现类, `AbstractHandlerMapping`。

源码如下:

```java
public abstract class AbstractHandlerMapping extends WebApplicationObjectSupport
		implements HandlerMapping, Ordered, BeanNameAware {

        ...

    public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
		Object handler = getHandlerInternal(request);
		if (handler == null) {
			handler = getDefaultHandler();
		}
		if (handler == null) {
			return null;
		}
		// Bean name or resolved handler?
		if (handler instanceof String) {
			String handlerName = (String) handler;
			handler = obtainApplicationContext().getBean(handlerName);
		}

		HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);

		if (logger.isTraceEnabled()) {
			logger.trace("Mapped to " + handler);
		}
		else if (logger.isDebugEnabled() && !request.getDispatcherType().equals(DispatcherType.ASYNC)) {
			logger.debug("Mapped to " + executionChain.getHandler());
		}

		if (hasCorsConfigurationSource(handler) || CorsUtils.isPreFlightRequest(request)) {
			CorsConfiguration config = (this.corsConfigurationSource != null ? this.corsConfigurationSource.getCorsConfiguration(request) : null);
			CorsConfiguration handlerConfig = getCorsConfiguration(handler, request);
			config = (config != null ? config.combine(handlerConfig) : handlerConfig);
			executionChain = getCorsHandlerExecutionChain(request, executionChain, config);
		}

		return executionChain;
	}

    ...

    protected abstract Object getHandlerInternal(HttpServletRequest request) throws Exception;

}


```

根据上面的源码可以看出, `getHandlerInternal`才是真正根据HttpServletRequest获取Handler的方法, 这是一个抽象方法, 由子类实现目的就是为了根据uri获取对应的Handler对象。


#### 4.3 getHandlerAdapter方法


```java
/**
* Return the HandlerAdapter for this handler object.
* @param handler the handler object to find an adapter for
* @throws ServletException if no HandlerAdapter can be found for the handler. This is a fatal error.
*/
protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
    if (this.handlerAdapters != null) {
        for (HandlerAdapter adapter : this.handlerAdapters) {
            if (adapter.supports(handler)) {
                return adapter;
            }
        }
    }
    throw new ServletException("No adapter for handler [" + handler +
            "]: The DispatcherServlet configuration needs to include a HandlerAdapter that supports this handler");
}
```

之前我们提到过, 声明Controller类的方式有三种, 最早的两种其实是实现了`Controller`和`HttpRequestHandler`接口, 通过实现其中的方法, 来实现具体的业务需求。

还有一种是加上`@Controller`注解, 通过反射的方式来调用具体的方法。

为了能够适配这三种方法调用, SpringMVC采用了适配器模式。根据handler的实现类型不同, 调用不同的实现方法。

比如, Spring MVC中封装了`HandlerAdapter`接口, 每个实现了这个接口的实现类, 都对应一种类型的Handler或者说是Controller。

![springmvc_handleradapter1](/image/springmvc_handleradapter1.png)

- `SimpleControllerHandlerAdapter`对应的就是, 实现了Controller接口的方式;

- `HttpRequestHandlerAdapter`对应的就是实现了HttpRequestHandler接口的方式;

- `RequestMappingHandlerAdapter`继承了`AbstractHandlerMethodAdapter`类, 这才是真正使用`@controller`注解的方式。

所以getHandlerAdapter就是根据当前的Handler获取到对应的适配器, 然后执行对应的handle方法。如下：

```java
mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
```

每一个适配器, 都对应这一个实现当前Handler的方法, 比如`SimpleControllerHandlerAdapter`和`HttpRequestHandlerAdapter`, 直接强转成对应的接口类型, 然后调用其实现方法即可。

```java
public class SimpleControllerHandlerAdapter implements HandlerAdapter {

	@Override
	@Nullable
	public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {

		return ((Controller) handler).handleRequest(request, response);
	}
}

```

```java
public class HttpRequestHandlerAdapter implements HandlerAdapter {

	@Override
	@Nullable
	public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {

		((HttpRequestHandler) handler).handleRequest(request, response);
		return null;
	}
}
```

```java
public abstract class AbstractHandlerMethodAdapter extends WebContentGenerator implements HandlerAdapter, Ordered {

    @Override
	@Nullable
	public final ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {

		return handleInternal(request, response, (HandlerMethod) handler);
	}
}
```

```java
public class RequestMappingHandlerAdapter extends AbstractHandlerMethodAdapter
		implements BeanFactoryAware, InitializingBean {

    @Override
	protected ModelAndView handleInternal(HttpServletRequest request,
			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

		ModelAndView mav;
		checkRequest(request);

		// Execute invokeHandlerMethod in synchronized block if required.
		if (this.synchronizeOnSession) {
			HttpSession session = request.getSession(false);
			if (session != null) {
				Object mutex = WebUtils.getSessionMutex(session);
				synchronized (mutex) {
					mav = invokeHandlerMethod(request, response, handlerMethod);
				}
			}
			else {
				// No HttpSession available -> no mutex necessary
				mav = invokeHandlerMethod(request, response, handlerMethod);
			}
		}
		else {
			// No synchronization on session demanded at all...
			mav = invokeHandlerMethod(request, response, handlerMethod);
		}

		if (!response.containsHeader(HEADER_CACHE_CONTROL)) {
			if (getSessionAttributesHandler(handlerMethod).hasSessionAttributes()) {
				applyCacheSeconds(response, this.cacheSecondsForSessionAttributeHandlers);
			}
			else {
				prepareResponse(response);
			}
		}

		return mav;
	}
}
```

### 5. 总结

上面的是SpringMVC中最核心的`DispatcherServlet`的原理, 其实看完之后, 也就明白了为什么说`DispatcherServlet`是SpringMVC的核心处理器。

看完这篇文章, 再回到之前[SpringMVC基本工作原理](develop_framework/Spring/SpringMVC.md)这篇文章, 对于SpringMVC执行流程应该会有一个更加深刻的认识。







