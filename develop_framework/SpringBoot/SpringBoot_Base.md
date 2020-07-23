## <center>Spring Boot 基础</center>

这篇文章主要介绍或者解决一些SpringBoot相关的基础问题, 不涉及代码。

### 1. 什么是Spring Boot Starters?

> 面试的时候可能会问这个问题, 你既然用过SpringBoot, 那你简单说下Spring Boot Starters是什么?

`Spring Boot Starters`是一系列依赖关系的集合。例如: 在没有`Spring Boot Starters`之前, 我们开发REST服务或者Web应用程序需要手动添加Spring MVC, Tomcat和Jackson这样的依赖库, 但是有了`Spring Boot Starts`我们只需要添加这一个依赖就可以, 因为它包含了我们开发REST服务需要的所有依赖。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

### 2. 什么是JavaConfig?

Spring JavaConfig提供了Spring IOC容器的纯java方法, 基本上替代了Spring使用XML配置。使用JavaConfig的优点在于:

- **面向对象的配置。** 由于配置被定义为JavaConfig类, 所以可以充分利用Java中的面向对象特性。一个配置类可以继承另一个, 重写@Bean方法等。
- **减少或消除XML配置**。

> 这里就引出来一个问题, XML配置有什么弊端?

1. XML配置学习的门槛较高, 而且XML文件不能做类型检查, 有时候你把一个类错误装配, 导致系统报错, 但是却很难发现。
2. XML配置的可读性和可维护性较低。
3. 至于`MyBatis`框架把SQL放入XML中, 我个人觉得是为了能够把SQL语句写的更加清晰

Spring JavaConfig的用法: 

- 用`@Configuration`注解标注的JavaConfig类;
- 用`@Bean`注解标注的方法, 标识Spring需要加载的Bean实例

