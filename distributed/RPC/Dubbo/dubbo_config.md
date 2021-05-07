## Dubbo 配置详解

其实Dubbo的配置详情, 在官方文档上[Dubbo配置官方文档](https://dubbo.apache.org/zh/docs/v2.7/user/examples/)写的已经很详细了, 这里只是做一个笔记, 加深一下印象。

### 1. 启动时检查

**在启动时检查依赖的服务是否可用**

> Dubbo 缺省会在启动时检查依赖的服务是否可用，不可用时会抛出异常，阻止 Spring 初始化完成，以便上线时，能及早发现问题，默认 check="true"。