## <center>Dubbo ExtensionLoader 流程解析</center>


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

### 1. 获取ExtensionLoader

在获取`ExtensionLoader`时, 要判断传入的Class是否为interface, 并且是否有`@SPI`注解。创建`ExtensionLoader`实例后在内存中缓存, 保证每次扩展点具有唯一的`ExtensionLoader`单例。

### 2. 获取扩展点

先检查缓存中是否存在扩展对象, 如果不存在,则创建扩展对象。

``` java
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
```

#### 2.1 创建扩展对象

`createExtension`方法逻辑为: 
1. 通过`getExtensionClasses`获取所有扩展类
2. 通过反射创建扩展对象
3. 向扩展对象注入依赖
4. 将扩展对象包装在相应的Wrapper对象中

```java
private T createExtension(String name) {
    // 从配置文件中加载所有的拓展类，可得到“配置项名称”到“配置类”的映射关系表
    Class<?> clazz = getExtensionClasses().get(name);
    if (clazz == null) {
        throw findException(name);
    }
    try {
        T instance = (T) EXTENSION_INSTANCES.get(clazz);
        if (instance == null) {
            // 通过反射创建实例
            EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        }
        // 向实例中注入依赖
        injectExtension(instance);
        Set<Class<?>> wrapperClasses = cachedWrapperClasses;
        if (wrapperClasses != null && !wrapperClasses.isEmpty()) {
            // 循环创建 Wrapper 实例
            for (Class<?> wrapperClass : wrapperClasses) {
                // 将当前 instance 作为参数传给 Wrapper 的构造方法，并通过反射创建 Wrapper 实例。
                // 然后向 Wrapper 实例中注入依赖，最后将 Wrapper 实例再次赋值给 instance 变量
                instance = injectExtension(
                    (T) wrapperClass.getConstructor(type).newInstance(instance));
            }
        }
        return instance;
    } catch (Throwable t) {
        throw new IllegalStateException("...");
    }
}
```

#### 2.2 获取所有的扩展类

在获取扩展类之前, 先需要根据配置文件解析出扩展项名称与扩展类的映射关系表(`Map<名称, 扩展类>`), 之后再根据扩展项名称从映射关系中取出对应的扩展类即可。

所以代码逻辑还是先检查缓存是否存在, 如果缓存为空, 则通过`synchronized`加锁。加锁之后再次判断缓存是否为空, 如果获取出的map仍然为空则通过`loadExtensionClasses`加载扩展类。

`loadExtensionClasses`方法主要对SPI注解进行解析, 而且调用`loadDirectory`方法加载指定文件夹配置文件。而`loadDirectory`方法通过`loadResource`方法加载资源。
```java
private Map<String, Class<?>> getExtensionClasses() {
    // 从缓存中获取已加载的拓展类
    Map<String, Class<?>> classes = cachedClasses.get();
    // 双重检查
    if (classes == null) {
        synchronized (cachedClasses) {
            classes = cachedClasses.get();
            if (classes == null) {
                // 加载拓展类
                classes = loadExtensionClasses();
                cachedClasses.set(classes);
            }
        }
    }
    return classes;
}
```

**`loadResource`方法: 用于读取和解析配置文件, 并通过反射加载类, 最后调用`loadClasses`方法将类加入缓存中** 

```java
private void loadClass(Map<String, Class<?>> extensionClasses, java.net.URL resourceURL, 
    Class<?> clazz, String name) throws NoSuchMethodException {
    
    if (!type.isAssignableFrom(clazz)) {
        throw new IllegalStateException("...");
    }

    // 检测目标类上是否有 Adaptive 注解
    if (clazz.isAnnotationPresent(Adaptive.class)) {
        if (cachedAdaptiveClass == null) {
            // 设置 cachedAdaptiveClass缓存
            cachedAdaptiveClass = clazz;
        } else if (!cachedAdaptiveClass.equals(clazz)) {
            throw new IllegalStateException("...");
        }
        
    // 检测 clazz 是否是 Wrapper 类型
    } else if (isWrapperClass(clazz)) {
        Set<Class<?>> wrappers = cachedWrapperClasses;
        if (wrappers == null) {
            cachedWrapperClasses = new ConcurrentHashSet<Class<?>>();
            wrappers = cachedWrapperClasses;
        }
        // 存储 clazz 到 cachedWrapperClasses 缓存中
        wrappers.add(clazz);
        
    // 程序进入此分支，表明 clazz 是一个普通的拓展类
    } else {
        // 检测 clazz 是否有默认的构造方法，如果没有，则抛出异常
        clazz.getConstructor();
        if (name == null || name.length() == 0) {
            // 如果 name 为空，则尝试从 Extension 注解中获取 name，或使用小写的类名作为 name
            name = findAnnotationName(clazz);
            if (name.length() == 0) {
                throw new IllegalStateException("...");
            }
        }
        // 切分 name
        String[] names = NAME_SEPARATOR.split(name);
        if (names != null && names.length > 0) {
            Activate activate = clazz.getAnnotation(Activate.class);
            if (activate != null) {
                // 如果类上有 Activate 注解，则使用 names 数组的第一个元素作为键，
                // 存储 name 到 Activate 注解对象的映射关系
                cachedActivates.put(names[0], activate);
            }
            for (String n : names) {
                if (!cachedNames.containsKey(clazz)) {
                    // 存储 Class 到名称的映射关系
                    cachedNames.put(clazz, n);
                }
                Class<?> c = extensionClasses.get(n);
                if (c == null) {
                    // 存储名称到 Class 的映射关系
                    extensionClasses.put(n, clazz);
                } else if (c != clazz) {
                    throw new IllegalStateException("...");
                }
            }
        }
    }
}
```

### 3. Setter 注入扩展点 & Wrapper 包装扩展点

在实例化扩展点的代码中, 我们可以看到有以下两个处理:

- Setter 注入: **扩展点实现类的成员如果为其他扩展点类型, `ExtensionLoader`会自动注入依赖扩展点, `ExtensionLoader`通过扫描扩展点实现类的所有set方法来判定其成员。**
- Wrapper 包装: **如果扩展点实现类有拷贝构造函数, 则认为是包装类。包装类持有实际扩展点实现类, 通过包装类可以吧所有扩展点的公共逻辑移到包装类, 类似AOP**

```java
//实例化扩展点
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

### 4. Adapive扩展点自适应 & Activate扩展点激活

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

### 5 Dubbo IOC

Dubbo IOC是通过setter方法注入依赖。Dubbo会通过反射获取到实例的所有方法, 然后遍历方法列表, 检测方法名是否具有setter方法特性, 如果有则通过`ObjectFactory`获取依赖对象, 最后通过反射调用setter方法将依赖设置到目标对象中。

```java
private T injectExtension(T instance) {
    try {
        if (objectFactory != null) {
            // 遍历目标类的所有方法
            for (Method method : instance.getClass().getMethods()) {
                // 检测方法是否以 set 开头，且方法仅有一个参数，且方法访问级别为 public
                if (method.getName().startsWith("set")
                    && method.getParameterTypes().length == 1
                    && Modifier.isPublic(method.getModifiers())) {
                    // 获取 setter 方法参数类型
                    Class<?> pt = method.getParameterTypes()[0];
                    try {
                        // 获取属性名，比如 setName 方法对应属性名 name
                        String property = method.getName().length() > 3 ? 
                            method.getName().substring(3, 4).toLowerCase() + 
                            	method.getName().substring(4) : "";
                        // 从 ObjectFactory 中获取依赖对象
                        Object object = objectFactory.getExtension(pt, property);
                        if (object != null) {
                            // 通过反射调用 setter 方法设置依赖
                            method.invoke(instance, object);
                        }
                    } catch (Exception e) {
                        logger.error("fail to inject via method...");
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