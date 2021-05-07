## MyBatis 面试题集锦

### 1. #{}和${}的区别是什么?

- `${}`是Properties文件中的**变量占位符**, 在SQL内部属于一种文本替换, 比如: ${driver}会被静态替换为com.mysql.jdbc.Driver。

- `#{}`是SQL的**参数占位符**, MyBatis会将SQL中的`#{}`替换为?号, 在SQL执行前会使用`PreparedStatement`的参数设置方法, 按照顺序给SQL的?号占位符设置参数值。
而`#{item.name}`的取值方式是使用反射从参数对象中获取item对象的name属性值, 相当于param.getItem().getName()。

### 2. Xml映射文件中, 除了常见的select|insert|update|delete标签之外, 还有哪些标签?

还有很对其他的标签, `<resultMap>`, `<parameterMap>`, `<sql>`, `<include>`, `<selectKey>`, 动态SQL的标签，
trim|where|set|foreach|if|choose|when|otherwise|bind等。其中`<sql>`为SQL片段标签, 通过`<include>`标签引入SQL
片段。`<selectKey>`为不支持自增的主键生成策略标签。

### 3. 在实际项目中, 通常一个XML映射文件, 都会写一个Mapper接口与之对应, 那么这个Mapper接口的工作原理是什么? Mapper接口里的方法, 参数不同时, 方法能重载吗?

- Xml映射文件中的namespace的值, 对应着接口的全限名, 也就是package包名+Mapper类名。接口的方法名, 就是Xml映射文件中的`MappedStatement`的id值, 接口方法内的参数就是传递给SQL的参数。

- Mapper接口是没有实现类的, 当调用接口方法时, 通过接口全限名+ 方法名拼接字符串作为key值, 定位唯一的`MappedStatement`。在MyBatis中, 每一个`<select>`, `<update>`, `<delete>`, `<insert>`标签都会被解析为一个`MappedStatement`对象。

- Mapper接口中的方法是不能重载的, 因为是全限名+方法名的策略。

- Mapper接口的工作原理是JDK的动态代理, MyBatis运行时使用动态代理为Mapper接口生成代理proxy对象, 代理对象proxy会拦截接口方法, 转而执行`MappedStatement`所代表的SQL, 然后将SQL执行结果返回。

这里的`MappedStatement`对象, 就是描述`<select>`, `<update>`, `<delete>`, `<insert>`标签中的所有属性, 比如: `resultType`, `parameterType`等等, 以及SQL语句。换句话说一个`MappedStatement`对象对应着mapper.xml中的一个sql节点。

### 4. MyBatis如何进行分页? 分页插件的原理是什么?

- MyBatis使用`RowBounds`对象进行分页, 它是针对ResultSet结果集执行的内存分页, 也就是逻辑上的分页, 而并非物理分页(执行分页SQL)。当然可以在SQL内直接编写带有物理分页的参数来完成物理分页功能, 也可以使用分页插件完成物理分页。

- 分页插件的基本原理是使用MyBatis提供的插件接口, 实现自定义插件, 在插件的拦截方法内拦截待执行的SQL, 然后重写SQL, 添加对应的物理分页语句和物理分页参数。
例如: `select * from student`, 拦截SQL后重写为: `select t.* from (select * from student) t limit 0, 10`

### 5. MyBatis动态SQL是做什么的? 都有哪些动态SQL? 简述一下动态SQL的执行原理?

- MyBatis动态SQL可以在XML映射文件内, 以标签的形式编写动态SQL, 完成逻辑判断和动态拼接SQL的功能。
- 动态SQL标签: `trim|where|set|foreach|if|choose|when|otherwise|bind`。
- 执行原理: 使用OGNL从SQL参数对象中计算表达式的值, 根据表达式的值动态拼接SQL。

### 6. MyBatis如何将SQL执行结果封装为目标对象并返回? 有哪些映射形式?

- 使用`<resultMap>`标签, 自定义列名和对象属性名之间的映射关系。
- 使用SQL语言的别名功能, 将列别名定义为对象属性名, 例如`T_NAME as name`。而且不区分大小写, MyBatis会忽略别名大小写

MyBatis通过反射创建对象, 同时使用反射给对象的属性赋值并返回, 如果找不到映射关系的属性, 无法完成赋值。

### 7. MyBatis是否支持延迟加载? 如果支持, 实现原理是什么?

MyBaits仅仅支持对关联对象进行懒加载, 即对子查询的子对象进行懒加载。

MyBatis仅支持`association`关联对象和`collection`关联集合对象的延迟加载。`association`指的是一对一, `collection`指的是一对多查询。在MyBatis中可以配置是否启用延迟加载`lazyLoadingEnabled=true/false`。

延迟加载的原理: 使用cglib创建目标对象的代理, 当调用目标方法时, 进入拦截器方法, 如果发现这个关联对象的值为null, 就会单独的查询关联对象的SQL, 然后调用set方法, 注入关联对象属性的值, 然后在调用get方法。

### 8. MyBatis的XML映射文件中, 不同的XML映射文件, id是否可以重复?

对于不同的XML映射文件, 如果配置了namespace, 那么id是可以重复的; 如果没有配置namespace, id不能重复, 因为namespace不是必须的, 只是最佳使用方法而已。

原因: MyBatis是将namespace+id名作为Map<String, MappedStatement>的key使用, 如果没有namespace, 就只有id, 那么id重复会导致数据相互覆盖。有了namespace, 且namespace不重复, 那么id就可以重复。

### 9. MyBatis如何执行批处理?

使用`BatchExecutor`完成批处理

### 10. MyBaits有哪些Executor执行器?

MyBatis有三种基本的Executor执行器, `SimpleExecutor`, `ReuseExecutor`, `BatchExecutor`。这些Executor都严格限制在SqlSession生命周期范围内。

- **`SimpleExecutor`**: 每执行一次update或select, 就开启一个Statement对象, 用完立刻关闭Statement对象。

- **`ReuseExecutor`**: 执行update或select, 以SQL作为key查找Statement对象, 存在就使用, 不存在就创建, 用完后不关闭Statement对象, 而是放置在Map<String, Statement>内。**换句话说, 就是重复使用Statement对象。**

- **`BatchExecutor`**: 执行update(JDBC批处理不支持select), 将所有SQL都添加到批处理中(addBatch()), 等待统一执行(executeBatch())。批处理中缓存了多个Statement对象, 每个Statement对象都是addBatch()完毕后, 等待依次执行executeBatch()批处理。

### 11. MyBatis如何指定一种Executor执行器?

在MyBatis配置属性中, 可以指定默认的ExecutorType执行器类型, 也可以手动给`DefaultSqlSessionFactory`的创建`SqlSession`的方法传递ExecutorType类型参数。

### 12. 简述MyBatis的XML映射文件和MyBatis内部数据结构之间的映射关系?

在XML映射文件中,
- `<parameterMap>`标签会被解析为`ParameterMap`对象, 每个子元素被解析成`ParameterMapping`对象。

- `<resultMap>`标签会被解析为`ResultMap`对象, 每个子元素会被解析为`ResultMapping`对象。

- 每一个`<select>, <insert>, <update>, <delete>`标签会被解析为`MappedStatement`对象, 标签内的SQL会被解析为`BoundSQL`对象。

### 13. 为什么说MyBatis是半自动的ORM映射工具? 它与全自动的区别在哪里?

- MyBatis在查询关联对象或者关联集合对象时, 需要手动编写SQL来完成, 所以称为半自动ORM映射框架
- Hibernate查询关联对象或者关联集合对象, 可以根据对象关系模型直接获取数据。