## <center>Spring Boot Interceptor学习</center>

**拦截器(Interceptor)**与Filter过滤器一样,它们都是面向切面编程。

### 1. 过滤器和拦截器的区别

> 这里也是面试的重点!!! 有面试官会问, 过滤器和拦截器的区别?

要理解它们的区别最主要是要知道它们在程序中处在什么位置? 这里可以参考这篇文章[过滤器与拦截器的区别](https://www.zhihu.com/question/35225845/answer/61876681)

**过滤器(Filter)**: 当过来一堆东西时, 只希望选择符合的对象。定义这些符合要求的工具。

**拦截器(Interceptor)**: 当一个流程进行时, 能够对它的进展干预, 甚至终止它的进行。

- `Filter`依赖于Servlet容器, 是Servlet的增强版, 所以只能在Servlet容器中实现; 而`Interceptor`是基于Java反射和动态代理的 可以适用于各种java环境。
- `Interceptor`可以在方法前后, 异常前后等调用(原理就是利用java反射, 动态代理技术), 而过滤器只能在请求前和请求后各调用一次。

要注意的是：

**`Filter`的作用只是筛选, 是不能改变对象的状态, 也就是修改对象, 这样就违背了`Filter`的约定; 而`Interceptor`几乎可以对流程做任何事, 所以开发过程中没有特别注意的。**

### 2. 自定义Interceptor

自定义`Interceptor`必须实现`HandlerInterceptor`接口或者继承`HandlerInterceptorAdapter`类, 并且要重写下面3个方法:

```java
public boolean preHandle(HttpServletRequest request,
                         HttpServletResponse response,
                         Object handler)


public void postHandle(HttpServletRequest request,
                       HttpServletResponse response,
                       Object handler,
                       ModelAndView modelAndView)


public void afterCompletion(HttpServletRequest request,
                            HttpServletResponse response,
                            Object handler,
                            Exception ex)
```

其中: `preHandle`方法返回true或false。如果返回true, 那么请求将继续到达Controller被处理。

![interceptor](/develop_framework/SpringBoot/img/interceptor.png)

`LogInterceptor`用于拦截请求, 打印日志:

```java
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import org.springframework.web.servlet.ModelAndView;
import org.springframework.web.servlet.handler.HandlerInterceptorAdapter;

public class LogInterceptor extends HandlerInterceptorAdapter {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception {
        long startTime = System.currentTimeMillis();
        System.out.println("\n-------- LogInterception.preHandle --- ");
        System.out.println("Request URL: " + request.getRequestURL());
        System.out.println("Start Time: " + System.currentTimeMillis());

        request.setAttribute("startTime", startTime);

        return true;
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, //
                           Object handler, ModelAndView modelAndView) throws Exception {

        System.out.println("\n-------- LogInterception.postHandle --- ");
        System.out.println("Request URL: " + request.getRequestURL());

        // You can add attributes in the modelAndView
        // and use that in the view page
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, //
                                Object handler, Exception ex) throws Exception {
        System.out.println("\n-------- LogInterception.afterCompletion --- ");

        long startTime = (Long) request.getAttribute("startTime");
        long endTime = System.currentTimeMillis();
        System.out.println("Request URL: " + request.getRequestURL());
        System.out.println("End Time: " + endTime);

        System.out.println("Time Taken: " + (endTime - startTime));
    }

}
```

`OldLoginInterceptor`是一个拦截器, 如果用户输入已经被废弃的链接"/admin/oldLogin", 它将重定向到新的url。

```java
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import org.springframework.web.servlet.ModelAndView;
import org.springframework.web.servlet.handler.HandlerInterceptorAdapter;

public class OldLoginInterceptor extends HandlerInterceptorAdapter {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception {

        System.out.println("\n-------- OldLoginInterceptor.preHandle --- ");
        System.out.println("Request URL: " + request.getRequestURL());
        System.out.println("Sorry! This URL is no longer used, Redirect to /admin/login");

        response.sendRedirect(request.getContextPath() + "/admin/login");
        return false;
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, //
                           Object handler, ModelAndView modelAndView) throws Exception {

        // This code will never be run.
        System.out.println("\n-------- OldLoginInterceptor.postHandle --- ");
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, //
                                Object handler, Exception ex) throws Exception {

        // This code will never be run.
        System.out.println("\n-------- QueryStringInterceptor.afterCompletion --- ");
    }

}
```

`AdminInterceptor`:

```java
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import org.springframework.web.servlet.ModelAndView;
import org.springframework.web.servlet.handler.HandlerInterceptorAdapter;

public class AdminInterceptor extends HandlerInterceptorAdapter {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception {

        System.out.println("\n-------- AdminInterceptor.preHandle --- ");
        return true;
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, //
            Object handler, ModelAndView modelAndView) throws Exception {

        System.out.println("\n-------- AdminInterceptor.postHandle --- ");
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, //
            Object handler, Exception ex) throws Exception {

        System.out.println("\n-------- AdminInterceptor.afterCompletion --- ");
    }

}
```

![new-old-LoginInterceptor](/develop_framework/SpringBoot/img/new-old-LoginInterceptor.png)

![new-old-LoginInterceptor2](/develop_framework/SpringBoot/img/new-old-LoginInterceptor2.png)

**配置拦截器**:

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        // LogInterceptor apply to all URLs.
        registry.addInterceptor(new LogInterceptor());

        // Old Login url, no longer use.
        // Use OldURLInterceptor to redirect to a new URL.
        registry.addInterceptor(new OldLoginInterceptor())//
                .addPathPatterns("/admin/oldLogin");

        // This interceptor apply to URL like /admin/*
        // Exclude /admin/oldLogin
        registry.addInterceptor(new AdminInterceptor())//
                .addPathPatterns("/admin/*")//
                .excludePathPatterns("/admin/oldLogin");
    }

}
```