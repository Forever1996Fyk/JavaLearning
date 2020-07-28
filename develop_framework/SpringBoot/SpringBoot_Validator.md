## <center>Spring Boot 参数校验</center>

数据的校验是前后端都需要进行的操作, **我们一般要对传入后端的数据再一次进行校验, 避免用户绕过浏览器直接通过一些HTTP工具直接向后端请求一些违法数据**。

我们现在学习Spring Boot, 只需要引入`spring-boot-starter-web`依赖即可。

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```