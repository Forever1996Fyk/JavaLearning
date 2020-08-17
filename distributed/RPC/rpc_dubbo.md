## <center>Dubbo深入学习</center>

SOA分布式架构的流行, 是Dubbo诞生的关键。SOA面向服务的架构(Service Oriented Architecture), 就是把项目按照业务逻辑拆分为服务层, 表现层两个工程。服务层包含业务逻辑, 只要对外提供服务即可。表现层只需要处理和页面的交互, 业务逻辑都是调用服务层的服务来实现。SOA架构主要两个角色: 服务提供者(Provider)和服务使用者(Consumer)。

### 1. 为什么要用Dubbo?

- **负载均衡**: 同一个服务部署在不同的机器该调用哪一台机器上的服务。
- **服务调用链路生成**: 随着系统的发展, 服务越来越多, 服务间依赖关系变得错综复杂, Dubbo可以解决服务之间相互是如何调用的。
- **服务访问压力以及时长统计, 资源调度和治理**: 基于访问压力实时管理集群容量, 提高集群利用率。
- **服务降级**: 某个服务挂掉之后调用服务。

Dubbo也可以运用在微服务系统中, 只不过由于Spring Cloud在微服务应用的广泛, 所以Dubbo大部分是非微服务的分布式系统下使用的RPC框架。

### 2. Dubbo架构

![Dubbo架构图](/distributed/RPC/img/Dubbo架构图.jpg)

调用关系:

1. 服务容器负责启动, 加载, 运行服务提供者。
2. 服务提供者在启动时, 向注册中心注册自己提供的服务。
3. 服务消费者在启动时, 向注册中心订阅自己所需的服务。
4. 注册中心返回服务提供者地址列表给消费者, 如果有变化, 注册中心将基于长连接推送最新数据给消费者。
5. 服务消费者, 从提供者列表中, 基于软负载均衡算法, 选择一台提供者进行调用, 如果调用失败, 再选另一台调用。
6. 服务消费者和提供者, 在内存中累计调用次数和调用时间, 定时每分钟发送一次统计数据到监控中心。

**注意点:**

- 注册中心, 服务提供者, 服务消费者三者之间均为长连接, 监控中心除外。
- 服务提供者全部宕机, 服务消费者应用将无法使用, 并无限次重连等待服务提供者恢复。
- 注册中心和监控中心全部宕机, 不影响已运行的提供者和消费者, 消费者在本地缓存了提供者列表。
- 注册中心通过长连接感知服务提供者的存在, 服务提供者宕机, 注册中心将立即推送事件通知消费者。

### 3. Dubbo原理

#### 3.1 Dubbo框架设计

对于Dubbo的框架设计以及解释, 其实官网上已经有了解释 ———— [官方Dubbo架构整体设计](http://dubbo.apache.org/zh-cn/docs/dev/design.html)

> 对于官网介绍的架构设计思想主要是两点

1. 采用URL作为配置信息的统一格式, 所有扩展点都通过传递URL携带配置信息

**各系统之间的参数传递基于URL来携带配置信息, 所有的参数都封装成Dubbo自定义的URL对象进行传递。**

2. Dubbo的所有功能点都可以被用户自定义扩展所替换

Dubbo的设计是**模块之间基于接口编程, 模块之间不对实现类进行硬编码。这段话的意思就是说, 一旦代码里涉及到具体的实现类, 如果需要替换一种实现, 就需要修改代码, 这样设计是非常糟糕的**  所以Dubbo针对这一问题提供了自己的SPI解决方案, 并重新命名为`ExtensionLoader`(扩展点机制), 按照用户配置来指定加载模块, 只需要约定路径即可(**在放置扩展点配置文件`META-INF/dubbo/接口全限定名`, 内容为: 配置名=扩展实现类全限定名, 多个实现类用换行符分隔**)。 

Java也有SPI机制, 可以做到服务发现和动态扩展, 但是弊端就是初始化就要把所有实现类给加载进去。

- SPI(Service Provide Interface): 服务提供接口, 是专门给扩展者用的。
- API(Application Programming Interface): 应用程序接口, 是给使用者用的。

#### 3.2 Dubbo启动解析, 加载配置信息

`ExtensionLoader`是Dubbo实现加载配置信息的核心。`ExtensionLoader`改进了Java ServiceLoader的问题:

- JDK的SPI会一次性实例化扩展点所有实现, 没用上也加载, 如果有扩展实现初始化很耗时, 会浪费资源。
- 如果扩展点加载失败, 连扩展点的名称都无法获取。
- 增加了对扩展点IOC和AOP的支持, 一个扩展点可以直接setter注入其他扩展点。

以`LoadBalance`为例, 文件`com.alibaba.dubbo.rpc.cluster.LoadBalance`中内容为:

```java
random=com.alibaba.dubbo.rpc.cluster.loadbalance.RandomLoadBalance
roundrobin=com.alibaba.dubbo.rpc.cluster.loadbalance.RoundRobinLoadBalance
leastactive=com.alibaba.dubbo.rpc.cluster.loadbalance.LeastActiveLoadBalance
consistenthash=com.alibaba.dubbo.rpc.cluster.loadbalance.ConsistentHashLoadBalance
```

用户使用时, 选择负载均衡策略, 只需要在XML中配置```java loadbalance="random"```, 那么Dubbo就会加载且仅加载`**RandomLoadBalance**`。

**Dubbo ExtensionLoader 流程解析:**

1. **获取`ExtensionLoader`:**

在获取`ExtensionLoader`时, 要判断传入的Class是否为interface, 并且是否有`@SPI`注解。创建`ExtensionLoader`实例后在内存中缓存, 保证每次扩展点具有唯一的`ExtensionLoader`单例。

2. **获取扩展点:**

```java
//根据名字获取扩展点实例
public T getExtension(String name) {
        if (name == null || name.length() == 0)
            throw new IllegalArgumentException("Extension name == null");
        if ("true".equals(name)) {
            return getDefaultExtension();
        }
        Holder<Object> holder = cachedInstances.get(name);
        if (holder == null) {
            cachedInstances.putIfAbsent(name, new Holder<Object>());
            holder = cachedInstances.get(name);
        }
        Object instance = holder.get();
        if (instance == null) {
            synchronized (holder) {
                instance = holder.get();
                if (instance == null) {
                    instance = createExtension(name);
                    holder.set(instance);
                }
            }
        }
        return (T) instance;
}
 
//实例化扩展点
private T createExtension(String name) {
    Class<?> clazz = getExtensionClasses().get(name);
    if (clazz == null) {
        throw findException(name);
    }
    try {
        T instance = (T) EXTENSION_INSTANCES.get(clazz);
        if (instance == null) {
            EXTENSION_INSTANCES.putIfAbsent(clazz, (T) clazz.newInstance());
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        }
        injectExtension(instance);
        Set<Class<?>> wrapperClasses = cachedWrapperClasses;
        if (wrapperClasses != null && wrapperClasses.size() > 0) {
            for (Class<?> wrapperClass : wrapperClasses) {
                instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
            }
        }
        return instance;
    } catch (Throwable t) {
        throw new IllegalStateException("Extension instance(name: " + name + ", class: " +
                type + ")  could not be instantiated: " + t.getMessage(), t);
    }
}
 
 
private Map<String, Class<?>> getExtensionClasses() {
      Map<String, Class<?>> classes = cachedClasses.get();
      if (classes == null) {
          synchronized (cachedClasses) {
              classes = cachedClasses.get();
              if (classes == null) {
                  classes = loadExtensionClasses();
                  cachedClasses.set(classes);
              }
          }
      }
      return classes;
}
 
//从配置文件中加载扩展点
private Map<String, Class<?>> loadExtensionClasses() {
    final SPI defaultAnnotation = type.getAnnotation(SPI.class);
    if(defaultAnnotation != null) {
        String value = defaultAnnotation.value();
        if(value != null && (value = value.trim()).length() > 0) {
            String[] names = NAME_SEPARATOR.split(value);
            if(names.length > 1) {
                throw new IllegalStateException("more than 1 default extension name on extension " + type.getName()
                        + ": " + Arrays.toString(names));
            }
            if(names.length == 1) cachedDefaultName = names[0];
        }
    }
     
    Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();
    loadFile(extensionClasses, DUBBO_INTERNAL_DIRECTORY);
    loadFile(extensionClasses, DUBBO_DIRECTORY);
    loadFile(extensionClasses, SERVICES_DIRECTORY);
    return extensionClasses;
}
```

获取扩展点时, 从内存缓存中获取扩展点实例。扩展点实例在进程中也是单例。Dubbo从三个路径中读取扩展点配置文件并加载:

- META-INF/services/
- META-INF/dubbo/
- META-INF/dubbo/internal/

3. **Setter 注入扩展点 & Wrapper 包装扩展点:**

在实例化扩展点的代码中, 我们可以看到有以下两个处理:

- Setter 注入: **扩展点实现类的成员如果为其他扩展点类型, `ExtensionLoader`会自动注入依赖扩展点, `ExtensionLoader`通过扫描扩展点实现类的所有set方法来判定其成员。**
- Wrapper 包装: **如果扩展点实现类有拷贝构造函数, 则认为是包装类。包装类持有实际扩展点实现类, 通过包装类可以吧所有扩展点的公共逻辑移到包装类, 类似AOP**

```java
/实例化扩展点
private T createExtension(String name) {
    //......
     
        injectExtension(instance);
        Set<Class<?>> wrapperClasses = cachedWrapperClasses;
        if (wrapperClasses != null && wrapperClasses.size() > 0) {
            for (Class<?> wrapperClass : wrapperClasses) {
                instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
            }
        }
        return instance;
         
    //......
}
 
//注入扩展点
private T injectExtension(T instance) {
    try {
        if (objectFactory != null) {
            for (Method method : instance.getClass().getMethods()) {
                if (method.getName().startsWith("set")
                        && method.getParameterTypes().length == 1
                        && Modifier.isPublic(method.getModifiers())) {
                    Class<?> pt = method.getParameterTypes()[0];
                    try {
                        String property = method.getName().length() > 3 ? method.getName().substring(3, 4).toLowerCase() + method.getName().substring(4) : "";
                        Object object = objectFactory.getExtension(pt, property);
                        if (object != null) {
                            method.invoke(instance, object);
                        }
                    } catch (Exception e) {
                        logger.error("fail to inject via method " + method.getName()
                                + " of interface " + type.getName() + ": " + e.getMessage(), e);
                    }
                }
            }
        }
    } catch (Exception e) {
        logger.error(e.getMessage(), e);
    }
    return instance;
}
```

4. **Adapive扩展点自适应 & Activate扩展点激活:**

从文件加载扩展点代码如下:

```java
private void loadFile(Map<String, Class<?>> extensionClasses, String dir) {
    //......
    BufferedReader reader = new BufferedReader(new InputStreamReader(url.openStream(), "utf-8"));
    try {
        String line = null;
        while ((line = reader.readLine()) != null) {
            final int ci = line.indexOf('#');
            if (ci >= 0) line = line.substring(0, ci);
            line = line.trim();
            if (line.length() > 0) {
                try {
                    String name = null;
                    int i = line.indexOf('=');
                    if (i > 0) {
                        name = line.substring(0, i).trim();
                        line = line.substring(i + 1).trim();
                    }
                    if (line.length() > 0) {
                        Class<?> clazz = Class.forName(line, true, classLoader);
                        if (! type.isAssignableFrom(clazz)) {
                            throw new IllegalStateException("Error when load extension class(interface: " +
                                    type + ", class line: " + clazz.getName() + "), class "
                                    + clazz.getName() + "is not subtype of interface.");
                        }
                        if (clazz.isAnnotationPresent(Adaptive.class)) {
                            if(cachedAdaptiveClass == null) {
                                cachedAdaptiveClass = clazz;
                            } else if (! cachedAdaptiveClass.equals(clazz)) {
                                throw new IllegalStateException("More than 1 adaptive class found: "
                                        + cachedAdaptiveClass.getClass().getName()
                                        + ", " + clazz.getClass().getName());
                            }
                        } else {
                            try {
                                clazz.getConstructor(type);
                                Set<Class<?>> wrappers = cachedWrapperClasses;
                                if (wrappers == null) {
                                    cachedWrapperClasses = new ConcurrentHashSet<Class<?>>();
                                    wrappers = cachedWrapperClasses;
                                }
                                wrappers.add(clazz);
                            } catch (NoSuchMethodException e) {
                                clazz.getConstructor();
                                if (name == null || name.length() == 0) {
                                    name = findAnnotationName(clazz);
                                    if (name == null || name.length() == 0) {
                                        if (clazz.getSimpleName().length() > type.getSimpleName().length()
                                                && clazz.getSimpleName().endsWith(type.getSimpleName())) {
                                            name = clazz.getSimpleName().substring(0, clazz.getSimpleName().length() - type.getSimpleName().length()).toLowerCase();
                                        } else {
                                            throw new IllegalStateException("No such extension name for the class " + clazz.getName() + " in the config " + url);
                                        }
                                    }
                                }
                                String[] names = NAME_SEPARATOR.split(name);
                                if (names != null && names.length > 0) {
                                    Activate activate = clazz.getAnnotation(Activate.class);
                                    if (activate != null) {
                                        cachedActivates.put(names[0], activate);
                                    }
                                    for (String n : names) {
                                        if (! cachedNames.containsKey(clazz)) {
                                            cachedNames.put(clazz, n);
                                        }
                                        Class<?> c = extensionClasses.get(n);
                                        if (c == null) {
                                            extensionClasses.put(n, clazz);
                                        } else if (c != clazz) {
                                            throw new IllegalStateException("Duplicate extension " + type.getName() + " name " + n + " on " + c.getName() + " and " + clazz.getName());
                                        }
                                    }
                                }
                            }
                        }
                    }
                } catch (Throwable t) {
                    IllegalStateException e = new IllegalStateException("Failed to load extension class(interface: " + type + ", class line: " + line + ") in " + url + ", cause: " + t.getMessage(), t);
                    exceptions.put(line, e);
                }
            }
        } // end of while read lines
    } finally {
        reader.close();
    }
    //......
}
```
上面一大串代码总结出来做了一下几件事:

1. 忽略已注释的行
2. 解析出名称和扩展点实现类名
3. 判断是否有`@Adaptive`注解
4. 匹配构造函数, 判断是否为Wrapper类
5. 判断是否有`@Activate`注解

- **`@Adaptive`**: 扩展点自适应, 直到扩展点方法执行时才决定调用哪一个扩展点实现。扩展点的调用会有URL作为参数, 通过`@Adaptive`注解可以提取约定key来决定调用哪个实现的方法。

- **`@Activate`**: 扩展点自动激活, 指定URL中激活扩展点的key, 未指定key表示无条件激活。比如: `LoadBalance`

```java

// 表示默认使用Random负载均衡策略, 同时会根据用户在XML中配置的loadbbalance参数来最终决定调用哪个扩展点实现类。
SPI(RandomLoadBalance.NAME)
public interface LoadBalance {
 
    @Adaptive("loadbalance")
    <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) throws RpcException;
 
}
```

再比如`AsyncFilter`:

```java

// 表示只有在Consumer端才会激活
@Activate(group = Constants.CONSUMER)
public class AsyncFilter implements Filter{
 
}
```


#### 3.3 Dubbo服务暴露

分析服务暴露的实现原理, 先看下暴露服务的时序图:

![Service_Exposure时序图](/distributed/RPC/img/Service_Exposure.jpg)

##### 3.3.1 标签解析

根据这篇文章《[Dubbo原理和源码解析之标签解析](https://www.cnblogs.com/cyfonly/p/9091857.html)》, `<dubbo:service>`标签会被解析成`ServiceBean`。

`ServiceBean`实现了`InitializingBean`接口, 在类加载完成之后会调用`afterPropertiesSet()`方法。在`afterPropertiesSet()`方法中, 依次解析一下标签信息:

- `<dubbo:provider>`
- `<dubbo:application>`
- `<dubbo:module>`
- `<dubbo:registry>`
- `<dubbo:monitor>`
- `<dubbo:protocol>`

`ServiceBean`还实现了`ApplicationListener`接口, 在Spring容器初始化时会调用`onapplicationEvent`方法。`ServiceBean`重写了`onApplictionEvent`方法, 实现了服务暴露的功能。

```java
public void onApplicationEvent(ApplicationEvent event) {
    if (ContextRefreshedEvent.class.getName().equals(event.getClass().getName())) {
        if (isDelay() && ! isExported() && ! isUnexported()) {
            if (logger.isInfoEnabled()) {
                logger.info("The service ready on spring started. service: " + getInterface());
            }
            export();
        }
    }
}
```

##### 3.3.2 延迟暴露

`ServiceBean`扩展了`ServiceConfig`, 调用export()方法时由`ServiceConfig`完成服务暴露功能的实现。

`**ServiceConfig.java**`:

```java
public synchronized void export() {
    if (provider != null) {
        if (export == null) {
            export = provider.getExport();
        }
        if (delay == null) {
            delay = provider.getDelay();
        }
    }
    if (export != null && ! export.booleanValue()) {
        return;
    }
    if (delay != null && delay > 0) {
        Thread thread = new Thread(new Runnable() {
            public void run() {
                try {
                    Thread.sleep(delay);
                } catch (Throwable e) {
                }
                doExport();
            }
        });
        thread.setDaemon(true);
        thread.setName("DelayExportServiceThread");
        thread.start();
    } else {
        doExport();
    }
}
```

根据代码可知, 如果设置了delay参数, Dubbo的处理凡是是启动一个守护线程在sleep指定时间后再doExport。

#### 3.4 Dubbo服务引用

#### 3.5 Dubbo服务调用