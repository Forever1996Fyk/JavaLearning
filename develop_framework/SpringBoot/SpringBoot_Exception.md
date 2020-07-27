## <center>Spring Boot 异常处理</center>

> 如何处理全局异常?

一般面试是不会问这种问题的, 但是我们也需要知道如何去处理, 以防万一。而且这对我们平时开发项目也有一定的帮助。

### 1. 使用`@ControllerAdvice`和`@ExceptionHandler`处理全局异常(常用)

这是一种目前常用的一种方式。所以一般处理全局异常的步骤为:

1. **新建异常信息实体类**

这是非必要的类, 主要用于包装异常信息。

```java
public class ErrorResponse {

    private String message;
    private String errorTypeName;

    public ErrorResponse(Exception e) {
        this(e.getClass().getName(), e.getMessage());
    }

    public ErrorResponse(String errorTypeName, String message) {
        this.errorTypeName = errorTypeName;
        this.message = message;
    }
    ......省略getter/setter方法
}
```

2. **自定义异常类**

一般我们处理的都是运行时异常, 所以自定义异常类需要继承`RuntimeException`。

```java
public class ResourceNotFoundException extends RuntimeException {
    private String message;

    public ResourceNotFoundException() {
        super();
    }

    public ResourceNotFoundException(String message) {
        super(message);
        this.message = message;
    }

    @Override
    public String getMessage() {
        return message;
    }

    public void setMessage(String message) {
        this.message = message;
    }
}
```

3. **新建异常处理类**

我们只需要在类上加上`@ControllerAdvice`注解, 该类就成为了全局异常处理类, 你也可以通过`assignableTypes`指定特定的`Controller`类, 让异常处理类只处理特定类抛出的异常

```java
@ControllerAdvice(assignableTypes = {ExceptionController.class})
@ResponseBody
public class GlobalExceptionHandler {

    ErrorResponse illegalArgumentResponse = new ErrorResponse(new IllegalArgumentException("参数错误!"));
    ErrorResponse resourseNotFoundResponse = new ErrorResponse(new ResourceNotFoundException("Sorry, the resourse not found!"));

    @ExceptionHandler(value = Exception.class)// 拦截所有异常, 这里只是为了演示，一般情况下一个方法特定处理一种异常
    public ResponseEntity<ErrorResponse> exceptionHandler(Exception e) {

        if (e instanceof IllegalArgumentException) {
            return ResponseEntity.status(400).body(illegalArgumentResponse);
        } else if (e instanceof ResourceNotFoundException) {
            return ResponseEntity.status(404).body(resourseNotFoundResponse);
        }
        return null;
    }
}
```

### 2. 使用@ExceptionHandler处理Controller级别异常

在实际开发中我们基本上都是用第一种方式处理异常了, 所以这种方式已经用的比较少了, 所以这里就不详细说明。

### 3. 使用ResponseStatusException

`ResponseStatusException`类会处理异常返回的信息, 将异常映射为状态码。

```java
@GetMapping("/resourceNotFoundException")
public void throwException3() {
    throw new ResponseStatusException(HttpStatus.NOT_FOUND, "Sorry, the resourse not found!", new ResourceNotFoundException());
}
```

返回结果如下:

```json
{
    "timestamp": "2019-08-21T07:28:12.017+0000",
    "status": 404,
    "error": "Not Found",
    "message": "Sorry, the resourse not found!",
    "path": "/api/resourceNotFoundException"
}
```

`ResponseStatusException`提供了三个构造方法:

```java
public ResponseStatusException(HttpStatus status) {
    this(status, null, null);
}

public ResponseStatusException(HttpStatus status, @Nullable String reason) {
    this(status, reason, null);
}

public ResponseStatusException(HttpStatus status, @Nullable String reason, @Nullable Throwable cause) {
    super(null, cause);
    Assert.notNull(status, "HttpStatus is required");
    this.status = status;
    this.reason = reason;
}
```