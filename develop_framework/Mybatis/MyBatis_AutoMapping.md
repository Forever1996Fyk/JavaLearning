## MyBatis 自动映射Mapper

这是面试官经常问的一个问题, MyBatis中声明一个interface接口, 没有编写任何实现类, MyBatis就能跟XML文件中的id对应并返回接口实例, 并调用接口方法返回数据库数据。

这里的原理就是: **动态代理**。之前也说过这方面的知识[动态代理](develop_framework/Spring/Spring_AOP.md)

### 1. MyBatis自动映射器Mapper的源码分析

测试类:

```java
 public static void main(String[] args) {
    SqlSession sqlSession = MybatisSqlSessionFactory.openSession();
    try {
        StudentMapper studentMapper = sqlSession.getMapper(StudentMapper.class);
        List<Student> students = studentMapper.findAllStudents();
        for (Student student : students) {
            System.out.println(student);
        }
    } finally {
        sqlSession.close();
    }
}
```

`StudentMapper.class`:

```java
public interface StudentMapper {
	List<Student> findAllStudents();
	Student findStudentById(Integer id);
	void insertStudent(Student student);
}
```


`sqlSession`的`getMapper`方法:

```java
  @Override
  public <T> T getMapper(Class<T> type) {
    return configuration.<T>getMapper(type, this);
  }

  @SuppressWarnings("unchecked")
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
      return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
  }
```

`MapperProxyFactory.class`源码:

```java
public class MapperProxyFactory<T> {

  private final Class<T> mapperInterface;
  private final Map<Method, MapperMethod> methodCache = new ConcurrentHashMap<Method, MapperMethod>();

  public MapperProxyFactory(Class<T> mapperInterface) {
    this.mapperInterface = mapperInterface;
  }

  public Class<T> getMapperInterface() {
    return mapperInterface;
  }

  public Map<Method, MapperMethod> getMethodCache() {
    return methodCache;
  }

  @SuppressWarnings("unchecked")
  protected T newInstance(MapperProxy<T> mapperProxy) {
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }

  public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }

}
```

`MapperProxy.class`部分源码:

```java
public class MapperProxy<T> implements InvocationHandler, Serializable {

  private static final long serialVersionUID = -6424540398559729838L;
  private final SqlSession sqlSession;
  private final Class<T> mapperInterface;
  private final Map<Method, MapperMethod> methodCache;

  public MapperProxy(SqlSession sqlSession, Class<T> mapperInterface, Map<Method, MapperMethod> methodCache) {
    this.sqlSession = sqlSession;
    this.mapperInterface = mapperInterface;
    this.methodCache = methodCache;
  }

  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    if (Object.class.equals(method.getDeclaringClass())) {
      try {
        return method.invoke(this, args);
      } catch (Throwable t) {
        throw ExceptionUtil.unwrapThrowable(t);
      }
    }
    // 投鞭断流
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    return mapperMethod.execute(sqlSession, args);
  }
  // ...
```

看到这就能明白, 所谓mapper接口不需要实现类就能返回数据, 就是使用了动态代理, 生成接口的代理类, 实现实例化接口并调用方法返回数据。



### 2. Mapper接口内的方法能重载吗?

例如:

```java
public User getUserById(Integer id);
public User getUserById(Integer id, String name);
```

**答案是: 不能**。

原因: 在进行动态代理时, MyBatis使用package+Mapper+method全限名作为key, 去XML内寻找唯一的sql来执行。类似: `key = x.y.UserMapper.getUserById`。如果使用重载方法, 就会产生矛盾。所以对于Mapper接口, MyBatis是禁止重载(overload)

### 3. Mapper接口如何对应XML?

这里面最核心的点就是XML配置解析器: `XMLConfigBuilder.parse()`, 这个方法会读取Mapper.xml文件, 并解析其中对应的xml元素, 封装成内部的数据结构对象。例如: `ParameterMap`, `ParameterMapping`, `ResultMap`, `ResultMapping`, `MappedStatement`, `BoundSql`等。

![MyBatis_MapperXML](/develop_framework/Mybatis/img/MyBatis_MapperXML.png)

`org.apache.ibatis.builder.xml.XMLMapperBuilder.parse()方法源码`:

```java
  private void configurationElement(XNode context) {
    try {
      String namespace = context.getStringAttribute("namespace");
      if (namespace == null || namespace.equals("")) {
        throw new BuilderException("Mapper's namespace cannot be empty");
      }
      builderAssistant.setCurrentNamespace(namespace);
      cacheRefElement(context.evalNode("cache-ref"));
      cacheElement(context.evalNode("cache"));
      parameterMapElement(context.evalNodes("/mapper/parameterMap"));
      resultMapElements(context.evalNodes("/mapper/resultMap"));
      sqlElement(context.evalNodes("/mapper/sql"));
      buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing Mapper XML. Cause: " + e, e);
    }
  }
```

通过上面的方法解析xml文件, 在调用`org.apache.ibatis.builder.xml.XMLMapperBuilder.bindMapperForNamespace()`方法, 根据namespace绑定Mapper接口上的方法。

```java
// namespace="com.mybatis3.mappers.StudentMapper"
private void bindMapperForNamespace() {
    String namespace = builderAssistant.getCurrentNamespace();
    if (namespace != null) {
        Class<?> boundType = null;
        try {
            boundType = Resources.classForName(namespace);
        } catch (ClassNotFoundException e) {
        //ignore, bound type is not required
        }
        if (boundType != null) {
            if (!configuration.hasMapper(boundType)) {
                configuration.addLoadedResource("namespace:" + namespace);
                configuration.addMapper(boundType);
            }
        }
    }
}
```

直接使用`Class.forName()`, 获取Mapper接口的名称, 找到就注册该接口, 否则什么也不做。

MyBatis的namespace有两个功能:

1. 作为命名空间使用。namespace + id, 就能找到对应的SQL。
2. 作为Mapper接口的全限名使用, 通过namespace, 就能找到对应的Mapper接口。


