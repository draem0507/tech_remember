## 一、Java IO系统

Java IO系统可以分为面向字节操作和字符操作两种，面向字节的是InputStream、OutputStream及其子类，面向字符的是Reader、Writer及其子类。

 （1）InputStream输入类：

- ByteArrayInputStream：字节流数据来源为字节数组，构造器参数为byte[ ]；
- StringBufferInputStream：字节流数据来源为字符串，将String转换成InputStream，构造器参数为String；
- FileInputStream：字节流数据来源为文件，构造器参数为String文件名或File对象；
- FilterInputStream：抽象类，作为装饰器的接口；

 

（2）OutputStream输出类：

- ByteArrayOutputStream；（相关功能与上面对应，下同）
- StringBufferOutputStream（无）；
- FileOutputStream；字节流输出到文件，构造时第二个参数设置为true追加内容到文件末尾，false覆盖原内容（FileWriter也类似);      
- FilterOutputStream；

（3）Filter装饰器相关子类：

DataInputStream、DataOutputStream：可以按照可移植的方式从流中读取基本类型数据，如readByte()，readInt()，readUTF()。

BufferedInputStream、BufferedOutputStream：以缓冲的方式读取或写出。

PrintStream：用于产生格式化输出，内有两个重要方法print()，println()，标准输出System.out，错误System.err均包装成此类型（标准输入[System.in](http://system.in/)为InputStream类），PrintStream不会抛出异常，它未完全国际化，有一些问题，已经在printWriter中得到解决。

InputStreamReader，OutputStreamWriter可以将InputStream，OutputStream转换成Reader，Writer。使用中应该尽量使用Reader和Writer 。

自我独立的类 RandomAccessFile，实现了DataInput，DataOutput接口，适用于大小已知的记录组成的文件，可以调用seek()将记录从一处转移到另一处，然后读取或修改记录。 

（4）NIO

新IO库提高了速度，因为使用的结构更接近于操作系统执行IO的方式：通道和缓冲器。

调用FileInputStream，FileOutputStream，RandomAccessFile的getChannel（）方法可以得到相应的FileChannel。 

Channel与ByteBuffer的读写操作：

 

代码块

Plain Text









```
   FileChannel in，out；
     ByteBuffer buf；
     while（in.read(buf)!=-1）{
          buf.flip()     //准备写
          out.write(buf);
          buf.clear();   //准备读
     } 
```



 



（5）内存映射文件：允许我们创建和修改那些因为太大而不能放入内存的文件，可以把它当做非常的大的数组来访问。

普通流读写与内存映射文件读写速度比较：

 

代码块

Plain Text









```
public class MapedIOTest {
    static int num=4000000;
    public static void streamReadWrite() throws IOException {
        RandomAccessFile raf=new RandomAccessFile(new File("tmp.txt"),"rw");
        raf.writeInt(1);
        for(int i=0;i<num;i++){
            raf.seek(raf.length()-4);
            raf.writeInt(raf.readInt());
        }
        raf.close();
    }

    public static void mapedReadWrite() throws IOException {
        FileChannel fileChannel=new RandomAccessFile(new File("tmp.txt"),"rw").getChannel();
        IntBuffer intBuffer=fileChannel.map(FileChannel.MapMode.READ_WRITE,0,fileChannel.size()).asIntBuffer();

        intBuffer.put(0);
        for(int i=1;i<num;i++){
            intBuffer.put(intBuffer.get(i-1));
        }
        fileChannel.close();

    }

    public static void main(String [] args) throws IOException {
        long start=System.nanoTime();
        streamReadWrite();
        long end=System.nanoTime();
        System.out.println("streamReadWrite "+(end-start));
        start=System.nanoTime();
        mapedReadWrite();
        end =System.nanoTime();
        System.out.println("mapedReadWrite "+(end-start));
    }
}
```



输出：

streamReadWrite 44344009983
mapedReadWrite 32076994

 

 

 （6）序列化

Java序列化API可以实现对象的持久化，保存对象的状态，允许我们将一个对象转换为流，并通过网络发送，或将其存入文件或数据库以便未来使用，反序列化则是将对象流转换为实际程序中使用的Java对象的过程。

实现一个类对象是可序列化的，所要做的是实现Serializable接口，序列化处理是通过ObjectInputStream和ObjectOutputStream实现的。

某个字段被声明为transient后，默认序列化机制就会忽略该字段。可以添加两个private方法writeObject()与readObject()对特定成员写入，序列化时ObjectOutputStream writeObject()时会检查Serializable对象是否实现了它自己的writeObject()，实现了则会自动调用，可以在自己的writeObject()方法中调用defaultWriteObject()执行默认的操作，反序列化也类似。

反序列化重构对象时必须确保该读取程序的CLASSPATH中包含有T.class。只要将任何对象序列化到单一流中，就可以恢复出与我们写出时一样的对象网，并且没有重复的对象。

static字段不是对象状态，而是属于类状态，序列化不会保存，对于要序列化static值，要添加serializeStaticState（ObjectOutputStream ）和deserializeStaticState（ObjectInputStream os）方法。

Externalizable接口
有时候我们想要去隐藏对象数据，来保持它的完整性，可以通过实现[java.io](http://java.io/).Externalizable接口，并提供writeExternal()和readExternal()方法的实现，它们被用于序列化处理。对于恢复一个Serializable对象，完全以存储的二进制位为基础构造，不调用构造器。而对于Externalizable对象恢复，所有普通的默认构造器会被调用（public 默认构造器），然后再调用readExternal( )。

 

代码块

Plain Text









```
class Data1 implements Serializable{
    private int i=1;
    private String s="data1";
    private  transient int j=9;

    @Override
    public String toString() {
        return "Data1{" +
                "i=" + i +
                ", s='" + s + '\'' +
                ", j=" + j +
                '}';
    }

    public Data1() {
    }

    public Data1(int i, String s) {
        this.i = i;
        this.s = s;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;

        Data1 data1 = (Data1) o;

        if (i != data1.i) return false;
        return s != null ? s.equals(data1.s) : data1.s == null;

    }

    @Override
    public int hashCode() {
        int result = i;
        result = 31 * result + (s != null ? s.hashCode() : 0);
        return result;
    }
}
class Data2 implements Serializable{
    private int i=2;
    private String s="data2";

    @Override
    public String toString() {
        return "Data2{" +
                "i=" + i +
                ", s='" + s + '\'' +
                '}';
    }
}

class Data3 implements  Serializable{
    private transient  int i=3;
    private String string="data3";
    public static int k=3;

    private void writeObject(ObjectOutputStream stream) throws IOException {
        stream.defaultWriteObject();
        stream.writeInt(i);
    }
    private void readObject(ObjectInputStream stream) throws IOException, ClassNotFoundException {
        stream.defaultReadObject();
        i=stream.readInt();
    }

    @Override
    public String toString() {
        return "Data3{" +
                "i=" + i +
                ", string='" + string + '\'' +
                ", k=" + k+
                '}';
    }
}

class Data4 implements Externalizable{
    private transient  int i=4;
    private String string="data4";

    @Override
    public String toString() {
        return "Data4{" +
                "i=" + i +
                ", string='" + string + '\'' +
                '}';
    }

    //必须要有public默认构造器
    public Data4() {
    }

    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        out.writeInt(i);
        out.writeObject(string);
    }

    @Override
    public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {
        i=in.readInt();
        string= (String) in.readObject();
    }
}

public class SerialTest {
    public static void main(String[] args) throws IOException, ClassNotFoundException {
        Data1 data1=new Data1();
        Data1 data11=new Data1(2,"data11");
        Data2 data2=new Data2();
        Data3 data3=new Data3();
        Data4 data4=new Data4();
        System.out.println(data1+"\n"+data11+"\n"+data2);
        ObjectOutputStream objectOutputStream=new ObjectOutputStream(new FileOutputStream("object.out"));
        objectOutputStream.writeObject(data1);
        objectOutputStream.writeObject(data11);
        objectOutputStream.writeObject(data2);
        objectOutputStream.writeObject(data3);
        objectOutputStream.writeObject(data4);
        objectOutputStream.close();
        System.out.println(Data3.k);
        Data3.k=4;

        //read
        ObjectInputStream inputStream=new ObjectInputStream(new FileInputStream("object.out"));
        Data1 data12= (Data1) inputStream.readObject();
        Data1 data13= (Data1) inputStream.readObject();
        Data2 data21= (Data2) inputStream.readObject();
        System.out.println(data12+"\n"+data13+"\n"+data21);
        System.out.println(data1.equals(data12)); //内容相同
        System.out.println(data11==data13);   //不是同一个对象
        //有方法writeObject()与readObject（）时通过调用默认的writeObject()、
        // readObject（）保存非transient字段，其他的要手动保存
        //static 数据没有保存
        Data3 data31= (Data3) inputStream.readObject();
        System.out.println(data3+"\n"+data31);
        Data4 data41= (Data4) inputStream.readObject();
        System.out.print(data41);
    }
}


```



输出：

Data1{i=1, s='data1', j=9}
Data1{i=2, s='data11', j=9}
Data2{i=2, s='data2'}
3
Data1{i=1, s='data1', j=0}
Data1{i=2, s='data11', j=0}
Data2{i=2, s='data2'}
true
false
Data3{i=3, string='data3', k=4}
Data3{i=3, string='data3', k=4}
Data4{i=4, string='data4'}

 

## 二、Java多线程

 

**线程可被中断**的调用：sleep( )，join( )，wait( )；

**不可中断调用**：synchronized( )，IO等待。

**多线程中的异常处理**：异常是不能跨线程传播的，对于受检异常checked exception必须在线程内处理，不能通过run( )抛出（重写的run()抛出异常编译错误）；对于非受检异常unchecked exception如果没有用try-catch捕获异常会直接传递到console，如果想在线程代码边界之外处理这个异常，可以在Thread start之前设置Thread的实例方法setUncaughtExceptionHandler(Thread.UncaughtExceptionHandler eh)，线程出现异常时会回调UncaughtExceptionHandler对象的uncaughtException（）方法。 

**线程的取消与关闭**

（1）volatile 标志位；

代码块

Plain Text









```
class StopTest1 implements Runnable{

  volatile boolean cancelled;

  public void cancle(){
   cancelled=true;
  }

  @Override
  public void run() {
    while(!cancelled){
    //do something
    }
  } 
}
```



 

（2）线程中断

 

代码块

Plain Text









```
class StopTest2 extends Thread{

  public void cancle(){
    interrupt();
  }

  @Override
  public void run() {
    while(!Thread.currentThread().isInterrupted()){
      //do something
      }
  }
 } 
```



 

（3）线程池ExecutorService的关闭

 使用shutdown关闭，调用了shutdown()方法，则线程池处于SHUTDOWN状态（初始时处于RUNNING状态），此时，不能再往线程池中添加任何任务，否则将会抛RejectedExecutionException异常。但是，此时线程池不会立刻退出，直到添加到线程池中的任务都已经处理完成，才会退出。

 使用shutdownNow关闭，调用了shutdownNow()方法，则线程池处于STOP状态，并试图停止所有正在执行的线程（向线程发送interrupt），不再处理还在池队列中等待的任务，当然，它会返回那些未执行的任务。

​     

（4）通过Future来关闭

ExecutorService.submit( )将返回一个Future<?>来描述任务，可以通过Future的cancel方法来取消任务。cancel（）方法有一个mayInterruptRunning参数，表示取消操作是否成功，若其为true时表示任务正在某个线程中运行，那么这个线程能被中断，若为false则意味着如果任务没有启动就不要运行它，这种方式应该用于那种不处理中断的任务中。

​     

（5）通过关闭底层底层连接关闭同步Socket IO，线程调用Selector.select方法时阻塞，调用close或wakeup会使线程抛出ClosedSelectorException并返回。

​     

（6）关闭生产者-消费者服务可以使用“毒丸”对象。 

一个多线程下载美团apk的例子:

代码块

Plain Text









```
class MultDownloadThread implements Runnable{
    private int threadId;
    private String urlPath;
    private int block;
    private File file;
    private int start;
    private int end;

    public MultDownloadThread(int threadId, String url, int block, File file) {
        this.threadId = threadId;
        this.urlPath = url;
        this.block = block;
        this.file = file;
        this.start=threadId*block;
        this.end=(threadId+1)*block-1;
    }

    @Override
    public void run() {
        BufferedInputStream inputStream=null;
        RandomAccessFile randomAccessFile=null;
        try {
            System.out.println("Thread "+threadId+" start");
            URL url=new URL(urlPath);
            HttpURLConnection connection= (HttpURLConnection) url.openConnection();
            connection.setRequestMethod("GET");
            connection.setReadTimeout(5000);
      //服务端要支持断点续传
            connection.setRequestProperty("Range", "bytes=" + start + "-" + end);
      //返回状态码为206
            if(connection.getResponseCode()==206){
                System.out.println("Thread "+threadId+" get success");
                randomAccessFile=new RandomAccessFile(file,"rw");
                randomAccessFile.seek(start);
                inputStream=new BufferedInputStream(connection.getInputStream());
                byte[] data=new byte[1024];
                int len=0;
                while((len=inputStream.read(data))!=-1){
                    randomAccessFile.write(data,0,len);
                }
                System.out.println("Thread: "+threadId+" complete");
            }
        } catch (MalformedURLException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }finally {
                try {
                    if(inputStream!=null)
                        inputStream.close();
                    if(randomAccessFile!=null)
                        randomAccessFile.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }

        }

    }
}


public class DownLoadTest {
  // static final String urlPath="https://apptest.sankuai.com/download/aimeituan-stage-81.apk";
    static final String urlPath="http://v.meituan.net/mobile/app/Android/group-381_4-meituan_.apk?channel=meituan&subchannel=";
    static final  String fileName="meituan.apk";

    public static void startDownload(int threadNum){
        try {
            URL url=new URL(urlPath);
            HttpURLConnection connection= (HttpURLConnection) url.openConnection();
            connection.setRequestMethod("GET");
            connection.setReadTimeout(5000);
            if(connection.getResponseCode()==200){
                int fileSize=connection.getContentLength();
                System.out.println(fileSize);
                int block=0;
                if(fileSize%threadNum==0){
                    block=fileSize/threadNum;
                }else {
                    block=fileSize/threadNum+1;
                }
                File file=new File(fileName);
                ExecutorService executorService= Executors.newCachedThreadPool();
                for(int i=0;i<threadNum;i++){
                    executorService.execute(new MultDownloadThread(i,urlPath,block,file));
                }
                executorService.shutdown();
            }
        } catch (MalformedURLException e) {
            e.printStackTrace();
        } catch (ProtocolException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public static void main(String [] args){
        startDownload(5);
    }
}
```



## 三、Java内存模型

 

**1、运行时数据区**

- **程序计数器**：线程私有，当前线程所执行字节码的行号指示器。
- **Java虚拟机栈**：线程私有，生命周期与线程相同，描述了Java方法执行的内存模型，每个方法执行时会创建一个栈帧用于存储局部变量表，操作数栈，动态链接，方法出口等信息。局部变量表存放了编译期可知的各种基本类型、对象引用和returnAddress类型。可能抛出StackOverflowError、OutOfMemoryError。
- **本地方法栈**：与虚拟机栈相似，为Native方法服务。
- **Java堆**：线程共享，用于存放对象实例，GC的主要区域。
- **方法区**：线程共享，用于存储虚拟机加载的类信息，运行时常量池，静态变量，即时编译后的代码等数据。可存在GC，主要针对的是常量池的回收和类型的卸载。运行时常量池用于存放Class文件中的常量池（字面量和符号引用），并可以支持动态拓展（如String的intern( )运行时将新常量放入常量池），方法区无法满足内存分配需求时抛出OutOfMemoryError。

 



**2、垃圾回收：**

**确定对象已死方法**

**引用计数法**：为对象添加一个引用计数器，每加个引用时计数器加一，无法解决对象间相互循环引用的问题。

**可达性分析算法**：通过一系列称为“GC Roots”的对象作为起始点，从这些节点向下开始搜索，搜索所走过的路径称为引用链，当一个对象到GC Roots没有任何引用链相连时，就说明此对象不可用，可以回收。

**Java中的GC Roots**

虚拟机栈（栈帧中的局部变量表）中引用的对象；方法区中静态属性引用的对象；方法区中常量引用的对象，本地方法栈中JNI（Native方法）引用的对象。

**垃圾收集算法**

​    （1）标记-清除算法

​     分为“标记”“清除”两个阶段，首先标记所有需要回收的对象，标记完成后统一回收所有标记的对象。有两个不足：一是效率问题，标记和清除的效率都不高。二是空间问题，清除标记后会产生大量不连续的内存碎片，空间碎片太多会导致以后分配大对象时无法找到连续的内存而不得不提前出发另外一次GC。

​    （2）复制算法

​     将可用的内存按容量划分为大小相等的两块，每次只使用其中的一块，当一块用完了，就将存活的对象复制到另一块上面，然后再把已使用过的内存空间一次清理掉。对整个半区内存回收，这样不用考虑内存碎片的情况。但是每次仅用一半的内存代价太高。现代商业虚拟机都采用这种算法来回收新生代，因为新生代大部分对象都是“朝生夕死”的，内存不用按照1:1的比例来划分内存，而是将内存分为一块较大的Eden空间和两块较小的Survivor空间，每次使用Eden和其中一块Survivor。当回收时将Eden空间可Survivor中还存活着的对象一次性的复制到另一块的Survivor空间上，最后清理掉Eden和刚才用过的Survivor空间，HotSpot虚拟机默认Eden 和Survivor的大小比例为8:1，当回收时Survivor空间不够时，需要依赖老年的内存进行担保。

​    （3）标记-整理算法     

​     复制收集算法在对象存活率较高（老年代）要进行较多的复制操作，效率会很低。标记-整理算法类似于标记-清除算法，但后续步骤不是直接对回收对象进行清理，而是让存活的对象都向一端移动，然后清理掉端边界以外的内存。

​    （4）分代收集算法

​     根据对象存活周期的不同将内存划分为几块，一般分为新生代和老年代，新生代对象存活率低，采用复制算法，老年代对象存活率高，没有额外的空间进行分配担保，必须使用“标记-清理”或“标记-整理”算法来回收。



**3、Java中的引用**：

- 强引用（Strong Reference），普通存在的引用，只要强引用存在，就不会回收引用的对象；
- 软引用（Soft Reference），内存不足时会回收引用对象；
-  弱引用（Weak Reference），弱引用关联的对象只能生存到下一次垃圾回收之前；
-  虚引用（Phantom Reference），对象的虚引用对对象的生存没有影响，为对象设置虚引用关联的唯一目的是这个对象被回收时会收到一个系统通知 。

## 四、Java反射

 

1、反射功能：

-  运行时判断任何一个对象所属的类
-  运行时构造任何一个类的对象
-  运行时判断任何一个类所具有的成员变量和方法
-  运行时调用任何一个对象的方法

2、Class类是反射的入口点，有下面3种方式获取：

-  Class.forName(“java.util.Data”)
- Tobj.getClass()
-  T.class

   JDK中，主要由以下类来实现Java反射机制，这些类都位于java.lang.reflect包中：

- Class类：代表一个类;
- Field 类：代表类的成员变量（成员变量也称为类的属性）;
- Method类：代表类的方法;
- Constructor 类：代表类的构造方法。
- Array类：提供了动态创建数组，以及访问数组的元素的静态方法;

3、编写Java反射程序的步骤：

​     1)必须首先获取一个类的Class对象

  　2)然后分别调用Class对象中的方法来获取一个类的属性/方法/构造方法的结构

4、反射调用慢的原因：反射调用是动态解析的，Jvm优化不能执行，所以反射调用比普通调用更慢，普通调用Jvm优化。

加快反射方法：setAccessible(true)取消安全检查，安全检查耗时较多.所以通过setAccessible(true)的方式关闭安全检查就可以达到提升反射速度的目的

 

5、RTTI（运行时类型信息）与反射的区别：

对RTTI来说，编译器在编译时打开和检查.class文件，而对于反射来说，.class文件在编译时是不可获取的，所以在运行时打开和检查.class文件。

 

6、getFields()与getDeclaredFields()的区别： 

getFields()获得某个类的所有的公共（public）的字段，包括父类。 

getDeclaredFields()获得某个类的所有申明的字段，即包括public、private和proteced，但是不包括父类的申明字段。 

同样类似的还有getConstructors()和getDeclaredConstructors()，getMethods()和getDeclaredMethods()。



一个完整的反射例子：

代码块

Plain Text











```
interface Speakable{
    void speak();
}


class Father{
    private int age;
    public String sex;
    public String name;

    public Father() {
    }

    public Father(String name) {
        this.name = name;
    }

    public Father(int age, String sex, String name) {
        this.age = age;
        this.sex = sex;
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public String getSex() {
        return sex;
    }

    public String getName() {
        return name;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public void setSex(String sex) {
        this.sex = sex;
    }

    public void setName(String name) {
        this.name = name;
    }

    @Override
    public String toString() {
        return "Father{" +
                "age=" + age +
                ", sex='" + sex + '\'' +
                ", name='" + name + '\'' +
                '}';
    }
}

class Sun extends Father implements Speakable{

    public Sun(int age, String sex, String name) {
        super(age, sex, name);
    }

    @Override
    public void speak() {
        System.out.println("Sun say hello");
    }

    public void speak(String s){
        System.out.println("Sun say "+s);
    }
}


public class ReflectTest {
    public static void main(String [] args) throws ClassNotFoundException, IllegalAccessException, InstantiationException, InvocationTargetException, NoSuchFieldException, NoSuchMethodException {
        Sun sun=new Sun(12,"male","Peter");
        Class<?> c= Class.forName("com.reflect.pzhao.Father");
        Father father= (Father) c.newInstance();
        System.out.println(father);
        //获取其他全部构造函数
        Constructor[] constructors=c.getConstructors();
        System.out.println(Arrays.toString(constructors));
        //利用构造函数创建实例
        Father father1= (Father) constructors[0].newInstance(12,"male","peter");
        Father father2= (Father) constructors[1].newInstance("peter");
        Father father3= (Father) constructors[2].newInstance();
        System.out.println(father1+", "+father2+", "+father3);
        //返回实例的接口
        Class<?> sc=sun.getClass();
        Class[] classes=sc.getInterfaces();
        System.out.println(Arrays.toString(classes));
        //获取父类
        Class<?> sup=sc.getSuperclass();
        System.out.println("super: "+sup.getName());

        //获取属性
        Field[] fields=c.getDeclaredFields();
        System.out.println(Arrays.toString(fields));
        System.out.println("获取属性");
        for(Field field:fields){
            int mo=field.getModifiers();
            String mod= Modifier.toString(mo);
            Class<?> type=field.getType();
            System.out.println(mod+" "+type.getName()+" "+field.getName());
        }
        //调用方法
        System.out.println(sun);
        Method method=sc.getMethod("setName",String.class);
        method.invoke(sun,"Ken");
        System.out.println(sun);

        //设置属性
        Field field=c.getDeclaredField("age");
        field.setAccessible(true);
        field.set(sun,13);
        System.out.println(sun);
    }
}
```



输出：

Father{age=0, sex='null', name='null'}
[public com.reflect.pzhao.Father(int,java.lang.String,java.lang.String), public com.reflect.pzhao.Father(java.lang.String), public com.reflect.pzhao.Father()]
Father{age=12, sex='male', name='peter'}, Father{age=0, sex='null', name='peter'}, Father{age=0, sex='null', name='null'}
[interface com.reflect.pzhao.Speakable]
super: com.reflect.pzhao.Father
[private int com.reflect.pzhao.Father.age, public java.lang.String com.reflect.pzhao.Father.sex, public java.lang.String [com.reflect.pzhao.Father.name](http://com.reflect.pzhao.father.name/)]
获取属性
private int age
public java.lang.String sex
public java.lang.String name
Father{age=12, sex='male', name='Peter'}
Father{age=12, sex='male', name='Ken'}
Father{age=13, sex='male', name='Ken'}