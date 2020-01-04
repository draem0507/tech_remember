# 背景

最近在[sqlreview](http://sqlreview.dba.dp/)上看到一些long sql，比如[一个案例](http://sqlreview.dba.dp/sqlreview/singleSlowLogDetail?time=2016-12-25&fpChecksum=7ba6f79c41d5cdaf1e40525ec9de38c7&dbName=SecondHand&owner=pengfei.shen&ip=10.1.101.169&port=3306&serverIp=10.3.27.39&total=1&count=1&dictIpCount={'max': 1, '10.3.27.39': 1})，这条SQL最近出现了三次long sql，少则140ms多则400ms。

 

# 案例SQL分析

先看一下案例SQL：

代码块

Plain Text









```
-- SQL last time occurred
/*id:adb9d2ad*/SELECT ID,
       ProductID,
       Url,
       Description,
       Status,
       AddTime,
       UpdateTime
FROM   SH_ProductPicture
WHERE  ProductID IN ( 969 )
       AND Status IN ( 1, 2 )
ORDER  BY ID ASC;
```



 

SH_ProductPicture这张表为ProductID和Status建立了一个联合索引名称为IX_ProductID_Status，这条SQL的WHERE条件命中索引没问题，通过EXPLAIN查看，Extra字段为Using index condition，Using filesort。排除其他的不确定性因素，这条SQL至少可以把filesort优化掉。

 

# Order By理论

B+树索引是按键顺序存储的，是有序的，因此当查询条件和排序键走相同的索引的时候就有可能通过索引来实现order by。而上面的案例where条件走的是IX_ProductID_Status索引，而排序走的是主键索引，因此当存储引起从IX_ProductID_Status索引上拿到数据之后还需要一次的额外排序，这个排序叫做filesort，显然如果通过索引直接实现order by是极好的。

# Filesort算法

这里通过了解filesort算法来理解filesort为什么不好，我们为什么要尽量避免filesort，filesort常见的有两种算法，另外还有一种这里不做解释。

## 算法一

1）先通过WHERE根据索引查询记录

2）对于每一条记录，构造一个包括排序key和row id的元组存储在sort buffer中

3）如果所有的元组都能够存储在sort buffer中，则不需要创建临时文件，否则当sort满了的时候需要将sort中的所有元组使用快速排序然后将排序结果存储在临时文件中，并保存指向这个临时文件的指针

4）重复上面的步骤知道所有的记录都被处理完

5）针对上面的临时文件在通过MERGEBUFF进行多路归并然后再存储于临时文件

6）最后一次归并只讲row id存储于结果文件

7）最后最后再通过有序的row id读取记录，row buffer大小通过read_rnd_buffer_size指定

算法一的问题在于需要两次读记录。

## 算法二

1）通过WHERE根据索引查询记录

2）对每条记录将构造排序key和查询关联的所有数据

3）通过sort buffer以及临时文件排序

4）归并排序，将结果返回

那么这两种算法如何选择呢，MySQL是通过max_length_for_sort_data参数确认。

# SQL优化

我们只要用把用来排序的字段加入索引并且让查询用到索引排序，那么filesort就被消除了，于是我们加个IX_ProductID_ID索引建立ProductID与ID的联合索引，这样理论上就可以支持索引排序了，添加索引后EXPLAIN的Extra为：

```
Using index condition; Using where
```

此时Using filesort被消灭了。

# 总结

案例SQL出现long sql的原因暂无定论，但是至少确定了这条SQL可以优化的地方，至于案例SQL为什么会导致filesort，用MySQL官网的一句话描述为：

`The index used to fetch the rows differs from the one used in the ORDER BY`

https://km.sankuai.com/page/20679296