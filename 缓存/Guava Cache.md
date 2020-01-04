# **背景**

项目中需要使用到很多个DataSource，就想着用一个LRU的算法来进行缓存。刚开始自己用LinkedHashMap，synchronized关键字实现了一个LRU的缓存类。但是被老大鄙视了，说还是google大法好，而且效率肯定也比我的高。好吧，所以就去学习了一个guava Cache的使用。

本篇文章主要是参考的是官方文档，有兴趣的同学可以自行去阅读。

# **Guava Cache简介**

Guava Cache底层就是用的ConcurrentHashMap来实现的。主要区别在于ConcurrentHashMap会一直保存所有添加的元素，直到显式地移除。而Guava Cache为了限制内存的使用，通常都是设定为自动回收元素。而自动回收元素的规则就为LRU算法。同时，Guavav Cache实现了缓存的自动加载，非常方便。

#  

# **新建一个Guava Cache**

Guava API中提供了CacheBuilder来初始化一个cache。通常有两种实现方式：



```java
private static final Cache<String,String> caches = CacheBuilder
        .newBuilder()
        .maximumSize(3)

       .removalListener(myListener)

       .build();

private static finalLoadingCache<String, String> caches = CacheBuilder.

       newBuilder()
       .maximumSize(3)

       .removalListener(myListener)

       .build(new CacheLoader<String, String>() {

           @Override

          public String load(String key) throws Exception {  // 注意，这里抛出了异常

               return MyLoadCache(key);
            }
      });
```



如果没有提供CacheLoader的实现，则返回的Cache，否则返回的是LoadingCache。

# **数据自动加载**

数据自动加载的使用场景在如果在get(key)的时候，没有该元素，那么Guava Cache就会调用相应方法自动去根据key值计算出value并加入到缓存中再返回。Guava提供了两种自动加载的实现。

## **1.CacheLoader**

如果你有合理的默认方法来根据Key加载或者计算出相应的Value，那么应当使用CacheLoader。

自己实现的的CacheLoader通常只需要简单地实现V load(K key) throws Exception方法。然后在load方法中实现根据key来计算出相应的value的逻辑即可。



在调用get(key)方法的时候，要么返回已有的缓存值，要么就使用CacheLoader向缓存中原子性的加载新增。由于CacheLoader可能抛出异常，LoadingCache.get(K)也声明为抛出ExecutionException异常。如果你定义的CacheLoader没有声明任何检查型异常，则可以通过getUnchecked(K)查找缓存；但必须注意，一旦CacheLoader声明了检查型异常，就不可以调用getUnchecked(K)。



```java
     private static final LoadingCache<Integer, String> cache1 = CacheBuilder.
        newBuilder()
        .maximumSize(3)
        .build(new CacheLoader<Integer, String>() {
     @Override

       public String load(Integer key){  // 没有抛出异常

            return null;

        }
    });
    private static String getUnchecked(Integer key) {
    return cache1.getUnchecked(key);    // 可以使用getUnchecked()
    }
```

 

getAll(Iterable<? extends K>)方法用来执行批量查询。默认情况下，对每个不在缓存中的键，getAll方法会单独调用CacheLoader.load来加载缓存项。



**2.Cache.get(K, Callable<V>)**

这个方法返回缓存中相应的值，或者用给定的Callable运算并把结果加入到缓存中。在整个加载方法完成前，缓存项相关的可观察状态都不会更改。这个方法简便地实现了模式"如果有缓存则返回；否则运算、缓存、然后返回"。



**显示插入**

Guava Cache同时也提供了put()方法用于显示插入数据使用cache.put(key,value)方法可以直接向缓存中插入值，这会直接覆盖掉给定键之前映射的值。



# **缓存回收**

Guava Cache提供了三种基本的缓存回收方式：基于容量回收，定时回收和基于引用回收。回收算法为LRU

## **1.基于容量回收**

如果要规定缓存项的数据不超过固定值，需要使用CacheBuilder.maximumSize(long num)。缓存将尝试回收最近没有使用或者总体上很少使用的缓存项。

注意的是，缓存项的数目在达到限定值之前，缓存就可能就进行回收操作，不过通常来说，这种情况发生在缓存项的数目要达到域值的时候。

另外，不同的缓存项可能有不同的“权重”(weights)，如果需要使用weights，首先需要使用CacheBuilder.maxmumWeight(long weight)来指定最大的权重值。然后需要使用CacheBuilder.weight(Weigher)指定一个权重函数，并重写weigh方法来计算每个缓存值的权重。在权重限定的场景中，除了要注意回收也是在重量逼近限定值就进行了，还要知道重量是在缓存创建时计算的，因此要考虑重量计算的复杂度。

## **2.定时回收**

CacheBuilder提供了两种定时回收的方法：

- expireAfterAccess(long, TimeUnit):缓存项在给定时间内没有被读/写访问，则回收。回收顺序也是根据LRU算法
- expireAfterWrite(long, TimeUnit):缓存项在给定时间内没有被写访问（创建或覆盖），则回收。如果任务缓存数据总是在固定后变成陈旧不可用，这种回收方式是可取的。

## **3.基于引用的回收**

通过使用弱引用的键、或弱引用的值、或软引用的值，Guava Cache可以把缓存设置为允许垃圾回收：

- CacheBuilder.weakKeys():使用弱引用存储键。当键没有其它（强或软）引用时，缓存项可以被垃圾回收。因为垃圾回收仅依赖恒等式（==），使用弱引用键的缓存用==而不是equals比较键。
- CacheBuilder.weakValues()：使用弱引用存储值。当值没有其它（强或软）引用时，缓存项可以被垃圾回收。因为垃圾回收仅依赖恒等式（==），使用弱引用值的缓存用==而不是equals比较值。
- CacheBuilder.softValues()：使用软引用存储值。软引用只有在响应内存需要时，才按照全局最近最少使用的顺序回收。考虑到使用软引用的性能影响，我们通常建议使用更有性能预测性的缓存大小限定（见上文，基于容量回收）。使用软引用值的缓存同样用==而不是equals比较值。

#  

# **显示清除**

Guava Cache也提供了三种方法用于显示的清除数据

- 个别清除：Cache.invalidate(key)
- 批量清除：Cache.invalidateAll(keys)
- 全部清除：Cache.invalidateAll()

# **移除监听器**

通过CacheBuilder.removalListener(RemovalListener), 可以申明一个监听器，以便缓存项被移除的时候做一些额外的操作。比如，如果缓存的是BasicDataSource或者Connection，那么在移除时，需要调用close方法将其关闭，否则就会造成内存泄露。

我们只需要实现RemovalListener接口并重写其onRemoval()方法即可，并且会获取通知RemovalNotification，RemovalNotification中包含了移除原因，建和值等。



注意：RemovalListener抛出的任何异常都会在记录到日志后被丢弃[swallowed]



注意：默认情况下，监听器方法是在移除缓存时同步调用的。因为缓存的维护和请求响应通常是同时进行的，代价高昂的监听器方法在同步模式下会拖慢正常的缓存请求。这种情况下，可以使用RemiValListener.asynchronous(RemovalListener, Executor)把监听器装饰为异步操作。



**什么时候对缓存进行清理？**

使用CacheBuilder构建的缓存缓存不会自己“自动”执行清理和回收工作，也不会在某个缓存过期后就立即清理，也没有相应的清理机制。相反，他会在写操作时顺带做少量的维护工作，或者如果写操作实在太少，那么偶尔在读操作时会做清理



这样做的原因是：如果要自动的持续清理缓存，就必须有一个线程，这个线程会和用户操作竞争共享锁。此外，某些环境线程创建可能受限制，这样CacheBuilder就不可用了。

如果你的缓存是高吞吐的，那就无需担心缓存的维护和清理等工作。如果你的缓存只会偶尔有写操作，而你又不想清理工作阻碍了读操作，那么可以创建自己的维护线程，定时调用Cache.cleanUp()方法。可以使用JDK 提供的ScheduledExecutorService类



https://km.sankuai.com/page/40969879