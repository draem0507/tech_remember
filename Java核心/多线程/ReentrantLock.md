ReentrantLock是一个实现了Lock接口的可重入锁。查看它的源码时会发现，ReentrantLock的方法大多是通过委托实现的（委托给了一个sync变量）。既然如此，要想了解ReentrantLock的原理，就要从sync入手。

sync变量的类型是Sync，它继承了AbstractQueuedSynchronizer。

AbstractQueuedSynchronizer(简称AQS)，是整个JUC包中最重要、最复杂、最难理解透彻的一个类。

AQS是许多同步器的抽象类，如CountDownLatch、FutureTask、ReentrantLock、ReentrantReadWriteLock、Semaphore等。

AQS提供了同步相关的一些共同机制：如原子性操作同步状态、阻塞线程、唤醒线程、队列等。

AQS采用了模板方法模式，它定义了tryAcquire，tryRelease，tryAcquireShared，tryReleaseShared等方法，代码如下：



```
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}
 
protected boolean tryRelease(int arg) {
    throw new UnsupportedOperationException();
}
 
protected int tryAcquireShared(int arg) {
    throw new UnsupportedOperationException();
}
 
protected boolean tryReleaseShared(int arg) {
    throw new UnsupportedOperationException();
}
```



AQS给出的默认实现，是直接抛出UnsupportedOperationException异常。子类需要根据自身需求，实现这些方法，自己决定acquire和release的逻辑。对于独占模式的同步器，只需实现tryAcquire和tryRelease。对于共享模式的同步器，只需实现tryAcquireShared和tryReleaseShared。

 

公平锁与非公平锁

对于公平锁，线程会按照请求锁的顺序来获取锁，这样能够保证公平，但是会牺牲一定的吞吐量。

对于非公平锁，可能会有线程突然闯入（因为tryAcquire是在线程入队之前执行的），打破顺序。非公平锁能够提高吞吐量，但是可能会带来饥饿问题。

在ReentrantLock中，公平锁与非公平锁是通过sync的不同类型来实现的。公平锁使用FairSync，而非公平锁使用NonFairSync。两者的却别在于tryAcquire方法的实现。

NonFairSync的tryAcquire方法的实现如下：



```
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
}
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```



FairSync的tryAcquire方法的实现如下：

```
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
    // 与NonFairSync的区别在这里
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```



通过对比可知，两者的代码基本一致，有一点重要的区别在于，FairSync的tryAcquire方法多了一个判断：!hasQueuedPredecessors()

通过方法名就可以看出，hasQueuedPredecessors()的作用是判断队列中有没有前驱结点。也就是说，只有当队列为空，或者当前结点就在队首的时候，tryAcquire才能成功，这就保证了线程获取锁的顺序，从而保证了公平性。