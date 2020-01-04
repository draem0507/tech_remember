# **MyBatis注解**

MyBatis可以利用SQL映射文件来配置，也可以利用Annotation来设置。MyBatis提供的一些基本注解如下表所示。

| 注解                                                         | 目标   | 相应的XML                                         | 描述                                                         |
| ------------------------------------------------------------ | ------ | ------------------------------------------------- | ------------------------------------------------------------ |
| @CacheNamespace                                              | **类** | <cache>                                           | 为给定的命名空间（比如类）配置缓存。属性：implemetation,eviction,flushInterval , size 和 readWrite 。 |
| @CacheNamespaceRef                                           | **类** | <cacheRef>                                        | 参照另外一个命名空间的缓存来使用。属性：value，也就是类的完全限定名。 |
| @ConstructorArgs                                             | 方法   | <constructor>                                     | 收集一组结果传递给对象构造方法。属性：value，是形式参数的数组 |
| @Arg                                                         | 方法   | <arg><idArg>                                      | 单独的构造方法参数，是ConstructorArgs  集合的一部分。属性：id,column,javaType，typeHandler。id属性是布尔值，来标识用于比较的属性，和<idArg>XML 元素相似 |
| @TypeDiscriminator                                           | 方法   | <discriminator>                                   | 一组实例值被用来决定结果映射的表现。属性：Column, javaType ,  jdbcType typeHandler，cases。cases属性就是实例的数组。 |
| @Case                                                        | 方法   | <case>                                            | 单独实例的值和它对应的映射。属性：value  ，type ，results 。Results 属性是结果数组，因此这个注解和实际的ResultMap 很相似，由下面的  Results注解指定 |
| @Results                                                     | 方法   | <resultMap>                                       | 结果映射的列表，包含了一个特别结果列如何被映射到属性或字段的详情。属性：value ，是Result注解的数组 |
| @Result                                                      | 方法   | <result><id>                                      | 在列和属性或字段之间的单独结果映射。属性：id ，column ， property ，javaType ，jdbcType ，type Handler ，one，many。id 属性是一个布尔值，表示了应该被用于比较的属性。one 属性是单独的联系，和 <association> 相似，而many 属性是对集合而言的，和<collection>相似。 |
| @One                                                         | 方法   | <association>                                     | 复杂类型的单独属性值映射。属性：select，已映射语句（也就是映射器方法）的完全限定名，它可以加载合适类型的实例。注意：联合映射在注解 API中是不支持的。 |
| @Many                                                        | 方法   | <collection>                                      | 复杂类型的集合属性映射。属性：select，是映射器方法的完全限定名，它可加载合适类型的一组实例。注意：联合映射在 Java注解中是不支持的。 |
| @Options                                                     | 方法   | 映射语句的属性                                    | 这个注解提供访问交换和配置选项的宽广范围，它们通常在映射语句上作为属性出现。而不是将每条语句注解变复杂，Options 注解提供连贯清晰的方式来访问它们。属性：useCache=true，flushCache=false，resultSetType=FORWARD_ONLY，statementType=PREPARED，fetchSize= -1，timeout=-1 ，useGeneratedKeys=false ，keyProperty=”id“。理解Java 注解是很重要的，因为没有办法来指定“null ”作为值。因此，一旦你使用了 Options注解，语句就受所有默认值的支配。要注意什么样的默认值来避免不期望的行为 |
| @Insert@Update@Delete                                        | 方法   | <insert><update><delete>                          | 这些注解中的每一个代表了执行的真实 SQL。它们每一个都使用字符串数组（或单独的字符串）。如果传递的是字符串数组，它们由每个分隔它们的单独空间串联起来。属性：value，这是字符串数组用来组成单独的SQL语句 |
| @InsertProvider@UpdateProvider@DeleteProvider@SelectProvider | 方法   | <insert><update><delete><select>允许创建动态SQL。 | 这些可选的SQL注解允许你指定一个类名和一个方法在执行时来返回运行的SQL。基于执行的映射语句， MyBatis会实例化这个类，然后执行由 provider指定的方法. 这个方法可以选择性的接受参数对象作为它的唯一参数，但是必须只指定该参数或者没有参数。属性：type，method。type 属性是类的完全限定名。method  是该类中的那个方法名。 |
| @Param                                                       | 参数   | N/A                                               | 当映射器方法需多个参数，这个注解可以被应用于映射器方法参数来给每个参数一个名字。否则，多参数将会以它们的顺序位置来被命名。比如#{1}，#{2} 等，这是默认的。使用@Param(“person”)，SQL中参数应该被命名为#{person}。 |

 

这些注解都是运用到传统意义上映射器接口中的方法、类或者方法参数中的。

# **使用方法**

## **1.insert方法**

我们可以使用@Insert注解来定义一个INSERT映射语句：



使用了@Insert注解的方法将返回insert语句执行后影响的行数。

## **2.自动生成主键**

使用@Options注解的userGeneratedKeys 和keyProperty属性让数据库产生auto_increment（自增长）列的值，然后将生成的值设置到输入参数对象的属性中。



这里stu_id主键列值将会通过MySQL数据库自动生成。并且生成的值将会被设置到student对象的studId属性中。

 



## **3.update方法**

可以使用@Update注解来定义一个UPDATE映射语句，如下所示：



使用了@Update的方法将会返回执行了update语句后影响的行数。

## **4.delete方法**

可以使用@Delete  注解来定义一个DELETE映射语句，如下所示：



使用了@Delete的方法将会返回执行了update语句后影响的行数。

## **5.select方法**

可以使用@ Select注解来定义一个SELECT映射语句。



## **6.结果映射**

可以将查询结果通过别名或者是@Results注解与JavaBean属性映射起来。





## **7.ResultMap**

6中的两个@Results配置完全相同，但是必须得重复它。这里有一个解决方法。我们可以创建一个映射器Mapper配置文件， 然后配置<resultMap>元素，然后使用@ResultMap注解引用此<resultMap>。

在StudentMapper.xml中定义一个ID为StudentResult的<resultMap>。





然后使用@ResultMap引用名为StudentResult的resultMap。



## **8.一对一映射**

MyBatis提供了@One注解来使用嵌套select语句（Nested-Select）加载一对一关联查询数据。让我们看看怎样使用@One注解获取学生及其地址信息。





这里我们使用了@One注解的select属性来指定一个使用了完全限定名的方法上，该方法会返回一个Address对象。使用column=”addr_id”,则STUEDNTS表中列addr_id的值将会作为输入参数传递给findAddressById()方法。如果@OneSELECT查询返回了多行结果，则会抛出TooManyResultsException异常。



我们可以通过基于XML的映射器配置，使用嵌套结果ResultMap来加载一对一关联的查询。而MyBatis3.2.2版本，并没有对应的注解支持。但是我们可以在映射器Mapper配置文件中配置<resultMap>并且使用@ResultMap注解来引用它。



在StudentMapper.xml中配置<resultMap>，如下所示：







## **9.一对多映射**

MyBatis提供了@Many注解，用来使用嵌套Select语句加载一对多关联查询。



这里我们使用了@Many注解的select属性来指向一个完全限定名称的方法，该方法将返回一个List<Course>对象。使用column=”tutor_id”，TUTORS表中的tutor_id列值将会作为输入参数传递给findCoursesByTutorId()方法。

这里使用了@Many注解的select属性来指向一个完全限定名称的方法，该方法将返回一个List<Course>对象。使用column=”tutor_id”，TUTORS表中的tutor_id列值将会作为输入参数传递给findCoursesByTutorId()方法。

我们可以通过基于XML的映射器配置，使用嵌套结果ResultMap来加载一对多关联的查询。而MyBatis3.2.2版本，并没有对应的注解支持。但是我们可以在映射器Mapper配置文件中配置<resultMap>并且使用@ResultMap注解来引用它。

在TutorMapper.xml中配置<resultMap>,如下所示：





# **动态SQL**

有时候我们需要根据输入条件动态地构建SQL语句。MyBatis提供了各种注解如@InsertProvider，@UpdateProvider

@DeleteProvider和@SelectProvider，来帮助构建动态SQL语句，然后让MyBatis执行这些SQL语句。

## **1.@SelectProvider**

现在来看一个使用@SelectProvider注解来创建一个简单的SELECT映射语句的例子。

创建一个TutorDynaSqlProvider.java类，以及findTutorByIdSql()方法，如下所示：





然后创建一个映射语句



这里我们使用了@SelectProvider来指定了一个类，及其内部的方法，用来提供需要执行的SQL语句。

但是使用字符串拼接的方法唉构建SQL语句是非常困难的，并且容易出错。所以MyBaits提供了一个SQL工具类不使用字符串拼接的方式，简化构造动态SQL语句。

现在，来看看如何使用org.apache.ibatis.jdbc.SQL工具类来准备相同的SQL语句。



SQL工具类会处理以合适的空格前缀和后缀来构造SQL语句。

动态**SQL provider**方法可以接收以下其中一种参数：

- ​           无参数
- ​           和映射器Mapper接口的方法同类型的参数
- ​           java.util.Map 

如果SQL语句的准备不取决于输入参数，你可以使用不带参数的SQL Provider方法。

例如：



这里我们没有使用输入参数构造SQL语句，所以它可以是一个无参方法。



如果映射器Mapper接口方法只有一个参数，那么可以定义SQLProvider方法，它接受一个与Mapper接口方法相同类型的参数。



例如映射器Mapper接口有如下定义：



 

这里findTutorById(int)方法只有一个int类型的参数。我们可以定义findTutorByIdSql(int)方法作为SQL provider方法。





如果映射器Mapper接口有多个输入参数，我们可以使用参数类型为java.util.Map的方法作为SQLprovider方法。然后映射器Mapper接口方法所有的输入参数将会被放到map中，以param1,param2等等作为key，将输入参数按序作为value。你也可以使用0，1，2等作为key值来取的输入参数。



SQL工具类也提供了其他的方法来表示JOINS，ORDER_BY，GROUP_BY等等。

让我们看一个使用LEFT_OUTER_JOIN的例子：



由于没有支持使用内嵌结果ResultMap的一对多关联映射的注解支持，我们可以使用基于XML的<resultMap>配置，然后与@ResultMap映射。

## **2.@InsertProvider**

可以使用@InsertProvider注解创建动态的INSERT语句，如下所示：



## **3.@UpdateProvider**

可以通过@UpdateProvider注解创建UPDATE语句，如下所示：



**4.DeleteProvider**

可以使用@DeleteProvider注解创建动态地DELETE语句,如下所示：





# 注意事项：

- 在利用注解配置映射器接口的时候，必须要通过 sqlSessionFactory.getConfiguration().addMapper(IBlogDAO.class);来对给映射器接口注册，如果映射器接口中使用了@ResultMap注解，则由于已经在mybatis-config.xml配置了Mapper，则就不需要再次在代码中添加mapper。



- 当方法有多个参数的时候，为了与SQL语句中的#{}对应，一般可以使用@Param("")来为每个参数命别名，使得该别名与#{}对应。当参数只有一个的时候，不需要别名。



- 在进行更新删除添加的时候，如果传递的是一个实体对象，则SQL可以直接使用实体的属性。



- 映射器接口调用SqlBuilder中的方法，都是将参数转换为Map中的key，可以在SqlBuilder的方法中利用Map来获取传递的参数值，进而进行逻辑操作判断。



- 注解中对于返回多条记录的查询可以直接利用@Results和@Result来配置映射，或者利用@ResultMap来调用SQL配置文件中的ResultMap。