## <center>Spring Transaction</center>

在日常开发中, 只要涉及到业务逻辑的开发都一定要去深入理解事务, 这不仅是面试高频提问点, 也是实际开发中必备技能。所以很重要!!!

我们既要知道事务的概念, 还要知道事务在Spring中是如何支持的。

### 1. 什么是事务?

**事务是逻辑上的一组操作, 要么都执行, 要么都不执行。**

系统中的每个业务方法可能包括了多个原子性的数据库操作, 这些操作是有依赖的, 它们要么都执行, 要么都不执行。

而且事务能否生效的关键在于数据库引擎是否支持事务。例如常用的MySQL数据库默认使用支持事务的`innodb`引擎。
但是, 如果数据库引擎变为`myisam`, 那么程序也无法再支持事务。

### 2. 事务的特性(ACID)了解吗?

![事务特性](/develop_framework/Spring/img/事务特性.png)
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

- 事务属性详解

> 谈谈你对`事务传播行为`的理解?

**事务传播行为是为了解决业务层方法之间相互调用的事务问题。** Spring中有7中事务传播行为, 其中常用的有3种。我们用代码来理解。
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

1. `TransactionDefinition.PROPAGATION_REQUIRED`:

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

2. `TransactionDefinition.PROPAGATION.REQUIRES_NEW`:

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

3. `TransactionDefinition.PROPAGATION_NESTED`:

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

4. `TransactionDefinition.PROPAGATION_MANDATORY`:

如果当前存在事务, 则加入该事务; 如果当前没有事务, 则抛出异常。(mandatory: 强制性)

**如果使用下面三种事务传播行为,事务将不会发生回滚。基本上不常用, 所以就简单说下:*

5. `TransactionDefinition.PROPAGATION_SUPPORTS`: 如果当前存在事务, 则加入事务; 如果当前没有事务, 则以非事务的方式继续运行
6. `TransactionDefinition.PROPAGATION_NOT_SUPPORTED`: 以非事务方式运行, 如果当前存在事务, 则把当前事务挂起。
7. `TransactionDefinition.PROPAGATION_NEVER`: 以非事务的方式运行, 如果当前存在事务, 则抛出异常。



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