# **1.对ThreadLocal的理解**

ThreadLocal，也叫线程本地变量或线程本地存储。ThreadLocal为变量在每个线程中都创建了一个副本，那么每个线程可以访问自己内部的副本变量。我们先来看一个例子：

```
`class` `ConnectionManager {``    ` `    ``private` `static` `Connection connect = ``null``;``    ` `    ``public` `static` `Connection openConnection() {``        ``if``(connect == ``null``){``            ``connect = DriverManager.getConnection();``        ``}``        ``return` `connect;``    ``}``    ` `    ``public` `static` `void` `closeConnection() {``        ``if``(connect!=``null``)``            ``connect.close();``    ``}``}`
```

假设有这样一个数据库连接管理类，这段代码在单线程中使用时没有任何问题的，但是如果在多线程中使用呢？很显然，在多线程中使用会存在线程安全问题：第一，这里面2个方法都没有进行同步，很可能在openConnection方法中多次创建connect；第二，由于connect是共享变量，那么必然在调用connect的地方需要使用同步来保障线程安全，因为很可能一个线程在使用connect进行数据库操作，而另外一个线程调用closeConnection关闭连接。

所以出于线程安全考虑，必须将这段代码的两个方法进行同步处理，并且在调用connect的地方需要进行同步处理

这将会大大影响程序的执行效率，因为一个线程在使用connect进行数据库操作的时候，其他线程只有等待

那么大家来仔细的分析一下这个问题，这地方到底需不需要将connect变量进行共享？事实上，是不需要的。假如每个线程中都有一个connect变量，各个线程之间对connect变量的访问实际上是没有依赖关系的，即一个线程不关心其他线程是否对这个connect进行了修改。

到这里，可能会有这样的考虑，既然不需要在线程之间共享这个变量，可以直接这样处理，在每个需要使用数据库连接的方法中具体使用时才创建数据库连接，然后在方法调用完毕在释放这个连接，比如下面这样：

```
`class` `ConnectionManager {``    ` `    ``private`  `Connection connect = ``null``;``    ` `    ``public` `Connection openConnection() {``        ``if``(connect == ``null``){``            ``connect = DriverManager.getConnection();``        ``}``        ``return` `connect;``    ``}``    ` `    ``public` `void` `closeConnection() {``        ``if``(connect!=``null``)``            ``connect.close();``    ``}``}`  `class` `Dao{``    ``public` `void` `insert() {``        ``ConnectionManager connectionManager = ``new` `ConnectionManager();``        ``Connection connection = connectionManager.openConnection();``        ` `        ``//使用connection进行操作``        ` `        ``connectionManager.closeConnection();``    ``}``}`
```

这样处理确实也没有任何问题，由于每次都是在方法内部创建的连接，那么线程之间自然不存在线程安全问题。但是这样会有一个致命影响：导致服务器压力非常大，并且严重影响程序执行性能。由于在方法中需要频繁地开启和关闭数据库连接，这样不仅严重影响程序执行效率，还可导致服务器压力巨大。

这种情况下使用ThreadLocal是再适合不过了，因为ThreadLocal在每个线程中对该变量会创建一个副本，即每个线程内部都会有一个该变量，且在线程内部都会有一个该变量，且在线程内部任何地方都可以使用，线程之间互不影响，这样一来就不存在线程安全问题，也不会严重影响程序执行性能

但是要注意，虽然ThreadLocal能够解决上面说的问题，但是由于在每个线程中都创建了副本，所以要考虑它对资源的消耗，比如内存的占用会不不使用ThreadLocal要大

# **2.深入理解ThreadLocal类**

现在来看看ThreadLocal是如何实现的

先了解一下ThreadLocal类提供的几个方法

```
`public` `T get() { }``public` `void` `set(T value) { }``public` `void` `remove() { }``protected` `T initialValue() { }`
```

get()方法是用来获取ThreadLocal在当前线程中保存的副本变量，set()用来设置当前线程中变量的副本，remove()用来移除当前线程中变量的副本，initialValue()是一个protected方法，一般用来在使用时进行重写，他是一个延时加载方法，下面会详细说明：

首先看一下ThreadLocal类是如何为每个线程创建一个变量的副本的

先看下get方法的实现：



第一句是获得当前线程，然后通过getMap(t)方法获取到一个map，map的类型为ThreadLocalMap。然后接着下面获取到<key,value>键值对，注意这里获取键值对传进行的是 this，而不是当前线程t



如果获取成功，则返回value值

如果map为空，则调用setInitialValue方法返回value。

我们来具体分析下getMap方法中做了什么：



其实就是返回当前线程t中的一个成员变量threadLocals，我们继续看下变量threadLocals是什么：



实际上就是一个ThreadLocalMap，这个类型是ThreadLocal类的一个内部类，我们继续取看ThreadLocalMap的实现：



可以看到ThreadLocalMap的Entry继承了WeakReference，并且使用ThreadLocal作为键值。

继续看看setInitialValue方法的具体实现：



很容易理解，就是如果map不为空，就设置键值，为空，就创建Map，看一个createMap的实现：



至此，差不多就明白了ThreadLocal是如何为每个线程创建变量的副本的：

首先，在每个线程Thread内部有一个ThreadLocal.ThreadLocalMap类型的成员变量threadLocals，这个threadLocals就是用来存储实际的变量副本的，键值为当前ThreadLocal变量，value为变量副本（即T类型的变量）

初始时，在Thread里面，ThreadLocals为空，当通过ThreadLocal变量调用get()方法或者set()方法，就会对Thread类中的threadLocals进行初始化，并且以当前ThreadLocal变量为键值，以ThreadLocal要保存的副本变量为value，存到threadLocals。

然后在当前线程里面，如果要使用副本变量，就可以通过get方法在threadLocals里面查找。

下面通过一个例子来证明通过ThreadLocal能达到在每个线程中创建变量副本的效果：

```
`public` `class` `Test {``    ``ThreadLocal<Long> longLocal = ``new` `ThreadLocal<Long>();``    ``ThreadLocal<String> stringLocal = ``new` `ThreadLocal<String>();` `    ` `    ``public` `void` `set() {``        ``longLocal.set(Thread.currentThread().getId());``        ``stringLocal.set(Thread.currentThread().getName());``    ``}``    ` `    ``public` `long` `getLong() {``        ``return` `longLocal.get();``    ``}``    ` `    ``public` `String getString() {``        ``return` `stringLocal.get();``    ``}``    ` `    ``public` `static` `void` `main(String[] args) ``throws` `InterruptedException {``        ``final` `Test test = ``new` `Test();``        ` `        ` `        ``test.set();``        ``System.out.println(test.getLong());``        ``System.out.println(test.getString());``    ` `        ` `        ``Thread thread1 = ``new` `Thread(){``            ``public` `void` `run() {``                ``test.set();``                ``System.out.println(test.getLong());``                ``System.out.println(test.getString());``            ``};``        ``};``        ``thread1.start();``        ``thread1.join();``        ` `        ``System.out.println(test.getLong());``        ``System.out.println(test.getString());``    ``}``}`
```



运行后会发现，在main线程中和thread1线程中，longLocal保存的副本和stringLocal保存的副本值都不一样。最后一次在main线程再次打印副本值是为了证明在main线程中和thread1线程中的副本值确实是不同的。



总结一下：

1. 实际上通过ThreadLocal创建的副本是存储在每个线程自己的threadLocals中
2. 为何threadLocals的类型ThreadLocalMap的键值为ThreadLocal对象，因为每个线程中可有多个threadLocal变量，就像上面代码中的longLocal和stringLocal
3. 在进行get之前，必须先set，否则会报空指针异常

​               如果想在get之前不需要调用set就能正常访问的话，就必须重写initialValue()方法

​               因为在上面的代码分析过程中，我们发现如果没有先set的话，即在map中查不到对应的存储，则会通过调用setInitialValue()方法返回i，而在setInitialValue方法中，有一个语句是T value = initialValue()，而默认情况下，initialValue返回的都是null。

看下面这个例子：

```
`public` `class` `Test {``    ``ThreadLocal<Long> longLocal = ``new` `ThreadLocal<Long>();``    ``ThreadLocal<String> stringLocal = ``new` `ThreadLocal<String>();` `    ``public` `void` `set() {``        ``longLocal.set(Thread.currentThread().getId());``        ``stringLocal.set(Thread.currentThread().getName());``    ``}``    ` `    ``public` `long` `getLong() {``        ``return` `longLocal.get();``    ``}``    ` `    ``public` `String getString() {``        ``return` `stringLocal.get();``    ``}``    ` `    ``public` `static` `void` `main(String[] args) ``throws` `InterruptedException {``        ``final` `Test test = ``new` `Test();``        ` `        ``System.out.println(test.getLong());``        ``System.out.println(test.getString());` `        ``Thread thread1 = ``new` `Thread(){``            ``public` `void` `run() {``                ``test.set();``                ``System.out.println(test.getLong());``                ``System.out.println(test.getString());``            ``};``        ``};``        ``thread1.start();``        ``thread1.join();``        ` `        ``System.out.println(test.getLong());``        ``System.out.println(test.getString());``    ``}``}`
```

在main线程中，没有先set，直接get的话，运行时会报空指针异常。



但是如果改成下面这段代码，即重写了initialValue方法：

```
`public` `class` `Test {``    ``ThreadLocal<Long> longLocal = ``new` `ThreadLocal<Long>(){``        ``protected` `Long initialValue() {``            ``return` `Thread.currentThread().getId();``        ``};``    ``};``    ``ThreadLocal<String> stringLocal = ``new` `ThreadLocal<String>(){;``        ``protected` `String initialValue() {``            ``return` `Thread.currentThread().getName();``        ``};``    ``};` `    ` `    ``public` `void` `set() {``        ``longLocal.set(Thread.currentThread().getId());``        ``stringLocal.set(Thread.currentThread().getName());``    ``}``    ` `    ``public` `long` `getLong() {``        ``return` `longLocal.get();``    ``}``    ` `    ``public` `String getString() {``        ``return` `stringLocal.get();``    ``}``    ` `    ``public` `static` `void` `main(String[] args) ``throws` `InterruptedException {``        ``final` `Test test = ``new` `Test();` `        ``test.set();``        ``System.out.println(test.getLong());``        ``System.out.println(test.getString());``    ` `        ` `        ``Thread thread1 = ``new` `Thread(){``            ``public` `void` `run() {``                ``test.set();``                ``System.out.println(test.getLong());``                ``System.out.println(test.getString());``            ``};``        ``};``        ``thread1.start();``        ``thread1.join();``        ` `        ``System.out.println(test.getLong());``        ``System.out.println(test.getString());``    ``}``}`
```



就可以直接不用先set而直接调用get了。



# **3.ThreadLocal的应用场景**

最常见的ThreadLocal使用场景为 用来解决 数据库连接、Session管理等。



```
`private` `static` `ThreadLocal<Connection> connectionHolder ``= ``new` `ThreadLocal<Connection>() {``  public` `Connection initialValue() {``    ``return` `DriverManager.getConnection(DB_URL);``  }``};` `public` `static` `Connection getConnection() {``  return` `connectionHolder.get();``}`
```





```
`private` `static` `final` `ThreadLocal threadSession = ``new` `ThreadLocal();` `public` `static` `Session getSession() ``throws` `InfrastructureException {``    ``Session s = (Session) threadSession.get();``    ``try` `{``        ``if` `(s == ``null``) {``            ``s = getSessionFactory().openSession();``            ``threadSession.set(s);``        ``}``    ``} ``catch` `(HibernateException ex) {``        ``throw` `new` `InfrastructureException(ex);``    ``}``    ``return` `s;``}`
```

# **4.ThreadLocal会不会造成内存泄露？**

​    为什么这样认为：(ThreadLocalMap是使用ThreadLocal的弱引用作为Key的。如果一个ThreadLocal没有外部强引用引用他，那么在系统GC的时候，这个对象会被回收，这样一来，ThreadLocalMap中的键就为null， 无法访问，如果当前线程迟迟不结束的话，这些key为null的Entry的value就会一直存在一条强引用链。永远无法回收)

​    但其实并不会：整理出ThreadLocalMap的getEntry函数的流程：

1.  首先从ThreadLocal的直接索引位置（位置通过ThreadLocal.threadLocalHashCode & (len-1)）获得Entry e，如果e不为null并且key相同则返回e。
2.  如果e为null或者key不一致则向下一个位置查询，如果下一个位置的key和当前需要查询的key相等，则返回对应的Entry，另外，如果key值为null，则擦除该位置的Entry，否则继续向下一个位置查询。

​    由上可以看出，在这个过程中遇到的key为null的Entry都会被擦除，那么Entry内的value也就没有强引用链，自然会被回收。set也有类型的思想。但是这个思路必须在调用set和get函数的前提下才能实现，所以很多情况下需要手动调用remove函数，手动删除不需要的TreadLocal。

​    JDK建议将ThreadLocal变量定义为private static的，这样的话ThreadLocal的生命周期就更长，由于一直存在ThreadLocal的强引用，所以ThreadLocal就不会回收，也就能保证任何时候都能根据ThreadLocal的弱引用访问到Entry的value值，然后remove，防止内存泄露 