## SqlSession相关原理

MyBatis一切的基础都从SqlSession和SqlSessionFactory接口开始。

### 1. SqlSession和SqlSessionFactory接口定义

`SqlSession`:

```java
public interface SqlSession extends Closeable {
    <T> T selectOne(String var1);
    <T> T selectOne(String var1, Object var2);
    <E> List<E> selectList(String var1);
    <E> List<E> selectList(String var1, Object var2);
    <E> List<E> selectList(String var1, Object var2, RowBounds var3);
    <K, V> Map<K, V> selectMap(String var1, String var2);
    <K, V> Map<K, V> selectMap(String var1, Object var2, String var3);
    <K, V> Map<K, V> selectMap(String var1, Object var2, String var3, RowBounds var4);
    void select(String var1, Object var2, ResultHandler var3);
    void select(String var1, ResultHandler var2);
    void select(String var1, Object var2, RowBounds var3, ResultHandler var4);
    int insert(String var1);
    int insert(String var1, Object var2);
    int update(String var1);
    int update(String var1, Object var2);
    int delete(String var1);
    int delete(String var1, Object var2);
    void commit();
    void commit(boolean var1);
    void rollback();
    void rollback(boolean var1);
    List<BatchResult> flushStatements();
    void close();
    void clearCache();
    Configuration getConfiguration();
    <T> T getMapper(Class<T> var1);
    Connection getConnection();
}
```

**从这个SqlSession接口源码就能看出, 这是最基本的C, U, R, D以及事务处理的方法。**

`SqlSessionFactory`: 从这个类的名字就能看出这是一个创建`SqlSession`对象的工厂类

```java
public interface SqlSessionFactory {
    SqlSession openSession();
    SqlSession openSession(boolean var1);
    SqlSession openSession(Connection var1);
    SqlSession openSession(TransactionIsolationLevel var1);
    SqlSession openSession(ExecutorType var1);
    SqlSession openSession(ExecutorType var1, boolean var2);
    SqlSession openSession(ExecutorType var1, TransactionIsolationLevel var2);
    SqlSession openSession(ExecutorType var1, Connection var2);
    Configuration getConfiguration();
}
```

![SqlSession&SqlSessionFactory](/develop_framework/Mybatis/img/SqlSession&SqlSessionFactory.png)

`SqlSession`实现类: `DefaultSqlSession`和`SqlSessionManager`

`SqlSessionFactory`实现类: `DefaultSqlSessionFactory`和`SqlSessionManager`

通过分析这两个接口的实现类, 可以发现, 有一个贯穿MyBatis执行流程的`Executor`接口。

### 2. SqlSessionManager源码分析(重点)

`SqlSessionManager`同时实现了`SqlSession`和`SqlSessionFactory`接口。

`org.apache.ibatis.session.SqlSessionManager.java`部分源码:

```java
public class SqlSessionManager implements SqlSessionFactory, SqlSession {

  private final SqlSessionFactory sqlSessionFactory;
  // proxy
  private final SqlSession sqlSessionProxy;
  // 保持线程局部变量SqlSession的地方
  private ThreadLocal<SqlSession> localSqlSession = new ThreadLocal<SqlSession>();

  private SqlSessionManager(SqlSessionFactory sqlSessionFactory) {
    this.sqlSessionFactory = sqlSessionFactory;
    // 这个proxy是重点
    this.sqlSessionProxy = (SqlSession) Proxy.newProxyInstance(
        SqlSessionFactory.class.getClassLoader(),
        new Class[]{SqlSession.class},
        new SqlSessionInterceptor());
  }

  public static SqlSessionManager newInstance(Reader reader) {
    return new SqlSessionManager(new SqlSessionFactoryBuilder().build(reader, null, null));
  }

  public static SqlSessionManager newInstance(Reader reader, String environment) {
    return new SqlSessionManager(new SqlSessionFactoryBuilder().build(reader, environment, null));
  }
  //...
  // 设置线程局部变量sqlSession的方法
  public void startManagedSession() {
    this.localSqlSession.set(openSession());
  }

  public void startManagedSession(boolean autoCommit) {
    this.localSqlSession.set(openSession(autoCommit));
  }
  //...
  @Override
  public <T> T selectOne(String statement, Object parameter) {
    return sqlSessionProxy.<T> selectOne(statement, parameter);
  }

  @Override
  public <K, V> Map<K, V> selectMap(String statement, String mapKey) {
    return sqlSessionProxy.<K, V> selectMap(statement, mapKey);
  }
  
private class SqlSessionInterceptor implements InvocationHandler {
    public SqlSessionInterceptor() {
        // Prevent Synthetic Access
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        final SqlSession sqlSession = SqlSessionManager.this.localSqlSession.get();
        if (sqlSession != null) {
            try {
                return method.invoke(sqlSession, args);
            } catch (Throwable t) {
                throw ExceptionUtil.unwrapThrowable(t);
            }
        } else {
            final SqlSession autoSqlSession = openSession();
            try {
                final Object result = method.invoke(autoSqlSession, args);
                autoSqlSession.commit();
                return result;
            } catch (Throwable t) {
                autoSqlSession.rollback();
                throw ExceptionUtil.unwrapThrowable(t);
            } finally {
                autoSqlSession.close();
            }
        }
    }
  }
```

变量`sqlSessionFactory`: 相当于`DefaultSqlSessionFactory`的实例, 并不是代理对象proxy。

变量`sqlSessionProxy`: 是JDK动态代理创建的代理对象proxy。动态代理的目的, 是为了通过拦截器`InvocationHandler`, 增强目标target的方法调用, 即DefaultSqlSession的代理实例。

所有的调用`sqlSessionProxy`代理对象的C, U, R, D及事务方法, 都会经过`SqlSessionInterceptor`拦截器, 并最终由目标对象target实际完成数据库操作。

**<font color=red>这里要注意: `SqlSession`的生命周期, 必须严格限制在方法内部获取Thread范围, 线程不安全, 线程之间不能共享。所以使用了`ThreadLocal<SqlSession>`保存每个线程的`SqlSession`变量副本</font>** 

