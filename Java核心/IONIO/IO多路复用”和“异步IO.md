## **曾经的VIP服务**

在网络的初期，网民很少，服务器完全无压力，那时的技术也没有现在先进，通常用一个线程来全程跟踪处理一个请求。因为这样最简单。

其实代码实现大家都知道，就是服务器上有个ServerSocket在某个端口监听，接收到客户端的连接后，会创建一个Socket，并把它交给一个线程进行后续处理。

线程主要从Socket读取客户端传过来的数据，然后进行业务处理，并把结果再写入Socket传回客户端。

由于网络的原因，Socket创建后并不一定能立刻从它上面读取数据，可能需要等一段时间，此时线程也必须一直阻塞着。在向Socket写入数据时，也可能会使线程阻塞。

这里准备了一个示例，主要逻辑如下：

客户端：创建20个Socket并连接到服务器上，再创建20个线程，每个线程负责一个Socket。

服务器端：接收到这20个连接，创建20个Socket，接着创建20个线程，每个线程负责一个Socket。

为了模拟服务器端的Socket在创建后不能立马读取数据，让客户端的20个线程分别休眠5-10之间的一个随机秒数。

客户端的20个线程会在第5秒到第10秒这段时间内陆陆续续的向服务器端发送数据，服务器端的20个线程也会陆陆续续接收到数据。

```java
/**
 * @author lixinjie
 * @since 2019-05-07
 */
public class BioServer {

  static AtomicInteger counter = new AtomicInteger(0);
  static SimpleDateFormat sdf = new SimpleDateFormat("HH:mm:ss"); 
  
  public static void main(String[] args) {
    try {
      ServerSocket ss = new ServerSocket();
      ss.bind(new InetSocketAddress("localhost", 8080));
      while (true) {
        Socket s = ss.accept();
        processWithNewThread(s);
      }
    } catch (Exception e) {
      e.printStackTrace();
    }
  }
  
  static void processWithNewThread(Socket s) {
    Runnable run = () -> {
      InetSocketAddress rsa = (InetSocketAddress)s.getRemoteSocketAddress();
      System.out.println(time() + "->" + rsa.getHostName() + ":" + rsa.getPort() + "->" + Thread.currentThread().getId() + ":" + counter.incrementAndGet());
      try {
        String result = readBytes(s.getInputStream());
        System.out.println(time() + "->" + result + "->" + Thread.currentThread().getId() + ":" + counter.getAndDecrement());
        s.close();
      } catch (Exception e) {
        e.printStackTrace();
      }
    };
    new Thread(run).start();
  }
  
  static String readBytes(InputStream is) throws Exception {
    long start = 0;
    int total = 0;
    int count = 0;
    byte[] bytes = new byte[1024];
    //开始读数据的时间
    long begin = System.currentTimeMillis();
    while ((count = is.read(bytes)) > -1) {
      if (start < 1) {
        //第一次读到数据的时间
        start = System.currentTimeMillis();
      }
      total += count;
    }
    //读完数据的时间
    long end = System.currentTimeMillis();
    return "wait=" + (start - begin) + "ms,read=" + (end - start) + "ms,total=" + total + "bs";
  }

  static String time() {
    return sdf.format(new Date());
  }
}
```



```java
/**
 * @author lixinjie
 * @since 2019-05-07
 */
public class Client {

  public static void main(String[] args) {
    try {
      for (int i = 0; i < 20; i++) {
        Socket s = new Socket();
        s.connect(new InetSocketAddress("localhost", 8080));
        processWithNewThread(s, i);
      }
    } catch (IOException e) {
      e.printStackTrace();
    }
  }

  static void processWithNewThread(Socket s, int i) {
    Runnable run = () -> {
      try {
        //睡眠随机的5-10秒，模拟数据尚未就绪
        Thread.sleep((new Random().nextInt(6) + 5) * 1000);
        //写1M数据，为了拉长服务器端读数据的过程
        s.getOutputStream().write(prepareBytes());
        //睡眠1秒，让服务器端把数据读完
        Thread.sleep(1000);
        s.close();
      } catch (Exception e) {
        e.printStackTrace();
      }
    };
    new Thread(run).start();
  }
  
  static byte[] prepareBytes() {
    byte[] bytes = new byte[1024*1024*1];
    for (int i = 0; i < bytes.length; i++) {
      bytes[i] = 1;
    }
    return bytes;
  }
}
```

执行结果如下：

```java
时间->IP:Port->线程Id:当前线程数
15:11:52->127.0.0.1:55201->10:1
15:11:52->127.0.0.1:55203->12:2
15:11:52->127.0.0.1:55204->13:3
15:11:52->127.0.0.1:55207->16:4
15:11:52->127.0.0.1:55208->17:5
15:11:52->127.0.0.1:55202->11:6
15:11:52->127.0.0.1:55205->14:7
15:11:52->127.0.0.1:55206->15:8
15:11:52->127.0.0.1:55209->18:9
15:11:52->127.0.0.1:55210->19:10
15:11:52->127.0.0.1:55213->22:11
15:11:52->127.0.0.1:55214->23:12
15:11:52->127.0.0.1:55217->26:13
15:11:52->127.0.0.1:55211->20:14
15:11:52->127.0.0.1:55218->27:15
15:11:52->127.0.0.1:55212->21:16
15:11:52->127.0.0.1:55215->24:17
15:11:52->127.0.0.1:55216->25:18
15:11:52->127.0.0.1:55219->28:19
15:11:52->127.0.0.1:55220->29:20

时间->等待数据的时间,读取数据的时间,总共读取的字节数->线程Id:当前线程数
15:11:58->wait=5012ms,read=1022ms,total=1048576bs->17:20
15:11:58->wait=5021ms,read=1022ms,total=1048576bs->13:19
15:11:58->wait=5034ms,read=1008ms,total=1048576bs->11:18
15:11:58->wait=5046ms,read=1003ms,total=1048576bs->12:17
15:11:58->wait=5038ms,read=1005ms,total=1048576bs->23:16
15:11:58->wait=5037ms,read=1010ms,total=1048576bs->22:15
15:11:59->wait=6001ms,read=1017ms,total=1048576bs->15:14
15:11:59->wait=6016ms,read=1013ms,total=1048576bs->27:13
15:11:59->wait=6011ms,read=1018ms,total=1048576bs->24:12
15:12:00->wait=7005ms,read=1008ms,total=1048576bs->20:11
15:12:00->wait=6999ms,read=1020ms,total=1048576bs->14:10
15:12:00->wait=7019ms,read=1007ms,total=1048576bs->26:9
15:12:00->wait=7012ms,read=1015ms,total=1048576bs->21:8
15:12:00->wait=7023ms,read=1008ms,total=1048576bs->25:7
15:12:01->wait=7999ms,read=1011ms,total=1048576bs->18:6
15:12:02->wait=9026ms,read=1014ms,total=1048576bs->10:5
15:12:02->wait=9005ms,read=1031ms,total=1048576bs->19:4
15:12:03->wait=10007ms,read=1011ms,total=1048576bs->16:3
15:12:03->wait=10006ms,read=1017ms,total=1048576bs->29:2
15:12:03->wait=10010ms,read=1022ms,total=1048576bs->28:1
```

可以看到服务器端确实为每个连接创建一个线程，共创建了20个线程。

客户端进入休眠约5-10秒，模拟连接上数据不就绪，服务器端线程在等待，等待时间约5-10秒。

客户端陆续结束休眠，往连接上写入1M数据，服务器端开始读取数据，整个读取过程约1秒。

可以看到，服务器端的工作线程会把时间花在“等待数据”和“读取数据”这两个过程上。

这有两个不好的地方：

一是有很多客户端同时发起请求的话，服务器端要创建很多的线程，可能会因为超过了上限而造成崩溃。

二是每个线程的大部分时光中都是在阻塞着，无事可干，造成极大的资源浪费。

开头已经说了那个年代网民很少，所以，不可能会有大量请求同时过来。至于资源浪费就浪费吧，反正闲着也是闲着。

来个简单的小例子：

饭店共有10张桌子，且配备了10位服务员。只要有客人来了，大堂经理就把客人带到一张桌子，并安排一位服务员全程陪同。

即使客人暂时不需要服务，服务员也一直在旁边站着。可能觉着是一种浪费，其实非也，这就是尊贵的VIP服务。

其实，VIP映射的是一对一的模型，主要体现在“专用”上或“私有”上。

## **真正的多路复用技术**

多路复用技术原本指的是，在通信方面，多种信号或数据（从宏观上看）交织在一起，使用同一条传输通道进行传输。

这样做的目的，一方面可以充分利用通道的传输能力，另一方面自然是省时省力省钱啦。

其实这个概念非常的“生活化”，随手就可以举个例子：

一条小水渠里水在流，在一端往里倒入大量乒乓球，在另一端用网进行过滤，把乒乓球和水流分开。

这就是一个比较“土”的多路复用，首先在发射端把多种信号或数据进行“混合”，接着是在通道上进行传输，最后在接收端“分离”出自己需要的信号或数据。

相信大家都看出来了，这里的重点其实就是处理好“混合”和“分离”，对于不同的信号或数据，有不同的处理方法。

比如以前的有线电视是模拟信号，即电磁波。一家一般只有一根信号线，但可以同时接多个电视，每个电视任意换台，互不影响。

这是由于不同频率的波可以混合和分离。（当然，可能不是十分准确，明白意思就行了。）

再比如城市的高铁站一般都有数个站台供高铁（同时）停靠，但城市间的高铁轨道单方向只有一条，如何保证那么多趟高铁安全运行呢？

很明显是分时使用，每趟高铁都有自己的时刻。多趟高铁按不同的时刻出站相当于混合，按不同的时刻进站相当于分离。

总结一下，多路指的是多种不同的信号或数据或其它事物，复用指的是共用同一个物理链路或通道或载体。

可见，多路复用技术是一种一对多的模型，“多”的这一方复用了“一”的这一方。

其实，一对多的模型主要体现在“公用”上或“共享”上。

## **您先看着，我一会再过来**

一对一服务是典型的有钱任性，虽然响应及时、服务周到，但不是每个人都能享受的，毕竟还是“屌丝”多嘛，那就来个共享服务吧。

所以实际当中更多的情况是，客人坐下后，会给他一个菜单，让他先看着，反正也不可能立马点餐，服务员就去忙别的了。

可能不时的会有服务员从客人身旁经过，发现客人还没有点餐，就会主动去询问现在需要点餐吗？

如果需要，服务员就给你写菜单，如果不需要，服务员就继续往前走了。

这种情况饭店整体运行的也很好，但是服务员人数少多了。现在服务10桌客人，4个服务员绰绰有余。（这节省的可都是纯利润呀。)

因为10桌客人同时需要服务的情况几乎是不会发生的，绝大部分情况都是错开的。如果真有的话，那就等会好了，又不是120/119，人命关天的。

回到代码里，情况与之非常相似，完全可以采用相同的理论去处理。

连接建立后，找个地方把它放到那里，可以暂时先不管它，反正此时也没有数据可读。

但是数据早晚会到来的，所以，要不时的去询问每个连接有数据没有，有的话就读取数据，没有的话就继续不管它。

其实这个模式在Java里早就有了，就是Java NIO，这里的大写字母“N”是单词“New”，即“新”的意思，主要是为了和上面的“一对一”进行区分。

## **先铺垫一下吧**

现在需要把Socket交互的过程再稍微细化一些。客户端先请求连接，connect，服务器端然后接受连接，accept，然后客户端再向连接写入数据，write，接着服务器端从连接上读出数据，read。

和打电话的场景一样，主叫拨号，connect，被叫接听，accept，主叫说话，speak，被叫聆听，listen。主叫给被叫打电话，说明主叫找被叫有事，所以被叫关注的是接通电话，听对方说。

客户端主动向服务器端发起请求，说明客户端找服务器端有事，所以服务器端关注的是接受请求，读取对方传来的数据。这里把接受请求，读取数据称为服务器端感兴趣的操作。

在Java NIO中，接受请求的操作，用OP_ACCEPT表示，读取数据的操作，用OP_READ表示。

我决定先过一遍饭店的场景，让首次接触Java NIO的同学不那么迷茫。就是把常规的场景进行了定向整理，稍微有点刻意，明白意思就行了。

1、专门设立一个“跑腿”服务员，工作职责单一，就是问问客人是否需要服务。

2、站在门口接待客人，本来是大堂经理的工作，但是他不愿意在门口盯着，于是就委托给跑腿服务员，你帮我盯着，有人来了告诉我。

*于是跑腿服务员就有了一个任务，替大堂经理盯梢。终于来客人了，跑腿服务员赶紧告诉了大堂经理。*

3、大堂经理把客人带到座位上，对跑腿服务员说，客人接下来肯定是要点餐的，但是现在在看菜单，不知道什么时候能看好，所以你不时的过来问问，看需不需要点餐，需要的话就再喊来一个“点餐”服务员给客人写菜单。

*于是跑腿服务员就又多了一个任务，就是盯着这桌客人，不时来问问，如果需要服务的话，就叫点餐服务员过来服务。*

4、跑腿服务员在某次询问中，客人终于决定点餐了，跑题服务员赶紧找来一个点餐服务员为客人写菜单。

5、就这样，跑腿服务员既要盯着门外新过来的客人，也要盯着门内已经就坐的客人。新客人来了，通知大堂经理去接待。就坐的客人决定点餐了，通知点餐服务员去写菜单。

事情就这样一直循环的持续下去，一切，都挺好。角色明确，职责单一，配合很好。

大堂经理和点餐服务员是需求的提供者或实现者，跑腿服务员是需求的发现者，并识别出需求的种类，需要接待的交给大堂经理，需要点餐的交给点餐服务员。

## **哈哈，Java NIO来啦**

代码的写法非常的固定，可以配合着后面的解说来看，这样就好理解了，如下：

```java
/**
 * @author lixinjie
 * @since 2019-05-07
 */
public class NioServer {

  static int clientCount = 0;
  static AtomicInteger counter = new AtomicInteger(0);
  static SimpleDateFormat sdf = new SimpleDateFormat("HH:mm:ss"); 
  
  public static void main(String[] args) {
    try {
      Selector selector = Selector.open();
      ServerSocketChannel ssc = ServerSocketChannel.open();
      ssc.configureBlocking(false);
      ssc.register(selector, SelectionKey.OP_ACCEPT);
      ssc.bind(new InetSocketAddress("localhost", 8080));
      while (true) {
        selector.select();
        Set<SelectionKey> keys = selector.selectedKeys();
        Iterator<SelectionKey> iterator = keys.iterator();
        while (iterator.hasNext()) {
          SelectionKey key = iterator.next();
          iterator.remove();
          if (key.isAcceptable()) {
            ServerSocketChannel ssc1 = (ServerSocketChannel)key.channel();
            SocketChannel sc = null;
            while ((sc = ssc1.accept()) != null) {
              sc.configureBlocking(false);
              sc.register(selector, SelectionKey.OP_READ);
              InetSocketAddress rsa = (InetSocketAddress)sc.socket().getRemoteSocketAddress();
              System.out.println(time() + "->" + rsa.getHostName() + ":" + rsa.getPort() + "->" + Thread.currentThread().getId() + ":" + (++clientCount));
            }
          } else if (key.isReadable()) {
            //先将“读”从感兴趣操作移出，待把数据从通道中读完后，再把“读”添加到感兴趣操作中
            //否则，该通道会一直被选出来
            key.interestOps(key.interestOps() & (~ SelectionKey.OP_READ));
            processWithNewThread((SocketChannel)key.channel(), key);
          }
        }
      }
    } catch (Exception e) {
      e.printStackTrace();
    }
  }

  static void processWithNewThread(SocketChannel sc, SelectionKey key) {
    Runnable run = () -> {
      counter.incrementAndGet();
      try {
        String result = readBytes(sc);
        //把“读”加进去
        key.interestOps(key.interestOps() | SelectionKey.OP_READ);
        System.out.println(time() + "->" + result + "->" + Thread.currentThread().getId() + ":" + counter.get());
        sc.close();
      } catch (Exception e) {
        e.printStackTrace();
      }
      counter.decrementAndGet();
    };
    new Thread(run).start();
  }
  
  static String readBytes(SocketChannel sc) throws Exception {
    long start = 0;
    int total = 0;
    int count = 0;
    ByteBuffer bb = ByteBuffer.allocate(1024);
    //开始读数据的时间
    long begin = System.currentTimeMillis();
    while ((count = sc.read(bb)) > -1) {
      if (start < 1) {
        //第一次读到数据的时间
        start = System.currentTimeMillis();
      }
      total += count;
      bb.clear();
    }
    //读完数据的时间
    long end = System.currentTimeMillis();
    return "wait=" + (start - begin) + "ms,read=" + (end - start) + "ms,total=" + total + "bs";
  }
  
  static String time() {
    return sdf.format(new Date());
  }
}
```

它的大致处理过程如下：

1、定义一个选择器，Selector。

*相当于设立一个跑腿服务员。*

2、定义一个服务器端套接字通道，ServerSocketChannel，并配置为非阻塞的。

*相等于聘请了一位大堂经理*。

3、将套接字通道注册到选择器上，并把感兴趣的操作设置为OP_ACCEPT。

*相当于大堂经理给跑腿服务员说，帮我盯着门外，有客人来了告诉我。*

4、进入死循环，选择器不时的进行选择。

*相当于跑腿服务员一遍又一遍的去询问、去转悠*。

5、选择器终于选择出了通道，发现通道是需要Acceptable的。

*相当于跑腿服务员终于发现门外来客人了，客人是需要接待的*。

6、于是服务器端套接字接受了这个通道，开始处理。

*相当于跑腿服务员把大堂经理叫来了，大堂经理开始着手接待*。

7、把新接受的通道配置为非阻塞的，并把它也注册到了选择器上，该通道感兴趣的操作为OP_READ。

*相当于大堂经理把客人带到座位上，给了客人菜单，并又把客人委托给跑腿服务员，说客人接下来肯定是要点餐的，你不时的来问问*。

8、选择器继续不时的进行选择着。

*相当于跑腿服务员继续不时的询问着、转悠着*。

9、选择器终于又选择出了通道，这次发现通道是需要Readable的。

*相当于跑腿服务员终于发现了一桌客人有了需求，是需要点餐的*。

10、把这个通道交给了一个新的工作线程去处理。

*相当于跑腿服务员叫来了点餐服务员，点餐服务员开始为客人写菜单*。

11、这个工作线程处理完后，就被回收了，可以再去处理其它通道。

*相当于点餐服务员写好菜单后，就走了，可以再去为其他客人写菜单*。

12、选择器继续着重复的选择工作，不知道什么时候是个头。

*相当于跑腿服务员继续着重复的询问、转悠，不知道未来在何方*。

相信你已经看出来了，大堂经理相当于服务器端套接字，跑腿服务员相当于选择器，点餐服务员相当于Worker线程。

启动服务器端代码，使用同一个客户端代码，按相同的套路发20个请求，结果如下：

```java
时间->IP:Port->主线程Id:当前连接数
16:34:39->127.0.0.1:56105->1:1
16:34:39->127.0.0.1:56106->1:2
16:34:39->127.0.0.1:56107->1:3
16:34:39->127.0.0.1:56108->1:4
16:34:39->127.0.0.1:56109->1:5
16:34:39->127.0.0.1:56110->1:6
16:34:39->127.0.0.1:56111->1:7
16:34:39->127.0.0.1:56112->1:8
16:34:39->127.0.0.1:56113->1:9
16:34:39->127.0.0.1:56114->1:10
16:34:39->127.0.0.1:56115->1:11
16:34:39->127.0.0.1:56116->1:12
16:34:39->127.0.0.1:56117->1:13
16:34:39->127.0.0.1:56118->1:14
16:34:39->127.0.0.1:56119->1:15
16:34:39->127.0.0.1:56120->1:16
16:34:39->127.0.0.1:56121->1:17
16:34:39->127.0.0.1:56122->1:18
16:34:39->127.0.0.1:56123->1:19
16:34:39->127.0.0.1:56124->1:20

时间->等待数据的时间,读取数据的时间,总共读取的字节数->线程Id:当前线程数
16:34:45->wait=1ms,read=1018ms,total=1048576bs->11:5
16:34:45->wait=0ms,read=1054ms,total=1048576bs->10:5
16:34:45->wait=0ms,read=1072ms,total=1048576bs->13:6
16:34:45->wait=0ms,read=1061ms,total=1048576bs->14:5
16:34:45->wait=0ms,read=1140ms,total=1048576bs->12:4
16:34:46->wait=0ms,read=1001ms,total=1048576bs->15:5
16:34:46->wait=0ms,read=1062ms,total=1048576bs->17:6
16:34:46->wait=0ms,read=1059ms,total=1048576bs->16:5
16:34:47->wait=0ms,read=1001ms,total=1048576bs->19:4
16:34:47->wait=0ms,read=1001ms,total=1048576bs->20:4
16:34:47->wait=0ms,read=1015ms,total=1048576bs->18:3
16:34:47->wait=0ms,read=1001ms,total=1048576bs->21:2
16:34:48->wait=0ms,read=1032ms,total=1048576bs->22:4
16:34:49->wait=0ms,read=1002ms,total=1048576bs->23:3
16:34:49->wait=0ms,read=1001ms,total=1048576bs->25:2
16:34:49->wait=0ms,read=1028ms,total=1048576bs->24:4
16:34:50->wait=0ms,read=1008ms,total=1048576bs->28:4
16:34:50->wait=0ms,read=1033ms,total=1048576bs->27:3
16:34:50->wait=1ms,read=1002ms,total=1048576bs->29:2
16:34:50->wait=0ms,read=1001ms,total=1048576bs->26:2
```

服务器端接受20个连接，创建20个通道，并把它们注册到选择器上，此时不需要额外线程。

当某个通道已经有数据时，才会用一个线程来处理它，所以，线程“等待数据”的时间是0，“读取数据”的时间还是约1秒。

因为20个通道是陆陆续续有数据的，所以服务器端最多时是6个线程在同时运行的，换句话说，用包含6个线程的线程池就可以了。

对比与结论：

处理同样的20个请求，一个需要用20个线程，一个需要用6个线程，节省了70%线程数。

在本例中，两种感兴趣的操作共用一个选择器，且选择器运行在主线程里，Worker线程是新的线程。

其实对于选择器的个数、选择器运行在哪个线程里、是否使用新的线程来处理请求都没有要求，要根据实际情况来定。

比如说redis，和处理请求相关的就一个线程，选择器运行在里面，处理请求的程序也运行在里面，所以这个线程既是I/O线程，也是Worker线程。

当然，也可以使用两个选择器，一个处理OP_ACCEPT，一个处理OP_READ，让它们分别运行在两个单独的I/O线程里。对于能快速完成的操作可以直接在I/O线程里做了，对于非常耗时的操作一定要使用Worker线程池来处理。

这种处理模式就是被称为的多路复用I/O，多路指的是多个Socket通道，复用指的是只用一个线程来管理它们。

## **再稍微分析一下**

一对一的形式，一个桌子配一个服务员，一个Socket分配一个线程，响应速度最快，毕竟是VIP嘛，但是效率很低，服务员大部分时间都是在站着，线程大部分时间都是在等待。

多路复用的形式，所有桌子共用一个跑腿服务员，所有Socket共用一个选择器线程，响应速度肯定变慢了，毕竟是一对多嘛。但是效率提高了，点餐服务员在需要点餐时才会过去，工作线程在数据就绪时才会开始工作。

从VIP到多路复用，形式上确实有很大的不同，其本质是从一对一到一对多的转变，其实就是牺牲了响应速度，换来了效率的提升，不过综合性能还是得到了极大的改进。

就饭店而言，究竟几张桌子配一个跑腿服务员，几张桌子配一个点餐服务员，经过一段时间运行，一定会有一个最优解。

就程序而言，究竟需要几个选择器线程，几个工作线程，经过评估测试后，也会有一个最优解。

一旦达到最优解后，就不可能再提升了，这同样是由多路复用这种一对多的形式所限制的。就像一对一的形式限制一样。

人们的追求是无止境的，如何对多路复用继续提升呢？答案一定是具有颠覆性的，即抛弃多路复用，采用全新的形式。

还以饭店为例，如何在最优解的情况下，既要继续减少服务员数量，还要使效率提升呢？可能有些朋友已经猜到了，那就是抛弃服务员服务客人这种模式，把饭店改成自助餐厅。

在客人进门时，把餐具给他，并告诉他就餐时长、不准浪费等这些规则，然后就不用管了。客人自己选餐，自己吃完，自己走人，不用再等服务员了，因此也不再需要服务员了。（收拾桌子的除外。）

这种模式对应到程序里，其实就是AIO，在Java里也早就有了。

## **嘻嘻，Java AIO来啦**

代码的写法非常的固定，可以配合着后面的解说来看，这样就好理解了，如下：

```java
/**
 * @author lixinjie
 * @since 2019-05-13
 */
public class AioServer {

  static int clientCount = 0;
  static AtomicInteger counter = new AtomicInteger(0);
  static SimpleDateFormat sdf = new SimpleDateFormat("HH:mm:ss"); 
  
  public static void main(String[] args) {
    try {
      AsynchronousServerSocketChannel assc = AsynchronousServerSocketChannel.open();
      assc.bind(new InetSocketAddress("localhost", 8080));
      //非阻塞方法，其实就是注册了个回调，而且只能接受一个连接
      assc.accept(null, new CompletionHandler<AsynchronousSocketChannel, Object>() {

        @Override
        public void completed(AsynchronousSocketChannel asc, Object attachment) {
          //再次注册，接受下一个连接
          assc.accept(null, this);
          try {
            InetSocketAddress rsa = (InetSocketAddress)asc.getRemoteAddress();
            System.out.println(time() + "->" + rsa.getHostName() + ":" + rsa.getPort() + "->" + Thread.currentThread().getId() + ":" + (++clientCount));
          } catch (Exception e) {
          }
          readFromChannelAsync(asc);
        }

        @Override
        public void failed(Throwable exc, Object attachment) {
          
        }
      });
      //不让主线程退出
      synchronized (AioServer.class) {
        AioServer.class.wait();
      }
    } catch (Exception e) {
      e.printStackTrace();
    }
  }

  static void readFromChannelAsync(AsynchronousSocketChannel asc) {
    //会把数据读入到该buffer之后，再触发工作线程来执行回调
    ByteBuffer bb = ByteBuffer.allocate(1024*1024*1 + 1);
    long begin = System.currentTimeMillis();
    //非阻塞方法，其实就是注册了个回调，而且只能接受一次读取
    asc.read(bb, null, new CompletionHandler<Integer, Object>() {
      //从该连接上一共读到的字节数
      int total = 0;
      /**
       * @param count 表示本次读取到的字节数，-1表示数据已读完
       */
      @Override
      public void completed(Integer count, Object attachment) {
        counter.incrementAndGet();
        if (count > -1) {
          total += count;
        }
        int size = bb.position();
        System.out.println(time() + "->count=" + count + ",total=" + total + "bs,buffer=" + size + "bs->" + Thread.currentThread().getId() + ":" + counter.get());
        if (count > -1) {//数据还没有读完
          //再次注册回调，接受下一次读取
          asc.read(bb, null, this);
        } else {//数据已读完
          try {
            asc.close();
          } catch (Exception e) {
            e.printStackTrace();
          }
        }
        counter.decrementAndGet();
      }

      @Override
      public void failed(Throwable exc, Object attachment) {
        
      }
    });
    long end = System.currentTimeMillis();
    System.out.println(time() + "->exe read req,use=" + (end -begin) + "ms" + "->" + Thread.currentThread().getId());
  }
  
  static String time() {
    return sdf.format(new Date());
  }
}
```

它的大致处理过程如下：

1、初始化一个AsynchronousServerSocketChannel对象，并开始监听

2、通过accept方法注册一个“完成处理器”的接受连接回调，即CompletionHandler，用于在接受到连接后的相关操作。

3、当客户端连接过来后，由系统来接受，并创建好AsynchronousSocketChannel对象，然后触发该回调，并把该对象传进该回调，该回调会在Worker线程中执行。

4、在接受连接回调里，再次使用accept方法注册一次相同的完成处理器对象，用于让系统接受下一个连接。就是这种注册只能使用一次，所以要不停的连续注册，人家就是这样设计的。

5、在接受连接回调里，使用AsynchronousSocketChannel对象的read方法注册另一个接受数据回调，用于在接受到数据后的相关操作。

6、当客户端数据过来后，由系统接受，并放入指定好的ByteBuffer中，然后触发该回调，并把本次接受到的数据字节数传入该回调，该回调会在Worker线程中执行。

7、在接受数据回调里，如果数据没有接受完，需要再次使用read方法把同一个对象注册一次，用于让系统接受下一次数据。这和上面的套路是一样的。

8、客户端的数据可能是分多次传到服务器端的，所以接受数据回调会被执行多次，直到数据接受完为止。多次接受到的数据合起来才是完整的数据，这个一定要处理好。

9、关于ByteBuffer，要么足够的大，能够装得下完整的客户端数据，这样多次接受的数据直接往里追加即可。要么每次把ByteBuffer中的数据移到别的地方存储起来，然后清空ByteBuffer，用于让系统往里装入下一次接受的数据。

注：如果出现ByteBuffer空间不足，则系统不会装入数据，就会导致客户端数据总是读不完，极有可能进入死循环。

启动服务器端代码，使用同一个客户端代码，按相同的套路发20个请求，结果如下：

```java
时间->IP:Port->回调线程Id:当前连接数
17:20:47->127.0.0.1:56454->15:1
时间->发起一个读请求，耗时->回调线程Id
17:20:47->exe read req,use=3ms->15
17:20:47->127.0.0.1:56455->15:2
17:20:47->exe read req,use=1ms->15
17:20:47->127.0.0.1:56456->15:3
17:20:47->exe read req,use=0ms->15
17:20:47->127.0.0.1:56457->16:4
17:20:47->127.0.0.1:56458->15:5
17:20:47->exe read req,use=1ms->16
17:20:47->exe read req,use=1ms->15
17:20:47->127.0.0.1:56460->15:6
17:20:47->127.0.0.1:56459->17:7
17:20:47->exe read req,use=0ms->15
17:20:47->127.0.0.1:56462->15:8
17:20:47->127.0.0.1:56461->16:9
17:20:47->exe read req,use=1ms->15
17:20:47->exe read req,use=0ms->16
17:20:47->exe read req,use=0ms->17
17:20:47->127.0.0.1:56465->16:10
17:20:47->127.0.0.1:56463->18:11
17:20:47->exe read req,use=0ms->18
17:20:47->127.0.0.1:56466->15:12
17:20:47->exe read req,use=1ms->16
17:20:47->127.0.0.1:56464->17:13
17:20:47->exe read req,use=1ms->15
17:20:47->127.0.0.1:56467->18:14
17:20:47->exe read req,use=2ms->17
17:20:47->exe read req,use=1ms->18
17:20:47->127.0.0.1:56468->15:15
17:20:47->exe read req,use=1ms->15
17:20:47->127.0.0.1:56469->16:16
17:20:47->127.0.0.1:56470->18:17
17:20:47->exe read req,use=1ms->18
17:20:47->exe read req,use=1ms->16
17:20:47->127.0.0.1:56472->15:18
17:20:47->127.0.0.1:56473->19:19
17:20:47->exe read req,use=2ms->15
17:20:47->127.0.0.1:56471->17:20
17:20:47->exe read req,use=1ms->19
17:20:47->exe read req,use=1ms->17

时间->本次接受到的字节数,截至到目前接受到的字节总数,buffer中的字节总数->回调线程Id:当前线程数
17:20:52->count=65536,total=65536bs,buffer=65536bs->14:1
17:20:52->count=65536,total=65536bs,buffer=65536bs->14:1
17:20:52->count=65536,total=65536bs,buffer=65536bs->14:1
17:20:52->count=230188,total=295724bs,buffer=295724bs->12:1
17:20:52->count=752852,total=1048576bs,buffer=1048576bs->14:3
17:20:52->count=131072,total=196608bs,buffer=196608bs->17:2

。。。。。。。。。。。。。。。。。。。。。。

17:20:57->count=-1,total=1048576bs,buffer=1048576bs->15:1
17:20:57->count=-1,total=1048576bs,buffer=1048576bs->15:1
17:20:57->count=-1,total=1048576bs,buffer=1048576bs->15:1
17:20:57->count=-1,total=1048576bs,buffer=1048576bs->15:1
17:20:58->count=-1,total=1048576bs,buffer=1048576bs->15:1
17:20:58->count=-1,total=1048576bs,buffer=1048576bs->15:1
17:20:58->count=-1,total=1048576bs,buffer=1048576bs->15:1
```

系统接受到连接后，在工作线程中执行了回调。并且在回调中执行了read方法，耗时是0，因为只是注册了个接受数据的回调而已。

系统接受到数据后，把数据放入ByteBuffer，在工作线程中执行了回调。并且回调中可以直接使用ByteBuffer中的数据。

接受数据的回调被执行了多次，多次接受到的数据加起来正好等于客户端传来的数据。

因为系统是接受到数据后才触发的回调，所以服务器端最多时是3个线程在同时运行回调的，换句话说，线程池包含3个线程就可以了。

对比与结论：

处理同样的20个请求，一个需要用20个线程，一个需要用6个线程，一个需要3个线程，又节省了50%线程数。

注：不用特别较真这个比较结果，这里只是为了说明问题而已。哈哈。

## **三种处理方式的对比**

第一种是阻塞IO，阻塞点有两个，等待数据就绪的过程和读取数据的过程。

第二种是阻塞IO，阻塞点有一个，读取数据的过程。

第三种是非阻塞IO，没有阻塞点，当工作线程启动时，数据已经（被系统）准备好可以直接用了。

可见，这是一个逐步消除阻塞点的过程。

再次来谈谈各种IO：

只有一个线程，接受一个连接，读取数据，处理业务，写回结果，再接受下一个连接，这是同步阻塞。这种用法几乎没有。

一个线程和一个线程池，线程接受到连接后，把它丢给线程池中的线程，再接受下一个连接，这是异步阻塞。对应示例一。

一个线程和一个线程池，线程运行selector，执行select操作，把就绪的连接拿出来丢给线程池中的线程，再执行下一次的select操作，就是多路复用，这是异步阻塞。对应示例二。

一个线程和一个线程池，线程注册一个accept回调，系统帮我们接受好连接后，才触发回调在线程池中执行，执行时再注册read回调，系统帮我们接受好数据后，才触发回调在线程池中执行，就是AIO，这是异步非阻塞。对应示例三。

redis也是多路复用，但它只有一个线程在执行select操作，处理就绪的连接，整个是串行化的，所以天然不存在并发问题。只能把它归为同步阻塞了。

BIO是阻塞IO，可以是同步阻塞，也可以是异步阻塞。AIO是异步IO，只有异步非阻塞这一种。因此没有同步非阻塞这种说法，因为同步一定是阻塞的。

注：以上的说法是站在用户程序/线程的立场上来说的。

**建议**把代码下载下来，自己运行一下，体会体会：

[https://github.com/coding-new-talking/java-code-demo.git](https://link.zhihu.com/?target=https%3A//github.com/coding-new-talking/java-code-demo.git)