## Spring Transaction

在日常开发中, 只要涉及到业务逻辑的开发都一定要去深入理解事务, 这不仅是面试高频提问点, 也是实际开发中必备技能。所以很重要!!!

我们既要知道事务的概念, 还要知道事务在Spring中是如何支持的。

### 1. 什么是事务?

**事务是逻辑上的一组操作, 要么都执行, 要么都不执行。**

系统中的每个业务方法可能包括了多个原子性的数据库操作, 这些操作是有依赖的, 它们要么都执行, 要么都不执行。

而且事务能否生效的关键在于数据库引擎是否支持事务。例如常用的MySQL数据库默认使用支持事务的`innodb`引擎。
但是, 如果数据库引擎变为`myisam`, 那么程序也无法再支持事务。

### 2. 事务的特性(ACID)了解吗?

![事务特性](/image/spring_transaction_characteristic.png)
- **原子性**: 事务是最小的执行单位, 不允许在分割。事务的原子性确保动作要么全部完成, 要么完全不起作用。
- **一致性**: 执行事务前后, 数据保持一致。
- **隔离性**: 并发访问数据库时, 一个事务的执行不应影响其他事务的执行。
- **持久性**: 一个事务被提交后, 对数据库中数据的改变是持久的, 即使数据库发生故障也不应该对其有任何影响。

> 在学习Spring的事务的支持和管理之前, 要先知道Mysql是怎样保证原子性的?

想要保证事务的原子性, 就需要在发生异常时, 对已经执行的操作进行回滚。在MySQL中, 恢复机制是通过**回滚日志(undo log)**实现的, 所有事务进行的修改都会先记录到这个回滚日志中, 然后在执行相关操作。如果执行过程中遇到异常的话, 我们直接利用回滚日志中的信息将数据回滚到修改之前的样子即可!而且, 回滚日志会先于数据持久化到磁盘上。这样就保证了即使遇到数据库突然宕机等情况, 当用户再次启动数据库的时候, 数据库还能够通过查询回滚日志来执行未完成的事务。

### 3. Spring对事务的支持

Spring对事务支持的前提, 就是数据库要使用支持事务的引擎!!!

#### 3.1 Spring支持两种方式的事务管理

1. 编程式事务管理
    通过`TransactionTemplate`或者`TransactionManager`手动管理事务, 但是实际应用很少使用, 因为跟业务逻辑代码的耦合太高了。

2. 声明式事务管理
    通过Spring AOP实现(基于`@Transactional`注解的方式最多)。 这是实际项目中最常用的方式, 也是利用AOP的特性, 贯穿整个业务模块, 实现统一的事务管理。

#### 3.2 Spring事务管理接口

Spring框架中, 事务管理相关的最重要的3个接口:
- `PlatformTransactionManager`: 事务管理器, Spring事务策略的核心。
- `TransactionDefinition`: 事务定义信息(事务隔离级别, 传播行为, 超时, 只读, 回滚规则)。
- `TransactionStatus`: 事务运行状态。

> 简单的说下Spring事务管理的机制?(我觉得说下核心的三个接口, 以及每个接口的用户的关系即可)

`PlatformTransactionManager`会根据`TransactionDefinition`的定义, 比如事务超时时间, 隔离级别, 传播行为等来进行事务管理, 而`TransactionStatus`接口提供方法获取事务
相应的状态比如是否新事务, 是否可以回滚。

##### 3.2.1 PlatformTransactionManager事务管理接口

Spring并不直接管理事务, 而是提供了多种事务管理器。即`PlatformTransactionManager`。通过这个接口Spring为各个平台提供了对应的事务管理器, 但是具体的实现都由各个平台实现。比如: JDBC(`DataSourceTransactionManager`), Hibernate(`HibernateTransactionManager`), JPA(JpaTransactionManager)等。

`PlatformTransactionManager`接口定义了三个方法, 获取事务, 提交事务, 回滚事务。
```java
    package org.springframework.transaction;

    import org.springframework.lang.Nullable;

    public interface PlatformTransactionManager {
        //获得事务
        TransactionStatus getTransaction(@Nullable TransactionDefinition var1) throws TransactionException;
        //提交事务
        void commit(TransactionStatus var1) throws TransactionException;
        //回滚事务
        void rollback(TransactionStatus var1) throws TransactionException;
    }

```

##### 3.2.2 TransactionDefinition事务属性

`PlatformTransactionManager`通过`getTransaction(@Nullable TransactionDefinition var1)`方法来获取事务, 其中`TransactionDefinition`类, 定义一些基本的**事务属性**。

![事务属性](/develop_framework/Spring/img/事务属性.png)



##### 3.2.3 TransactionStatus事务状态

`TransactionStatus`接口用来记录事务的状态, `PlatformTransactionManager.getTransaction(@Nullable TransactionDefinition var1)` 方法返回一个`TransactionStatus`对象。

- TransactionStatus 接口内容如下:
```java
    public interface TransactionStatus{
        boolean isNewTransaction(); // 是否是新的事物
        boolean hasSavepoint(); // 是否有恢复点
        void setRollbackOnly();  // 设置为只回滚
        boolean isRollbackOnly(); // 是否为只回滚
        boolean isCompleted; // 是否已完成
    }
```

#### 3.3 事务属性详解

##### 3.3.1 事务传播行为

Spring中有7中事务传播行为, 其中常用的有3种。我们用代码来理解。
```java
    Class A {
    @Transactional(propagation=propagation.xxx)
    public void aMethod {
        //do something
        B b = new B();
        b.bMethod();
        }
    }

    Class B {
        @Transactional(propagation=propagation.xxx)
        public void bMethod {
        //do something
        }
    }
```

在`TransactionDefinition`定义中包括了如下几个表示传播行为的常量：
```java
public interface TransactionDefinition {
    int PROPAGATION_REQUIRED = 0;
    int PROPAGATION_SUPPORTS = 1;
    int PROPAGATION_MANDATORY = 2;
    int PROPAGATION_REQUIRES_NEW = 3;
    int PROPAGATION_NOT_SUPPORTED = 4;
    int PROPAGATION_NEVER = 5;
    int PROPAGATION_NESTED = 6;
    ......
}
```
但是, Spring为了方便使用定义一个枚举类: `Propagation`:
```java
package org.springframework.transaction.annotation;

import org.springframework.transaction.TransactionDefinition;

public enum Propagation {

    REQUIRED(TransactionDefinition.PROPAGATION_REQUIRED),

    SUPPORTS(TransactionDefinition.PROPAGATION_SUPPORTS),

    MANDATORY(TransactionDefinition.PROPAGATION_MANDATORY),

    REQUIRES_NEW(TransactionDefinition.PROPAGATION_REQUIRES_NEW),

    NOT_SUPPORTED(TransactionDefinition.PROPAGATION_NOT_SUPPORTED),

    NEVER(TransactionDefinition.PROPAGATION_NEVER),

    NESTED(TransactionDefinition.PROPAGATION_NESTED);


    private final int value;

    Propagation(int value) {
        this.value = value;
    }

    public int value() {
        return this.value;
    }

}
```

> 谈谈你对`事务传播行为`的理解?

**事务传播行为是为了解决业务层方法之间相互调用的事务问题。** 意思就是如果业务方法A调用了业务方法B, 那么这两个方法在调用过程中, 对事务是如何操作的, 就是事务传播行为.

1 `TransactionDefinition.PROPAGATION_REQUIRED`:

使用最多的一个事务传播行为, `@Transactional`注解默认使用的传播行为。如果当前存在事务, 则加入该事务; 如果当前没有事务, 则创建一个新的事务。

具体来说:
    1. 如果外部方法没有开始事务, `PROPAGATION_REQUIRED`修饰的内部方法会开启自己的事务, 并且开启的事务相互独立, 互不干扰。
    2. 如果外部方法开启事务并且是`PROPAGATION_REQUIRED`类型, 所有的`PROPAGATION_REQUIRED`修饰的内部方法和外部方法都属于同一事务, 只要一个方法回滚, 整个事务都回滚。

举个例子：如果我们上面的`aMethod()`和`bMethod()`使用的都是`PROPAGATION_REQUIRED`传播行为的话，两者使用的就是同一个事务，只要其中一个方法回滚，整个事务均回滚。
```java
Class A {
    @Transactional(propagation=propagation.PROPAGATION_REQUIRED)
    public void aMethod {
        //do something
        B b = new B();
        b.bMethod();
    }
}

Class B {
    @Transactional(propagation=propagation.PROPAGATION_REQUIRED)
    public void bMethod {
       //do something
    }
}

```

2 `TransactionDefinition.PROPAGATION.REQUIRES_NEW`:

创建一个新的事务, 如果当前存在事务, 则把当前事务挂起。也就是不管外部方法是否开启事务, `PROPAGATION.REQUIRES_NEW`修饰的内部方法会新开启自己的事务, 并且开启的事务相互独立, 互不干扰。

举个例子：如果我们上面的`bMethod()`使用`PROPAGATION_REQUIRES_NEW`事务传播行为修饰，`aMethod`还是用`PROPAGATION_REQUIRED`修饰的话。如果`aMethod()`发生异常回滚，`bMethod()`不会跟着回滚，因为`bMethod()`开启了独立的事务。但是，如果`bMethod()`抛出了未被捕获的异常并且这个异常满足事务回滚规则的话, `aMethod()`同样也会回滚，因为这个异常被`aMethod()`的事务管理机制检测到了。

```java
Class A {
    @Transactional(propagation=propagation.PROPAGATION_REQUIRED)
    public void aMethod {
        //do something
        B b = new B();
        b.bMethod();
    }
}

Class B {
    @Transactional(propagation=propagation.REQUIRES_NEW)
    public void bMethod {
       //do something
    }
}

```

3 `TransactionDefinition.PROPAGATION_NESTED`:

如果当前存在事务, 会创建一个事务作为当前事务的嵌套事务来运行, 如果当前没有事务, 则该事务等价于`PROPAGATION_REQUIRED`。也就是说:
    1. 在外部方法未开启事务的情况下`PROPAGATION_NESTED`和`PROPAGATION_REQUIRED`作用相同, 修饰的内部方法都会开启自己事务, 并且开启的事务相互独立, 互不干扰。
    2. 如果外部方法开启事务的话, `PROPAGATION_NESTED`修饰的内部方法属于外部事务的子事务, 外部主事务回滚的话, 子事务也会回滚, 而内部子事务可以单独回滚而不影响外部主事务和其他子事务。

如果`aMethod()`回滚的话，`bMethod()`和`bMethod2()`都要回滚，而`bMethod()`回滚的话，并不会造成`aMethod()`和`bMethod()`回滚。
```java
Class A {
    @Transactional(propagation=propagation.PROPAGATION_REQUIRED)
    public void aMethod {
        //do something
        B b = new B();
        b.bMethod();
        b.bMethod2();
    }
}

Class B {
    @Transactional(propagation=propagation.PROPAGATION_NESTED)
    public void bMethod {
       //do something
    }
    @Transactional(propagation=propagation.PROPAGATION_NESTED)
    public void bMethod2 {
       //do something
    }
}

```

4 `TransactionDefinition.PROPAGATION_MANDATORY`:

如果当前存在事务, 则加入该事务; 如果当前没有事务, 则抛出异常。(mandatory: 强制性)

**如果使用下面三种事务传播行为,事务将不会发生回滚。基本上不常用, 所以就简单说下:*

5 `TransactionDefinition.PROPAGATION_SUPPORTS`: 如果当前存在事务, 则加入事务; 如果当前没有事务, 则以非事务的方式继续运行

6 `TransactionDefinition.PROPAGATION_NOT_SUPPORTED`: 以非事务方式运行, 如果当前存在事务, 则把当前事务挂起。

7 `TransactionDefinition.PROPAGATION_NEVER`: 以非事务的方式运行, 如果当前存在事务, 则抛出异常。

如果想要深入了解事务传播行为的具体表现, 可以看看这篇文章[Spring事务传播行为的理解](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247486668&idx=2&sn=0381e8c836442f46bdc5367170234abb&chksm=cea24307f9d5ca11c96943b3ccfa1fc70dc97dd87d9c540388581f8fe6d805ff548dff5f6b5b&token=1776990505&lang=zh_CN#rd)

#### 3.3.2 事务隔离级别

`TransactionDefinition`接口定义了5个隔离级别常量, 与事务传播行为相同, Spring也相应的定义了一个枚举类: `Isolation`

```java
public enum Isolation {

    DEFAULT(TransactionDefinition.ISOLATION_DEFAULT),

    READ_UNCOMMITTED(TransactionDefinition.ISOLATION_READ_UNCOMMITTED),

    READ_COMMITTED(TransactionDefinition.ISOLATION_READ_COMMITTED),

    REPEATABLE_READ(TransactionDefinition.ISOLATION_REPEATABLE_READ),

    SERIALIZABLE(TransactionDefinition.ISOLATION_SERIALIZABLE);

    private final int value;

    Isolation(int value) {
        this.value = value;
    }

    public int value() {
        return this.value;
    }

}
```

> 谈谈对事务隔离级别的理解?

事务隔离级别是为了解决并发事务所带来的问题, 比如: 脏读, 丢失修改, 不可重复读, 幻读。

- **<font color="red">脏读(Dirty Read)</font>**: 当事务A正在访问数据并对数据进行修改, 而这种修改还没有提交到数据库中, 事务B也访问这个数据然后使用这个数据进行操作, 由于这个数据还没有被事务A提交, 那么事务B读到的数据就是"脏数据", 用这个"脏数据"所做的操作可能是不正确的。

- **<font color="red">丢失修改(Lost to Modify)</font>**: 事务A读取并修改一个数据时, 事务B也读取并修改了这个数据, 导致事务A的修改丢失了。

- **<font color="red">不可重复读(Unrepeatableread)</font>**: 当事务A多次读取一个数据时, 由于事务B也访问该数据并修改了数据, 导致事务A多次读取的数据不一样。

- **<font color="red">幻读(Phantom Read)</font>**: 事务A读取几行数据, 由于并发事务B插入了一些数据, 导致之后的查询中, 事务A就会发现多了一些原本不存在的记录, 就像发生幻觉一样, 所以称为幻读。

> **<font color="red">不可重复读和幻读的区别是什么(Phantom Read)?</font>**

**不可重复读的重点是对数据修改, 幻读的重点是多表中数据新增或者删除。**

例1（同样的条件, 你读取过的数据, 再次读取出来发现值不一样了 ）：事务1中的A先生读取自己的工资为 1000的操作还没完成，事务2中的B先生就修改了A的工资为2000，导 致A再读自己的工资时工资变为 2000；这就是不可重复读。

例2（同样的条件, 第1次和第2次读出来的记录数不一样 ）：假某工资单表中工资大于3000的有4人，事务1读取了所有工资大于3000的人，共查到4条记录，这时事务2 又插入了一条工资大于3000的记录，事务1再次读取时查到的记录就变为了5条，这样就导致了幻读。

介绍一下Spring事务隔离级别:

- `TransactionDefinition.ISOLATION_DEFAULT`: 使用后端连接的数据库默认的隔离级别。MySQL默认采用`repeatable_read`隔离级别, Oracle默认采用`read_commited`隔离级别。

- `TransactionDefinition.ISOLATION_READ_UNCOMMITTED`: 最低隔离级别。一般很少使用这个隔离级别, 因为它允许读取尚未提交的数据变更, **可能导致脏读, 幻读, 不可重复读**。

- `TransactionDefinition.ISOLATION_READ_COMMITTED`: 允许读取并发事务已提交的数据, **可以阻止脏读, 但是幻读和不可重复读仍有可能发生。**

- `TransactionDefinition.ISOLATION_REPEATABLE_READ`: 对同一字段的多次读取结果都是一致的, 除非数据被本身事务所修改。**根据我们上面对 脏读, 丢失修改, 不可重复读, 幻读的解释, 这种隔离级别我们可以阻止脏读和不可重复读, 但是还是有可能发生幻读。**

- `TransactionDefinition.ISOLATION_SERIALIZABLE`: 最高的隔离级别, 完全服从ACID的隔离级别。所有事务依次逐个执行, 保证事务之间完全不会产生干扰。可以防止脏读, 不可重复读, 幻读, 丢失修改。但是这么多操作, 严重影响程序的性能。所以一般情况下, 也不会用到这个级别。

我们平常使用最多的数据库就是MySQL了, MySQL InnoDB存储引擎的默认支持的隔离级别是`REPEATABLE-READ(可重读)`(上面也介绍了这种事务隔离级别, 可以阻止脏读和不可重复读, 但是还是有可能发生幻读), 但是MySQL在使用默认隔离级别的情况下, 还采用了Next-Key Lock锁算法, 因此可以避免幻读的产生。所以MySQL的InnoDB引擎默认的隔离级别`REPEATABLE-READ(可重读)`已经完全保证事务的隔离性要求, 也就是达到了Spring中的`ISOLATION_SERIALIZABLE`级别。

因为隔离级别越低, 事务请求的锁越少, 性能也就越好。 所以InnoDB存储引擎默认使用`REPEATABLE-READ(可重读)` 既完全保证事务的隔离性要求, 又不会有性能上的损失。 

#### 3.3.3 事务超时属性

所谓事务超时, 就是指一个事务所允许执行的最长时间, 如果超多改时间限制但事务还没有完成, 则自动回滚事务。

#### 3.3.4 事务只读属性

对于只有读取数据查询的事务，可以指定事务类型为 readonly，即只读事务。只读事务不涉及数据的修改，数据库会提供一些优化手段，适合用在有多条数据库查询操作的方法中。

> 为什么一个数据查询操作还要启用事务支持?

1. 如果你一次执行单条查询语句，则没有必要启用事务支持，数据库默认支持 SQL 执行期间的读一致性。

2. 如果你一次执行多条查询语句，例如统计查询，报表查询，在这种场景下，多条查询 SQL 必须保证整体的读一致性，否则，在前条 SQL 查询之后，后条 SQL 查询之前，数据被其他用户改变，则该次整体的统计查询将会出现读数据不一致的状态，此时，应该启用事务支持

#### 3.3.5 事务回滚规则

默认情况下, 事务只有遇到运行期异常(RuntimeException的子类)时才会回滚, Error也会导致事务回滚。但是, 遇到检查型(Checked)异常时不会回滚。


### 4. @Transactional注解使用详解

#### 1) `@Transactional`的作用范围

1. **方法**: 推荐将注解使用与方法上, 但是**该注解只能应用到public方法上, 否则不生效**。
2. **类**: 注解使用在类上, 表明该注解对该类中所有的public方法都生效
3. **接口**: 不推荐在接口上使用。

#### 2) `@Transactional`常用配置参数

`@Transactional`源码如下:

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Transactional {

    @AliasFor("transactionManager")
    String value() default "";

    @AliasFor("value")
    String transactionManager() default "";

    Propagation propagation() default Propagation.REQUIRED;

    Isolation isolation() default Isolation.DEFAULT;

    int timeout() default TransactionDefinition.TIMEOUT_DEFAULT;

    boolean readOnly() default false;

    Class<? extends Throwable>[] rollbackFor() default {};

    String[] rollbackForClassName() default {};

    Class<? extends Throwable>[] noRollbackFor() default {};

    String[] noRollbackForClassName() default {};

}

```

比较常用的参数配置:

| 属性名 | 说明 |
|  ----  | ----  |
| propagation | 事务的传播行为, 默认值为REQUIRED  |
| isolation | 事务的隔离级别, 默认值为DEFAULT |
| timeout | 事务的超时时间, 默认值为-1(不超时)。如果超过该时间闲置但事务没有完成, 则自动回滚事务 |
| readOnly | 指定事务是否为只读事务, 默认为false |
| rollbackFor | 用于指定能够触发事务回滚的异常类型, 并且可以指定多个异常类型。 |

#### 3) `@Transactional`事务注解原理

我们之前已经学过, `@Transactional`的工作机制是基于AOP实现的, AOP有事使用动态代理实现的。
所以如果一个类或者类中的public方法上被标注`@Transactional`注解, 
Spring容器就会在启动的时候为其创建一个代理类, 在调用被`@Transactional`注解的public方法时, 
实际调用的是, `TransactionInterceptor`类的`invoke()`方法。这个方法的作用就是目标方法之前开启事务, 
方法执行过程中如果遇到异常时回滚事务, 方法调用完成之后提交事务。

#### 4) Spring AOP自调用问题

如果类中没有标注`@Transactional`注解的方法`methodA()`, 调用了标注了`@Transactional`注解方法`methodB()`, 那么`methodB()`方法的事务会失效。

因为Spring AOP代理只有当`@Transactional`注解的方法在类以外被调用时, Spring事务管理才会生效。

`MyService`类中的`method1()`调用`method2()`就会导致`method2()`的事务失效。解决的办法就是避免同一类中自调用或者使用AspectJ取代Spring AOP代理。
```java
@Service
public class MyService {

private void method1() {
     method2();
     //......
}
@Transactional
 public void method2() {
     //......
  }
}

```

