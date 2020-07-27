## <center>Spring Boot Filter学习</center>

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
2. 过滤器会对请求00进行一些处理。比如过滤请求的参数, 修改返回给客户端的response的内容, 判断是否该用户是否有权限访问该接口;
3. 用户请求响应结束;
4. 进行自己的其他操作。

![SpringBoot_filter](/develop_framework/SpringBoot/SpringBoot_Filter.md)