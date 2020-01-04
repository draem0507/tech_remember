**1、MyBatis中#{}和${}的区别详解**

{}值替换，可以防止SQL注入；${}一般是order by



**2、Mybatis是如何进行分页的？分页插件的原理是什么？**

通过pagehelper插件实现物理分页。实现原理是通过PageInterceptor对其后面的第一个SQL语句进行拦截，拼装上分页属性

**3、通常一个Xml映射文件，都会写一个Dao接口与之对应，请问，这个Dao接口的工作原理是什么？Dao接口里的方法，参数不同时，方法能重载吗？**

接口全限名+方法名拼接字符串作为key值，可唯一定位一个MappedStatement

 Dao接口的工作原理是JDK动态代理，Mybatis运行时会使用JDK动态代理为Dao接口生成代理proxy对象，代理对象proxy会拦截接口方法，转而执行MappedStatement所代表的sql，然后将sql执行结果返回。

Dao接口里的方法，是不能重载的，因为是全限名+方法名的保存和寻找策略。

#### 4、Mybatis动态sql是做什么的？都有哪些动态sql？能简述一下动态sql的执行原理不？

一个sqlsession会由静态和动态sqlnode组成。通过OGNL语言识别解析，最后组装生成一个可执行的SQL语句



#### 5、Mybatis是如何解决sql字段名与实体类名不一致导致的冲突？

通过sql语句的别名或则resultSet标签



一级、二级缓存

mybatis的一级缓存是SQLSession级别的缓存

mybatis的一级缓存是mapper级别的缓存

Mybatis是否支持延迟加载？如果支持，它的实现原理是什么？

支持；a.getB().getName()  。通过JDK的动态代理，发现getB()返回空，则进行查询，并赋值



更多

https://blog.51cto.com/14153136/2399697

