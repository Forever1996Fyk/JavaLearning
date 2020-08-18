## <center>Dubbo服务暴露原理解析</center>

分析服务暴露的实现原理, 先看下暴露服务的时序图:

![Service_Exposure时序图](/distributed/RPC/img/Service_Exposure.jpg)

### 1. 标签解析

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


### 2. 延迟暴露

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

根据代码可知, 如果设置了delay参数, Dubbo的处理就是启动一个守护线程在sleep指定时间后再doExport。

### 3. 参数校验

在`ServiceConfig`的`doExport()`方法中会进行参数检查和设置, 包括:

- 泛化调用
- 本地实现
- 本地存根
- 本地伪装
- 配置(application, registry, protocol等)

### 4. 多协议, 多注册中心
在检查完参数之后，开始暴露服务。Dubbo支持多协议和多注册中心:

`**ServiceConfig.java**`:

```java
private void doExportUrls() {
    List<URL> registryURLs = loadRegistries(true);
    for (ProtocolConfig protocolConfig : protocols) {
        doExportUrlsFor1Protocol(protocolConfig, registryURLs);
    }
}
```

通过`loadRegistries`加载注册中心链接, 然后遍历`ProtocolConfig`集合导出每个服务, 并在导出服务的过程中将服务注册到注册中心。


### 5. 组装URL

针对每个协议，每个注册中心, 开始组装URL。

`**ServiceConfig.java**`:

```java
private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
    String name = protocolConfig.getName();
    if (name == null || name.length() == 0) {
        name = "dubbo";
    }

    //处理host

    //处理port

    Map<String, String> map = new HashMap<String, String>();
    //设置参数到map

    // 导出服务
    String contextPath = protocolConfig.getContextpath();
    if ((contextPath == null || contextPath.length() == 0) && provider != null) {
        contextPath = provider.getContextpath();
    }
    URL url = new URL(name, host, port, (contextPath == null || contextPath.length() == 0 ? "" : contextPath + "/") + path, map);

    if (ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
            .hasExtension(url.getProtocol())) {
        url = ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
                .getExtension(url.getProtocol()).getConfigurator(url).configure(url);
    }

    //此处省略：服务暴露（详见 2.6 和 2.7）
    
    this.urls.add(url);
}
```

### 6. 本地暴露

如果配置`scope=none`, 则不会进行服务暴露; 如果`scope!=remote`, 则会进行本地暴露。

> 暴露服务时, 会通过代理创建`Invoker`;
> 本地暴露时使用`injvm`协议, `injvm`协议是一个伪协议, 它不开启端口, 不能被远程调用, 只在JVM内直接关联, 但执行Duboo的Filter链。

`**ServiceConfig.java**`:

```java
//public static final String LOCAL_PROTOCOL = "injvm";
//public static final String LOCALHOST = "127.0.0.1";

private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
    //......
    String scope = url.getParameter(Constants.SCOPE_KEY);
    //配置为none不暴露
    if (! Constants.SCOPE_NONE.toString().equalsIgnoreCase(scope)) {
        //配置不是remote的情况下做本地暴露 (配置为remote，则表示只暴露远程服务)
        if (!Constants.SCOPE_REMOTE.toString().equalsIgnoreCase(scope)) {
            exportLocal(url);
        }
        //......
    }
    //......
}

private void exportLocal(URL url) {
    if (!Constants.LOCAL_PROTOCOL.equalsIgnoreCase(url.getProtocol())) {
        URL local = URL.valueOf(url.toFullString())
                .setProtocol(Constants.LOCAL_PROTOCOL)
                .setHost(NetUtils.LOCALHOST)
                .setPort(0);
        Exporter<?> exporter = protocol.export(
                proxyFactory.getInvoker(ref, (Class) interfaceClass, local));
        exporters.add(exporter);
        logger.info("Export dubbo service " + interfaceClass.getName() +" to local registry");
    }
}
```


### 7. 远程暴露

如果配置`scope!=local`, 则会进行远程暴露。


在服务暴露时, 有两种情况:

    - 不使用注册中心: 直接暴露对应协议的服务, 引用服务时只能通过直连方式引用。
    - 使用注册中心: 暴露对应协议的服务后, 会将服务节点注册到注册中心, 引用服务时可以通过注册中心动态获取服务提供者列表, 也可以通过直连方式引用。


`**ServiceConfig.java**`:

    ```java
        private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
            String scope = url.getParameter(Constants.SCOPE_KEY);
            //配置为none不暴露
            if (! Constants.SCOPE_NONE.toString().equalsIgnoreCase(scope)) {
                //......
                //如果配置不是local则暴露为远程服务.(配置为local，则表示只暴露远程服务)
                if (! Constants.SCOPE_LOCAL.toString().equalsIgnoreCase(scope) ){
                    if (logger.isInfoEnabled()) {
                        logger.info("Export dubbo service " + interfaceClass.getName() + " to url " + url);
                    }
                    if (registryURLs != null && registryURLs.size() > 0
                            && url.getParameter("register", true)) {
                        for (URL registryURL : registryURLs) {
                            url = url.addParameterIfAbsent("dynamic", registryURL.getParameter("dynamic"));
                            URL monitorUrl = loadMonitor(registryURL);
                            if (monitorUrl != null) {
                                url = url.addParameterAndEncoded(Constants.MONITOR_KEY, monitorUrl.toFullString());
                            }
                            if (logger.isInfoEnabled()) {
                                logger.info("Register dubbo service " + interfaceClass.getName() + " url " + url + " to registry " + registryURL);
                            }
                            Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));
            
                            Exporter<?> exporter = protocol.export(invoker);
                            exporters.add(exporter);
                        }
                    } else {
                        Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, url);
            
                        Exporter<?> exporter = protocol.export(invoker);
                        exporters.add(exporter);
                    }
                }
            }
        }
    
    ```

### 8. 暴露服务

不使用注册中心时, 直接调用对应协议(如Dubbo协议)的export()暴露服务。以Dubbo协议为例:

`**DubboProtocol.java**`:

```java
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
    URL url = invoker.getUrl();
    
    // export service.
    String key = serviceKey(url);
    DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
    exporterMap.put(key, exporter);
    
    //export an stub service for dispaching event
    Boolean isStubSupportEvent = url.getParameter(Constants.STUB_EVENT_KEY,Constants.DEFAULT_STUB_EVENT);
    Boolean isCallbackservice = url.getParameter(Constants.IS_CALLBACK_SERVICE, false);
    if (isStubSupportEvent && !isCallbackservice){
        String stubServiceMethods = url.getParameter(Constants.STUB_EVENT_METHODS_KEY);
        if (stubServiceMethods == null || stubServiceMethods.length() == 0 ){
            if (logger.isWarnEnabled()){
                logger.warn(new IllegalStateException("consumer [" +url.getParameter(Constants.INTERFACE_KEY) +
                        "], has set stubproxy support event ,but no stub methods founded."));
            }
        } else {
            stubServiceMethodsMap.put(url.getServiceKey(), stubServiceMethods);
        }
    }

    // 创建并启动Server
    openServer(url);
    
    return exporter;
}

private void openServer(URL url) {
    // find server.
    String key = url.getAddress();
    //client 也可以暴露一个只有server可以调用的服务。
    boolean isServer = url.getParameter(Constants.IS_SERVER_KEY,true);
    if (isServer) {
        ExchangeServer server = serverMap.get(key);
        if (server == null) {
            serverMap.put(key, createServer(url));
        } else {
            //server支持reset,配合override功能使用
            server.reset(url);
        }
    }
}

private ExchangeServer createServer(URL url) {
    //默认开启server关闭时发送readonly事件
    url = url.addParameterIfAbsent(Constants.CHANNEL_READONLYEVENT_SENT_KEY, Boolean.TRUE.toString());
    //默认开启heartbeat
    url = url.addParameterIfAbsent(Constants.HEARTBEAT_KEY, String.valueOf(Constants.DEFAULT_HEARTBEAT));
    String str = url.getParameter(Constants.SERVER_KEY, Constants.DEFAULT_REMOTING_SERVER);

    if (str != null && str.length() > 0 && ! ExtensionLoader.getExtensionLoader(Transporter.class).hasExtension(str))
        throw new RpcException("Unsupported server type: " + str + ", url: " + url);

    url = url.addParameter(Constants.CODEC_KEY, Version.isCompatibleVersion() ? COMPATIBLE_CODEC_NAME : DubboCodec.NAME);
    ExchangeServer server;
    try {
        server = Exchangers.bind(url, requestHandler);
    } catch (RemotingException e) {
        throw new RpcException("Fail to start server(url: " + url + ") " + e.getMessage(), e);
    }
    str = url.getParameter(Constants.CLIENT_KEY);
    if (str != null && str.length() > 0) {
        Set<String> supportedTypes = ExtensionLoader.getExtensionLoader(Transporter.class).getSupportedExtensions();
        if (!supportedTypes.contains(str)) {
            throw new RpcException("Unsupported client type: " + str);
        }
    }
    return server;
}
```

### 9. 服务注册

如果使用了注册中心, 则在通过具体协议(如Dubbo协议)暴露服务之后, 进入服务注册流程, 将服务注册到注册中心。

`getRegistry()`方法根据注册中心类型(默认Zookeeper)获取注册中心客户端, 由注册中心客户端实例来进行真正的服务注册。

注册中心客户端将节点注册到注册中心, 同时订阅对应的override数据, 实时监听服务的属性变动实现动态配置功能。

最终返回`Exporter`实现了`unexport()`方法, 这样在服务下线时清理相关资源

`**RegistryProtocol.java**`:

```java
public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
    //export invoker
    final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker);
    //registry provider
    final Registry registry = getRegistry(originInvoker);
    final URL registedProviderUrl = getRegistedProviderUrl(originInvoker);
    //向Zookeeper注册节点
    registry.register(registedProviderUrl);
    // 订阅override数据
    // FIXME 提供者订阅时，会影响同一JVM即暴露服务，又引用同一服务的的场景，因为subscribed以服务名为缓存的key，导致订阅信息覆盖。
    final URL overrideSubscribeUrl = getSubscribedOverrideUrl(registedProviderUrl);
    final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl);
    overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);
    registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);
    //保证每次export都返回一个新的exporter实例
    return new Exporter<T>() {
        public Invoker<T> getInvoker() {
            return exporter.getInvoker();
        }
        public void unexport() {
            try {
                exporter.unexport();
            } catch (Throwable t) {
                logger.warn(t.getMessage(), t);
            }
            try {
                registry.unregister(registedProviderUrl);
            } catch (Throwable t) {
                logger.warn(t.getMessage(), t);
            }
            try {
                overrideListeners.remove(overrideSubscribeUrl);
                registry.unsubscribe(overrideSubscribeUrl, overrideSubscribeListener);
            } catch (Throwable t) {
                logger.warn(t.getMessage(), t);
            }
        }
    };
}

private Registry getRegistry(final Invoker<?> originInvoker){
    URL registryUrl = originInvoker.getUrl();
    if (Constants.REGISTRY_PROTOCOL.equals(registryUrl.getProtocol())) {
        String protocol = registryUrl.getParameter(Constants.REGISTRY_KEY, Constants.DEFAULT_DIRECTORY);
        registryUrl = registryUrl.setProtocol(protocol).removeParameter(Constants.REGISTRY_KEY);
    }
    return registryFactory.getRegistry(registryUrl);
}
```