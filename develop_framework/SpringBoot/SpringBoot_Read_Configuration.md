## <center>Spring Boot 读取配置文件 </center>

之前我们知道Spring Boot通过自动配置简化了Spring开发流程, 所以我们需要知道SpringBoot自动配置的原理以及读取配置文件的方式。首先我们要了解YAML的基本语法。

```yaml

name: Michaelkai

my-profile:
  name: SpringBoot配置
  email: xxxx@aliyun.com

user:
  location: 安徽合肥
  props:
    - name: test1
      age: 25
    - name: test2
      age: 30
```

### 1. 通过`@Value`读取比较简单的配置信息

使用`@Value(${property})`读取比较简单的配置信息。

```java
@Value("${name}")
String name;
```

> 需要注意的是 @value这种方式是不被推荐的，Spring 比较建议的是下面几种读取配置信息的方式。

### 2. 通过`@ConfigurationProperties`读取并与Bean绑定

> `UserProperties`类加上`@Component`注解, 就能够像普通bean一样注入类中.

```java
@Component
@ConfigurationProperties(prefix = "user")
@Setter
@Getter
@ToString
class LibraryProperties {
    private String location;
    private List<Prop> props;

    @Setter
    @Getter
    @ToString
    static class Prop {
        String name;
        Integer age;
    }
}

```

### 3. `@PropertySource`读取指定properties文件

这里要注意一点使用`@PropertySource`只能读取`.properties`文件, `.yml`文件是无法读取的

```java
@Component
@PropertySource("classpath:website.properties")
@Getter
@Setter
class WebSite {
    @Value("${url}")
    private String url;
}
```

### 4. Spring Boot加载配置文件的优先级

如下图:

![Spring Boot加载配置文件的优先级](JavaLearning\develop_framework\SpringBoot\img\SpringBootReadConfiurationOrder.jpg)
