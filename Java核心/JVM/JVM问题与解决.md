## concurrent mode failure与promotion fail

**-XX:+UseConcMarkSweepGC 一般使用** PraNew+cms+Parallel Old （作为后援，当发生**concurrent mode failure时退化成**Parallel Old ）

**concurrent mode failure  ：1、cms在回收老年代，但是还没完成回收，此时yonggc 晋升的对象在老年代放不下时；2、直接在老年代申请空间不能满足时；**

两个原因：

- 在 CMS 启动过程中，新生代提升速度过快，老年代收集速度赶不上新生代提升速度
- 在 CMS 启动过程中，老年代碎片化严重，无法容纳新生代提升上来的大对象

解决办法：

- 新生代提升过快问题：（1）如果频率太快的话，说明空间不足，首先可以尝试调大新生代空间和晋升阈值。（2）如果内存有限，可以设置 CMS 垃圾收集在老年代占比达到多少时启动来减少问题发生频率（越早启动问题发生频率越低，但是会降低吞吐量，具体得多调整几次找到平衡点），参数如下：如果没有第二个参数，会随着 JVM 动态调节 CMS 启动时间

-XX:CMSInitiatingOccupancyFraction=68 （默认是 68）

-XX:+UseCMSInitiatingOccupancyOnly



- 老年代碎片严重问题：（1）如果频率太快或者 Full GC 后空间释放不多的话，说明空间不足，首先可以尝试调大老年代空间（2）如果内存不足，可以设置进行 n 次 CMS 后进行一次压缩式 Full GC，参数如下：

-XX:+UseCMSCompactAtFullCollection：允许在 Full GC 时，启用压缩式 GC

-XX:CMSFullGCBeforeCompaction=n     在进行 n 次，CMS 后，进行一次压缩的 Full GC，用以减少 CMS 产生的碎片

 

**promotion fail：在进行yonggc时，空闲的s1 放不下s0和Eden的对象，并且老年代也放不下或老年代碎片较多时；**

在 Minor GC 过程中，Survivor Unused 可能不足以容纳 Eden 和另一个 Survivor 中的存活对象， 那么多余的将被移到老年代， 称为过早提升（Premature Promotion）。 这会导致老年代中短期存活对象的增长， 可能会引发严重的性能问题。  再进一步， 如果老年代满了， Minor GC 后会进行 Full GC， 这将导致遍历整个堆， 称为提升失败（Promotion Failure）。



提升失败原因：Minor GC 时发现 Survivor 空间放不下，而老年代的空闲也不够

- 新生代提升太快
- 老年代碎片太多，放不下大对象提升（表现为老年代还有很多空间但是，出现了 promotion failed）

解决方法：

​       两条和上面 concurrent mode failure 一样

​       另一条，是因为 Survivor Unused 不足，那么可以尝试调大 Survivor 来尝试下 



**CMSInitiatingOccupancyFraction值与Xmn的关系公式**

上面介绍了promontion faild产生的原因是EDEN空间不足的情况下将EDEN与From survivor中的存活对象存入To survivor区时,To survivor区的空间不足，再次晋升到old gen区，而old gen区内存也不够的情况下产生了promontion faild从而导致full gc.那可以推断出：eden+from survivor < old gen区剩余内存时，不会出现promontion faild的情况，即：
(Xmx-Xmn)*(1-CMSInitiatingOccupancyFraction/100)>=(Xmn-Xmn/(SurvivorRatior+2))  进而推断出：

CMSInitiatingOccupancyFraction <=((Xmx-Xmn)-(Xmn-Xmn/(SurvivorRatior+2)))/(Xmx-Xmn)*100

例如：

当xmx=128 xmn=36 SurvivorRatior=1时 CMSInitiatingOccupancyFraction<=((128.0-36)-(36-36/(1+2)))/(128-36)*100 =73.913

当xmx=128 xmn=24 SurvivorRatior=1时 CMSInitiatingOccupancyFraction<=((128.0-24)-(24-24/(1+2)))/(128-24)*100=84.615…

当xmx=3000 xmn=600 SurvivorRatior=1时  CMSInitiatingOccupancyFraction<=((3000.0-600)-(600-600/(1+2)))/(3000-600)*100=83.33

CMSInitiatingOccupancyFraction低于70% 需要调整xmn或SurvivorRatior值。



参考：<https://www.cnblogs.com/redcreen/archive/2011/05/04/2037057.html>