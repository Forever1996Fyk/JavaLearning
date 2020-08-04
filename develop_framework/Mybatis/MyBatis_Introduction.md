## <center>Mybatis 整体简介</center>

### 1. MyBatis整体架构

MyBatis整体可为三层: 基础支持层, 核心处理层, 接口层

![Mybatis_Structure](/develop_framework/Mybatis/img/Mybatis_Structure.png)

### 2. MyBatis SQL执行流程

SQL语句的执行会涉及到各个组件, 其中最重要的是`Executor`, `StatementHandler`, `ParameterHandler`和`ResultSetHandler`。

`Executor`主要负责一级缓存和二级缓存, 并提供事务管理的相关操作, `Executor`会将数据库相关操作委托给`StatementHandler`完成。
`StatementHandler`首先通过`ParmameterHandler`完成SQL的实参绑定, 然后通过`java.sql.Statement`对象执行SQL语句并得到结果集`ResultSet`, 最后通过
`ResultSetHandler`完成结果集的映射, 得到对象并返回。

![Mybatis_Process](/develop_framework/Mybatis/img/Mybatis_Process.png)

### 3. MyBatis相关连接

- [MyBatis源码GitHub地址](https://github.com/mybatis/mybatis-3)
- [MyBatis3中文文档](https://mybatis.org/mybatis-3/zh/index.html)
- [MyBatis相关博客](https://my.oschina.net/zudajun?tab=newest&catalogId=3532897)