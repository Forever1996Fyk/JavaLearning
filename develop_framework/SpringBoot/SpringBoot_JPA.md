## <center>Spring Data JPA</center>

首先JPA是一个比较容易上手的框架, 但是涉及的东西还是很多的。而且我个人是不喜欢使用JPA或者Hibernate这样的高度封装的ORM框架, 虽然很方便, 虽然也有自定义SQL使用, 但是从SQL的阅读上, 书写上, 我感觉都是XML更好一点。而且我认为JPA不适合表结构比较复杂的项目, 因为一旦需求比较复杂, 就会涉及到表连接, 而JPA对于表连接的操作是比较少的。

> 如何学习Spring Data JPA?

直接参考[官方文档](https://spring.io/projects/spring-data-jpa)。文档已经非常详细了, 所以如果是新手也可以直接阅读。因为本人并没有研究过JPA的原理, 仅仅只是会用。 所以这里记录一下JPA的用法。

### 1. 相关依赖

```xml
 <dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### 2. 配置数据库连接信息和JPA配置

这里的配置属性有一个需要了解一下, `spring.jpa.hibernate.ddl-auto=create`这个配置。

这个属性常用的有四种:

1. `create`: 每次重新启动项目都会重新创建表结构, 会导致数据丢失
2. `create-drop`: 每次启动项目创建表结构, 关闭项目删除表结构
3. `update`: 每次启动项目会更新表结构
4. `validate`: 验证表结构, 不会数据库进行任何更改

但是这里要注意的是, **在生产环境下一定不要使用dll自动生成表结构, 一般推荐手写**。

```properties

spring.datasource.url=jdbc:mysql://localhost:3306/springboot_jpa?useSSL=false&serverTimezone=CTT
spring.datasource.username=root
spring.datasource.password=123456
# 打印出 sql 语句
spring.jpa.show-sql=true
spring.jpa.hibernate.ddl-auto=create
spring.jpa.open-in-view=false
# 创建的表的 ENGINE 为 InnoDB
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQL55Dialect

```

### 3. 实体类

我们在实体类上添加`@Entity`注解代表它是数据库持久化类, 而且需要配置主键id。

```java
import lombok.Data;
import lombok.NoArgsConstructor;

import javax.persistence.Column;
import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;

@Entity
@Data
@NoArgsConstructor
public class Person {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @Column(unique = true)
    private String name;
    private Integer age;

    public Person(String name, Integer age) {
        this.name = name;
        this.age = age;
    }

}
```

如果运行项目, 查看数据如果发现控制台打印出创建表的sql语句, 并且数据库中表真的被创建出来的话, 说明Spring Data JPA基本上配置成功。

### 4. 创建操作数据库的Repository接口

```java
@Repository
public interface PersonRepository extends JpaRepository<Person, Long> {
}
```

对于这段代码, 首先添加了`@Repository`注解, 代表是数据访问接口, 而且继承了`JpaRepository<Person, Long>`接口, 这个接口定义了很多有关数据库的操作方法。如下:

```java
@NoRepositoryBean
public interface JpaRepository<T, ID> extends PagingAndSortingRepository<T, ID>, QueryByExampleExecutor<T> {
    List<T> findAll();

    List<T> findAll(Sort var1);

    List<T> findAllById(Iterable<ID> var1);

    <S extends T> List<S> saveAll(Iterable<S> var1);

    void flush();

    <S extends T> S saveAndFlush(S var1);

    void deleteInBatch(Iterable<T> var1);

    void deleteAllInBatch();

    T getOne(ID var1);

    <S extends T> List<S> findAll(Example<S> var1);

    <S extends T> List<S> findAll(Example<S> var1, Sort var2);
}
```

所以我们只要继承了`JpaRepository<T, ID>`就具有了JPA为我们提供好的增删盖茶, 分页查询以及根据条件查询等方法。

#### 4.1 JPA具体方法实战(这里可以直接用到实际项目中)

##### 1) 增删改查

**1. 保存用户到数据库**

```java
Person person = new Person("YuKai Fan", 23);
personRepository.save(person);
```

**2. 根据id查找用户**

```java
Person person = personRepository.findById(id).get();
```

**3. 根据id删除用户**

```java
personRepository.deleteById(id);
```

**4. 更新用户**

更新操作是比较麻烦的, 用的也是`save()`方法。更新操作要注意的是, 

- 首先id是必须要传的
- 如果只传递修改的数据, 那么那些未修改的数据会被直接置为null, 所以这里需要进行数据的处理, 通过Spring的BeanUtils类, 进行属性copy

```java
Person person = personRepository.findById(entity.getId()).get();

// getNullField方法是获取属性中为空的字段 源码我也贴在下面
BeanUtils.copyProperties(entity, person, BeanUtil.getNullField(entity));

personRepository.save(person);
```

`BeanUtil.getNullField`:

```java

/**
    * 获取属性中为空的字段
    *
    * @param target
    * @return
    */
public static String[] getNullField(Object target) {
    BeanWrapper beanWrapper = new BeanWrapperImpl(target);
    PropertyDescriptor[] propertyDescriptors = beanWrapper.getPropertyDescriptors();
    Set<String> notNullFieldSet = new HashSet<>();
    if (propertyDescriptors.length > 0) {
        for (PropertyDescriptor p : propertyDescriptors) {
            String name = p.getName();
            Object value = beanWrapper.getPropertyValue(name);
            if (Objects.isNull(value)) {
                notNullFieldSet.add(name);
            }
        }
    }
    String[] notNullField = new String[notNullFieldSet.size()];
    return notNullFieldSet.toArray(notNullField);
}
```

**5. 根据条件查询用户**
 
这里需要根据JPA提供的语法自定义, 自己在`PersonRepository `中按照JPA的规则进行定义:

 ```java

 // 根据名称查找Person
 Optional<Person> findByName(String name);

 // 找到年龄大于某个值的人
 List<Person> findByAgeGreaterThan(int age);
 ```

**6. 集合查询**

这里是进行条件的集合查询, 方式有很多。这里我是封装了条件查询的java代码跟分页代码一起用的。


一般情况下, 根据《阿里巴巴Java开发手册》返回值是不能用Map的, 而请求值在大于两个的情况下也是不能用Map的, 但是我为了方便就都用了Map作为参数, 以及返回结果。

这里根据自己的项目情况使用。

```java

//不分页
public PageResult<Map<String, Object>> getPersions(int pageNum, int pageSize, Map<String, Object> map) {
    StringBuilder sql = new StringBuilder();
    sql.append(" SELECT a.id id, a.name name, a.age age");
    sql.append(" from tb_person a where 1 = 1");

    Map<String, Object> params = Maps.newHashMap();
    if (StringUtils.isNotBlank(map.get("name"))) {
        sql.append(" and a.name like :name");
        params.put("name", "%" + map.get("name") + "%");
    }
    if (StringUtils.isNotBlank(map.get("age"))) {
        sql.append(" and a.age = :age");
        params.put("status", map.get("age"));
    }
    sql.append(" and a.status = 1");
    sql.append(" order by a.create_time desc");

    //pageRepository 源码我会贴在下面
    return pageRepository.search(pageNum, pageSize, sql.toString(), params);
}

//分页
public List<Map<String, Object>> getPersions(Map<String, Object> map) {
    StringBuilder sql = new StringBuilder();
    sql.append(" SELECT a.id id, a.name name, a.age age");
    sql.append(" from tb_person a where 1 = 1");

    Map<String, Object> params = Maps.newHashMap();
    if (StringUtils.isNotBlank(map.get("name"))) {
        sql.append(" and a.name like :name");
        params.put("name", "%" + map.get("name") + "%");
    }
    if (StringUtils.isNotBlank(map.get("age"))) {
        sql.append(" and a.age = :age");
        params.put("status", map.get("age"));
    }
    sql.append(" and a.status = 1");
    sql.append(" order by a.create_time desc");

    //pageRepository 源码我会贴在下面
    return pageRepository.search(sql.toString(), params);
}
```

**分页返回结果`PageResult`类:**

```java
@Data
public class PageResult<T> {
    //当前页
    private Integer currentPage;
    //记录总数
    private Integer total;
    //当前页大小
    private Integer pageSize;
    //总页数
    private Integer totalPage;
    //数据
    private List<T> rows;

    public PageResult(Integer currentPage, Integer total, Integer pageSize) {
        super();
        this.currentPage = currentPage;
        this.total = total;
        this.pageSize = pageSize;
        if (this.currentPage == null) {
            this.currentPage = 1;
        }

        if (this.pageSize == null) {
            this.pageSize = 30;
        }

        //计算总页数
        this.totalPage = (this.total + this.pageSize - 1)/this.pageSize;
        //判断当前页数是否超过范围
        //不能小于1
        if (this.currentPage < 1) {
            this.currentPage = 1;
        }
        if (this.currentPage > this.totalPage) {
            this.currentPage = this.totalPage;
        }
    }

    public int getStart() {
        return (this.currentPage - 1) * this.pageSize;
    }
}
```

**集合查询返回结果处理`PageRepository`类:**

```java
@Component
@Transactional
public class PageRepository {
    @PersistenceContext
    private EntityManager entityManager;

    /**
     * 分页查询
     * @param pageNumber
     * @param pageSize
     * @param sql
     * @param params
     * @return
     */
    public PageResult<Map<String, Object>> search(Integer pageNumber, Integer pageSize, String sql, Map<String, Object> params) {
        String countSql = getCountSql(sql);
        //查询结果总条数
        int total = Integer.parseInt(getQueryWithParameters(entityManager.createNativeQuery(countSql), params).getSingleResult().toString());
        // 初始化分页返回
        PageResult<Map<String, Object>> pageResult = new PageResult<>(pageNumber, total, pageSize);
        if (total == 0) {
            return pageResult;
        }

        Query query = getQueryWithParameters(entityManager.createNativeQuery(sql), params).unwrap(NativeQueryImpl.class)
                .setResultTransformer(Transformers.ALIAS_TO_ENTITY_MAP);
        query.setFirstResult(pageResult.getStart());
        query.setMaxResults(pageSize);
        List<Map<String, Object>> list = query.getResultList();
        pageResult.setRows(list);
        return pageResult;
    }

    /**
     * 不分页查询
     * @param sql
     * @param params
     * @return
     */
    public List<Map<String, Object>> search(String sql, Map<String, Object> params) {
        Query query = getQueryWithParameters(entityManager.createNativeQuery(sql), params).unwrap(NativeQueryImpl.class)
                .setResultTransformer(Transformers.ALIAS_TO_ENTITY_MAP);
        List<Map<String, Object>> list = query.getResultList();
        return list;
    }

    /**
     * 不分页查询(无参数)
     * @param sql
     * @return
     */
    public List<Map<String, Object>> search(String sql) {
        Query query = entityManager.createNativeQuery(sql).unwrap(NativeQueryImpl.class)
                .setResultTransformer(Transformers.ALIAS_TO_ENTITY_MAP);
        List<Map<String, Object>> list = query.getResultList();
        return list;
    }

    /**
     * 获取统计sql
     * @param sql
     * @return
     */
    private String getCountSql(String sql) {
        StringBuilder countSql = new StringBuilder(" select count(*) from (").append(sql).append(") t");
        return countSql.toString();
    }

    /**
     * 拼接参数
     * @param query
     * @param params
     * @return
     */
    private Query getQueryWithParameters(Query query, Map<String, Object> params) {
        if (!CollectionUtils.isEmpty(params)) {
            for (Map.Entry<String, Object> entry : params.entrySet()) {
                query.setParameter(entry.getKey(), entry.getValue());
            }
        }

        return query;
    }
}
```

#### 4.2 自定义SQL语句实战

很多时候我们需要用到表连接, 但是我觉得在java代码中写SQL, 它的格式各方面都不是特别方便, 尤其是比较复杂的SQL写起来各种转义符, 简直要人命, 这也是我不喜欢用JPA的原因。
但是为了学这门技术, 还是要了解一下。

```java

//根据name查找Person
@Query("select p from Person p where p.name = :name")
Optional<Person> findByNameCustomeQuery(@Param("name") String name);

// Person部分属性查询, 避免`select *` 操作
@Query("select p.name from Person p where p.id = :id")
String findPersonNameById(@Param("id") Long id);

// 根据id更新Person name
@Modifying
@Query("update Person p set p.name = ?1 where p.id = ?2")
void updatePersonNameById(String name, Long id);
```

看到这大家应该知道, 这跟上面的集合查询参数的方法是差不多的, 都是通过自定义SQL来做的, 只不过这里是通过注解将SQL映射到接口中, 上面的方法是通过java代码进行操作。








