RU全称是Least Recently Used，即最近最久未使用的意思。LRU算法的设计原则是：**如果一个数据在最近一段时间没有被访问到，那么在将来它被访问的可能性也很小**。也就是说，当限定的空间已存满数据时，应当把最久没有被访问到的数据淘汰。

# 实现LRU

​     

1.用一个数组来存储数据，给每一个数据项标记一个访问时间戳，每次插入新数据项的时候，先把数组中存在的数据项的时间戳自增，并将新数据项的时间戳置为0并插入到数组中。每次访问数组中的数据项的时候，将被访问的数据项的时间戳置为0。当数组空间已满时，将时间戳最大的数据项淘汰。



2.利用一个链表来实现，每次新插入数据的时候将新数据插到链表的头部；每次缓存命中（即数据被访问），则将数据移到链表头部；那么当链表满的时候，就将链表尾部的数据丢弃。



3.利用链表和hashmap。当需要插入新的数据项的时候，如果新数据项在链表中存在（一般称为命中），则把该节点移到链表头部，如果不存在，则新建一个节点，放到链表头部，若缓存满了，则把链表最后一个节点删除即可。在访问数据的时候，如果数据项在链表中存在，则把该节点移到链表头部，否则返回-1。这样一来在链表尾部的节点就是最近最久未访问的数据项。



对于第一种方法，需要不停地维护数据项的访问时间戳，另外，在插入数据、删除数据以及访问数据时，时间复杂度都是O(n)。对于第二种方法，链表在定位数据的时候时间复杂度为O(n)。所以在一般使用第三种方式来是实现LRU算法。

# 实现方案



使用LinkedHashMap实现

​     LinkedHashMap底层就是用的HashMap加双链表实现的，而且本身已经实现了按照访问顺序的存储。此外，LinkedHashMap中本身就实现了一个方法removeEldestEntry用于判断是否需要移除最不常读取的数，方法默认是直接返回false，不会移除元素，所以需要重写该方法。即当缓存满后就移除最不常用的数。

​    

```java
public class LRU<K,V> {

  private static final float hashLoadFactory = 0.75f;
  private LinkedHashMap<K,V> map;
  private int cacheSize;

  public LRU(int cacheSize) {
    this.cacheSize = cacheSize;
    int capacity = (int)Math.ceil(cacheSize / hashLoadFactory) + 1;
    map = new LinkedHashMap<K,V>(capacity, hashLoadFactory, true){
      private static final long serialVersionUID = 1;

      @Override
     protected boolean removeEldestEntry(Map.Entry eldest) {
        return size() > LRU.this.cacheSize;
      }
   };
  }

  public synchronized V get(K key) {
    return map.get(key);
  }

  public synchronized void put(K key, V value) {
    map.put(key, value);
  }

  public synchronized void clear() {
    map.clear();
  }

  public synchronized int usedSize() {
    return map.size();
  }

  public void print() {
    for (Map.Entry<K, V> entry : map.entrySet()) {
      System.out.print(entry.getValue() + "--");
    }
    System.out.println();

  }

}
```

当存在热点数据时，LRU的效率很好，但偶发性的、周期性的批量操作会导致LRU命中率急剧下降，缓存污染情况比较严重。

# 扩展

## 1.LRU-K



## 2.two queue



## 3.Multi Queue(MQ)