# **背景**

项目中使用的是MyBatis的注解实现和数据库的交互，但是在配置文件中来声明SQL更加的灵活，现在总结下MyBatis中配置文件的语法。

# **Insert**

一个INSERT SQL语句可以在<insert>元素在映射器XML配置文件中配置，如下所示：



这里我们使用一个ID insertStudent，可以在名空间com.mybatis3.mappers.StudentMapper.insertStudent中唯一标识。parameterType属性应该是一个完全限定类名或者是一个类型别名（alias）。

接口中定义如下：



该方法返回执行INSERT语句后所影响的行数。

# **自动生成主键**

在上述的INSERT语句中，我们为可以自动生成（auto-generated）主键的列STUD_ID插入值。我们可以使用useGeneratedKeys 和 keyProperty属性让数据库生成 auto_increment列的值，并将生成的值设置到其中一个输入对象属性内，如下所示：



这里 STUD_ID列值将会被MySQL 数据库自动生成，并且生成的值会被设置到student对象的studId属性上。

# **Update**

一个UPDATE SQL语句可以在<update>元素在映射器XML配置文件中配置，如下所示：





接口定义如下



该方法返回执行UPDATE语句后所影响的行数。

# **Delete**

一个DELETE SQL语句可以在<delete>元素在映射器XML配置文件中配置，如下所示：





该方法返回delete语句执行后影响的行数。

# **Select**

MyBatis真正强大的功能，在于映射SELECT查询结果到JavaBeans 方面的极大灵活性。

让我们看看一个简单的select查询是如何（在MyBatis中）配置的，如下所示：



接口为



但是执行结果生成的Student对象，

如果你检查Student对象的属性值，你会发现studId属性值并没有被stud_id列值填充。这是因为MyBatis自动对JavaBean中和列名匹配的属性进行填充。这就是为什么name ,email,和phone属性被填充，而studId属性没有被填充。



为了解决这一问题，我们可以为列名起一个可以与JavaBean中属性名匹配的别名，如下所示：



现在，Student 这个Bean对象中的值将会恰当地被stud_id,name,email,phone列填充了。

现在，让我们看一下如何执行返回多条结果的SELECT语句查询，如下所示：



接口定义为：



如果你注意到上述的SELECT映射定义，你可以看到，我们为所有的映射语句中的stud_id起了别名。

我们可以使用ResultMaps，来避免上述的到处重复别名。我们稍后会继续讨论。

除了java.util.List，你也可以是由其他类型的集合类，如Set,Map，以及（SortedSet）。MyBatis根据集合的类型，会采用适当的集合实现，如下所示：

- 对于List，Collection，Iterable类型，MyBatis将返回java.util.ArrayList
- 对于Map类型，MyBatis将返回java.util.HashMap
- 对于Set类型，MyBatis将返回 java.util.HashSet
- 对于SortedSet类型，MyBatis将返回java.util.TreeSet

# **ResultMaps**

ResultMaps被用来 将SQL SELECT语句的结果集映射到 JavaBeans的属性中。我们可以定义结果集映射ResultMaps并且在一些SELECT语句上引用resultMap。MyBatis的结果集映射ResultMaps特性非常强大，你可以使用它将简单的SELECT语句映射到复杂的一对一和一对多关系的SELECT语句上。

## **简单ResultMap**

一个映射了查询结果和Student JavaBean的简单的resultMap定义如下：



表示resultMap的StudentResult id值应该在此名空间内是唯一的。并且type属性应该是完全限定类名或者是返回类型的别名。



<result>子元素被用来将一个resultset列映射到JavaBean的一个属性中。



<id>元素和<result>元素功能相同，不过它被用来映射到唯一标识属性，用来区分和比较对象（一般和主键列相对应）。



在<select>语句中，我们使用了resultMap属性，而不是resultType来引用StudentResult映射。当<select>语句中配置了resutlMap属性，MyBatis会使用此数据库列名与对象属性映射关系来填充JavaBean中的属性。



ResultMap和ResultType一次只能用一个

## **查询结果填充到HashMap中**





在上述的<select>语句中，我们将resultType配置成map，即java.util.HashMap的别名。在这种情况下，结果集的列名将会作为Map中的key值，而列值将作为Map的value值。





让我们再看一个 使用resultType=”map”,返回多行结果的例子：





由于resultType=”map”和语句返回多行，则最终返回的数据类型应该是List<HashMap<String,Object>>，如下所示：



## **拓展ResultMap**

我们可以从从另外一个<resultMap>，拓展出一个新的<resultMap>，这样，原先的属性映射可以继承过来实现。



id为StudentWithAddressResult的resultMap拓展了id为StudentResult的resultMap。

如果你只想映射Student数据，你可以使用id为StudentResult的resultMap,如下所示





如果你想将映射Student数据和Address数据，你可以使用id为StudentWithAddressResult的resultMap：



## **一对一映射**

在我们的域模型样例中，每一个学生都有一个与之关联的地址信息。表STUDENTS有一个ADDR_ID列，是ADDRESSES表的外键。

STUDENTS表的样例数据如下所示：

| STUD_ID | NAME | EMAIL                                   | PHONE        | ADDR_ID |
| ------- | ---- | --------------------------------------- | ------------ | ------- |
| 1       | John | [john@gmail.com](mailto:john@gmail.com) | 123-456-7890 | 1       |
| 2       | Paul | [paul@gmail.com](mailto:paul@gmail.com) | 111-222-3333 | 2       |

 

ADDRESSES表的样例输入如下所示：

| ADDR_ID | STREET     | CITY    | STATE | ZIP   | COUNTRY |
| ------- | ---------- | ------- | ----- | ----- | ------- |
| 1       | Naperville | CHICAGO | IL    | 60515 | USA     |
| 2       | Paul       | CHICAGO | IL    | 60515 | USA     |

下面让我们看一下怎样取Student明细和其Address明细。

Student和Address 的JavaBean以及映射器Mapper XML文件定义如下所示：





我们可以使用圆点记法为内嵌的对象的属性赋值。在上述的resultMap中，Student的address属性使用了圆点记法被赋上了address对应列的值。同样地，我们可以访问任意深度的内嵌对象的属性。

上述样例展示了一对一关联映射的一种方法。然而，使用这种方式映射，如果address结果需要在其他的SELECT映射语句中映射成Address对象，我们需要为每一个语句重复这种映射关系。MyBatis提供了更好地实现一对一关联映射的方法：嵌套结果ResultMap和嵌套select查询语句。接下来，我们将讨论这两种方式



**嵌套结果ResultMap实现一对一关系映射**

我们可以使用一个嵌套结果ResultMap方式来获取Student及其Address信息，代码如下：



元素<association>被用来导入“有一个”(has-one)类型的关联。在上述的例子中，我们使用了<association>元素引用了另外的在同一个XML文件中定义的<resultMap>。

我们也可以使用<association定义内联的resultMap，代码如下所示：





**使用嵌套查询实现一对一关系映射**

我们可以通过使用嵌套select查询来获取Student及其Address信息，代码如下：



在此方式中，<association>元素的 select属性被设置成了id为 findAddressById的语句。这里，两个分开的SQL语句将会在数据库中执行，第一个调用findStudentById加载student信息，而第二个调用findAddressById来加载address信息。

Addr_id列的值将会被作为输入参数传递给selectAddressById语句。

这种方式会查询数据库两次！



**一对多映射**

在我们的域模型样例中，一个讲师可以教授一个或者多个课程。这意味着讲师和课程之间存在一对多的映射关系。

我们可以使用<collection>元素将 一对多类型的结果 映射到 一个对象集合上。

TUTORS表的样例数据如下：

| TUTOR_ID | NAME | EMAIL                                   | PHONE        | ADDR_ID |
| -------- | ---- | --------------------------------------- | ------------ | ------- |
| 1        | John | [john@gmail.com](mailto:john@gmail.com) | 123-456-7890 | 1       |
| 2        | Ying | [ying@gmail.com](mailto:ying@gmail.com) | 111-222-3333 | 2       |

 

COURSE表的样例数据如下：

| COURSE_ID | NAME    | DESCRIPTION | START_DATE | END_DATE   | TUTOR_ID |
| --------- | ------- | ----------- | ---------- | ---------- | -------- |
| 1         | JavaSE  | Java SE     | 2013-01-10 | 2013-02-10 | 1        |
| 2         | JavaEE  | Java EE 6   | 2013-01-10 | 2013-03-10 | 2        |
| 3         | MyBatis | MyBatis     | 2013-01-10 | 2013-02-20 | 2        |

 

 在上述的表数据中，John讲师教授一个课程，而Ying讲师教授两个课程。

Course和Tutor的JavaBean定义如下：



现在让我们看看如何获取讲师信息以及其所教授的课程列表信息。

<collection>元素被用来将多行课程结果映射成一个课程Course对象的一个集合。和一对一映射一样，我们可以使用**嵌套结果ResultMap**和**嵌套Select语句**两种方式映射实现一对多映射。



**嵌套结果ResultMap实现一对多映射**

我们可以使用嵌套结果resultMap方式获得讲师及其课程信息，代码如下：



这里我们使用了一个简单的使用了JOINS连接的Select语句获取讲师及其所教课程信息。<collection>元素的resultMap属性设置成了CourseResult，CourseResult包含了Course对象属性与表列名之间的映射。



**使用嵌套Select语句实现一对多映射**

我们可以使用嵌套Select语句方式获得讲师及其课程信息，代码如下：



在这种方式中，<aossication>元素的select属性被设置为id 为findCourseByTutor的语句，用来触发单独的SQL查询加载课程信息。tutor_id这一列值将会作为输入参数传递给findCouresByTutor语句。

注意：嵌套Select语句查询会导致N+1选择问题。首先，主查询将会执行(1次)，对于主查询返回的每一行，另外一个查询将会被执行（主查询N行，则此查询N次）。对于大型数据库而言，这会导致很差的性能问题。

# **动态SQL**

有时候，静态的SQL语句并不能满足应用程序的需求。我们可以根据一些条件，来动态地构建SQL语句。

例如，在Web应用程序中，有可能有一些搜索界面，需要输入一个或多个选项，然后根据这些已选择的条件去执行检索操作。在实现这种类型的搜索功能，我们可能需要根据这些条件来构建动态的SQL语句。如果用户提供了任何输入条件，我们需要将那个条件 添加到SQL语句的WHERE子句中。

MyBatis通过使用<if>,<choose>,<where>,<foreach>,<trim>元素提供了对构造动态SQL语句的高级别支持。



**IF条件**

<if>元素被用来有条件地嵌入SQL片段，如果测试条件被赋值为true，则相应地SQL片段将会被添加到SQL语句中。

假定我们有一个课程搜索界面，设置了 **讲师（Tutor）**下拉列表框，**课程名称（CourseName）**文本输入框，**开始时间（StartDate）**输入框，**结束时间（EndDate）**输入框，作为搜索条件。假定课讲师下拉列表是必须选的，其他的都是可选的。

当用户点击 搜索按钮时，我们需要显示符合以下条件的成列表：

- 特定讲师的课程
- 课程名 包含输入的课程名称关键字的课程；如果课程名称输入为空，则取所有课程
- 在开始时间和结束时间段内的课程

我们可以对应的映射语句，如下所示：





Mybatis是使用OGNL来构建动态的SQL表达式

此处将生成查询语句 SELECT * FROM COURSES WHERE TUTOR_ID= ? AND NAME like? AND START_DATE >= ?。准备根据给定条件的动态SQL查询将会派上用场。



**choose，when和otherwise条件**

有时候，查询功能是以查询类别为基础的。首先，用户需要选择是否希望通过选择讲师，课程名称，开始时间，或结束时间作为查询条件类别来进行查询，然后根据选择的查询类别，输入相应的参数。在这样的情景中，我们需要只使用其中一种查询类别。



MyBatis 提供了<choose>元素支持此类型的SQL预处理。



现在让我们书写一个适用此情景的SQL映射语句。如果没有选择查询类别，则查询开始时间在今天之后的课程，代码如下：





**where条件**

有时候，所有的查询条件（criteria）应该是可选的。在需要使用至少一种查询条件的情况下，我们应该使用WHERE子句。并且， 如果有多个条件，我们需要在条件中添加AND或OR。MyBatis提供了<where>元素支持这种类型的动态SQL语句。

在我们查询课程界面，我们假设所有的查询条件是可选的。进而，当需要提供一个或多个查询条件时，应该改使用WHERE子句。





<where>元素只有在其内部标签有返回内容时才会在动态语句上插入WHERE条件语句。并且，如果WHERE子句以AND或者OR打头，则打头的AND或OR将会被移除。

如果tutor_id参数值为null，并且courseName参数值不为null，则<where>标签会将AND name like#{courseName} 中的AND移除掉，生成的SQL WHERE子句为：where name like #{courseName}。



**trim条件**

<trim>元素和<where>元素类似，但是<trim>提供了在添加前缀/后缀 或者移除前缀/后缀方面提供更大的灵活性。



这里如果任意一个<if>条件为true，<trim>元素会插入WHERE,并且移除紧跟WHERE后面的AND或OR。



**foreach循环**

另外一个强大的动态SQL语句构造标签即是<foreach>。它可以迭代遍历一个数组或者列表，构造AND/OR条件或一个IN子句。



假设我们想找到tutor_id为1，3，6的讲师所教授的课程，我们可以传递一个tutor_id组成的列表给映射语句，然后通过<foreach>遍历此列表构造动态SQL。









现在让我们来看一下怎样使用<foreach>生成 IN子句：





**Set条件**

<set>元素和<where>元素类似，如果其内部条件判断有任何内容返回时，他会插入SET SQL片段。



这里，如果<if>条件返回了任何文本内容，<set>将会插入set关键字和其文本内容，并且会剔除将末尾的 “，”。

 在上述的例子中，如果 phone!=null,<set>将会让会移除  phone=#{phone}后的逗号“,”，

生成 set phone=#{phone} 。



**传入多个输入参数**

MyBatis中的映射语句有一个parameterType属性来制定输入参数的类型。如果我们想给映射语句传入多个参数的话，我们可以将所有的输入参数放到HashMap中，将HashMap传递给映射语句。



MyBatis 还提供了另外一种传递多个输入参数给映射语句的方法。假设我们想通过给定的name和email信息查找学生信息，定义查询接口如下：



MyBatis 支持将多个输入参数传递给映射语句，并以#{param}的语法形式引用它们：



这里#{param1}引用第一个参数name，而#{param2}引用了第二个参数email。

这里也可以在接口定义的方法中使用@Param注解



**使用RowBounds对结果集进行分页**

有时候，我们会需要跟海量的数据打交道，比如一个有数百万条数据级别的表。由于计算机内存的现实我们不可能一次性加载这么多数据，我们可以获取到数据的一部分。特别是在Web应用程序中，分页机制被用来以一页一页的形式展示海量的数据。



MyBatis可以使用RowBounds逐页加载表数据。RowBounds对象可以使用offset和limit参数来构建。参数offset表示开始位置，而limit表示要取的记录的数目。



假设如果你想每页加载并显示25条学生的记录，你可以使用如下的代码：





然后，你可以加载如下加载第一页数据（前25条）





若要展示第二页，使用offset=25,limit=25;第三页，则为offset=50，limit=25。

## **缓存**

将从数据库中加载的数据缓存到内存中，是很多应用程序为了提高性能而采取的一贯做法。MyBatis对通过映射的SELECT语句加载的查询结果提供了内建的缓存支持。默认情况下，启用一级缓存；即，如果你使用同一个SqlSession接口对象调用了相同的SELECT语句，则直接会从缓存中返回结果，而不是再查询一次数据库。

我们可以在SQL映射器XML配置文件中使用<cache />元素添加全局二级缓存。

当你加入了<cache/>元素，将会出现以下情况：

- 所有的在映射语句文件定义的<select>语句的查询结果都会被缓存
- 所有的在映射语句文件定义的<insert>,<update> 和<delete>语句将会刷新缓存
- 缓存根据最近最少被使用（Least Recently Used，LRU）算法管理
- 缓存不会被任何形式的基于时间表的刷新（没有刷新时间间隔），即不支持定时刷新机制
- 缓存将存储1024个 查询方法返回的列表或者对象的引用
- 缓存会被当作一个读/写缓存这是指检索出的对象不会被共享，并且可以被调用者安全地修改，不会其他潜在的调用者或者线程的潜在修改干扰。（即，缓存是线程安全的）

你也可以通过复写默认属性来自定义缓存的行为，如下所示：



以下是对上述属性的描述：

-  eviction:此处定义缓存的移除机制。默认值是LRU，其可能的值有：LRU（least recently used,最近最少使用）,FIFO(first infirst out,先进先出)，SOFT(soft reference,软引用)，WEAK（weak reference,弱引用）。
-  flushInterval:定义缓存刷新间隔，以毫秒计。默认情况下不设置。所以不使用刷新间隔，缓存cache只有调用语句的时候刷新。
-  size:此表示缓存cache中能容纳的最大元素数。默认值是1024，你可以设置成任意的正整数。
-  readOnly:一个只读的缓存cache会对所有的调用者返回被缓存对象的同一个实例（实际返回的是被返回对象的一份引用）。一个读/写缓存cache将会返回被返回对象的一分拷贝（通过序列化）。默认情况下设置为false。可能的值有false和true。

一个缓存的配置和缓存实例被绑定到映射器配置文件所在的名空间（namespace）上，所以在相同名空间内的所有语句被绑定到一个cache中。

默认的映射语句的cache配置如下



你可以为任意特定的映射语句复写默认的cache行为；例如，对一个select语句不使用缓存，可以设置useCache=“false”。