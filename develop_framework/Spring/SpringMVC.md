## <center>Spring MVC工作原理详解 </center>

如果用到Spring, 那就不得不提Spring MVC。这两个框架是我们开发学习路程中必不可少的, 即使现在SpringBoot大火, 但仍然是在这两个框架的基础上进行开发的。之前我们已经学习了Spring的核心知识, 所以学习SpringMVC也是紧接着Spring框架。

> 对于Spring MVC与Struts2 两个框架的区别以及选择

其实从现在技术的趋势来看, Spring MVC的使用程度已经远远超过了Struts2。如果一个公司还在使用Struts2框架, 那只能说这个公司的技术栈已经很落后了, 如果是我那是肯定不会去的。再加上SpringBoot的大火, 就更加体现出SpringMVC是更符合开发者的。
所以, 我建议现在还在使用Struts2的开发者, 可以去了解, 可以去学习, 但是没必要深入学习, 更没必要用在实际的项目中。

至于这两者的区别, 其实网上已经有很多的文章博客了, 大部分都是相同的观点。这里就贴出一篇博客。[SpringMVC与Struts2区别比较总结](https://blog.csdn.net/chenleixing/article/details/44570681)

### 1. MVC 模式

MVC是一种设计模式, 它与三层架构是两个概念。

MVC: 
- **Model(M)**: 模型, 应用程序中处理数据逻辑的部分
- **View(V)**: 视图, 程序中处理数据显示的部分。
- **Controller(C)**: 控制器, 处理用户交互的部分。通常是从视图中读取数据, 控制用户输入, 并向模型发送数据, 最终有可能在返回视图。

MVC原理图:

![MVC模式原理图](/develop_framework/Spring/img/MVC模式图.jpg)

> 面试可能会问, 三层架构与MVC之间的关系?

对于三层架构, 我们通常是指: 控制层(Web层), 业务逻辑层(Service), 数据访问层(Dao)。这是一种分层开发的思想, 主要是为了减轻对象之间的耦合, 减少代码之间的依赖性。

MVC模式是项目的整体结构, 将后端代码和前台展示代码进行分离。

下图比较清晰的展示了MVC与三层架构的关系:

![MVC与三层架构的关系图](/develop_framework/Spring/img/MVC与三层架构的关系图.png)

这里面有一点要注意就是这个Model——模型。在这里我的理解是, Model模型是整个项目的基础, 几乎存在于每个部分。所以不管是Controller, View, 还是Service, Dao都属于Model所表达的部分。

View视图中对数据的定义是根据pojo或者其他类型的实体, Controller控制层中接收数据也是根据实体, Dao数据访问层更不用说, pojo就是根据数据库字段定义的, 而Service业务逻辑层, 也需要依据Model中的实体来进行业务处理。

### 2.Spring MVC核心原理

通过上面对MVC模式的简单了解, SpringMVC框架就是在此基础上设计的。
SpringMVC框架是以请求为驱动, 围绕着Servlet设计, 将请求发给控制器, 然后通过模型对象, 分派器来展示请求结果视图。其中核心类是`DispatcherServlet`, 它是一个Servlet, 顶层是实现的Servlet接口。

> 谈谈SpringMVC的工作原理? (这里应该是面试官经常提问并且比较难的部分, 这个问题的重视程度与Spring IOC&DI, 以及Spring AOP相当)

SpringMVC的核心是`DispatcherServlet`前端控制器。具体的流程如下:

![SpringMVC请求流程原理图](/develop_framework/Spring/img/SpringMVC请求流程原理图.png)

(1) `DispatcherServlet`接收客户端的请求, 根据请求信息调用处理映射器`HandlerMapping`, 解析并返回对应的Handler处理器;

(2) 通过返回的Handler处理器, `DispatcherServlet`再去请求处理适配器`HandlerAdapter`, 

(3) `HandlerAdapter`处理适配器根据`Handler`处理器, 执行真正的处理器也就是我们所说的Controller, 并处理相应的业务逻辑;

(4) 处理器Controller处理完成后, 会返回一个`ModelAndView`对象, Model是返回的数据对象, View是逻辑上的视图;

(5) `ModelAndView`通过返回处理适配器`HandlerAdapter`和前端控制器`DispatcherServlet`, 再传入视图解析器`ViewResolver`, 解析`ModelAndView`对象, 查找并返回实际的View视图(可以理解为页面所在的路径等信息);

(6) `DispatcherServlet`把返回的Model传给View, 进行视图渲染, 将模型数据填充到request域中;

(7) 把View通过response响应给客户端。

### 3. Spring MVC组件

- **前端控制器`DispatcherServlet`**

> 这里也是面试高频点, 谈谈你对DispatcherServlet的理解?

这是SpringMVC的核心组件。**用于接收请求, 响应结果, 相当于转发器, 中央处理器。`DispatcherServlet`控制其他组件, 统一调度, 降低了其他组件之间的耦合性, 提高了每个组件的扩展性。**

对源码的解析也许在面试中并不需要, 但是在学习过程中是必须要知道的。

```java
package org.springframework.web.servlet;

@SuppressWarnings("serial")
public class DispatcherServlet extends FrameworkServlet {

    public static final String MULTIPART_RESOLVER_BEAN_NAME = "multipartResolver";
    public static final String LOCALE_RESOLVER_BEAN_NAME = "localeResolver";
    public static final String THEME_RESOLVER_BEAN_NAME = "themeResolver";
    public static final String HANDLER_MAPPING_BEAN_NAME = "handlerMapping";
    public static final String HANDLER_ADAPTER_BEAN_NAME = "handlerAdapter";
    public static final String HANDLER_EXCEPTION_RESOLVER_BEAN_NAME = "handlerExceptionResolver";
    public static final String REQUEST_TO_VIEW_NAME_TRANSLATOR_BEAN_NAME = "viewNameTranslator";
    public static final String VIEW_RESOLVER_BEAN_NAME = "viewResolver";
    public static final String FLASH_MAP_MANAGER_BEAN_NAME = "flashMapManager";
    public static final String WEB_APPLICATION_CONTEXT_ATTRIBUTE = DispatcherServlet.class.getName() + ".CONTEXT";
    public static final String LOCALE_RESOLVER_ATTRIBUTE = DispatcherServlet.class.getName() + ".LOCALE_RESOLVER";
    public static final String THEME_RESOLVER_ATTRIBUTE = DispatcherServlet.class.getName() + ".THEME_RESOLVER";
    public static final String THEME_SOURCE_ATTRIBUTE = DispatcherServlet.class.getName() + ".THEME_SOURCE";
    public static final String INPUT_FLASH_MAP_ATTRIBUTE = DispatcherServlet.class.getName() + ".INPUT_FLASH_MAP";
    public static final String OUTPUT_FLASH_MAP_ATTRIBUTE = DispatcherServlet.class.getName() + ".OUTPUT_FLASH_MAP";
    public static final String FLASH_MAP_MANAGER_ATTRIBUTE = DispatcherServlet.class.getName() + ".FLASH_MAP_MANAGER";
    public static final String EXCEPTION_ATTRIBUTE = DispatcherServlet.class.getName() + ".EXCEPTION";
    public static final String PAGE_NOT_FOUND_LOG_CATEGORY = "org.springframework.web.servlet.PageNotFound";
    private static final String DEFAULT_STRATEGIES_PATH = "DispatcherServlet.properties";
    protected static final Log pageNotFoundLogger = LogFactory.getLog(PAGE_NOT_FOUND_LOG_CATEGORY);
    private static final Properties defaultStrategies;
    static {
        try {
            ClassPathResource resource = new ClassPathResource(DEFAULT_STRATEGIES_PATH, DispatcherServlet.class);
            defaultStrategies = PropertiesLoaderUtils.loadProperties(resource);
        }
        catch (IOException ex) {
            throw new IllegalStateException("Could not load 'DispatcherServlet.properties': " + ex.getMessage());
        }
    }

    /** Detect all HandlerMappings or just expect "handlerMapping" bean? */
    private boolean detectAllHandlerMappings = true;

    /** Detect all HandlerAdapters or just expect "handlerAdapter" bean? */
    private boolean detectAllHandlerAdapters = true;

    /** Detect all HandlerExceptionResolvers or just expect "handlerExceptionResolver" bean? */
    private boolean detectAllHandlerExceptionResolvers = true;

    /** Detect all ViewResolvers or just expect "viewResolver" bean? */
    private boolean detectAllViewResolvers = true;

    /** Throw a NoHandlerFoundException if no Handler was found to process this request? **/
    private boolean throwExceptionIfNoHandlerFound = false;

    /** Perform cleanup of request attributes after include request? */
    private boolean cleanupAfterInclude = true;

    /** MultipartResolver used by this servlet */
    private MultipartResolver multipartResolver;

    /** LocaleResolver used by this servlet */
    private LocaleResolver localeResolver;

    /** ThemeResolver used by this servlet */
    private ThemeResolver themeResolver;

    /** List of HandlerMappings used by this servlet */
    private List<HandlerMapping> handlerMappings;

    /** List of HandlerAdapters used by this servlet */
    private List<HandlerAdapter> handlerAdapters;

    /** List of HandlerExceptionResolvers used by this servlet */
    private List<HandlerExceptionResolver> handlerExceptionResolvers;

    /** RequestToViewNameTranslator used by this servlet */
    private RequestToViewNameTranslator viewNameTranslator;

    private FlashMapManager flashMapManager;

    /** List of ViewResolvers used by this servlet */
    private List<ViewResolver> viewResolvers;

    public DispatcherServlet() {
        super();
    }

    public DispatcherServlet(WebApplicationContext webApplicationContext) {
        super(webApplicationContext);
    }
    @Override
    protected void onRefresh(ApplicationContext context) {
        initStrategies(context);
    }

    protected void initStrategies(ApplicationContext context) {
        initMultipartResolver(context);
        initLocaleResolver(context);
        initThemeResolver(context);
        initHandlerMappings(context);
        initHandlerAdapters(context);
        initHandlerExceptionResolvers(context);
        initRequestToViewNameTranslator(context);
        initViewResolvers(context);
        initFlashMapManager(context);
    }
}
```
上面是`DispatcherServlet`的源码, 其中比较重要的属性Beans:
    - `HandlerMapping`: 用于handlers映射请求和一系列对于拦截器的处理。大部分都用`@Controller`注解。
    - `HandlerApapter`: 帮助`DispatcherServlet`处理映射请求处理器`Handler`的适配器。一般我们是不需要考虑这部分。
    - `ViewResolver`: 根据实际配置解析实际的视图`View`类型。
    - `ThemeResolver`: 解决Web应用程序的主题, 比如提供个性化布局。
    - `MultipartResolver`: 解析多部分请求, 用来支持从HTML表单上传的文件。

在Web MVC框架中, 每个`DispatcherServlet`都有自己的`WebApplicationContext`, 它继承了`ApplicationContext`。所以`WebApplicationContext`包含其上下文和Servlet实例之间共享的所有基础框架的`Beans`信息。

- **处理映射器`HandlerMapping`**

`HandlerMapping`根据请求的URL查找到对应的Handler。SpringMVC提供了不同的映射器, 用于实现不同种类的映射方法。例如: 配置文件方式, 实现接口方式, 注解方式等。

HandlerMapping接口处理请求的映射实现类:
    - `SimpleUrlHandlerMapping`类通过配置文件把Url映射到Controller类中。
    - `DefaultAnnotationMapping`类通过注解吧Url映射到Controller类中。

- **处理适配器`HandlerAdapter`**

通过`HandlerAdapter`适配器的规则去执行Handler。这是适配器模式的应用, 通过扩展适配器可以对更多类型的处理器进行执行。

`AnnotationMethodHandlerAdapter`: 通过注解吧请求Url映射到Controller类的方法上。

- **处理器`Handler`**

因为处理器`Handler`涉及到具体的用户业务请求, 所以是需要开发者自行开发的, 而且编写`Handler`时要按照`HandlerAdapter`的要求去做, 这样适配器才可以去正确执行`Handler`。在前端控制器`DispatcherServlet`的控制下, `Handler`对具体的用户请求进行处理。

- **视图解析器`View Resolver`**

`View Resolver`负责将处理结果`ModelAndView`先根据逻辑视图名解析成具体实现视图名即具体的页面地址, 再生称View视图对象, 最后对View进行渲染将处理结果通过页面展示给用户。
SpringMVC框架提供很多的View视图类型, 包括: jstl, freemarker, pdf等。

`UrlBasedViewResolver`类 通过配置文件，把一个视图名交给到一个View来处理。

- **视图`View`**

View是一个接口, 实现类支持不同的`View`类型(jsp, freemarker, pdf...)

**需要注意的时, 处理器`Handler`(也就是我们开发中的Controller控制器)以及视图层`View`都是需要开发者自己开发的。而其他的一些组件都是框架提供的。**

### 4. SpringMVC与Servlet的关系

