# 1.背景

Spring提供了编程式事务和声明式事务，但由于编程性事务的侵入性，开发中普遍会使用Spring的声明式事务，下文中所说的Spring事务也都是指声明式事务。

Spring声明式事务底层是建立在AOP的基础上的，其本质就是对方法前后进行拦截，然后在目标方法之前创建或加入一个事务，在执行完目标方法之后根据执行执行情况提交或回滚事务。

声明式事务最大的优点就是不需要在业务逻辑代码中掺杂事务管理的代码，Spring事务提供两种事务管理方式：基于注解和基于XML配置，我们只需要简单的配置即可使用。

相比于编程式事务，由于声明式事务是基于AOP实现的，所以只能实现对方法的粗粒度的控制，而无法做到代码块级别的细粒度控制。

Spring事务包括5中参数，分别是：传播行为，隔离级别，是否只读，事务超时时间，回滚规则

本文分析下5种事务参数，然后对具体的传播行为给出Demo，以此给出其“细粒度”的事务实现。

# 2.Spring事务的5种参数

## 2.1 传播行为

传播行为定义了关于客户端和被调用方法的事务边界，Spring中定义了7中传播行为。

| 传播行为                      | 意义                                                         |
| :---------------------------- | :----------------------------------------------------------- |
| **PROPAGATION_REQUIRED**      | 默认的Spring事务传播规则；表示当前方法必须在一个事务中运行，如果一个现有事务正在运行中，该方法将在那个事务中运行，否则就会新建一个事务 |
| **PROPAGATION_REQUIRES_NEW**  | 表示当前方法必须在它自己的事务里运行。每次都会新建一个事务，如果当前有一个事务在运行的话，则将在这个方法运行期间挂起，待新事务执行完成后，上下文事务再恢复执行 |
| **PROPAGATION_SUPPORTS**      | 表示如果上下文中存在事务，就在当前事务中执行，否则使用非事务的方式执行 |
| **PROPAGATION_NOT_SUPPORTED** | 表示该方法不应该在一个事务中运行。如果上下文中存在事务，它将在该方法执行期间被挂起 |
| **PROPAGATION_MANDATORY**     | 表示该方法必须在一个事务中运行，如果没有事务，将抛出异常     |
| **PROPAGATION_NEVER**         | 表示该方法不能在一个事务中运行，如果有事务，则会抛出异常     |
| **PROPAGATION_NESTED**        | 表示如果当前正有一个事务在运行中，则该方法会运行在一个嵌套事务中。被嵌套的事务可以独立于封装事务进行提交或回滚。如果不存在事务，则新建事务。 |

**对嵌套事务的理解：**

嵌套是子事务在父事务中执行，子事务是父事务的一部分，在进入子事务之前，父事务建立一个回滚点，叫save point，然后执行子事务，这个子事务也算是父事务的一部分，然后子事务执行结束，父事务继续执行。下面看几个问题就懂了

**如果子事务回滚，会发生什么？**

答：父事务会回滚到save point，然后尝试其他的事务或者其他的业务逻辑，父事务之前的操作不会受到影响，更不会自动回滚

**如果父事务回滚，会发生什么？**

答：父事务回滚，子事务也会跟着回滚。父事务结束之前，子事务不会提交。

**事务的提交是什么情况？**

答：子事务是父事务的一部分，所以子事务先提交，父事务再提交

## 2.2 隔离级别

隔离级别定义一个事务可能接受其他并发事务活动受影响的程度。也可以想象成事务对于数据处理的自私程度。

多个事务同时运行，经常会为了完成他们的工作而操作同一个数据。并发虽然是必需的，但是会导致以下问题：

- 胀读——A事务读取数据并修改，未提交之间B事务又读取了数据
- 不可重复读——A事务读取数据，B事务读取数据并修改，A事务再读取数据时发现两次数据不一致
- 幻读——事务A读取或删除了全部数据，事务B又insert一条数据，这时事务A发现莫名其妙的多了一条数据

理想状态下，事务之间是完全隔离的。但是完全隔离会影响性能，因为隔离的实现依赖于数据库中的锁，侵占性锁会阻碍并发，要求事务互相等待

考虑到完全隔离会影响性能，而且并不是所有的情况都要求完全隔离，所以有时候可以在事务隔离方面灵活处理。因此，就有好几个隔离级别。

| 隔离级别                   | 含义                                                         |
| :------------------------- | :----------------------------------------------------------- |
| ISOLATION_DEFAULT          | 使用后端数据库默认的隔离级别                                 |
| ISOLATION_READ_UNCOMMITTED | 允许读取未提交的更改。 可能导致脏读、幻读或不可重复读。      |
| ISOLATION_READ_COMMITTED   | 允许从已经提交的并发事务读取。可防止脏读，但幻读和不可重复读仍可能会发生。 |
| ISOLATION_REPEATABLE_READ  | 对相同字段的多次读取的结果是一致的，除非数据被当前事务本身改变。可防止脏读和不可重复读，但幻影读仍可能发生。 |
| ISOLATION_SERIALIZABLE     | 完全服从ACID的隔离级别，确保不发生脏读、不可重复读和幻影读。这在所有隔离级别中也是最慢的，因为它通常是通过完全锁定当前事务所涉及的数据表来完成的。 |

## 2.3 是否只读

如果一个事务只对后端数据库执行读操作，那么该数据库就可能利用那个事务的只读特性，采取某些优化措施。

由于只读的优化措施是在一个事务启动时由后端数据库实施的，因此，只有对于那些具有可能启动一个新事物的传播行为（PROPAGATION_REQUIRED，PROPAGATION_REQUIRES_NEW，PROPAGATION_NESTED）的方法来说，将事务声明为只读才有意义

## 2.4 事务超时

假设事务的运行时间变得格外的长，由于事务可能涉及对后端数据库的锁定，所以长时间运行的事务会不必要地占用数据库资源，这时就可以声明一个事务在特定时间后自动回滚

由于超时时钟在一个事务启动的时候开始的，因此，有对于那些具有可能启动一个新事物的传播行为（PROPAGATION_REQUIRED，PROPAGATION_REQUIRES_NEW，PROPAGATION_NESTED）的方法来说，声明事务超时才有意义，默认为30S

## 2.5 回滚规则

回滚规则定义了哪些异常引起回滚，哪些不引起。在默认情况下，事务只出现运行时异常（Runtime Exception）时回滚，而在出现受检查异常（Checked Exception）时不回滚。

# 3.事务传播特性实例

在本节中，我们只会考虑事务的传播规则，不会考虑到其隔离级别等其他参数。测试Service为TestServiceA和TestServiceB

## 3.1 PROPAGATION_REQUIRED示例

TestSeviceA代码为：

代码块

java









```
@Autowired
private TransactionDAO transactionDAO;
@Autowired
private TestServiceB testService;

// 会开启事务,在事务范围内使用同一个事务,否则开启新事务
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void addOrder() {
    transactionDAO.insertOrder();
    testService.addVoucher();
    throw new RuntimeException();
}
```



TestServiceB代码为：

代码块

java









```
@Autowired
    private TransactionDAO transactionDAO;

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void addVoucher() {
        transactionDAO.insertOrderVoucher();
//        throw new RuntimeException();
    }
```



经测试，无论是在addOrder还是addVoucher中抛出异常，数据都会提交失败；当且仅当两个方法都成功时，数据提交成功。

这说明这两个Service使用的是同一个事务，并且只要方法被调用就会开启事务。

注意：一旦有前置方法开启了Spring事务，在调用链中的方法即使没有声明事务，也都会加入到事务中。

 

如果在ServiceA中使用了try catch语句后会是什么情况呢？即代码如下：

TestSeviceA代码为：

代码块

java









```
@Autowired
private TransactionDAO transactionDAO;
@Autowired
private TestServiceB testService;

// 会开启事务,在事务范围内使用同一个事务,否则开启新事务
@Transactional(propagation = Propagation.REQUIRED)
public void addOrder() {
    transactionDAO.insertOrder();
    try {
      testService.addVoucher();
  } catch (Exception e) {
  }
}
```



TestServiceB代码为：

代码块

java









```
@Autowired
    private TransactionDAO transactionDAO;

    @Transactional(propagation = Propagation.REQUIRED)
    public void addVoucher() {
        transactionDAO.insertOrderVoucher();
        throw new RuntimeException();
    }
```



在ServiceB中抛出RuntimeException异常，但是在ServiceA中使用try catch捕获该异常。

执行后发现数据没有提交，但是出现了一个异常：org.springframework.transaction.UnexpectedRollbackException: Transaction rolled back because it has been marked as rollback-only

原因：在addVoucher()方法return时会将Transactional标记为Rollback only， 而addOrder()方法捕获了bar的RuntimeException，所以Spring将会commit addOrder()的事务，但是这两个方法使用的是同一事务，因此在commit addOrder()事务时，将会抛出UnexpectedRollbackException。

## 3.2 PROPAGATION_REQUIRES_NEW示例

TestServiceA代码为：

代码块

java









```
@Autowired
private TransactionDAO transactionDAO;
@Autowired
private TestServiceB testService;

// 无论何时都会开启事务
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void addOrder() {
    transactionDAO.insertOrder();
    testService.addVoucher();
    throw new RuntimeException();
}
```



TestServiceB代码为：

代码块

java









```
@Autowired
    private TransactionDAO transactionDAO;

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void addVoucher() {
        transactionDAO.insertOrderVoucher();
//        throw new RuntimeException();
    }
```



经测试如果在addOrder中抛出异常，order数据不能提交，voucher数据被正确提交。相反addVoucher中抛出异常，就只能提交order数据。

说明两个Service是在两个独立是事务中运行，并且只要方法被调用就开启事务。

## 3.3 PROPAGATION_SUPPORTS示例

TestServiceA代码为：

代码块

java









```
@Autowired
private TransactionDAO transactionDAO;
@Autowired
private TestServiceB testService;

// 自身不会开启事务，在事务范围内则使用相同事务，否则不使用事务
@Transactional(propagation = Propagation.SUPPORTS)
public void addOrder() {
    transactionDAO.insertOrder();
    testService.addVoucher();
    throw new RuntimeException();
}
```



TestServiceB代码为：

代码块

java









```
@Autowired
    private TransactionDAO transactionDAO;

    @Transactional(propagation = Propagation.SUPPORTS)
    public void addVoucher() {
        transactionDAO.insertOrderVoucher();
//        throw new RuntimeException();
    }
```



由于两个方法都被申明为SUPPORTS的传播规则，所以即使抛出异常，数据都也都能被正确提交。说明两个方法并没有使用Spring事务，而是使用本地事务，需要注意的是，这两个方法的本地事务并不是同一个事务，一旦出现异常将导致数据不一致。

我们来考虑一下更多的组合情况：

1. 1. REQUIRED → SUPPORTS  这时开启事务，并使用同一个事务
   2. REQUIRES_NEW → SUPPORTS 同a
   3. SUPPORTS → REQUIRED  ServiceA由本地事务管理，ServiceB由Spring事务管理
   4. SUPPORTS → REQUIRES_NEW 同c

## 3.4 PROPAGATION_NOT_SUPPORTED示例

TestServiceA代码为：

代码块

java









```
@Autowired
private TransactionDAO transactionDAO;
@Autowired
private TestServiceB testService;

// 自身不会开启事务，在事务范围内使用会挂起事务执行，运行完毕恢复事务
@Transactional(propagation = Propagation.REQUIRED)
public void addOrder() {
    transactionDAO.insertOrder();
    testService.addVoucher();
}
```



TestServiceB代码为：

代码块

java









```
@Autowired
    private TransactionDAO transactionDAO;

    @Transactional(propagation = Propagation.SUPPORTS)
    public void addVoucher() {
        transactionDAO.insertOrderVoucher();
        throw new RuntimeException();
    }
```



经测试如果在addVoucher中抛出异常，addOrder数据提交失败，addVoucher数据会提交成功。说明ServiceA开了事务，ServiceB没有开启事务，而是使用了本地事务。

## 3.5 PROPAGATION_MANDATORY示例

TestServiceA代码

代码块

java









```
@Autowired
private TransactionDAO transactionDAO;
@Autowired
private TestServiceB testService;

// 自身不会开启事务，必须在事务环境使用，否则报错
@Transactional(propagation = Propagation.MANDATORY)
public void addOrder() {
    transactionDAO.insertOrder();
    testService.addVoucher();
}


```



经测试以上代码报错

org.springframework.transaction.IllegalTransactionStateException: No existing transaction found for transaction marked with propagation 'mandatory'   没有找到事务环境

即该注解必须要和能创建事务的注解使用（PROPAGATION_REQUIRED，PROPAGATION_REQUIRES_NEW，PROPAGATION_NESTED）

## 3.6 PROPAGATION_NEVER 示例

 TestServiceA代码为： 

代码块

java









```
@Autowired
private TransactionDAO transactionDAO;
@Autowired
private TestServiceB testService;

@Transactional(propagation = Propagation.REQUIRED)
public void addOrder() {
    transactionDAO.insertOrder();
    testService.addVoucher();
}
```



TestServiceB代码为：

代码块

java









```
@Autowired
    private TransactionDAO transactionDAO;

    // 自身不会开启事务，在事务范围内使用抛出异常
    @Transactional(propagation = Propagation.NEVER)
    public void addVoucher() {
        transactionDAO.insertOrderVoucher();
        throw new RuntimeException();
    }
```



经测试代码抛出异常

org.springframework.transaction.IllegalTransactionStateException: Existing transaction found for transaction marked with propagation 'never'  存在事务环境

## 3.7 PROPAGATION_NESTED示例

 

TestServiceA代码为： 

 

代码块

java









```
@Autowired
private TransactionDAO transactionDAO;
@Autowired
private TestServiceB testService;

// 新开启一个事务
    @Transactional(propagation = Propagation.REQUIRED)
    public void addOrder() {

        transactionDAO.insertOrder();
        try {
            testService.addVoucher();
        } catch (Exception e) {
        }
//        throw new RuntimeException();
    }
```



 

TestServiceB代码为：

 

代码块

java









```
@Autowired
    private TransactionDAO transactionDAO;

    // 启用嵌套事务
    @Transactional(propagation = Propagation.NESTED)
    public void addVoucher() {
        transactionDAO.insertOrderVoucher();
        throw new RuntimeException();
    }
```



经测试，在ServiceB中抛出异常后，addVoucher数据提交失败，addOrder数据提交成功。

如果在ServiceA中抛出异常，那么数据都会提交失败。

# 4.注意事项

1. @Transactional 只能被应用到public方法上, 对于其它非public的方法,如果标记了@Transactional也不会报错,但方法没有事务功能.
2. 仅仅 @Transactional 注解的出现不足于开启事务行为，需要在spring中开启事务：<tx:annotation-driven/>
3. 默认遇到运行期例外(throw new RuntimeException("注释");)会回滚，即遇到不受检查（unchecked）的例外时回滚；
4. 事务的传播特性会在调用同一类中不同方法时失效

# 5.参考文献

https://www.ibm.com/developerworks/cn/education/opensource/os-cn-spring-trans/

https://my.oschina.net/dongli/blog/56904

[http://blog.csdn.net/aya19880214/article/details/50640596](http://blog.csdn.net/aya19880214/article/details/50640596)