## <center>Dubbo原理解析</center>

### 1. Dubbo框架设计

对于Dubbo的框架设计以及解释, 其实官网上已经有了解释 ———— [官方Dubbo架构整体设计](http://dubbo.apache.org/zh-cn/docs/dev/design.html)

> 对于官网介绍的架构设计思想主要是两点

1. 采用URL作为配置信息的统一格式, 所有扩展点都通过传递URL携带配置信息
    
    **各系统之间的参数传递基于URL来携带配置信息, 所有的参数都封装成Dubbo自定义的URL对象进行传递。**

2. Dubbo的所有功能点都可以被用户自定义扩展所替换

    Dubbo的设计是**模块之间基于接口编程, 模块之间不对实现类进行硬编码。这段话的意思就是说, 一旦代码里涉及到具体的实现类, 如果需要替换一种实现, 就需要修改代码, 这样设计是非常糟糕的**  所以Dubbo针对这一问题提供了自己的SPI解决方案, 并重新命名为`ExtensionLoader`(扩展点机制), 按照用户配置来指定加载模块, 只需要约定路径即可(**在放置扩展点配置文件`META-INF/dubbo/接口全限定名`, 内容为: 配置名=扩展实现类全限定名, 多个实现类用换行符分隔**)。 

Java也有SPI机制, 可以做到服务发现和动态扩展, 但是弊端就是初始化就要把所有实现类给加载进去。

- SPI(Service Provide Interface): 服务提供接口, 是专门给扩展者用的。
- API(Application Programming Interface): 应用程序接口, 是给使用者用的。

### 2. Dubbo启动解析, 加载配置信息

[Dubbo_ExtensionLoader](/distributed/RPC/Dubbo/dubbo_extensionLoader.md)

### 3. Dubbo服务暴露

[Dubbo_Service_Exposed](/distributed/RPC/Dubbo/dubbo_service_exposed.md)

### 4. Dubbo服务引用

[Dubbo_Service_Reference](/distributed/RPC/Dubbo/dubbo_service_reference.md)

### 5. Dubbo "微内核+插件"机制

[Dubbo_Microkernel_Plugin](/distributed/RPC/Dubbo/dubbo_microkernel_plugin.md)

