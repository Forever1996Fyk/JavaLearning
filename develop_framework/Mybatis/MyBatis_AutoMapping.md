## <center>MyBatis 自动映射Mapper</center>

这是面试官经常问的一个问题, MyBatis中声明一个interface接口, 没有编写任何实现类, MyBatis就能跟XML文件中的id对应并返回接口实例, 并调用接口方法返回数据库数据。

这里的原理就是: **动态代理**。之前也说过这方面的知识[动态代理](develop_framework/Spring/Spring_AOP.md)