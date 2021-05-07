## Spring Boot Filter学习

`Filter`这个概念应该是很熟悉的东西, 特别是在学习Servlet的时候, 就知道了对请求的过滤。

**这也是Filter过滤器的用途, 用来过滤用户请求的, 它允许对用请求进行前置处理和后置处理。比如实现URL级别的权限控制, 过滤非法请求。**

Filter过滤器是面向切面编程————AOP的具体实现。

如果需要自定义Filter的话, 只需要实现`javax.Servlet.Filter`接口, 然后重写里面的3个方法即可!

`Filter.java`

```java
public interface Filter {

   //初始化过滤器后执行的操作
    default void init(FilterConfig filterConfig) throws ServletException {
    }
   // 对请求进行过滤
    void doFilter(ServletRequest var1, ServletResponse var2, FilterChain var3) throws IOException, ServletException;
   // 销毁过滤器后执行的操作，主要用户对某些资源的回收
    default void destroy() {
    }
}
```

### 1. Filter如何实现拦截?

> 面试题: 说一下过滤器的原理? 到底是怎么拦截的?

`Filter`接口有一个`doFilter`方法, 这个方法实现了对用户请求的过滤。流程如下:

1. 用户发送请求到web服务器, 请求会先到过滤器;
2. 过滤器会对请求进行一些处理。比如过滤请求的参数, 修改返回给客户端的response的内容, 判断是否该用户是否有权限访问该接口;
3. 用户请求响应结束;
4. 进行自己的其他操作。

![SpringBoot_filter](/image/filter.png)

### 2. 自定义Filter

有两种方法可以自定义Filter: (1) 实现`javax.Servlet.Filter`接口, 并重写方法; (2) 在过滤器类上加上`@WebFilter`注解, 并提供对应的参数(例如filterName, urlPatterns)

#### 2.1 实现javax.Servlet.Filter接口

`MyFilter.java`:

```java
@Component
public class MyFilter implements Filter {
    private static final Logger logger = LoggerFactory.getLogger(MyFilter.class);

    @Override
    public void init(FilterConfig filterConfig) {
        logger.info("初始化过滤器：", filterConfig.getFilterName());
    }

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        //对请求进行预处理
        logger.info("过滤器开始对请求进行预处理：");
        HttpServletRequest request = (HttpServletRequest) servletRequest;
        String requestUri = request.getRequestURI();
        System.out.println("请求的接口为：" + requestUri);
        long startTime = System.currentTimeMillis();
        //通过 doFilter 方法实现过滤功能
        filterChain.doFilter(servletRequest, servletResponse);
        // 上面的 doFilter 方法执行结束后用户的请求已经返回
        long endTime = System.currentTimeMillis();
        System.out.println("该用户的请求已经处理完毕，请求花费的时间为：" + (endTime - startTime));
    }

    @Override
    public void destroy() {
        logger.info("销毁过滤器");
    }
}
```

`MyFilterConfig.java`: **在配置类中注册自定义的过滤器**

```java
@Configuration
public class MyFilterConfig {
    @Autowired
    MyFilter myFilter;
    @Bean
    public FilterRegistrationBean<MyFilter> thirdFilter() {
        FilterRegistrationBean<MyFilter> filterRegistrationBean = new FilterRegistrationBean<>();

        filterRegistrationBean.setFilter(myFilter);

        filterRegistrationBean.setUrlPatterns(new ArrayList<>(Arrays.asList("/api/*")));

        return filterRegistrationBean;
    }
}
```

#### 2.2 通过@WebFilter注解实现自定义Filter

```java
@WebFilter(filterName = "MyFilterWithAnnotation", urlPatterns = "/api/*")
public class MyFilterWithAnnotation implements Filter {

   ......
}
```

这里要注意, 为了让Spring能够在加载的时候把这个类加载为Bean, 需要在启动类上加上`@ServletComponentScan`注解。

### 3. 定义多个过滤器的执行顺序

> 如果此时我们有两个过滤器, 怎样决定这两个过滤器的顺序?

**在配置类中注册自定义的过滤器, 通过`FilterRegistrationBean`的`setOrder`方法可以决定过滤器的执行顺序**。

```java
@Configuration
public class MyFilterConfig {
    @Autowired
    MyFilter myFilter;

    @Autowired
    MyFilter2 myFilter2;

    @Bean
    public FilterRegistrationBean<MyFilter> setUpMyFilter() {
        FilterRegistrationBean<MyFilter> filterRegistrationBean = new FilterRegistrationBean<>();
        filterRegistrationBean.setOrder(2);
        filterRegistrationBean.setFilter(myFilter);
        filterRegistrationBean.setUrlPatterns(new ArrayList<>(Arrays.asList("/api/*")));

        return filterRegistrationBean;
    }

    @Bean
    public FilterRegistrationBean<MyFilter2> setUpMyFilter2() {
        FilterRegistrationBean<MyFilter2> filterRegistrationBean = new FilterRegistrationBean<>();
        filterRegistrationBean.setOrder(1);
        filterRegistrationBean.setFilter(myFilter2);
        filterRegistrationBean.setUrlPatterns(new ArrayList<>(Arrays.asList("/api/*")));
        return filterRegistrationBean;
    }
}
```
