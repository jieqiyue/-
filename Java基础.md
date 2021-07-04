- 面向对象特性：封装，多态（动态绑定，向上转型），继承
- 泛型，类型擦除
- 反射，原理，优缺点
- `static`，`final` 关键字
- `String`，`StringBuffer`，`StringBuilder`底层区别
- BIO、NIO、AIO
- `Object` 类的方法
- 自动拆箱和自动装箱

## string对象

string是一个不可变对象。

```
private final byte[] value;
```

string的一些replaceall等方法都是创建了一个新的string再返回的所以原有的string对象并没有改变。

string中的 ”+“ 运算符：

只要+拼接时，存在已经定义的变量，则会调用StringBuilder进行拼接操作，所以在定义字符串时，尽量不应使用变量。定义字符串的时候，如果是String s = "a" + "b"; 那么编译器在编译的时候会进行优化。直接在常量池中放入"ab"。

stringbuffer ：

 线程安全。通过预先分配一些缓冲区来提升性能。对于经常需要变更的string。建议使用。

## 内部类

1. 内部类分为四种：成员内部类、局部内部类、匿名内部类和静态内部类。

2. 成员内部类有三种访问权限可以修饰。就可以把这种内部类看成是外部类的一个成员变量。成员内部类，如果你去观察它生成的class文件，你会发现，内部类有一个独立的class文件，然后，这个class文件中保存了一个指向外部类的指针。这个指针是在内部类创建的时候，在构造函数中由编译器给加进去的。所以，这里也可以看出来成员内部类是依赖于外部类的。即内部类必须在外部类创建好了之后，才能被创建。

3. 匿名内部类一般用的最多。一般是用来实现某一个接口，或者继承某一个父类。

4. 为什么局部内部类和匿名内部类只能访问局部final变量？

   这个过程是在编译期间由编译器默认进行，如果这个变量的值在编译期间可以确定，则编译器默认会在匿名内部类（局部内部类）的常量池中添加一个内容相等的字面量或直接将相应的字节码嵌入到执行字节码中。这样一来，匿名内部类使用的变量是另一个局部变量，只不过值和方法中局部变量的值相等，因此和方法中的局部变量完全独立开。

5. 

## NIO

Channel就像是一个管道一样，根据不同类型的管道，连接了不同的缓冲区和数据源。缓冲区是在内存中的。而另外一边可能是file或者是socket。

##### 缓冲区 （ Buffer ）

Fields

所有缓冲区都有4个属性：capacity、limit、position、mark，并遵循：mark <= position <= limit <= capacity，下表格是对着4个属性的解释：

属性 描述

| Capacity | 容量，即可以容纳的最大数据量；在缓冲区创建时被设定并且不能改变 |
| -------- | :----------------------------------------------------------- |
| Limit    | 表示缓冲区的当前终点，不能对缓冲区超过极限的位置进行读写操作。且极限是可以修改的 |
| Position | 位置，下一个要被读或写的元素的索引，每次读写缓冲区数据时都会改变改值，为下次读写作准备 |
| Mark     | 标记，调用mark()来设置mark=position，再调用reset()可以让position恢复到标记的位置 |

![image-20210120212300546](https://gitee.com/qiyuejie/qiyue/raw/master/img/image-20210120212300546.png)

| flip() | limit = position;position = 0;mark = -1;  翻转，也就是让flip之后的position到limit这块区域变成之前的0到position这块，翻转就是将一个处于存数据状态的缓冲区变为一个处于准备取数据的状态 |
| ------ | ------------------------------------------------------------ |
|        |                                                              |

##### 文件通道

创建通道

FileChannel 是一个用于连接文件的通道，通过该通道，既可以从文件中读取，也可以向文件中写入数据。与SocketChannel 不同，FileChannel 无法设置为非阻塞模式，这意味着它只能运行在阻塞模式下。在使用FileChannel 之前，需要先打开它。由于 FileChannel 是一个抽象类，所以不能通过直接创建而来。必须通过像 InputStream、OutputStream 或 RandomAccessFile 等实例获取一个 FileChannel 实例。

```java
FileInputStream fis = new FileInputStream(FILE_PATH);
FileChannel channel = fis.getChannel();

FileOutputStream fos = new FileOutputStream(FILE_PATH);
FileChannel channel = fis.getChannel();

RandomAccessFile raf = new RandomAccessFile(FILE_PATH , "rw");
FileChannel channel = raf.getChannel();
```

文件通道就是一个数据通道，通过这个通道可以将数据放入上面那个缓冲区。

Chanel可以设置为非阻塞的，如果设置为非阻塞一般需要绑定到Selector上。

#### Selector

Java NIO 引入 Selector ( 选择器 )的概念，它是 Java NIO 得以实现非阻塞 IO 操作的**最最最关键**。

## BIO与NIO、AIO的区别

 IO的方式通常分为几种，同步阻塞的BIO、同步非阻塞的NIO、异步非阻塞的AIO。

首先分清楚同步和异步，阻塞和非阻塞的概念。

Unix IO模型的语境下，同步和异步的区别在于数据拷贝阶段是否需要完全由操作系统处理。阻塞和非阻塞操作是针对发起IO请求操作后是否有立刻返回一个标志信息而不让请求线程等待。

> nio （同步非阻塞）

NIO基于Reactor，当socket有流可读或可写入socket时，操作系统会相应的通知引用程序进行处理，应用再将流读取到缓冲区或写入操作系统。NIO是使用单线程或者只使用少量的多线程，每个连接共用一个线程。NIO的最重要的地方是当一个连接创建后，不需要对应一个线程，这个连接会被注册到多路复用器上面，所以所有的连接只需要一个线程就可以搞定，当这个线程中的多路复用器进行轮询的时候，发现连接上有请求的话，才开启一个线程进行处理，也就是一个请求一个线程模式。

同步非阻塞IO:在此种方式下，用户进程发起一个IO操作以后边可返回做其它事情，但是用户进程需要时不时的询问IO操作是否就绪，这就要求用户进程不停的去询问，从而引入不必要的CPU资源浪费。其中目前JAVA的NIO就属于同步非阻塞IO。

## 序列化与反序列化

https://www.cnblogs.com/9dragon/p/10901448.html

### 一、序列化的含义、意义及使用场景

- **序列化：将对象写入到IO流中**
- **反序列化：从IO流中恢复对象**
- **意义：序列化机制允许将实现序列化的Java对象转换位字节序列，这些字节序列可以保存在磁盘上，或通过网络传输，以达到以后恢复成原来的对象。序列化机制使得对象可以脱离程序的运行而独立存在。**
- **使用场景：所有可在网络上传输的对象都必须是可序列化的，**比如RMI（remote method invoke,即远程方法调用），传入的参数或返回的对象都是可序列化的，否则会出错；**所有需要保存到磁盘的java对象都必须是可序列化的。通常建议：程序创建的每个JavaBean类都实现Serializeable接口。**

### 二、序列化实现的方式

如果需要将某个对象保存到磁盘上或者通过网络传输，那么这个类应该实现**Serializable**接口或者**Externalizable**接口之一。

### 三 、序列化要注意的地方

**反序列化并不会调用构造方法。反序列的对象是由JVM自己生成的对象，不通过构造方法生成。**

**如果一个可序列化的类的成员不是基本类型，也不是String类型，那这个引用类型也必须是可序列化的；否则，会导致此类不能序列化。**

**Java序列化同一对象，并不会将此对象序列化多次得到多个对象。**

- **Java序列化算法**

1. **所有保存到磁盘的对象都有一个序列化编码号**

2. **当程序试图序列化一个对象时，会先检查此对象是否已经序列化过，只有此对象从未（在此虚拟机）被序列化过，才会将此对象序列化为字节序列输出。**

3. **如果此对象已经序列化过，则直接输出编号即可。**

   

由于java序利化算法不会重复序列化同一个对象，只会记录已序列化对象的编号。**如果序列化一个可变对象（对象内的内容可更改）后，更改了对象内容，再次序列化，并不会再次将此对象转换为字节序列，而只是保存序列化编号。**

## 枚举

```java
enum Weekday {
    MON("1",1){
        @Override
        public String toString() {
            return super.toString();
        }
        // 注意这里重写了
        @Override
        public boolean isOrdered() {
            return true;
        }
    },STA("2",2){
        @Override
        public String toString() {
            return super.toString();
        }
    };

    public boolean isOrdered() {
        return false;
    }

    public final String dayValue;
    public int num;

    private Weekday(String dayValue,int num) {
        this.dayValue = dayValue;
        this.num = num;
    }
}

public static void main(String[] args) {
    Weekday weekday = Weekday.MON;
    System.out.println(weekday.isOrdered());
    Weekday sta = Weekday.STA;
    System.out.println(sta.isOrdered());
    // 可以像下面这样进行比较，因为jvm中，仅仅存在单个实例
    Weekday weekday = Weekday.MON;
    assert weekday == Weekday.MON;
}
```

在enum中可以定义自己的变量和方法。并且每一个实例都可以自己去重写定义在enum中的方法。

在枚举中可以很方便的使用 `== ` 来比较两个枚举是否相等。因为在jvm中，保证了枚举实例的唯一性。它是唯一存在的所以可以直接使用 `== ` 来进行比较。

并且枚举可以说是实现单例模式最好的模式。先复习一下其它的方式：

- 静态内部类

  利用一个特性：静态内部类在没有访问它的成员属性或者成员方法的时候，static代码块或者static变量是不会被初始化的，那么就可以做到懒加载。（只有主动去访问了，才会加载）并且能够做到线程安全。代码如下：

  ```java
  public class Singleton {  
  	// 静态内部类
      private static class SingletonHolder {  
      	private static final Singleton INSTANCE = new Singleton();  
      }  
      private Singleton (){}  
  	
      public static final Singleton getInstance() { 
          // 只有调用了本方法才会进行静态内部类的初始化
      	return SingletonHolder.INSTANCE;  
      }  
  }  
  ```

  

