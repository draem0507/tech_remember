昨天，笔者在一篇面经中突然看到阿里的这样一道面试题：

> Mybatis中的Dao接口和XML文件里的SQL是如何建立关系的？ 如果有两个XML文件和这个DAO建立关系，岂不是冲突了？

如果你看过笔者关于Mybatis源码分析的往期博文，相信你肯定可以给出一个不错的答案。

但鉴于系列文章篇幅较大，而且重点是源码部分的解读，所以笔者想再针对这个问题，再梳理下整个流程。

本文配合下列文章，食用更佳。

Mybatis源码分析（二）XML的解析和Annotation的支持 - 掘金juejin.im



Mybatis源码分析（四）mapper接口方法是怎样被调用到的 - 掘金juejin.im



Mybatis源码分析（五）探究SQL语句的执行过程 - 掘金juejin.im





## 一、解析XML

首先，Mybatis在初始化`SqlSessionFactoryBean`的时候，找到`mapperLocations`路径去解析里面所有的XML文件，这里我们重点关注两部分。



### 1、创建SqlSource

Mybatis会把每个SQL标签封装成SqlSource对象。然后根据SQL语句的不同，又分为动态SQL和静态SQL。其中，静态SQL包含一段String类型的sql语句；而动态SQL则是由一个个SqlNode组成。

![img](https://pic4.zhimg.com/80/v2-7c926984c4b31f1ff24624b7137ea327_hd.jpg)

假如我们有这样一个SQL：

```java
<select id="getUserById" resultType="user">
	select * from user 
	<where>
		<if test="uid!=null">
			and uid=#{uid}
		</if>
	</where>
</select>	
```

它对应的SqlSource对象看起来应该是这样的：

![img](https://pic1.zhimg.com/80/v2-62b07bd1e8c7a8e6874c0d264147cee4_hd.jpg)

### 2、创建MappedStatement

XML文件中的每一个SQL标签就对应一个MappedStatement对象，这里面有两个属性很重要。

- id

全限定类名+方法名组成的ID。

- sqlSource

当前SQL标签对应的SqlSource对象。

创建完`MappedStatement`对象，将它缓存到`Configuration#mappedStatements`中。
Configuration对象，我们知道它就是Mybatis中的大管家，基本所有的配置信息都维护在这里。把所有的XML都解析完成之后，Configuration就包含了所有的SQL信息。

![img](https://pic4.zhimg.com/80/v2-8bad0d4a1032687df9c3cfe8b14f9f2f_hd.jpg)

到目前为止，XML就解析完成了。看到上面的图示，聪明如你，也许就大概知道了。当我们执行Mybatis方法的时候，就通过`全限定类名+方法名`找到`MappedStatement`对象，然后解析里面的SQL内容，执行即可。



## 二、Dao接口代理

我们的Dao接口并没有实现类，那么，我们在调用它的时候，它是怎样最终执行到我们的SQL语句的呢？

首先，我们在Spring配置文件中，一般会这样配置：

```text
<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
	<property name="basePackage" value="com.viewscenes.netsupervisor.dao" />
	<property name="sqlSessionFactoryBeanName" value="sqlSessionFactory"></property>
</bean>
```

或者你的项目是基于SpringBoot的，那么肯定也见过这种:MapperScan("com.xxx.dao")

它们的作用是一样的。将包路径下的所有类注册到Spring Bean中，并且将它们的beanClass设置为`MapperFactoryBean`。

有意思的是，`MapperFactoryBean`实现了`FactoryBean`接口，俗称工厂Bean。那么，当我们通过`@Autowired`注入这个Dao接口的时候，返回的对象就是`MapperFactoryBean`这个工厂Bean中的`getObject()`方法对象。

那么，这个方法干了些什么呢？
简单来说，它就是通过JDK动态代理，返回了一个Dao接口的代理对象，这个代理对象的处理器是`MapperProxy`对象。所有，我们通过`@Autowired`注入Dao接口的时候，注入的就是这个代理对象，我们调用到Dao接口的方法时，则会调用到`MapperProxy`对象的invoke方法。

**对这块内容不太能理解的朋友，可以先看看Spring中的FactoryBean 和 JDK动态代理相关知识。**



曾经有个朋友问过这样一个问题：

> 对于有实现的dao接口，mapper还会用代理么？

**答案是肯定的，只要你配置了MapperScan，它就会去扫描，然后生成代理。但是，如果你的dao接口有实现类，并且这个实现类也是一个Spring Bean，那就要看你在Autowired的时候，去注入哪一个了。**

具体什么意思呢？我们来到一个例子。
如果我们给userDao搞一个实现类，并且把它注册到Spring。

```text
@Component
public class UserDaoImpl implements UserDao{
	public List<User> getUserList(Map<String,Object> map){
		return new ArrayList<User>();
	}
}
```

然后我们在Service方法中，注入这个userDao。猜猜会发生什么？

```text
@Service
public class UserServiceImpl implements UserService{

	@Autowired
	UserMapper userDao1;

	public List<User> getUserList(Map<String,Object> map) {
		return userDao1.getUserList(map);
	}
}
```

也许你已经猜到了，是的，它会启动报错。因为在注入的时候，找到了两个UserMapper的实例对象。日志是这样的：

```
No qualifying bean of type [com.viewscenes.netsupervisor.dao.UserDao] is defined: expected single matching bean but found 2: userDaoImpl,userDao
```


当然了，也许我们的命名不太规范。其实我们通过名字注入就可以了，像这样： `@Autowired UserMapper userDao;` 或者 `@Autowired UserMapper userDaoImpl;` 再或者给你其中一个Bean加上@Primary注解。

具体原理，请参看笔者文章：

彻底搞明白Spring中的自动装配和Autowired - 掘金juejin.im



说着说着可能扯远了，我们继续回到Mybatis。那么，目前为止，我们通过Dao接口也有了代理实现，所以就可以执行到它里面的方法了。



## 三、执行

如上所述，当我们调用Dao接口方法的时候，实际调用到代理对象的invoke方法。 在这里，实际上调用的就是SqlSession里面的东西了。

```text
public class DefaultSqlSession implements SqlSession {

	public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
		try {
			MappedStatement ms = configuration.getMappedStatement(statement);
			return executor.query(ms, 
				wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
		}
	}
}
```

看到以上代码，说明我们想的不错。它就是通过statement`全限定类名+方法名`拿到MappedStatement 对象，然后通过执行器Executor去执行具体SQL并返回。

![img](https://pic3.zhimg.com/80/v2-7eb3a1d550cbc34b73b3940beddb806a_hd.jpg)

## 四、总结

到这里，再回到开头我们提到的问题，也许你能更好的回答。同时笔者觉得，这道题目，如果你覆盖到以下几个关键词，面试官可能会觉得很满意。

- SqlSource以及动态标签SqlNode
- MappedStatement对象
- Spring 工厂Bean 以及动态代理
- SqlSession以及执行器

那么，针对第二个问题：如果有两个XML文件和这个Dao建立关系，岂不是冲突了？
答案也是显而易见，不管有几个XML和Dao建立关系，只要保证`namespace+id`唯一即可。