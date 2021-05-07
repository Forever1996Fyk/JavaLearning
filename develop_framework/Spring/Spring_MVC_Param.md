## Spring MVC 参数解析原理

根据前篇文章[Spring MVC源码解析](develop_framework/Spring/Spring_MVC_Source.md)的分析, 我们应该了解了SpringMVC整体处理请求流程的原理。

但是对于SpringMVC的学习还没有结束, 包括SpringMVC的参数解析。

我们都知道SpringMVC有多种类型的参数解析方式, 我们平常在项目中常用的就是注解形式的:

- @RequestParam

- @PathVariable

- @RequestBody

但是SpringMVC是如何根据这些注解来解析对应的参数。比如: 为什么`@RequestBody`可以绑定对应的实体, `@RequestParam`既可以绑定单个参数, 也可以绑定map?
这些都是需要去探讨的问题。

### 1. 参数解析

我们知道, 如果我们使用@Controller注解来声明一个Controller类时, 

SpringMVC使用的`AbstractHandlerMethodAdapter`适配器来调用对应的handle方法, 并且有它的子类`RequestMappingHandlerAdapter` 执行内部的handleInternal方法。源码如下;

`RequestMappingHandlerAdapter.java`:

```java
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
```

其中最核心的调用方法就是`invokeHandlerMethod`,

```java
@Nullable
protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
        HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

    ServletWebRequest webRequest = new ServletWebRequest(request, response);
    try {
        WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
        ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);

        ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
        if (this.argumentResolvers != null) {
            invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
        }
        if (this.returnValueHandlers != null) {
            invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
        }
        invocableMethod.setDataBinderFactory(binderFactory);
        invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);

        ...

        // 真正的执行逻辑
        invocableMethod.invokeAndHandle(webRequest, mavContainer);

        return getModelAndView(mavContainer, modelFactory, webRequest);
    }
    finally {
        webRequest.requestCompleted();
    }
}
```

我们可以看到`invokeHandlerMethod`中有一个`argumentResolvers`参数解析器。

参数解析器`HandlerMethodArgumentResolver`是一个接口, 它有很多个实现类, 每一个实现类其实都对应的一个参数类型的解析:

![springmvc_paramresolver1](/image/springmvc_paramresolver1.png)

我们从名字上就可以判断出, 每个参数解析器对应哪种类型的参数, 包括`@RequestParam`, `@PathVariable`等等。

#### 1.1 什么时候注入这些参数解析器?

其实很简单, 我们可以看到`RequestMappingHandlerAdapter`它实现了InitializingBean接口, 也就是说, 当Bean初始化结束时, Spring容器就会调用到`RequestMappingHandlerAdapter`的`afterPropertiesSet`方法, 我们看其中的源码:

```java
@Override
public void afterPropertiesSet() {

    ...

    if (this.argumentResolvers == null) {
        List<HandlerMethodArgumentResolver> resolvers = getDefaultArgumentResolvers();
        this.argumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
    }

    ...

}

/**
* Return the list of argument resolvers to use including built-in resolvers
* and custom resolvers provided via {@link #setCustomArgumentResolvers}.
*/
private List<HandlerMethodArgumentResolver> getDefaultArgumentResolvers() {
    List<HandlerMethodArgumentResolver> resolvers = new ArrayList<>(30);

    // Annotation-based argument resolution
    resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), false));
    resolvers.add(new RequestParamMapMethodArgumentResolver());
    resolvers.add(new PathVariableMethodArgumentResolver());
    resolvers.add(new PathVariableMapMethodArgumentResolver());
    resolvers.add(new MatrixVariableMethodArgumentResolver());
    resolvers.add(new MatrixVariableMapMethodArgumentResolver());
    resolvers.add(new ServletModelAttributeMethodProcessor(false));
    resolvers.add(new RequestResponseBodyMethodProcessor(getMessageConverters(), this.requestResponseBodyAdvice));
    resolvers.add(new RequestPartMethodArgumentResolver(getMessageConverters(), this.requestResponseBodyAdvice));
    resolvers.add(new RequestHeaderMethodArgumentResolver(getBeanFactory()));
    resolvers.add(new RequestHeaderMapMethodArgumentResolver());
    resolvers.add(new ServletCookieValueMethodArgumentResolver(getBeanFactory()));
    resolvers.add(new ExpressionValueMethodArgumentResolver(getBeanFactory()));
    resolvers.add(new SessionAttributeMethodArgumentResolver());
    resolvers.add(new RequestAttributeMethodArgumentResolver());

    // Type-based argument resolution
    resolvers.add(new ServletRequestMethodArgumentResolver());
    resolvers.add(new ServletResponseMethodArgumentResolver());
    resolvers.add(new HttpEntityMethodProcessor(getMessageConverters(), this.requestResponseBodyAdvice));
    resolvers.add(new RedirectAttributesMethodArgumentResolver());
    resolvers.add(new ModelMethodProcessor());
    resolvers.add(new MapMethodProcessor());
    resolvers.add(new ErrorsMethodArgumentResolver());
    resolvers.add(new SessionStatusMethodArgumentResolver());
    resolvers.add(new UriComponentsBuilderMethodArgumentResolver());

    // Custom arguments
    if (getCustomArgumentResolvers() != null) {
        resolvers.addAll(getCustomArgumentResolvers());
    }

    // Catch-all
    resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), true));
    resolvers.add(new ServletModelAttributeMethodProcessor(true));

    return resolvers;
}
```

可以看到, 这个方法直接获取到所有的参数解析器, 并放入集合缓存当中, 所以, 当Bean初始化结束的时候, 容器就会调用这个方法, 将所有的参数解析器加入List缓存, 以便后面的判断。 

#### 1.2 问题2: 这么多参数解析器, SpringMVC到底要使用哪一个,难道if else每个判断么?


我们不断深入`invocableMethod.invokeAndHandle`方法, 会找到`InvocableHandlerMethod.getMethodArgumentValues`,

```java

public class InvocableHandlerMethod extends HandlerMethod {

    /**
	 * Get the method argument values for the current request, checking the provided
	 * argument values and falling back to the configured argument resolvers.
	 * <p>The resulting array will be passed into {@link #doInvoke}.
	 * @since 5.1.2
	 */
    protected Object[] getMethodArgumentValues(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {

		MethodParameter[] parameters = getMethodParameters();
		if (ObjectUtils.isEmpty(parameters)) {
			return EMPTY_ARGS;
		}

		Object[] args = new Object[parameters.length];
		for (int i = 0; i < parameters.length; i++) {
			MethodParameter parameter = parameters[i];
			parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
			args[i] = findProvidedArgument(parameter, providedArgs);
			if (args[i] != null) {
				continue;
			}
			if (!this.resolvers.supportsParameter(parameter)) {
				throw new IllegalStateException(formatArgumentError(parameter, "No suitable resolver"));
			}
			try {
				args[i] = this.resolvers.resolveArgument(parameter, mavContainer, request, this.dataBinderFactory);
			}
			catch (Exception ex) {
				// Leave stack trace for later, exception may actually be resolved and handled...
				if (logger.isDebugEnabled()) {
					String exMsg = ex.getMessage();
					if (exMsg != null && !exMsg.contains(parameter.getExecutable().toGenericString())) {
						logger.debug(formatArgumentError(parameter, exMsg));
					}
				}
				throw ex;
			}
		}
		return args;
	}

}

```

我们跟踪代码, 可以看到有一个判断逻辑`this.resolvers.supportsParameter(parameter)`, 我们在深入这个方法, 最终定位到`HandlerMethodArgumentResolverComposite.getArgumentResolver`方法, 源码如下:

```java
public class HandlerMethodArgumentResolverComposite implements HandlerMethodArgumentResolver {

    private HandlerMethodArgumentResolver getArgumentResolver(MethodParameter parameter) {
		HandlerMethodArgumentResolver result = this.argumentResolverCache.get(parameter);
        if (result == null) {
            for (HandlerMethodArgumentResolver resolver : this.argumentResolvers) {
                if (resolver.supportsParameter(parameter)) {
                    result = resolver;
                    this.argumentResolverCache.put(parameter, result);
                    break;
                }
            }
        }
        return result;
    }

}

```

我们可以看到, 这个方法先根据这个参数信息从缓存中看能不能获取到当前的解析结果, 如果获取不到就遍历参数解析器的List集合, 调用每个实现类的`supportsParameter`方法, 来判断当前参数解析是否符合这个解析器, 如果符合还会把当前参数的信息放进一个Map缓存, 然后直接break, 返回。

有了这种缓存机制, 每次SpringMVC可能第一次请求会比较慢, 但是之后的请求只要参数信息不变, 就会直接从缓存中获取结果。

#### 1.3 开始解析参数

经过上面`getMethodArgumentValues`的参数解析判断, 可以判断出当前的请求参数是否存在不符合解析条件的, 如果不符合就直接抛出异常:

```java
if (!this.resolvers.supportsParameter(parameter)) {
    throw new IllegalStateException(formatArgumentError(parameter, "No suitable resolver"));
}
```

如果都符合, 就开始参数解析:

```java
args[i] = this.resolvers.resolveArgument(parameter, mavContainer, request, this.dataBinderFactory);
```

其中的原理, 也是跟上面一样, 方法参数信息, 获取到会员的参数解析器, 利用对应参数解析器的`resolveArgument`方法, 解析不同的参数。

```java
public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
        NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {

    HandlerMethodArgumentResolver resolver = getArgumentResolver(parameter);
    if (resolver == null) {
        throw new IllegalArgumentException("Unsupported parameter type [" +
                parameter.getParameterType().getName() + "]. supportsParameter should be called first.");
    }
    return resolver.resolveArgument(parameter, mavContainer, webRequest, binderFactory);
}
```

> 具体的实现流程, 就没必要在继续下去了。我觉得了解参数解析的最重要的就是腰知道其中的设计思想, 比如SpringMVC对于参数解析器的处理, 非常巧妙的用到了SpringIOC的`InitializingBean`, 然后利用**策略模式**根据不同的参数信息,获取到不同的解析器, 从而解析参数。

