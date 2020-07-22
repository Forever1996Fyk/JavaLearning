## <center><font color="red">SpringBoot的核心——自动装配</font></center>

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

### 2. SpringBoot自动配置的原理

通过上面我们看到的`@SpringBootApplication`注解的源码, 我们知道SpringBoot自动配置最核心的注解是`@EnableAutoConfiguration`注解

```java
EnableAutoConfiguration注解源码

```
而这个`@EnableAutoConfiguration`注解, 也是一个派生注解, 其中最关键的就是`@Import`注解中导入的**AutoConfigurationImportSelector**类。这个类有一个`selectImports()`方法, 其作用就是扫描**META-INF/spring.factories**配置文件。 

**META-INF/spring.factories**配置文件就是所有需要自动配置类的全类名列表

![spring.factories](/develop_framework/SpringBoot/img/spring_factories.png)
