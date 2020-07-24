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

> 什么是`Spring Boot starter Parent`?

创建Spring Boot项目, 默认都是有parent。这个spring-boot-starter-parent的作用是:

- 定义Java的编译版本为1.8
- 使用UTF-8编码
- 继承spring-boot-dependencies, 其中定义了依赖的版本, 所以我们在用SpringBoot相关依赖时不需要写版本号
- 执行打包操作的配置
- 自动化资源过滤
- 自动化的插件配置

### 2. 什么是JavaConfig?

Spring JavaConfig提供了Spring IOC容器的纯java方法, 基本上替代了Spring使用XML配置。使用JavaConfig的优点在于:

- **面向对象的配置。** 由于配置被定义为JavaConfig类, 所以可以充分利用Java中的面向对象特性。一个配置类可以继承另一个, 重写@Bean方法等。
- **减少或消除XML配置**。

> 这里就引出来一个问题, XML配置有什么弊端?

1. XML配置学习的门槛较高, 而且XML文件不能做类型检查, 有时候你把一个类错误装配, 导致系统报错, 但是却很难发现。
2. XML配置的可读性和可维护性较低。
3. 至于`MyBatis`框架把SQL放入XML中, 我个人觉得如果是较为复杂的业务需要些比较复杂的SQL, 为了把SQL语句写的更加清晰, 所以才用XML, 否则的话用注解更加方便。

Spring JavaConfig的用法: 

- 用`@Configuration`注解标注的JavaConfig类;
- 用`@Bean`注解标注的方法, 标识Spring需要加载的Bean实例

### 3. 为什么要用YAML配置(application.yml)?

- 配置有序, 在一些特殊场景下, 配置有序很关键。
- 支持数组, 数组中的元素可以是基本数据类型也可以是对象
- 简洁, 没有JSON格式的{}, []等要求。

### 4. application.yml与bootstrap.yml的区别?

- bootstrap(.yml或者.properties): bootstrap是由父`ApplicationContext`加载的, 比application优先加载, 而且bootstrap里面的属性不能被覆盖;
- application(.yml或者.properties): 由`ApplicaitonContext`加载, 用于SpringBoot项目的自动化配置。

### 5. 什么事Spring Profiles?

Spring Profiles允许用户根据配置文件(dev, test, prod等)来注册Bean。所以, 当程序开发中运行时, 可以让某些Bean加载, 而在生产环境下, 让其他Bean加载。

### 6. Spring Boot中如何解决跨域问题?

跨域可以在前端通过`JSONP`来解决, 但是`JSONP`只能发送GET请求, 所以还是在后端通过(`CORS`, `Cross-origin resource sharing`)来解决跨域问题。

### 7. 什么是CSRF攻击?

`CSRF`代表跨站请求伪造。可以这么理解: <font color="red">攻击者盗用了你的认证身份, 以你的认证身份发送恶意请求</font>。

完成一次`CSRF`攻击, 受害者必须一次完成两个步骤:

1. 登录受信任网站A, 并在本地上午生成Cookie;
2. 在不登出A的情况下, 访问危险网站B。

> 例如通过银行网站A, 它以GET或者POST请求来完成转账操作, 有一个危险网站B, 它里面存在隐藏的请求银行网站A的接口机制, 你如果访问了危险网站B, 就会请求接口进行转账操作。

**<font color="red">CSRF攻击是源于WEB的隐式身份验证机制!!! 这个机制可以保证一个请求是来自某个用户的浏览器, 但是无法保证这个请求是用户批准发送的!!!</font>**

具体的原理可以参考这篇文章: [浅谈CSRF攻击方式](https://www.cnblogs.com/hyddd/archive/2009/04/09/1432744.html)
### 8. 如何防御CSRF攻击?

可以从服务端和客户端两方面着手, 但是从服务端着手效果比较好。

**(1) Cookie Hashing(所有提交的表单都包含同一个伪随机值)**: 

    这是最简单的解决方案, 因为攻击者不能获得第三方的Cookie。在表单里增加Hash值, 来认证这是用户发送的请求, 然后在服务端进行Hash值验证。

**(2) 验证码**:

    每次用户提交都需要用户在表单中填写一个图片上的随机字符串, 这个可以完全解决CSRF。

**(3) One-Time Tokens(不同表单包含一个不同的伪随机值)**:

    这个方法跟(1)是一样的, 只是如果打开不同的表单, 所包含的都是一个新的随机值, 服务端对每次提交的表单随机值进行验证。

### 9. 运行Spring Boot有哪几种方式?

- 打包用命令或者放到容器中运行
- 用Maven/Gradle插件运行
- 直接执行main方式运行

