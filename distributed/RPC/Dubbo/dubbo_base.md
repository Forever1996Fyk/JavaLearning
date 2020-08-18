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

### 3. Dubbo SPI

SPI(Service Provider Interface), 是一种服务发现机制。SPI的本质是将接口实现类的全限定名配置在文件中, 并由服务加载器读取配置文件, 加载实现类。这样可以在运行时, 动态为接口替换实现类。

具体相关`ExtensionLoader`原理, 会在[Dubbo_ExtensionLoader](/distributed/RPC/Dubbo/dubbo_extensionLoader.md)中解释

#### 3.1 Java SPI实例

先定义一个接口

```java
public interface Robot {
    void sayHello();
}
```

定义两个实现类, 实现这个接口

```java
public class OptimusPrime implements Robot {
    
    @Override
    public void sayHello() {
        System.out.println("Hello, I am Optimus Prime.");
    }
}

public class Bumblebee implements Robot {

    @Override
    public void sayHello() {
        System.out.println("Hello, I am Bumblebee.");
    }
}
```

在`NETA-INF/services`文件夹下创建一个文件, 名称为Robot的全限定名`org.apache.spi.Robot`。文件内容为实现类的全限定的类名:

```properties
org.apache.spi.OptimusPrime
org.apache.spi.Bumblebee
```

测试:

```java
public class JavaSPITest {

    @Test
    public void sayHello() throws Exception {
        ServiceLoader<Robot> serviceLoader = ServiceLoader.load(Robot.class);
        System.out.println("Java SPI");
        serviceLoader.forEach(Robot::sayHello);
    }
}

// 结果:

Java SPI 
Hello, I am Optimus Prime.
Hello, I an Bumlebee.
```

#### 3.2 Dubbo SPI示例

Dubbo SPI的相关逻辑封装在`ExtensionLoader`类中, 通过`ExtensionLoader`可以加载指定的实现类。Dubbo SPI所需的配置文件需要放置在`META-INF/dubbo`路径下,

Dubbo SPI实现类胚子通过键值对的方式进行配置, 而且需要对接口标注`@SPI`注解

```properties
optimusPrime = org.apache.spi.OptimusPrime
bumblebee = org.apache.spi.Bumblebee
```

测试

```java
public class DubboSPITest {

    @Test
    public void sayHello() throws Exception {
        ExtensionLoader<Robot> extensionLoader = 
            ExtensionLoader.getExtensionLoader(Robot.class);
        Robot optimusPrime = extensionLoader.getExtension("optimusPrime");
        optimusPrime.sayHello();
        Robot bumblebee = extensionLoader.getExtension("bumblebee");
        bumblebee.sayHello();
    }
}
```

