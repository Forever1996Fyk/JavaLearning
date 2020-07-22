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