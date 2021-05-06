## <center><font color="red">SpringBoot的核心——自动装配(重点)</font></center>

这是一个十分及其特别重要的知识点, 不仅仅是高频面试点, 也是我们学习SpringBoot必须要知道的知识, 因为这是SpringBoot的核心。可以说SpringBoot学的如何, 就看对自动装配的流程是否熟悉。

### 1. `@SpringBootApplication`注解

查看`@SpringBootApplication`源码, 会发现这是一个组合注解

```java
package org.springframework.boot.autoconfigure;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {
        @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
        @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
   ......
}

package org.springframework.boot;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration
public @interface SpringBootConfiguration {

}
```

通过源码分析, `@SpringBootApplication`注解基本上可以分解为`@EnableAutoConfiguration`, `@ComponentScan`, `@Configuration`这个三个注解。

- `@EnableAutoConfiguration`: 启用SpringBoot的自动配置机制
- `@ComponentScan`: 扫描被`@Component`(`@Service`, `@COntroller`等)注解的Bean, 注解默认会扫描该类下所在包下的所有类。
- `@Configuration`: 允许在上下文中注册额外的Bean或者导入其他配置类。

### 2. SpringBoot自动配置的流程

- 通过上面我们看到的`@SpringBootApplication`注解的源码, 我们知道SpringBoot自动配置最核心的注解是`@EnableAutoConfiguration`注解

    ```java
    package org.springframework.boot.autoconfigure;

    @Target({ElementType.TYPE})
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    @Inherited
    @AutoConfigurationPackage
    @Import({AutoConfigurationImportSelector.class})
    public @interface EnableAutoConfiguration {
        String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

        Class<?>[] exclude() default {};

        String[] excludeName() default {};
    }

    ```

- 而这个`@EnableAutoConfiguration`注解, 也是一个派生注解, 其中最关键的就是`@Import`注解中导入的**AutoConfigurationImportSelector**类。这个类有一个`selectImports()`方法, 其作用就是扫描并读取**META-INF/spring.factories**配置文件。 

    **META-INF/spring.factories**配置文件就是所有需要自动配置类的全类名列表, 这些类名以逗号分割, 以xxxAutoConfiguration形式命名。如下图:

    ![spring.factories](/image/spring_factories.png)

- 对于这些自动配置类XxxxAutoConfiguration都是在某些条件下才会生效, 这些条件也是以`@Conditional`注解的形式体现。

    1. `@ConditionalOnBean`: 当容器理由指定的Bean的条件下, 才会自动配置。
    2. `@ConditionalOnMissingBean`: 当容器中不存在指定Bean的条件下, 才会自动配置。
    3. `@ConditionalOnClass`: 当类的路径下有指定类的条件下, 才会自动配置。
    4. `@ConditionalOnMissingClass`: 当类的路径下不存在指定类的条件下, 才会自动配置。
    5. `@ConditionalOnProperty`: 指定的属性是否有指定值的条件下, 才会自动配置。比如: `@ConditionalOnProperties(prefix="xxx.xxx", value="enable", matchIfMissing=true)`, 就表示当xxx.xxx为enable时, 且它的值为true, 如果没有设置的情况下默认为true, 才会进行自动配置。

    - 而每一个自动配置类上都有一个`@EnableConfigurationProperties`注解: 开启配置属性, 而它所带的参数是一个XxxxProperties属性配置类。这是习惯优于配置的最终落点。

    ```java
    package org.springframework.boot.autoconfigure.web.servlet;

    @Configuration
    @AutoConfigureOrder(-2147483648)
    @ConditionalOnClass({ServletRequest.class})
    @ConditionalOnWebApplication(
        type = Type.SERVLET
    )
    @EnableConfigurationProperties({ServerProperties.class})
    @Import({ServletWebServerFactoryAutoConfiguration.BeanPostProcessorsRegistrar.class, EmbeddedTomcat.class, EmbeddedJetty.class, EmbeddedUndertow.class})
    public class ServletWebServerFactoryAutoConfiguration { 
        ... 
    }
    ```

- 在XxxxProperties类中, 发现了我们非常熟悉的注解`@ConfigurationProperties`, 它的作用就是将配置文件(.yml/.properties)绑定到对应的Bean上, 而`@EnableConfigurationProperties`将这个绑定属性的Bean注入到Spring容器中。也就是说, 我们在`applicaiton.yml/application.properties`中配置的属性, 最终会落实到XxxxProperties类中。

    ```java
    package org.springframework.boot.autoconfigure.web;

    @ConfigurationProperties(
        prefix = "server",
        ignoreUnknownFields = true
    )
    public class ServerProperties {
        private Integer port;
        private InetAddress address;

        ...
        
    }
    ```

所以整个SpringBoot自动配置的流程大致如下:

1. 将`@SpringBootApplicaiton`注解标记在SpringBoot启动类上, 并启动`SpringApplicaiton.run(...)`方法;
2. 通过`@EnableConfiguration`注解, 就会找到内部注解`@Import`导入的**AutoConfigurationImportSelector**类的`selectImports()`方法; 
3. 执行`selectImports()`方法, 加载**META-INFO/spring.factories**配置文件;
4. **META-INFO/spring.factories**配置文件中的自动配置类XxxxAutoConfiguration根据相应的条件注解`@Conditional`等等, 判断是否需要进行自动配置;
5. 在通过满足条件的自动配置类上的`@EnableCOnfigurationProperties`注解, 开启配置属性, 导入对应的配置属性类XxxxProperties类;
6. 通过配置属性类XxxxProperties类上的`@ConfigurationProperties`注解, 读取配置文件(.yml/.properties)中绑定对应的属性到配置属性类上, 并注入到Spring容器中, 如果没有配置文件将使用默认配置;
7. 自动配置类XxxxAutoConfiguration获取这个配置属性类的Bean注入属性,最终完成自动配置。

所以SpringBoot的自动配置原理是比较复杂的, 但是一层层的不断揭开就会发现SpringBoot的最终通过XxxxProperties配置类, 读取我们经常用的配置文件绑定配置。

> **<font color="red">简单谈谈SpringBoot自动配置的原理? (面试过程中, 不需要把上面的具体流程说出来, 只要说到点子上即可)</font>**

**<font color="red">Spring Boot启动时, 通过`@EnableAutoConfiguration`注解找到并加载`META-INFO/spring.factories`配置文件中的所有自动配置类, 这些自动配置类可以通过Properties属性类获取在全局配置文件中配置的属性。比如server.port, 而这些Properties属性类是通过`@ConfigurationProperties`注解与全局配置文件中对应的属性进行绑定。</font>**

![SpringBoot自动配置流程图](/image/SpringBootAutoConfig流程图.png)