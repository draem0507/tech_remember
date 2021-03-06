提高CPU的使用率，一个普遍的思路就是任务分解，使用多线程并发执行任务，而问题重点在于如何设定线程的数量。

影响线程数量的因素有：

1.CPU的数量

需要的线程数与CPU的数量直接相关。例如，在一个具有4个CPU的机器上跑3个线程，那么显然CPU的使用率至多只能达到75%而已。所以要想跑满CPU，线程的数量不能少于CPU的数量。

2.任务的类型

任务可以分为计算密集型和IO密集型。对于计算密集型任务，可以近似认为，线程执行时对单个CPU的使用率是100%。对于IO密集型任务，设运算与IO等待的时间比例是a:b。那么任务执行时对单个CPU的使用率是a/(a+b)。

综合以上两点因素，可以得出如下结论：

设共有n个CPU，则对于计算密集型任务，线程数可以设为n+1。之所以线程数比CPU个数多1，是为了保证在某个线程因为缺页或其他原因发生阻塞时，仍然能有候补线程顶替上来，避免时间片的浪费。

而对于IO密集型任务，单个线程的CPU使用率是a/(a+b)，那么为了使该CPU的使用率达到100%，需要(a+b)/a个线程。再考虑CPU的个数n，那么线程数可以设为n*（a+b)/a。