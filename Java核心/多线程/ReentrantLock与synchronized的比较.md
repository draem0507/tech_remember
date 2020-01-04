ReentrantLock在加锁和内存上提供的语义内置锁相同，但是ReentrantLock还提供了一些其他功能。本文将从以下几个方面来对比ReentrantLock和synchronized的区别：

1.性能

ReentrantLock在性能上优于内置锁（尤其是在Java 5.0中），但是随着版本的升级，Java对内置锁进行了优化，两者的性能差距也随之缩小。

2.代码风格

内置锁的代码更加简洁紧凑，只需使用synchronized关键字即可。而ReentrantLock必须显示的lock和unlock，而且unlock必须放在finally块中。所以，synchronized在使用时比较不容易出错，因为JVM负责加锁和解锁。而使用ReentrantLock时，如果忘记了在finally块中unlock，那么虽然代码表面上能正常运行，但实际上已经埋下一颗定时炸弹。

3.等待条件

Condition作为等待条件，可以和ReentrantLock一同使用。与Object的wait()，notify()，notifyAll()类似，Condition有await()，signal()，singnalAll()等方法，功能基本相同。但是同时可以有多个Condition与ReentrantLock绑定，所以ReentrantLock可以支持多个等待条件。也就是说ReentrantLock的等待集粒度更细，更加灵活。

4.定时的锁等待和可中断的锁等待

ReentrantLock支持定时的锁等待和可中断的锁等待，使用起来更加灵活，而且能在一定程度上避免死锁。

5.公平性

内置锁不能保证公平性，而ReentrantLock同时提供了公平的和非公平的实现，可以通过布尔类型的构造方法参数来指定是否是公平锁。公平锁会按照线程请求锁的顺序来获取锁，虽然牺牲了一定的吞吐量，但是保证了公平性。

 

ReentrantLock与synchronized间的抉择

虽然 `ReentrantLock` 相对 synchronized 来说，有一些重要的优势，但是急于把 synchronized 抛弃，绝对是个严重的错误。 *java.util.concurrent.lock 中的锁定类是用于高级用户和高级情况的工具* 。一般来说，除非对 `Lock` 的某个高级特性有明确的需要，或者有明确的证据（而不是仅仅是怀疑）表明在特定情况下，同步已经成为可伸缩性的瓶颈，否则还是应当继续使用 synchronized。

为什么我在一个显然“更好的”实现的使用上主张保守呢？因为对于 `java.util.concurrent.lock` 中的锁定类来说，synchronized 仍然有一些优势。比如，在使用 synchronized 的时候，不能忘记释放锁；在退出 `synchronized` 块时，JVM 会为您做这件事。您很容易忘记用 `finally` 块释放锁，这对程序非常有害。您的程序能够通过测试，但会在实际工作中出现死锁，那时会很难指出原因（这也是为什么根本不让初级开发人员使用 `Lock` 的一个好理由。）

另一个原因是因为，当 JVM 用 synchronized 管理锁定请求和释放时，JVM 在生成线程转储时能够包括锁定信息。这些对调试非常有价值，因为它们能标识死锁或者其他异常行为的来源。 `Lock` 类只是普通的类，JVM 不知道具体哪个线程拥有 `Lock` 对象。而且，几乎每个开发人员都熟悉 synchronized，它可以在 JVM 的所有版本中工作。

既然如此，我们什么时候才应该使用 `ReentrantLock` 呢？答案非常简单 —— 在确实需要一些 synchronized 所没有的特性的时候，比如时间锁等候、可中断锁等候、无块结构锁、公平队列。 `ReentrantLock` 还具有可伸缩性的好处，应当在高度争用的情况下使用它，但是请记住，大多数 synchronized 块几乎从来没有出现过争用，所以可以把高度争用放在一边。建议用 synchronized 开发，直到确实证明 synchronized 不合适，而不要仅仅是假设如果使用 `ReentrantLock` “性能会更好”。