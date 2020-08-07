## <center>MyBatis 面试题集锦</center>

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

### 6. 