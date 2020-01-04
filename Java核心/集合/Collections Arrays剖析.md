# **Collections**

## **Collection与Collections**

​     Collection是集合的最顶层接口，提供了对集合对象进行基本操作的通用接口方法。为各种具体的集合提供了最大化的统一操作方式。

​     Collentions是一个工具类。它包含各种有关集合操作的静态多态方法，此类的构造方法为private，不能被实例化。

## **Collections方法：**

### **空集合**

空集合指的是没有元素在这些集合中，特别需要主要的是返回的集合都是只读的。只要更改值就会抛出UnsupportedOperationException异常

Collections.EMPTY_LIST,Collections.emptyList()——返回

**只读** 

的空LIST 集合

Collections.EMPTY_MAP,Collections.emptyMap()——返回

只读 

的空MAP集合

Collections.EMPTY_SET,Collections.emptySet()——返回**只读** 的空SET集合

其实底层有一个Empty*类，就是返回一个新的实例。

注：只读且为空，不知道干嘛用

### **单元素集合**

Collections中的单元素集合指的是集合只有一个元素而且集合只读。

Collections.singletonList——用来生成

**只读** 

的单一元素的List

Collections.singletonMap——用来生成

**只读** 

的单Key和Value组成的Map

Collections.singleton——用来生成**只读** 的单一元素的Set

其实底层有一个singleton*类，调用方法就返回一个实例

### **只读集合**

这些集合一旦初始化以后就不能修改，任何修改这些集合的方法都会抛出UnsupportedOperationException异常。

unmodifiableCollection

unmodifiableList

unmodifiableMap

unmodifiableSet

unmodifiableSortedMap

unmodifiableSortedSet

以上方法都顾名思义，不在赘述。

### **Checked集合**

Checked集合具有检查插入集合元素类型的特性，例如当我们设定checkedList中元素的类型是String的时候，如果插入起来类型的元素就会抛出ClassCastExceptions异常。

checkedCollection

checkedList

checkedMap

checkedSet

checkedSortedMap

checkedSortedSet

以上方法都顾名思义，不在赘述。

### **同步集合**

Collections的synchronizedXxxxx系列方法顾名思义会返回同步化集合类（SynchronizedMap，

SynchronizedList等等）。

这些集合类内部实现都是通过一个mutex（互斥体）来实现对这些集合操作的同步化。

### **替换查找**

#### fill——使用指定元素替换指定列表中的所有元素。

public static <T> void fill(List<? super T> list, T obj) {

​    int size = list.size();

​    if (size < FILL_THRESHOLD || list instanceof RandomAccess) {

​        for (int i=0; i<size; i++)

​            list.set(i, obj);

​    } else {

​        ListIterator<? super T> itr = list.listIterator();

​        for (int i=0; i<size; i++) {

​            itr.next();

​            itr.set(obj);

​        }

​    }

}

这里可以看出有一个域值FILL_THRESHOLD，这个值为25，如果小于25，或者实现随机访问接口的，直接set。否则使用迭代器的set。

#### frequency——返回指定 collection 中等于指定对象的元素数。

public static int frequency(Collection<?> c, Object o) {

​    int result = 0;

​    if (o == null) {

​        for (Object e : c)

​            if (e == null)

​                result++;

​    } else {

​        for (Object e : c)

​            if (o.equals(e))

​                result++;

​    }

​    return result;

}

为null的时候用==判断，否则用equals(Object中equals也是用的==判断)。 

#### indexOfSubList—— 返回指定源列表中第一次出现指定目标列表的起始位置，如果没有出现这样的列表，则返回 -1。

#### lastIndexOfSubList——返回指定源列表中最后一次出现指定目标列表的起始位置，如果没有出现这样的列表，则返回-1。

#### max—— 返回给定 collection 的最大元素。

public static <T> T max(Collection<? extends T> coll, Comparator<? super T> comp) {

​    if (comp==null)

​        return (T)max((Collection) coll);

​    Iterator<? extends T> i = coll.iterator();

​    T candidate = i.next();

​    while (i.hasNext()) {

​        T next = i.next();

​        if (comp.compare(next, candidate) > 0)

​            candidate = next;

​    }

​    return candidate;

}

如果没有实现comparator接口，就以自然顺序排序。是否使用compare比较。

注：comparable和comparator的区别。

​     Comparator在util包下，Comparable在lang包下。java中的对象排序都是以comparable接口为标准的。comparator是在对象外部实现排序。

#### min—— 返回给定 collection 的最小元素。

#### replaceAll——使用另一个值替换列表中出现的所有某一指定值。

## **排序** 

#### reverse——对List中的元素倒序排列

public static void reverse(List<?> list) {

​    int size = list.size();

​    if (size < REVERSE_THRESHOLD || list instanceof RandomAccess) {

​        for (int i=0, mid=size>>1, j=size-1; i<mid; i++, j--)

​            swap(list, i, j);

​    } else {

​        // instead of using a raw type here, it's possible to capture

​        // the wildcard but it will require a call to a supplementary

​        // private method

​        ListIterator fwd = list.listIterator();

​        ListIterator rev = list.listIterator(size);

​        for (int i=0, mid=list.size()>>1; i<mid; i++) {

​            Object tmp = fwd.next();

​            fwd.set(rev.previous());

​            rev.set(tmp);

​        }

​    }

}

都是分为两种情况进行处理，感觉是数组和链表的关系。不使用迭代器的直接使用swap方法，使用迭代器的需要两个迭代器。

#### shuffle——对List中的元素随机排列，这个方法让我想到了Apple的iPod Shuffle

#### sort——对List中的元素排序

public static <T extends Comparable<? super T>> void sort(List<T> list) {

​    list.sort(null);

}

public static <T> void sort(List<T> list, Comparator<? super T> c) {

​    list.sort(c);

}

sort方法底层其实就是调用List接口的sort方法（java8中List接口使用default关键字实现了sort方法）,使用sort方法可以根据元素的自然顺序 对指定列表按升序进行排序。列表中的所有元素都必须实现 Comparable 接口。此列表内的所有元素都必须是使用指定比较器可相互比较的 。下面看看list.sort()的代码：

 

default void sort(Comparator<? super E> c) {

​    Object[] a = this.toArray();
​    Arrays.sort(a, (Comparator) c);
​    ListIterator<E> i = this.listIterator();
​    for (Object e : a) {
​        i.next();
​        i.set((E) e);
​    }

}

发现其实底层是调用的Arrays.sort()方法，然后调用迭代器给现在的list赋值。 

 

#### swap——交换List中某两个指定下标位元素在集合中的位置。

public static void swap(List<?> list, int i, int j) {

​    // instead of using a raw type here, it's possible to capture

​    // the wildcard but it will require a call to a supplementary

​    // private method

​    final List l = list;

​    l.set(i, l.set(j, l.get(i)));

}

/**

 \* Swaps the two specified elements in the specified array.

 */

private static void swap(Object[] arr, int i, int j) {

​    Object tmp = arr[i];

​    arr[i] = arr[j];

​    arr[j] = tmp;

}

同样，对于链表和数组有不同的操作方式。

#### rotate——循环移动。

​     假设list包含[t, a, b, k, s]。在调用Collection.rotate(list, 1) 或者 Collection.rotate(list, -4) 后， list将包含[s, t, a, b, k]。

## **Arrays**

Arrays位于java.util包下，是一个工具类，主要提供了对数组的一些操作。

### **Arrays方法：**

**fill——给数组赋值**。fill的重载函数非常的多，这里只给出某一个的代码：

public static void fill(double[] a, double val) {

​    for (int i = 0, len = a.length; i < len; i++)

​        a[i] = val;

}

其实就是一个for循环来赋值，清晰明了

**sort——排序**，sort的重载函数也很多，下面主要看两种实现。

public static void sort(byte[] a) {

​    DualPivotQuicksort.sort(a, 0, a.length - 1);

}

public static void sort(byte[] a, int fromIndex, int toIndex) {
    rangeCheck(a.length, fromIndex, toIndex);
    DualPivotQuicksort.sort(a, fromIndex, toIndex - 1);

}

如果说指定了排序范围，先要check一下是否合法，然后都是调用这个DualPivotQuicksort.sort方法，下面就来看看这个方法：

static void sort(byte[] a, int left, int right) {
    // Use counting sort on large arrays
    if (right - left > COUNTING_SORT_THRESHOLD_FOR_BYTE) {
        int[] count = new int[NUM_BYTE_VALUES];

​        for (int i = left - 1; ++i <= right;
​            count[a[i] - Byte.MIN_VALUE]++
​        );
​        for (int i = NUM_BYTE_VALUES, k = right + 1; k > left; ) {
​            while (count[--i] == 0);
​            byte value = (byte) (i + Byte.MIN_VALUE);
​            int s = count[i];

​            do {
​                a[--k] = value;
​            } while (--s > 0);
​        }
​    } else { // Use insertion sort on small arrays
​        for (int i = left, j = i; i < right; j = ++i) {
​            byte ai = a[i + 1];

​            while (ai < a[j]) {

​                a[j + 1] = a[j];

​                if (j-- == left) {
​                    break;
​                }
​            }
​            a[j + 1] = ai;
​        }
​    }

}

这个方法里面有个域值，当大于域值的时候会新建一个数组来进行操作，然后先把byte转化为int，再转化过去。当小于域值的时候，就是简单的值交换。

**copyOf——复制数组**：其实理解为集合扩容即可

public static byte[] copyOf(byte[] original, int newLength) {
    byte[] copy = new byte[newLength];
    System.arraycopy(original, 0, copy, 0,
                     Math.min(original.length, newLength));
    return copy;

}

​     可以看出底层其实就是调用System.arraycopy方法，这是个本地方法，作用就是实现两个集合之间的复制。

**equals——比较数组**：通过equals方法比较数组中的元素是否相等

public static boolean equals(byte[] a, byte[] a2) {

​    if (a==a2)

​        return true;

​    if (a==null || a2==null)

​        return false;

​    int length = a.length;

​    if (a2.length != length)

​        return false;

​    for (int i=0; i<length; i++)

​        if (a[i] != a2[i])

​            return false;

​    return true;

}

先用==判断，如果一致就马上返回（==对于复杂类型是比较地址，地址一致肯定相等），否则的话就一个一个比较，代码比较清晰，不再说了。

**binarySearch——查找数组元素**：binarySearch中最主要的方法就是binarySearch0，所有的入口方法最后都会调用这个方法。方法如下：

private static int binarySearch0(byte[] a, int fromIndex, int toIndex,

​                                 byte key) {

​    int low = fromIndex;

​    int high = toIndex - 1;

​    while (low <= high) {

​        int mid = (low + high) >>> 1;

​        byte midVal = a[mid];

​        if (midVal < key)

​            low = mid + 1;

​        else if (midVal > key)

​            high = mid - 1;

​        else

​            return mid; // key found

​    }

​    return -(low + 1);  // key not found.

}

典型的二分查找算法，注意的是求mid时用的是移位操作（感觉是不是java8把* /都改为移位操作了啊，今后不再赘述了）

## **总结：**

​     这两个都是工具类，熟悉一下有什么操作即可。另外看源码的过程应该是学习这些java大牛们的编码规范和培养怎么写出漂亮代码的能力。

参考文献：

http://java.sun.com/javase/6/docs/api/java/util/Collections.html

http://gceclub.sun.com.cn/Java_Docs/html/zh_CN/api/java/util/Collections.html

java.util.Collections类包的学习

[The Java Collections Class](http://marxsoftware.blogspot.com/2009/03/java-collections-class.html)

[http://blog.sina.com.cn/s/blog_93daad41010115yq.html](http://blog.sina.com.cn/s/blog_93daad41010115yq.html)