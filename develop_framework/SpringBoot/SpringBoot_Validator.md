## Spring Boot 参数校验

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
### 1. 参数校验详情

实体类:

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Person {

    @NotNull(message = "classId 不能为空")
    private String classId;

    @Size(max = 33)
    @NotNull(message = "name 不能为空")
    private String name;

    @Pattern(regexp = "((^Man$|^Woman$|^UGM$))", message = "sex 值不在可选范围")
    @NotNull(message = "sex 不能为空")
    private String sex;

    @Email(message = "email 格式不正确")
    @NotNull(message = "email 不能为空")
    private String email;

}
```

> 正则表达式说明:
 ```txt
    - ^string : 匹配以 string 开头的字符串
    - string$ ：匹配以 string 结尾的字符串
    - ^string$ ：精确匹配 string 字符串
    - ((^Man$|^Woman$|^UGM$)) : 值只能在 Man,Woman,UGM 这三个值中选择
 ```

 **JSR的校验注解:**

 - `@Null`被注释的元素必须为null
 - `@NotNull`被注释的元素必须不能为null
 - `@AssertTrue`被注释的元素必须为true
 - `@AssertFalse`被注释的元素必须为false
 - `@Min(value)`被注释的元素必须是一个数字, 其值必须大于等于指定的最小值
 - `@Max(value)`被注释的元素必须是一个数字, 其值必须小于等于指定的最大值
 - `@DecimalMin(value)`被注释的元素必须是一个数字, 其值必须大于等于指定的最小值
 - `@DecimalMax(value)`被注释的元素必须是一个数字, 其值必须小于等于指定的最大值
 - `@Size(max=, min=)`被注释的元素的大小必须在指定的范围内
 - `@Digits(integer, fraction)`被注释的元素必须是一个数字, 其值必须在可接受的范围内
 - `@Past`被注释的元素必须是一个过去的日期
 - `@Future`被注释的元素必须是一个将来的日期
 - `@Pattern(regex=, flag=)`被注释的元素必须符合指定的正则表达式

 **Hibernate Validator提供的校验注解:**

 - `@NotBlank(message =)`验证字符串非null, 且长度必须大于0
 - `@Email`被注释的元素必须是电子邮箱地址
 - `@Length(min=,max=)`被注释的字符串的大小必须在指定的范围内
 - `@NotEmpty`被注释的字符串必须非空
 - `@Range(min=, max=, message=)`被注释的元素必须在合适的范围内

### 2. 参数校验相关用法

#### 2.1 验证Controller输入

需要在验证的参数上加上`@Valid`注解, 如果验证失败会抛出`MethodArgumentNotValidException`。默认情况下, Spring将此异常转换为HTTP Status 400(请求错误)

```java
@RestController
@RequestMapping("/api")
public class PersonController {

    @PostMapping("/person")
    public ResponseEntity<Person> getPerson(@RequestBody @Valid Person person) {
        return ResponseEntity.ok().body(person);
    }
}
```

`ExceptionHandler`:

自定义异常处理器, 可以捕获异常进行处理。

```java
@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, String>> handleValidationExceptions(
            MethodArgumentNotValidException ex) {

        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getAllErrors().forEach((error) -> {
            String fieldName = ((FieldError) error).getField();
            String errorMessage = error.getDefaultMessage();
            errors.put(fieldName, errorMessage);
        });
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(errors);

    }
}
```

#### 2.2 验证请求参数(PathVariables和Request Parameters)

使用这种方式, 必须要在类上加上`@Validated`注解。

```java
@RestController
@RequestMapping("/api")
@Validated
public class PersonController {

    @GetMapping("/person/{id}")
    public ResponseEntity<Integer> getPersonByID(@Valid @PathVariable("id") @Max(value = 5,message = "超过 id 的范围了") Integer id) {
        return ResponseEntity.ok().body(id);
    }

    @PutMapping("/person")
    public ResponseEntity<String> getPersonByName(@Valid @RequestParam("name") @Size(max = 6,message = "超过 name 的范围了") String name) {
        return ResponseEntity.ok().body(name);
    }
}
```

### 3. 自定义Validator

如果自带的参数校验注解无法满足需求, 还可以自定义实现注解。

第一步需要创建一个注解:

```java
@Target({FIELD})
@Retention(RUNTIME)
@Constraint(validatedBy = RegionValidator.class)
@Documented
public @interface Region {

    String message() default "Region 值不在可选范围内";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};
}
```

第二步需要实现`ConstraintValidator`接口, 并重写`isValid`方法:

```java
public class RegionValidator implements ConstraintValidator<Region, String> {

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        HashSet<Object> regions = new HashSet<>();
        regions.add("China");
        regions.add("China-Taiwan");
        regions.add("China-HongKong");
        return regions.contains(value);
    }
}
```

使用自定义注解:

```java
@Region
private String region;
```

### 4. `@NotNull`与`@Column(nullable = false)`区别?(重要)

在使用JPA操作数据时, 会碰到`@Column(nullable = false)`这种类型约束, 但是这与`@NotNull`是完全不同的两种东西。

- `@NotNull`: 是JSR Bean验证的注解, 它与数据库约束无关
- `@Column(nullable = false)`: 是JPA声明该数据表字段非空

也就是说`@NotNull`用于验证, 而`@Column(nullable = false)`用于指示数据库创建表时对表字段的约束

