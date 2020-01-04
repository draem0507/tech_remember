```java
新生代1500m
-Xmn1500m

元数据区256M

-XX:MetaspaceSize=256M -XX:MaxMetaspaceSize=256M

初始堆4g,最大堆4g

-Xmx4g -Xms4g

新生代采用ParNew，老年代采用CMS回收器，每次发生full GC后都做一次压缩，第一次当内存使用率达到80%时触发GC，之后由JVM自行判定触发时机

-XX:+UseConcMarkSweepGC
-XX:CMSFullGCsBeforeCompaction=0
-XX:+UseCMSCompactAtFullCollection
-XX:CMSInitiatingOccupancyFraction=80
```

[技术团队博客](https://km.sankuai.com/space/blog)/[到店-平台技术部-平台业...](https://km.sankuai.com/page/145262572)/[度假客户系统 GC 优化](https://km.sankuai.com/page/165199991)



度假客户系统 GC 优化

C-2创建:付磊, 最后修改: 付磊06-17 15:07

目录

[前言](https://km.sankuai.com/page/165199991#id-前言)

[年轻代](https://km.sankuai.com/page/165199991#id-年轻代)

[老年代](https://km.sankuai.com/page/165199991#id-老年代)

[方法区](https://km.sankuai.com/page/165199991#id-方法区)

[垃圾回收器](https://km.sankuai.com/page/165199991#id-垃圾回收器)

[现象与挑战](https://km.sankuai.com/page/165199991#id-现象与挑战)

[第一次优化](https://km.sankuai.com/page/165199991#id-第一次优化)

[确定目标](https://km.sankuai.com/page/165199991#id-确定目标)

[找到根源](https://km.sankuai.com/page/165199991#id-找到根源)

[参数优化](https://km.sankuai.com/page/165199991#id-参数优化)

[验收结果](https://km.sankuai.com/page/165199991#id-验收结果)

[第二次优化](https://km.sankuai.com/page/165199991#id-第二次优化)

[确定目标](https://km.sankuai.com/page/165199991#id-确定目标-1)

[找到根源](https://km.sankuai.com/page/165199991#id-找到根源-1)

[参数优化](https://km.sankuai.com/page/165199991#id-参数优化-1)

[验收结果](https://km.sankuai.com/page/165199991#id-验收结果-1)

[总结](https://km.sankuai.com/page/165199991#id-总结)

### 前言

首先简单介绍一下JVM的GC回收，GC即Garbage Collection，中文翻译称之为“垃圾回收”，他的作用是识别出内存中“不可再使用的”的对象，并自动回收，解放开发人员的双手。

但是，GC时常常伴随着GC停顿，称为Stop the world（下称STW），STW时应用程序的其他所有线程都会被挂起，所以一旦GC停顿时间越长，系统的可用性将越差。

GC作用的区域主要分为三块：年轻代（eden+s0+s1）、老年代（old）、方法区（Metaspace）。

![img](https://km.sankuai.com/api/file/cdn/165199991/165266327?contentType=1&isNewContent=true&isNewContent=false)



#### 年轻代

年轻代的主要存放刚刚新建的对象，其中的大多数对象都“朝生夕死”，GC后的存活率比较低。年轻代又分为三小块，eden区、s0区、s1区，其中s0、s1又统称为Survivor区。一般情况下，一个对象创建后首先放入eden区，经历一次GC后再复制到servior区，之后如果该对象一直不可被回收的话，每次GC后就会在s0、s1之间复制，直到复制次数达到设定的阈值，就会被认定为是“很难被回收的老对象”，最终复制到GC频率更低的老年代去。



#### 老年代

老年代主要存放一些经历过多次GC的对象，因为对象都是经历过多次GC的，所以可以认为是很难被回收，所以GC频率相对与年轻代来说要低的多。因为本身占用空间较大，每次GC后对象的存活率也高，所以一般采用的是标记清除+整理的GC策略。



#### 方法区

方法区在JDK7及JDK7之前采用永久代的方式Perm区实现，后来采用元数据区的方式Metaspace区。主要存放类定义、字节码和常量等比较稳健、几乎不会被回收的对象。



#### 垃圾回收器

垃圾回收器是指垃圾回收的具体实现，现如今较为流行的垃圾回收器搭配是新生代使用ParNew回收器，老年代使用CMS回收器。

ParNew回收器的特点就是充分利用多核CPU，利用多线程并行回收，以此提高回收效率。

![img](https://km.sankuai.com/api/file/cdn/165199991/165246344?contentType=1&isNewContent=true&isNewContent=false)



CMS则更加充分利用多核CPU，它会和用户线程交替着并发执行。其整个回收阶段主要分为四步

1.初始标记（CMS initial mark）

在初始标记阶段，CMS仅会标记一些GC Roots能直接关联的对象，会产生STW现象。

2.并发标记（CMS concurrent mark）

在初始标记的基础上标记那些可达的对象，与用户线程并行运行，不会产生STW现象。

3.重新标记（CMS remark）

因为在并发标记阶段，用户线程任在运行，所以可能产生新的垃圾对象需要回收，所以需要重新标记一下。这些垃圾对象成为浮动垃圾，浮动垃圾的多少会影响CMS remark阶段的效率，改阶段会产生STW现象。

4.并发清除（CMS concurrent mark）

标记完后，就已经知道哪些需要回收了，并发清理阶段就是回收这部分对象，它与用户线程并行运行，不会产生STW现象。

![img](https://km.sankuai.com/api/file/cdn/165199991/165271769?contentType=1&isNewContent=true&isNewContent=false)





### 现象与挑战

首先简单介绍下度假客户系统的一些情况，该系统有12台机器，QPS在400左右，平均每台机器的QPS为30+，垃圾回收器采用的是典型的ParNew+CMS。

当时的现象是每天都会这么一个有个时间点，其他调用端调用度假客户系统会发生超时，而且时间点不固定。通过在cat上的异常日志统计发现超时的异常都集中在一台机器上，故排除网络抖动的问题。再查看机器的GC情况，发现超时的时间点发生了时间较长时的GC。故认定超时的元凶是因为GC造成STW导致的，围绕这个结论展开一系列优化。

![img](https://km.sankuai.com/api/file/cdn/165199991/165291636?contentType=1&isNewContent=true&isNewContent=false)

![img](https://km.sankuai.com/api/file/cdn/165199991/165246234?contentType=1&isNewContent=true&isNewContent=false)



### 第一次优化

#### 确定目标

减少JVM GC时间，保证GC时调用端的调用不会超时。

#### 找到根源

前言中介绍过，GC作用于年轻代、老年代、方法区三个区域，首先需要确认的是，到底是GC回收哪个代的效率低下。

是方法区吗？通过cat查看当时metaspace的使用率，一直稳定在80%左右，不会因为元空间不足而触发STW，所以方法区的回收没有问题。

![img](https://km.sankuai.com/api/file/cdn/165199991/165254670?contentType=1&isNewContent=true&isNewContent=false)

是年轻代吗？查看younggc time的统计信息发现，一直稳定在40ms左右，所以年轻代的回收没有问题。

是老年代吗？通过fullgc time的统计信息发现，那个时间点发生了fullgc，且高达1600ms，所以可以认定是老年代回收时的fullgc导致的问题。



当时JVM的主要参数如下，年轻代1500M，老年代大约有2500M，CMSInitiatingOccupancyFraction的回收阈值是80%，没有什么明显的问题。

代码块

Plain Text









```
# 新生代1500m
-Xmn1500m

#元数据区256M
-XX:MetaspaceSize=256M -XX:MaxMetaspaceSize=256M

# 初始堆4g,最大堆4g
-Xmx4g -Xms4g

# 新生代采用ParNew，老年代采用CMS回收器，每次发生full GC后都做一次压缩，第一次当内存使用率达到80%时触发GC，之后由JVM自行判定触发时机
-XX:+UseConcMarkSweepGC
-XX:CMSFullGCsBeforeCompaction=0
-XX:+UseCMSCompactAtFullCollection
-XX:CMSInitiatingOccupancyFraction=80

```





遂通过运行下面几个命令，翻下GC日志查看导致STW的几个可能，发现有”promotion failed“晋升失败相关的日志，而一旦晋升失败将会引起Serial Old形式的Full gc。

再回头看，老年代最小的内存空间为：Old*（1-CMSInitiatingOccupancyFraction）=2500*（1-80%）= 500M，

Survivor区域大小 Survivor=1500/（1+1+8）* 2 = 300M

Eden区大小 New=1500/（1+1+8）* 8 = 1200M

新生代向老生代一次晋升最大可达1500M，极有可能发生promotion failed，所以需要调低CMS回收阈值、减小年轻代大小，减少发生promotion failed可能性。





```
# 查看具体的fullgc日志

less virgo.gc.log | grep "full"

# 查看CMS remark阶段的耗时

less virgo.gc.log | grep "CMS-final-remark"

# 查看有没有CMF

less virgo.gc.log | grep "concurrent mode failure"

# 查看有没有晋升失败

less virgo.gc.log | grep "promotion failed"
```

[技术团队博客](https://km.sankuai.com/space/blog)/[到店-平台技术部-平台业...](https://km.sankuai.com/page/145262572)/[度假客户系统 GC 优化](https://km.sankuai.com/page/165199991)



度假客户系统 GC 优化

C-2创建:付磊, 最后修改: 付磊06-17 15:07

目录

[前言](https://km.sankuai.com/page/165199991#id-前言)

[年轻代](https://km.sankuai.com/page/165199991#id-年轻代)

[老年代](https://km.sankuai.com/page/165199991#id-老年代)

[方法区](https://km.sankuai.com/page/165199991#id-方法区)

[垃圾回收器](https://km.sankuai.com/page/165199991#id-垃圾回收器)

[现象与挑战](https://km.sankuai.com/page/165199991#id-现象与挑战)

[第一次优化](https://km.sankuai.com/page/165199991#id-第一次优化)

[确定目标](https://km.sankuai.com/page/165199991#id-确定目标)

[找到根源](https://km.sankuai.com/page/165199991#id-找到根源)

[参数优化](https://km.sankuai.com/page/165199991#id-参数优化)

[验收结果](https://km.sankuai.com/page/165199991#id-验收结果)

[第二次优化](https://km.sankuai.com/page/165199991#id-第二次优化)

[确定目标](https://km.sankuai.com/page/165199991#id-确定目标-1)

[找到根源](https://km.sankuai.com/page/165199991#id-找到根源-1)

[参数优化](https://km.sankuai.com/page/165199991#id-参数优化-1)

[验收结果](https://km.sankuai.com/page/165199991#id-验收结果-1)

[总结](https://km.sankuai.com/page/165199991#id-总结)

### 前言

首先简单介绍一下JVM的GC回收，GC即Garbage Collection，中文翻译称之为“垃圾回收”，他的作用是识别出内存中“不可再使用的”的对象，并自动回收，解放开发人员的双手。

但是，GC时常常伴随着GC停顿，称为Stop the world（下称STW），STW时应用程序的其他所有线程都会被挂起，所以一旦GC停顿时间越长，系统的可用性将越差。

GC作用的区域主要分为三块：年轻代（eden+s0+s1）、老年代（old）、方法区（Metaspace）。

![img](https://km.sankuai.com/api/file/cdn/165199991/165266327?contentType=1&isNewContent=true&isNewContent=false)



#### 年轻代

年轻代的主要存放刚刚新建的对象，其中的大多数对象都“朝生夕死”，GC后的存活率比较低。年轻代又分为三小块，eden区、s0区、s1区，其中s0、s1又统称为Survivor区。一般情况下，一个对象创建后首先放入eden区，经历一次GC后再复制到servior区，之后如果该对象一直不可被回收的话，每次GC后就会在s0、s1之间复制，直到复制次数达到设定的阈值，就会被认定为是“很难被回收的老对象”，最终复制到GC频率更低的老年代去。



#### 老年代

老年代主要存放一些经历过多次GC的对象，因为对象都是经历过多次GC的，所以可以认为是很难被回收，所以GC频率相对与年轻代来说要低的多。因为本身占用空间较大，每次GC后对象的存活率也高，所以一般采用的是标记清除+整理的GC策略。



#### 方法区

方法区在JDK7及JDK7之前采用永久代的方式Perm区实现，后来采用元数据区的方式Metaspace区。主要存放类定义、字节码和常量等比较稳健、几乎不会被回收的对象。



#### 垃圾回收器

垃圾回收器是指垃圾回收的具体实现，现如今较为流行的垃圾回收器搭配是新生代使用ParNew回收器，老年代使用CMS回收器。

ParNew回收器的特点就是充分利用多核CPU，利用多线程并行回收，以此提高回收效率。

![img](https://km.sankuai.com/api/file/cdn/165199991/165246344?contentType=1&isNewContent=true&isNewContent=false)



CMS则更加充分利用多核CPU，它会和用户线程交替着并发执行。其整个回收阶段主要分为四步

1.初始标记（CMS initial mark）

在初始标记阶段，CMS仅会标记一些GC Roots能直接关联的对象，会产生STW现象。

2.并发标记（CMS concurrent mark）

在初始标记的基础上标记那些可达的对象，与用户线程并行运行，不会产生STW现象。

3.重新标记（CMS remark）

因为在并发标记阶段，用户线程任在运行，所以可能产生新的垃圾对象需要回收，所以需要重新标记一下。这些垃圾对象成为浮动垃圾，浮动垃圾的多少会影响CMS remark阶段的效率，改阶段会产生STW现象。

4.并发清除（CMS concurrent mark）

标记完后，就已经知道哪些需要回收了，并发清理阶段就是回收这部分对象，它与用户线程并行运行，不会产生STW现象。

![img](https://km.sankuai.com/api/file/cdn/165199991/165271769?contentType=1&isNewContent=true&isNewContent=false)





### 现象与挑战

首先简单介绍下度假客户系统的一些情况，该系统有12台机器，QPS在400左右，平均每台机器的QPS为30+，垃圾回收器采用的是典型的ParNew+CMS。

当时的现象是每天都会这么一个有个时间点，其他调用端调用度假客户系统会发生超时，而且时间点不固定。通过在cat上的异常日志统计发现超时的异常都集中在一台机器上，故排除网络抖动的问题。再查看机器的GC情况，发现超时的时间点发生了时间较长时的GC。故认定超时的元凶是因为GC造成STW导致的，围绕这个结论展开一系列优化。

![img](https://km.sankuai.com/api/file/cdn/165199991/165291636?contentType=1&isNewContent=true&isNewContent=false)

![img](https://km.sankuai.com/api/file/cdn/165199991/165246234?contentType=1&isNewContent=true&isNewContent=false)



### 第一次优化

#### 确定目标

减少JVM GC时间，保证GC时调用端的调用不会超时。

#### 找到根源

前言中介绍过，GC作用于年轻代、老年代、方法区三个区域，首先需要确认的是，到底是GC回收哪个代的效率低下。

是方法区吗？通过cat查看当时metaspace的使用率，一直稳定在80%左右，不会因为元空间不足而触发STW，所以方法区的回收没有问题。

![img](https://km.sankuai.com/api/file/cdn/165199991/165254670?contentType=1&isNewContent=true&isNewContent=false)

是年轻代吗？查看younggc time的统计信息发现，一直稳定在40ms左右，所以年轻代的回收没有问题。

是老年代吗？通过fullgc time的统计信息发现，那个时间点发生了fullgc，且高达1600ms，所以可以认定是老年代回收时的fullgc导致的问题。



当时JVM的主要参数如下，年轻代1500M，老年代大约有2500M，CMSInitiatingOccupancyFraction的回收阈值是80%，没有什么明显的问题。

代码块

Plain Text









```
# 新生代1500m
-Xmn1500m

#元数据区256M
-XX:MetaspaceSize=256M -XX:MaxMetaspaceSize=256M

# 初始堆4g,最大堆4g
-Xmx4g -Xms4g

# 新生代采用ParNew，老年代采用CMS回收器，每次发生full GC后都做一次压缩，第一次当内存使用率达到80%时触发GC，之后由JVM自行判定触发时机
-XX:+UseConcMarkSweepGC
-XX:CMSFullGCsBeforeCompaction=0
-XX:+UseCMSCompactAtFullCollection
-XX:CMSInitiatingOccupancyFraction=80

```





遂通过运行下面几个命令，翻下GC日志查看导致STW的几个可能，发现有”promotion failed“晋升失败相关的日志，而一旦晋升失败将会引起Serial Old形式的Full gc。

再回头看，老年代最小的内存空间为：Old*（1-CMSInitiatingOccupancyFraction）=2500*（1-80%）= 500M，

Survivor区域大小 Survivor=1500/（1+1+8）* 2 = 300M

Eden区大小 New=1500/（1+1+8）* 8 = 1200M

新生代向老生代一次晋升最大可达1500M，极有可能发生promotion failed，所以需要调低CMS回收阈值、减小年轻代大小，减少发生promotion failed可能性。

代码块

Plain Text









```
# 查看具体的fullgc日志
less virgo.gc.log | grep "full"

# 查看CMS remark阶段的耗时
less virgo.gc.log | grep "CMS-final-remark"

# 查看有没有CMF
less virgo.gc.log | grep "concurrent mode failure"

# 查看有没有晋升失败
less virgo.gc.log | grep "promotion failed"

```





之后经新宇提示，又加上“-XX:+PrintReferenceGC”参数，细化GC日志中对系统内的软引用、弱引用、虚引用的回收耗时。加上后发现，发现回收这些引用的耗时+类卸载的耗时成为了整个CMS reamrk阶段的瓶颈。由于JDK8在Linux环境会默认开启CMSClassUnloadingEnabled，这会使得CMS在CMS-remark阶段尝试进行类的卸载，然而通过cat可以得知Metaspace空间稳定，无需每次fullgc都去主动卸载，故需要显示关闭CMSClassUnloadingEnabled。



优化

```
# 关闭类卸载

-XX:-CMSClassUnloadingEnabled

# 调整CMS回收阈值，改阈值后期由CMS自动调整

-XX:CMSInitiatingOccupancyFraction=65
-XX:+UseCMSInitiatingOccupancyOnly

# 新生代1000m

-Xmn1000m

# eden:from:to = 8:1:1

-XX:SurvivorRatio=8
```



结果：full  gc 控制在500ms左右



[技术团队博客](https://km.sankuai.com/space/blog)/[到店-平台技术部-平台业...](https://km.sankuai.com/page/145262572)/[度假客户系统 GC 优化](https://km.sankuai.com/page/165199991)



度假客户系统 GC 优化

C-2创建:付磊, 最后修改: 付磊06-17 15:07

目录

[前言](https://km.sankuai.com/page/165199991#id-前言)

[年轻代](https://km.sankuai.com/page/165199991#id-年轻代)

[老年代](https://km.sankuai.com/page/165199991#id-老年代)

[方法区](https://km.sankuai.com/page/165199991#id-方法区)

[垃圾回收器](https://km.sankuai.com/page/165199991#id-垃圾回收器)

[现象与挑战](https://km.sankuai.com/page/165199991#id-现象与挑战)

[第一次优化](https://km.sankuai.com/page/165199991#id-第一次优化)

[确定目标](https://km.sankuai.com/page/165199991#id-确定目标)

[找到根源](https://km.sankuai.com/page/165199991#id-找到根源)

[参数优化](https://km.sankuai.com/page/165199991#id-参数优化)

[验收结果](https://km.sankuai.com/page/165199991#id-验收结果)

[第二次优化](https://km.sankuai.com/page/165199991#id-第二次优化)

[确定目标](https://km.sankuai.com/page/165199991#id-确定目标-1)

[找到根源](https://km.sankuai.com/page/165199991#id-找到根源-1)

[参数优化](https://km.sankuai.com/page/165199991#id-参数优化-1)

[验收结果](https://km.sankuai.com/page/165199991#id-验收结果-1)

[总结](https://km.sankuai.com/page/165199991#id-总结)

### 前言

首先简单介绍一下JVM的GC回收，GC即Garbage Collection，中文翻译称之为“垃圾回收”，他的作用是识别出内存中“不可再使用的”的对象，并自动回收，解放开发人员的双手。

但是，GC时常常伴随着GC停顿，称为Stop the world（下称STW），STW时应用程序的其他所有线程都会被挂起，所以一旦GC停顿时间越长，系统的可用性将越差。

GC作用的区域主要分为三块：年轻代（eden+s0+s1）、老年代（old）、方法区（Metaspace）。

![img](https://km.sankuai.com/api/file/cdn/165199991/165266327?contentType=1&isNewContent=true&isNewContent=false)



#### 年轻代

年轻代的主要存放刚刚新建的对象，其中的大多数对象都“朝生夕死”，GC后的存活率比较低。年轻代又分为三小块，eden区、s0区、s1区，其中s0、s1又统称为Survivor区。一般情况下，一个对象创建后首先放入eden区，经历一次GC后再复制到servior区，之后如果该对象一直不可被回收的话，每次GC后就会在s0、s1之间复制，直到复制次数达到设定的阈值，就会被认定为是“很难被回收的老对象”，最终复制到GC频率更低的老年代去。



#### 老年代

老年代主要存放一些经历过多次GC的对象，因为对象都是经历过多次GC的，所以可以认为是很难被回收，所以GC频率相对与年轻代来说要低的多。因为本身占用空间较大，每次GC后对象的存活率也高，所以一般采用的是标记清除+整理的GC策略。



#### 方法区

方法区在JDK7及JDK7之前采用永久代的方式Perm区实现，后来采用元数据区的方式Metaspace区。主要存放类定义、字节码和常量等比较稳健、几乎不会被回收的对象。



#### 垃圾回收器

垃圾回收器是指垃圾回收的具体实现，现如今较为流行的垃圾回收器搭配是新生代使用ParNew回收器，老年代使用CMS回收器。

ParNew回收器的特点就是充分利用多核CPU，利用多线程并行回收，以此提高回收效率。

![img](https://km.sankuai.com/api/file/cdn/165199991/165246344?contentType=1&isNewContent=true&isNewContent=false)



CMS则更加充分利用多核CPU，它会和用户线程交替着并发执行。其整个回收阶段主要分为四步

1.初始标记（CMS initial mark）

在初始标记阶段，CMS仅会标记一些GC Roots能直接关联的对象，会产生STW现象。

2.并发标记（CMS concurrent mark）

在初始标记的基础上标记那些可达的对象，与用户线程并行运行，不会产生STW现象。

3.重新标记（CMS remark）

因为在并发标记阶段，用户线程任在运行，所以可能产生新的垃圾对象需要回收，所以需要重新标记一下。这些垃圾对象成为浮动垃圾，浮动垃圾的多少会影响CMS remark阶段的效率，改阶段会产生STW现象。

4.并发清除（CMS concurrent mark）

标记完后，就已经知道哪些需要回收了，并发清理阶段就是回收这部分对象，它与用户线程并行运行，不会产生STW现象。

![img](https://km.sankuai.com/api/file/cdn/165199991/165271769?contentType=1&isNewContent=true&isNewContent=false)





### 现象与挑战

首先简单介绍下度假客户系统的一些情况，该系统有12台机器，QPS在400左右，平均每台机器的QPS为30+，垃圾回收器采用的是典型的ParNew+CMS。

当时的现象是每天都会这么一个有个时间点，其他调用端调用度假客户系统会发生超时，而且时间点不固定。通过在cat上的异常日志统计发现超时的异常都集中在一台机器上，故排除网络抖动的问题。再查看机器的GC情况，发现超时的时间点发生了时间较长时的GC。故认定超时的元凶是因为GC造成STW导致的，围绕这个结论展开一系列优化。

![img](https://km.sankuai.com/api/file/cdn/165199991/165291636?contentType=1&isNewContent=true&isNewContent=false)

![img](https://km.sankuai.com/api/file/cdn/165199991/165246234?contentType=1&isNewContent=true&isNewContent=false)



### 第一次优化

#### 确定目标

减少JVM GC时间，保证GC时调用端的调用不会超时。

#### 找到根源

前言中介绍过，GC作用于年轻代、老年代、方法区三个区域，首先需要确认的是，到底是GC回收哪个代的效率低下。

是方法区吗？通过cat查看当时metaspace的使用率，一直稳定在80%左右，不会因为元空间不足而触发STW，所以方法区的回收没有问题。

![img](https://km.sankuai.com/api/file/cdn/165199991/165254670?contentType=1&isNewContent=true&isNewContent=false)

是年轻代吗？查看younggc time的统计信息发现，一直稳定在40ms左右，所以年轻代的回收没有问题。

是老年代吗？通过fullgc time的统计信息发现，那个时间点发生了fullgc，且高达1600ms，所以可以认定是老年代回收时的fullgc导致的问题。



当时JVM的主要参数如下，年轻代1500M，老年代大约有2500M，CMSInitiatingOccupancyFraction的回收阈值是80%，没有什么明显的问题。

代码块

Plain Text









```
# 新生代1500m
-Xmn1500m

#元数据区256M
-XX:MetaspaceSize=256M -XX:MaxMetaspaceSize=256M

# 初始堆4g,最大堆4g
-Xmx4g -Xms4g

# 新生代采用ParNew，老年代采用CMS回收器，每次发生full GC后都做一次压缩，第一次当内存使用率达到80%时触发GC，之后由JVM自行判定触发时机
-XX:+UseConcMarkSweepGC
-XX:CMSFullGCsBeforeCompaction=0
-XX:+UseCMSCompactAtFullCollection
-XX:CMSInitiatingOccupancyFraction=80

```





遂通过运行下面几个命令，翻下GC日志查看导致STW的几个可能，发现有”promotion failed“晋升失败相关的日志，而一旦晋升失败将会引起Serial Old形式的Full gc。

再回头看，老年代最小的内存空间为：Old*（1-CMSInitiatingOccupancyFraction）=2500*（1-80%）= 500M，

Survivor区域大小 Survivor=1500/（1+1+8）* 2 = 300M

Eden区大小 New=1500/（1+1+8）* 8 = 1200M

新生代向老生代一次晋升最大可达1500M，极有可能发生promotion failed，所以需要调低CMS回收阈值、减小年轻代大小，减少发生promotion failed可能性。

代码块

Plain Text









```
# 查看具体的fullgc日志
less virgo.gc.log | grep "full"

# 查看CMS remark阶段的耗时
less virgo.gc.log | grep "CMS-final-remark"

# 查看有没有CMF
less virgo.gc.log | grep "concurrent mode failure"

# 查看有没有晋升失败
less virgo.gc.log | grep "promotion failed"

```





之后经新宇提示，又加上“-XX:+PrintReferenceGC”参数，细化GC日志中对系统内的软引用、弱引用、虚引用的回收耗时。加上后发现，发现回收这些引用的耗时+类卸载的耗时成为了整个CMS reamrk阶段的瓶颈。由于JDK8在Linux环境会默认开启CMSClassUnloadingEnabled，这会使得CMS在CMS-remark阶段尝试进行类的卸载，然而通过cat可以得知Metaspace空间稳定，无需每次fullgc都去主动卸载，故需要显示关闭CMSClassUnloadingEnabled。

代码块

Plain Text









```
弱引用处理：[weak refs processing, 3.2391236 secs]75139.975:
类卸载： [class unloading, 2.8267804 secs]75142.802: 
清理MetaSpace符号表：[scrub symbol table, 1.8053134 secs]

```





#### 参数优化

加上或者调整这些参数，各机房灰度发布一台机器查看fullgc情况

代码块

Plain Text









```
# 关闭类卸载
-XX:-CMSClassUnloadingEnabled

# 调整CMS回收阈值，改阈值后期由CMS自动调整
-XX:CMSInitiatingOccupancyFraction=65
-XX:+UseCMSInitiatingOccupancyOnly

# 新生代1000m
-Xmn1000m

# eden:from:to = 8:1:1
-XX:SurvivorRatio=8

```





#### 验收结果

箭头处为正式上线的时间点，由图可知java fullgc time指标已稳定在500ms左右，查看每次GC时间点的异常日志统计也没有发现超时告警，此次目标达成。

![img](https://km.sankuai.com/api/file/cdn/165199991/165254586?contentType=2&isNewContent=true&isNewContent=false)





### 第二次优化

#### 确定目标

第一次优化后，发现几乎所有机器的jvm.fullgc time 指标都超过了300ms，偶尔会出现超过500ms的情况，考虑到每台机器平均30+的QPS，这种表现不太可接受。估需要进一步优化，确定目标为：保证JVM GC时间在300ms以下，且GC频率不变。



#### 找到根源

继续分析GC日志找fullgc的瓶颈，随机拿三台看下日志，以xr03机器举例，发现JVM回收FinalReference时消耗了0.3852042 s，而整个CMS-remark阶段也才耗时0.43 s，其他机器的日志也是类似的情况，故JVM回收FinalReference的耗时是CMS-remark的瓶颈所在。



```
2019-05-14T05:13:30.481+0800: 43847.581: 
[GC (CMS Final Remark) 
[YG occupancy: 90688 K (921600 K)]
43847.581: [Rescan (parallel) , 0.0304719 secs]
43847.611: [weak refs processing
    43847.611: [SoftReference, 17765 refs, 0.0034378 secs]
    43847.615: [WeakReference, 23542 refs, 0.0015734 secs]
    43847.616: [FinalReference, 333319 refs, 0.3852042 secs]
    43848.001: [PhantomReference, 40 refs, 3 refs, 0.0038779 secs]
    43848.005: [JNI Weak Reference, 0.0000273 secs], 0.3941897 secs]
```



**那么为什么FinalReference会这么多呢？**

FinalReference对象的产生是因为程序中使用了实现finalize()方法的类，常见的如SocketImpl、IO流等，实现了finalize()方法的类实例会影响GC效率。

通过命令jmap -histo常看当前堆的统计信息，发现SocketImpl对象的数量相比于其他类似的服务多很多，而SocketImpl实现了finalize()方法，所以会产生大量的FinalReference对象，这下就就通了。

**可SocketImp又是哪里来的？**

通过MAT工具，发现SocketImpl最终都被Thrift连接池所引用，所以确认是Thrift连接池的问题。





原文地址：https://km.sankuai.com/page/165199991

