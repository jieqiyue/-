- 三种线程初始化方法（`Thread`、`Callable`，`Runnable`）区别

- 线程池（`ThreadPoolExecutor`，7 大参数，原理，四种拒绝策略，四个变型：Fixed，Single，Cached，Scheduled）

- - `Synchronized`：
  - 使用：方法（静态，一般方法），代码块（this，`ClassName.class`）
  - 1.6 优化：锁粗化，锁消除，自适应自旋锁，偏向锁，轻量级锁
  - 锁升级的过程和细节：无锁->偏向锁->轻量级锁->重量级锁（不可逆）
  - 重量级锁的原理（`monitor`对象，`monitorenter`,`monitorexit`）
  - `ReentrantLock`：和`Synchronized`区别？（公平锁、非公平锁、可中断锁....）、原理、用法
  - 有界、无界任务队列，手写`BlockingQueue`。
  - 乐观锁：CAS（优缺点，ABA 问题，DCAS）
  - 悲观锁：
  - `ThreadLocal` ：底层数据结构：`ThreadLocalMap`、原理、应用场景。
  - `Atomic` 类（原理，应用场景）
  - AQS：原理、`Semaphore`、`CountDownLatch`、`CyclicBarrier`
  - `Volatile`：原理：有序性，可见性

https://tech.meituan.com/2018/11/15/java-lock.html

### 线程的创建和启动

在jvm启动的时候，通常有一个非守护线程被启动。这个线程就是main线程。

> 守护线程

> 创建线程

1. 创建一个继承与thread的类，并重写run方法。那么创建好这个类之后，这个类是有能力成为一个线程的，但是要调用start方法。如果只是单纯的创建一个类继承thread并且重写run方法是没有成为一个线程的。
2. 实现runnable接口，传入thread类的构造函数。



### synchronized

https://blog.csdn.net/javazejian/article/details/72828483

https://zhuanlan.zhihu.com/p/358879110

synchronized关键字最主要有以下3种应用方式，下面分别介绍

- 修饰实例方法，作用于当前实例加锁，进入同步代码前要获得当前实例的锁

  实例方法就是非static修饰的方法。必须要创建对象实例才能调用的方法。当修饰实例方法的时候。这个锁就是一个实例一把锁。当一个线程获取到了这把锁，就是说一个线程在执行某一个实例的同步方法的时候。另外一个线程是无法继续执行这个对象的同步方法的。但是另外一个线程可以去执行这个实例的非同步方法。但如果你要去执行这个实例的非同步方法的话，但是这个非同步方法也对共享数据进行了修改的话，那么线程安全还是无法得到保障的。

  **总结：修饰实例方法，锁是实例对象！**

- 修饰静态方法，作用于当前类对象加锁，进入同步代码前要获得当前类对象的锁

  那么，在这种情况下，如果一个线程去调用被static修饰的同步方法。另外一个线程调用该对象实例的非static修饰的同步方法是可行的。代码如下：（此时会发生线程安全问题，虽然可以调用）

  ```java
  public class AccountingSyncClass implements Runnable{
      static int i=0;
  
      /**
       * 作用于静态方法,锁是当前class对象,也就是
       * AccountingSyncClass类对应的class对象
       */
      public static synchronized void increase(){
          i++;
      }
  
      /**
       * 非静态,访问时锁不一样不会发生互斥
       */
      public synchronized void increase4Obj(){
          i++;
      }
  
      @Override
      public void run() {
          for(int j=0;j<1000000;j++){
              increase();
          }
      }
      public static void main(String[] args) throws InterruptedException {
          //new新实例
          Thread t1=new Thread(new AccountingSyncClass());
          //new心事了
          Thread t2=new Thread(new AccountingSyncClass());
          //启动线程
          t1.start();t2.start();
  
          t1.join();t2.join();
          System.out.println(i);
      }
  }
  ```

  **总结：如果修饰的是静态方法的话。那么这个锁就是当前类的class锁！**

- 修饰代码块，指定加锁对象，对给定对象加锁，进入同步代码库前要获得给定对象的锁。

  修饰代码块就简单了。了解了前面两种方式之后。对于第三种修饰代码块的情况，就是说，修饰代码块可以自己制定用哪个锁。可以是类的class锁，或者是当前实例的this锁。代码如下：

  ```java
  //this,当前实例对象锁
  synchronized(this){
      for(int j=0;j<1000000;j++){
          i++;
      }
  }
  
  //class对象锁
  synchronized(AccountingSync.class){
      for(int j=0;j<1000000;j++){
          i++;
      }
  }
  ```

  总结： 

  ​		总的来说就是，先去判断这个synchronized关键字是锁什么的。到底是锁class对象，还是说锁实例对象。而且各个不同的锁法之间互不关联。就是说不同的锁法，会产生多把锁。那么，多个线程在获取这多把锁的时候是不会出现互斥现象的。就是说，互斥现象只是出现在争抢同一把锁的时候。

##### synchronized实现原理：

实现是在对象的对象头中实现的。一个对象在内存中大概分为对象头，实例变量和填充数据。

那么在对象头中还有一个Mark world。这个Mark world中有一个指针。是指向一个monitor对象的。一个对象和一个monitor 对应。那么在这个monitor 中有两个队列。_WaitSet 和 _EntryList。用来保存ObjectWaiter对象列表( 每个等待锁的线程都会被封装成ObjectWaiter对象)，_owner指向持有ObjectMonitor对象的线程，当多个线程同时访问一段同步代码时，首先会进入 _EntryList 集合，当线程获取到对象的monitor 后进入 _Owner 区域并把monitor中的owner变量设置为当前线程同时monitor中的计数器count加1，若线程调用 wait() 方法，将释放当前持有的monitor，owner变量恢复为null，count自减1，同时该线程进入 WaitSe t集合中等待被唤醒。若当前线程执行完毕也将释放monitor(锁)并复位变量的值，以便其他线程进入获取monitor(锁)。如下图所示![img](https://img-blog.csdn.net/20170604114223462?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvamF2YXplamlhbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

由此看来，monitor对象存在于每个Java对象的对象头中(存储的指针的指向)，synchronized锁便是通过这种方式获取锁的，也是为什么Java中任意对象可以作为锁的原因。

当一个线程获取到某个锁的时候，就是将这个对象的对象头中指向的monitor给获取到了。

那么同步代码块是使用的是monitorenter 和 monitorexit 指令。当monitor对象的计数器为0的时候，某一个线程就有机会去获得这个monitor（锁）对象了。当执行monitorexit 指令的时候，计数器会重置为0.此时其它线程又有机会去获得这个锁了。

那么同步方法是通过使用ACC_SYNCHRONIZED标示来判断是否是一个同步方法。进而来执行相应的同步调用。

以上这些都是重量级锁的讨论。

那么java官方也对synchronized实现了一些优化：

锁的状态总共有四种，无锁状态、偏向锁、轻量级锁和重量级锁。锁只会升级，不会降级。

其它一些点:

可重入性：

当一个线程获取到锁之后，调用其它同一把锁的同步方法。那么是可以调用成功的。而且monitor的count会++；

中断：

当一个线程是非阻塞状态的时候，去调用interrupt方法。那么这个方法会继续的执行下去，而不会发生任何改变。但是，该线程的isInterrupted()返回true。即非阻塞状态调用interrupt方法，中断状态不会被重置。

而当一个线程阻塞的时候，去调用interrupt方法。会抛出异常，并且中断状态重置。

中断与synchronized

事实上线程的中断操作对于正在等待获取的锁对象的synchronized方法或者代码块并不起作用，也就是对于synchronized来说，如果一个线程在等待锁，那么结果只有两种，要么它获得这把锁继续执行，要么它就保存等待，即使调用中断线程的方法，也不会生效。

等待唤醒机制与synchronized

所谓等待唤醒机制本篇主要指的是notify/notifyAll和wait方法，在使用这3个方法时，必须处于synchronized代码块或者synchronized方法中，否则就会抛出IllegalMonitorStateException异常，这是因为调用这几个方法前必须拿到当前对象的监视器monitor对象，也就是说notify/notifyAll和wait方法依赖于monitor对象，在前面的分析中，我们知道monitor 存在于对象头的Mark Word 中(存储monitor引用指针)，而synchronized关键字可以获取 monitor ，这也就是为什么notify/notifyAll和wait方法必须在synchronized代码块或者synchronized方法调用的原因。

```java
synchronized (obj) {
       obj.wait();
       obj.notify();
       obj.notifyAll();         
 }12345
```

需要特别理解的一点是，与sleep方法不同的是wait方法调用完成后，线程将被暂停，但wait方法将会释放当前持有的监视器锁(monitor)，直到有线程调用notify/notifyAll方法后方能继续执行，而sleep方法只让线程休眠并不释放锁。同时notify/notifyAll方法调用后，并不会马上释放监视器锁，而是在相应的synchronized(){}/synchronized方法执行结束后才自动释放锁。

### Volatile 

https://blog.csdn.net/javazejian/article/details/72772461

> 禁止指令重排序优化

内存屏障(Memory Barrier）。
内存屏障，又称内存栅栏，是一个CPU指令，它的作用有两个，一是保证特定操作的执行顺序，二是保证某些变量的内存可见性（利用该特性实现volatile的内存可见性）。由于编译器和处理器都能执行指令重排优化。如果在指令间插入一条Memory Barrier则会告诉编译器和CPU，不管什么指令都不能和这条Memory Barrier指令重排序，也就是说通过插入内存屏障禁止在内存屏障前后的指令执行重排序优化。Memory Barrier的另外一个作用是强制刷出各种CPU的缓存数据，因此任何CPU上的线程都能读取到这些数据的最新版本。总之，volatile变量正是通过内存屏障实现其在内存中的语义，即可见性和禁止重排优化。

这里引入一个例子：  单例模式

```java
public class DoubleCheckLock {

    private  volatile static DoubleCheckLock instance; 
	// 此处的instance需要用 volatile来修饰。防止指令重排序。
    private DoubleCheckLock(){}

    public static DoubleCheckLock getInstance(){

        //第一次检测
        if (instance==null){
            //同步
            synchronized (DoubleCheckLock.class){
                if (instance == null){
                    //多线程环境下可能会出现问题的地方
                    instance = new DoubleCheckLock();
                }
            }
        }
        return instance;
    }
}
```

为什么要用 volatile修饰呢？原因就是在第二个if里面的new 对象（）; 的操作会分成三个步骤执行。

```java
memory = allocate(); //1.分配对象内存空间
instance(memory);    //2.初始化对象
instance = memory;   //3.设置instance指向刚分配的内存地址，此时instance！=null
```

当进行指令重排序的时候可能第三步排到了第二。但是实例还未初始化，导致出错。但是用了禁止重排序就不会调换顺序了。

> 实现可见性？

Volatile 是**轻量级的 synchronized**，它在多处理器开发中保证了共享变量的“可见性”。可见性的意思是当一个线程修改一个共享变量时，另外一个线程能读到这个修改的值。java 线程内存模型确保所有线程看到这个变量的值是一致的。

lock 前缀的指令在多核处理器下会引发了两件事情。

- 将当前处理器缓存行的数据会写回到系统内存。
- 这个写回内存的操作会引起在其他 CPU 里缓存了该内存地址的数据无效。

能保证可见性，有序性，不能保证原子性。

>  为什么不能保证原子性？

1. jmm保证了基本数据类型赋值的原子性。一定是基本数据类型。
2. 如i = 1的赋值操作，但是像j = i或者i++这样的操作都不是原子操作，因为他们都进行了多次原子操作，比如先读取i的值，再将i的值赋值给j，两个原子操作加起来就不是原子操作了。
3. 如果一个变量被volatile修饰了，那么肯定可以保证每次读取这个变量值的时候得到的值是最新的，但是一旦需要对变量进行自增这样的非原子操作，就不会保证这个变量的原子性了。
4. 主要问题就是说，给一个变量做自增操作或者其它非原子性操作的时候出的问题。就拿自增来讲吧。a线程在把i自增1之后，没来得及把它写回内存，此时另外一个b线程读取了i。那么这个i是之前那个i，并没有进行自增。此时b线程也自增。然后写入内存。使得a线程内存中的i值失效。但是。此时a线程做自增操作不需要重新去内存中读取了啊。因为刚刚已经读取并且加一了。所以，只是剩下a线程的最后一个写入内存。那么，a线程写入内存成功。导致，两个线程都执行了++操作。但是值只被增加了一次。

```java
public class errot {
    static volatile int a = 0;
    public static void main(String[] args) {
        new Thread("thread_my1"){
            @Override
            public void run() {
                for (int i = 0;i < 50;i++){
                    a++;
                }
            }
        }.start();

        new Thread("thread_my2"){
            @Override
            public void run() {
                for (int i = 0;i < 55;i++){
                    System.out.println(a);
                }
            }
        }.start();

    }
}
```

如上程序，输出结果一定全部都是50，因为根据happens-before原则第三条。

如果对声明了Volatile变量进行写操作，JVM就会向处理器发送一条Lock前缀的指令，将这个变量所在缓存行的数据写回到系统内存。但是就算写回到内存，如果其他处理器缓存的值还是旧的，再执行计算操作就会有问题，所以在多处理器下，为了保证各个处理器的缓存是一致的，就会实现缓存一致性协议，每个处理器通过嗅探在总线上传播的数据来检查自己缓存的值是不是过期了，当处理器发现自己缓存行对应的内存地址被修改，就会将当前处理器的缓存行设置成无效状态，当处理器要对这个数据进行修改操作的时候，会强制重新从系统内存里把数据读到处理器缓存里。

### happen-before 原则

happen-before原则有八条。这些原则都是为了保证在不加锁的情况下也能保证一部分的线程安全。这些原则都是针对于指令重排序来说的。就是说不能随意的进行指令的重排序，必须得满足这些原则。

比如说其中一条是：volatile变量规则（volatile Variable Rule） A write to a volatile field happens-before every subsequent read of that volatile.

那么一个读的操作一定不会被重排序到一个读的操作的前面。

### 无锁CAS与Unsafe类及其并发包Atomic

https://blog.csdn.net/javazejian/article/details/72772470

synchronized =====> 有锁并发

CAS ======> 无锁并发

###### CAS 核心算法原理

首先介绍两点。悲观派和乐观派。对于悲观派来说，每一次都假设有线程之间的争抢，每一次都需要加锁来保证安全。但是乐观派不一样，乐观派觉得不用加锁，当出现线程安全问题的时候，就用cas算法来解决问题。

CAS的全称是Compare And Swap 即比较交换，其算法核心思想如下

> 执行函数：CAS(V,E,N)

其包含3个参数

- V表示要更新的变量
- E表示预期值
- N表示新值

![这里写图片描述](https://img-blog.csdn.net/20170701155737036?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvamF2YXplamlhbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

从上图可以看出，即使更新失败，那么也有两种情况，继续读取变量，继续更新，或者放弃更新。

那么，这样的cas是一个无锁的状态，那么就天生的不会出现死锁的现象。

而且，cas不会出现数据不一致的现象，因为涉及的操作都是操作系统的原语操作。

###### UNsafe类

UNsafe中的几个方法。cas操作是基于以下三个方法实现的。

```java
//第一个参数o为给定对象，offset为对象内存的偏移量，通过这个偏移量迅速定位字段并设置或获取该字段的值，
//expected表示期望值，x表示要设置的值，下面3个方法都通过CAS原子指令执行操作。
public final native boolean compareAndSwapObject(Object o, long offset,Object expected, Object x);                                                                               
public final native boolean compareAndSwapInt(Object o, long offset,int expected,int x);
public final native boolean compareAndSwapLong(Object o, long offset,long expected,long x);
```

###### Atomic包

- AtomicInteger：原子更新整型

其它的几个atomic的类都差不都，我们来分析以下AtomicInteger的incrementAndGet方法。

```java
//当前值加1，返回新值，底层CAS操作
public final int incrementAndGet() {
     return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
 }

//Unsafe类中的getAndAddInt方法
public final int getAndAddInt(Object o, long offset, int delta) {
        int v;
        do {
            v = getIntVolatile(o, offset);
        } while (!compareAndSwapInt(o, offset, v, v + delta));
        return v;
    }
```

通过上面的代码就很容易的看出来了cas到底是怎样辅助atomic类来进行操作的。

aba问题？

可以通过增加一个时间戳来破解。

### ReentrantLock（可重入锁）

synchronize是隐式锁，锁的加锁和解锁都没有我们干预。而reentrantLock是显式锁。Lock其实是juc下的一个接口。实现的类是ReentrantLock。Lock对象锁还提供了synchronized所不具备的其他同步特性，如可中断锁的获取(synchronized在等待获取锁时是不可中断的)，超时中断锁的获取，等待唤醒机制的多条件变量Condition等，这也使得Lock锁在使用上具有更大的灵活性。

下面看几个要点：

1. 可重入性。可以在同一个线程中多次加锁，但是多次加锁就必须要多次解锁。
2. 可以设置公平锁（顺序相关），或者非公平锁（获取锁与请求顺序无关）。

##### ReetrantLock的内部实现原理

ReetrantLock分为公平和非公平锁两种。ReetrantLock是借助AQS来做的。在AQS中已经定义了入队列，判断前驱结点是否是头结点等方法，以及定义了双向列表的头尾节点，锁的状态，锁的持有者，对于Thread构造成一个Node节点，以及表示该线程状态的一些变量。所以说，ReetrantLock仅仅只是提供了一个获取锁的实现。

公平锁：

在公平锁中，获取锁的方法是lock()方法。在这个方法中，会调用到ReetrantLock中定义的tryacquire()方法获取锁。在这个tryacquire中就会去判断当前的state的值是否为0，以及当state为非0的时候，这个时候代表已经有线程获取到锁了，那么就会去判断一下当前持有锁的线程是不是就是当前请求锁的线程。如果是，那就直接增加state的值，这里就实现了可重入。那么，如果此时没有获取到锁呢？那么就会将当前请求锁的线程给创建出一个Node对象，再放到双向链表的末尾。在成功放到链表尾部之后，就会去判断一下刚刚放进去的那个节点的前驱节点是否是头结点，如果是头结点，那么就进行一个获取锁的操作。当然也可能是获取锁失败。那么，如果再次获取锁失败了之后，就会考虑是否要挂起当前节点了。此时，就会将刚刚在末尾的那个节点给挂到一个waitStatus为SIGNAL的节点上，然后就被park了。在前驱节点release的时候，或者是超时退出的时候，就会唤醒自己。那么，还有一点就是一个Node刚刚创建的时候的waitState是0的，它是在什么时候被置位SIGNAL的呢？就是在下面的倒数第二行代码处。

```java
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus; // 获得前驱节点的ws
        if (ws == Node.SIGNAL)
            // 前驱节点的状态已经是SIGNAL了，说明闹钟已经设了，可以直接睡了
            return true;
        if (ws > 0) {
            // 当前节点的 ws > 0, 则为 Node.CANCELLED 说明前驱节点已经取消了等待锁(由于超时或者中断等原因)
            // 既然前驱节点不等了, 那就继续往前找, 直到找到一个还在等待锁的节点
            // 然后我们跨过这些不等待锁的节点, 直接排在等待锁的节点的后面 (是不是很开心!!!)
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            // 前驱节点的状态既不是SIGNAL，也不是CANCELLED
            // 用CAS设置前驱节点的ws为 Node.SIGNAL，给自己定一个闹钟
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);// 这里置位-1
        }
        return false;
    }
```

释放锁：

在释放锁的时候，相对来说比较简单。具体做法是将state的值减去1，如果state变为0，的代表当前锁已经没人占用了。此时，会去队列里面从后往前找一个waitState小于等于0的节点，将它唤醒。此时有两点需要注意，一个是为啥要从后往前找。因为在将一个节点连接到链表尾部的时候分为三步，而第三步才是将原来尾结点的next赋值为新节点。这就导致了，从前往后可能会漏掉最后的那个节点。所以从后往前。还有一点就是，在唤醒之后，还会去判断在等待锁期间是否被中断过。是否被中断的判断就是指判断一下是否将中断置位了。如果有人中断过的话，那么在被唤醒后会去中断一下自己。就相当于是把这个中断延后执行了。

公平锁和非公平的实现：

非公平的锁在调用lock获取锁的时候不管链表中有没有节点都会去尝试获取锁，而公平锁在获取锁的时候回先去判断一下链表中是否有节点在等待锁，如果有则直接执行入队列操作。如果没有才会去尝试得到锁。

### wait、join、sleep

调用wait、join、sleep这三个函数的时候，都会去检查线程的中断状态是否是true。如果为true的话，则会抛出中断异常。如果线程在等待synchronize锁的话，此时去调用这个线程的interrupt方法。只会将这个线程的中断状态置位true。但是否真的中断了呢，这个还得看线程自己的状态。当在这三个函数中调用中断方法，则会抛出异常。并将中断状态置位false。又可以重新去中断这个线程。

### interrupt

中断操作只是给线程的一个建议，最终怎么执行看线程本身的状态，那么什么状态做什么事情呢？

- 若线程被中断前，如果该线程处于非阻塞状态(未调用过`wait`,`sleep`,`join`方法)，那么该线程的中断状态将被设为true, 除此之外，不会发生任何事。
- 若线程被中断前，该线程处于阻塞状态(调用了`wait`,`sleep`,`join`方法)，那么该线程将会立即从阻塞状态中退出，并抛出一个`InterruptedException`异常，同时，该线程的中断状态被设为false, 除此之外，不会发生任何事。

可以通过isinterrupted来判断自己的中断状态是否是true。

### 线程池

> 七大参数

```java
 public ThreadPoolExecutor(int corePoolSize,
                           int maximumPoolSize,
                           long keepAliveTime,
                           TimeUnit unit,
                           BlockingQueue<Runnable> workQueue,
                           ThreadFactory threadFactory,
                           RejectedExecutionHandler handler)
```

1. 核心线程数
2. 最大线程数
3. 空闲线程最多生存时间
4. 第三个参数的单位，秒或者分钟或者小时
5. 任务队列
6. 线程工厂
7. 拒绝策略

> 四种拒绝策略

1. 直接拒绝，并且抛出异常
2. 直接拒绝，什么都不做
3. 将队列中最老的那个线程给扔掉，将新的线程加入队列
4. 将线程交给调用者执行，比如说main线程

   当然也可以自己定义拒绝策略。

> 如何关闭线程池

1. 调用shutdown方法。此方法会等待所有任务执行完再去关闭线程池。即使是在任务队列中的任务也会被执行完。
2. 调用shutdownNow方法。此方法会打断线程池中的线程。并且关闭空闲线程。并且将任务队列中的任务给返回。也就是说，任务队列中的任务是不能够被执行的。

> 四个生成线程池的工厂方法（不推荐）

不推荐的原因是不够灵活。因为参数都是已经写死的了。但是好处是创建比较简单，不用自己写七个参数。其实底层也是调用那个七个参数的构造方法。

1. ```java
   ExecutorService service = Executors.newCachedThreadPool();
   ```

   创建一个初始线程数为0的线程池。适合**短任务**。它的任务队列的特点是一次只能存入一个，并且只有等上一个任务拿走了之后，下一个任务才能放进去。这种方式可能会创建大量的线程。而且在执行完任务之后，这个线程池会自动结束。

2. ```java
   ExecutorService service = Executors.newFixedThreadPool(10);
   ```

   特点是核心线程数量和最大线程数都是一样的。所以线程一直是那么多个。任务队列是integer的最大数量。

   任务执行完毕之后，不会结束线程池。

3. ```java
   ExecutorService service = Executors.newSingleThreadExecutor();
   ```

   这个方式被FinalizableDelegatedExecutorService这个类给包装了。目的是只暴露ExecutorService中的那些方法。这个只会去创建一个线程。最大的也是1。

4. ```java
   ExecutorService service = Executors.newWorkStealingPool();
   ```

   充分利用你的多核CPU。其实返回的是ForkJoinPool。ForkJoinPool也是实现了ExecutorService接口的一个类。

可以在创建好线程池之后，可以execute任务也可以submit任务。submit任务的返回值是Future。

> 三种任务队列

![图片](https://mmbiz.qpic.cn/mmbiz_png/A3ibcic1Xe0iaRNBshy5yC0KdbGAK4icrSTWsglZpUC0aNXy6dtwGK4TDtUC24ibzHib54vCJW8ia2ymQQibE1rtVez03g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- `SynchronousQueue`: 基于`阻塞队列(BlockingQueue)`的实现，它会直接将任务交给消费者，必须等队列中的添加元素被消费后才能继续添加新的元素。使用 SynchronousQueue 阻塞队列一般要求maximumPoolSizes 为无界，也就是 Integer.MAX_VALUE，避免线程拒绝执行操作。
- `LinkedBlockingQueue`：LinkedBlockingQueue 是一个**无界缓存等待队列**。当前执行的线程数量达到 corePoolSize 的数量时，剩余的元素会在阻塞队列里等待。
- `ArrayBlockingQueue`：ArrayBlockingQueue 是一个**有界缓存等待队列**，可以指定缓存队列的大小，当正在执行的线程数等于 corePoolSize 时，多余的元素缓存在 ArrayBlockingQueue 队列中等待有空闲的线程时继续执行，当 ArrayBlockingQueue 已满时，加入 ArrayBlockingQueue 失败，会开启新的线程去执行，当线程数已经达到最大的 maximumPoolSizes 时，再有新的元素尝试加入 ArrayBlockingQueue时会报错

> ExecutorService API

1. 

```java
void shutdown();
```

在shutdown之后就不能再提交任务了。shutdown会将线程池关闭。这个方法不是阻塞的。

2. 在任务执行过程中出现错误怎么办？

可以自定义一个线程工厂。在创建线程池的时候使用我们自己的线程工厂。不推荐这种方式。

```java
private static class MyThreadFactory implements ThreadFactory{
    @Override
    public Thread newThread(Runnable r) {
        Thread t = new Thread(r);
        t.setUncaughtExceptionHandler((thread,exception)->{
            System.out.println(thread.getName() + "出现异常了 .. ");
        });
        return t;
    }
}
```

或者是你定义一个抽象类去继承runnable接口。再在这个抽象类中定义一些打印异常信息的方法。这样的话，你在出异常的时候，在run方法中出异常的时候，你就可以去调用自己定义的那些打印异常信息的方法（在catch代码块中）。用到了模板设计模式。

3. ```java
   public void allowCoreThreadTimeOut(boolean value)
   ```

   这个方法表示可以将核心线程给终结。

4. ```java
   public boolean prestartCoreThread() {
   ```

   我们在刚刚启动一个线程池的时候，core thread 是0 。当我们提交任务的时候才会去创建线程。但是我想让线程池刚刚启动的时候就去创建一个线程的话，那么可以使用这个方法。预先创建线程。

5. ```java
   <T> T invokeAny(Collection<? extends Callable<T>> tasks)
   ```

   将传入的tasks中所有的tasks都执行，但是返回值只有一个。返回值是所有task中返回的最快的那个。当这些task中有一个返回了，那么其他的task都会被结束。

6. ```java
   <T> Future<T> submit(Runnable task, T result);
   ```

   提交一个任务，这个runnable接口本来是没有返回值的。我们可以用这个submit给它一个返回值。当然future可以调用get（）方法来获取到T类型的返回值。这个get方法是阻塞的方法。当get得到result之后，就可以认为这个task执行结束了。

7. ```java
   public BlockingQueue<Runnable> getQueue() {
   ```

   这是一个挺有趣的方法。我们可以直接得到任务队列。那么我们如果直接通过得到的这个任务队列往里面去添加任务呢，这个任务会不会被执行呢？答案是，只有当线程池中有在wait的线程的时候，才会被执行。

   ```java
   ThreadPoolExecutor executorService1 = (ThreadPoolExecutor) Executors.newFixedThreadPool(3);
   executorService1.prestartCoreThread(); // 如果注释掉这句，那么下面的任务不会被执行。
   executorService1.getQueue().put(()->{
       System.out.println(123);
   });
   ```

> future

提交一个任务，直接立马返回一个结果，这个结果类似于一个凭证一样的东西。你等下拿着这个凭证去那任务的结果。

```java
ExecutorService service = Executors.newWorkStealingPool();

Future<String> submit = service.submit(() -> {
     // get data from db ... it will takes long time ... 
     TimeUnit.SECONDS.sleep(10);
     return "hello ";
});
 // ==================================
 //  I can do something when the submit is not return 
 // ==================================
 
 // then I can get the result ... 
 String s = submit.get();
```

这个get方法还有一个重载的timeout的方法。当timeout之后，任务还会被继续执行完毕，但是被get给wait住的那个线程不再wait继续等下去了。

### 并发容器

### 并发工具

#### AQS

AQS 作为一个顶层的逻辑，提供了一些通用的方法。继承它的类需要根据自己的实际情况来实现某一些方法。

```
独占模式需要实现的 
<li> {@link #tryAcquire}	
<li> {@link #tryRelease}
共享模式需要实现的
<li> {@link #tryAcquireShared}    	
<li> {@link #tryReleaseShared}
<li> {@link #isHeldExclusively}
```

实现AQS的子类需要考虑的几个方面有：

是否可重入，是否是独占模式（独占模式有Condition），是否公平。

> 属性

1. 在AQS里面有一个state的属性，这个属性表示什么意思，是由实现了AQS的类来决定的。就比如说我自定义一个锁，这个锁是独占的。可以重入的。那么这个state的意思就是重入的次数，当state为0的时候，代表的意思就是说当前没有锁占用。
2. queue 同步队列。就比如说第一个线程占用到锁了，那么第二个线程占用锁失败。那么第二个线程就会入队。此时会先去初始化这个队列。那么也有可能是，这个队列永远也不会出现。就比如说，当第二个线程要去得到锁的时候，就直接得到了，那么第二个线程也就不用入队了。所以也就没有初始化队列的操作。那个入队也有两个操作。第一步是构造队列的数据结构，就是在数据结构的层面上创建好一个队列。然后再去判断刚刚创建的这个节点是否有资格去获取锁。如果是头结点的下一个节点的话，那么就有资格尝试获取锁。如果获取成功，那么就从双向链表中删除刚刚创建好的那个节点。
3. waitState。这个waitStatus在刚开始的时候是0，当某个线程入队之后，它会将它的前一个Node节点的waitStatus给设置为-1。表示在前一个节点结束释放锁的时候，要将自己唤醒。
4. 头结点。这个头结点就是双向列表中的dummy节点。实际不存放有用的数据，只是为了表示的方便。
5. 条件队列。Condition。条件队列中的Node节点用的和前面的双向链表中的节点是同一个Node ADT。这个时候Node节点中的nextWaiter就派上用场了。当调用Condition的await操作的时候，会创建一个Condition节点，并且加入到条件队列中去。并且waitState设置为-2。当调用Condition的signal的时候，会将同步队列中的节点转移到之前的双向链表中。

> 方法

1. 释放锁 release操作

   1. 在释放锁的时候，首先会检查当前线程是否是持有锁的那个线程。

   2. 然后修改AQS的state属性。

   3. 唤醒FIFO队列中的头结点的下一个节点中的线程。

   4. 当头结点的下一个节点被唤醒之后，该节点首先会判断自己的前驱结点是不是头结点。然后再去争抢锁。

   5. 当头结点的下一个节点争抢到锁之后，会去修改state的值。再去修改head指针。将原来的head节点的next设置为null，争抢到锁的那个节点的pre设置为null。此时原来的head节点将会被GC回收。

> ReentrantLock源码

ReentrantLock是一个可重入锁。可重入的意思是一个线程可以重复的调用` lock() `方法，在每一次调用` lock` 方法的时候都会对state增加1。用这种方式来实现重入。当state为 0 的时候看做锁没有线程持有。上面也说了，这个state是AQS维护的一个变量。具体含义由AQS的子类来实现和定义。所以，在这里state被认为有没有线程持有锁。

第二点是，ReentrantLock是一个可以通过在构造函数中传入一个true或者false来使用公平锁，或者非公平锁。默认是非公平锁。

第三点，ReentrantLock是一个独占锁。也就是说一个时刻只能有一个线程获得到锁。

> lock() 方法

以下是非公平锁的实现方式：

1. 首先，调用 compareAndSetState(0, 1) 如果能够成功设置则表示该线程抢到了锁。如果设置失败，则会调用`acquire(int arg)` 方法。

```java
    final void lock() {
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
    }


	public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

2. 在 `acquire(int arg)`中首先还是会`tryAcquire(arg)`去尝试获取一次锁。并且判断一下是不是持有锁的线程可以当前的线程是不是同一个，这样来实现可重入的。如果没有成功的获得锁，而且当前线程也不是持有锁的线程，那么就会执行入队列操作。先是构建一个Node，并加入队列中（先把数据结构给创建好）。并且返回链表中新创建的那个node节点的前一个节点。或者新创建的node节点。
3. 然后再会去调用` acquireQueued(addWaiter(Node.EXCLUSIVE), arg)` 申请入队列。刚刚已经把数据结构创建好了。在这个申请入队列的时候，如果刚刚那个加入链表的节点的前一个节点是head节点的话，还是有一次机会去获取锁的。如果没有获取到锁，则会被阻塞。直到被前一个节点唤醒。

> unlock() 方法

调用AQS类的release方法。release相对来说很简单。就是先去判断一下当前持有锁的线程是不是调用release的线程。如果不是则会抛出异常。然后将state的值减1.如果当state减到 0 了之后，那么就可以认为当前锁已经被释放了 。然后就会去唤醒头结点的下一个节点。那么，公平与非公平也是在这里实现的。非公平的话，当去唤醒头结点的下一个节点去争抢锁，但是此时可能有新的线程跟它抢，所以可能抢失败了。那么，公平锁在新节点抢锁的时候会去判断链表中是否有需要抢锁的节点，如果有则会放弃抢锁，直接加入队列。这样保证了公平性。

#### CountDownLatch

CountDownLatch 就是在一个线程调用了await之后，只有当CountDownLatch 的值为0的时候，这个wait的线程才能够继续的执行。就比如说main线程等待所有的线程都执行完毕之后，再去执行后面的代码，此时就可以用CountDownLatch 。每一个线程执行完自己的逻辑之后，都调用一次countDown()。当state的值为0时，main线程开始执行。

第一点：CountDownLatch 也是通过AQS来实现的。使用了AQS的共享模式。可以说CountDownLatch 就是使用了AQS的一些共享模式的方法来实现的。

在CountDownLatch 中，这个state字段的意思就是最多共享的线程。它这个CountDownLatch 就是拿来做一种类似于线程间同步的，线程间通知的作用的。所以并不会记录当前持有锁的线程是谁。它的state字段的意思是资源能够被多少个线程占用。

#### ThreadLocal

在Thread类中，有一个ThreadLocalMap。这里的这个map的key就是ThreadLocal，而value就是这个ThreadLocal里面要存放的值。

### 面试题：

#### sleep() 和 wait() 的区别

sleep() 方法是线程类（Thread）的静态方法，让调用线程进入睡眠状态，让出执行机会给其他线程，等到休眠时间结束后，线程进入就绪状态和其他线程一起竞争cpu的执行时间。并且不会让出锁。因为是静态方法。不能改变对象锁。对象是在实例化的时候初始化的，静态方法是在java虚拟机加载的时候初始化，当然静态方法不能影响到对象锁。

wait()是Object类的方法，当一个线程执行到wait方法时，它就进入到一个和该对象相关的等待池，同时释放对象的机锁，使得其他线程能够访问，可以通过notify，notifyAll方法来唤醒等待的线程。

sleep不需要被唤醒。wait需要被唤醒。

sleep不需要抢到锁，而wait需要。

#### java中线程一共有几种状态？6种!

https://blog.csdn.net/Baisitao_/article/details/99766322?utm_medium=distribute.pc_relevant.none-task-blog-OPENSEARCH-2.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-OPENSEARCH-2.control

```java 
/**
 * A thread can be in only one state at a given point in time.
 * These states are virtual machine states which do not reflect
 * any operating system thread states.
 * 一个线程在特定的时刻，只能有一个状态。
 * 这些状态是虚拟机的状态，不能反映任何操作系统的线程状态
 * @since   1.5
 */
public enum State {
    /**
     * Thread state for a thread which has not yet started.
     * 线程未开始的状态
     */
    NEW,

    /**
     * Thread state for a runnable thread.  A thread in the runnable
     * state is executing in the Java virtual machine but it may
     * be waiting for other resources from the operating system
     * such as processor.
     * 可运行的线程状态，此状态的线程正在JVM中运行，
     * 但也有可能在等待操作系统的其他资源，例如处理器
     */
    RUNNABLE,

    /**
     * Thread state for a thread blocked waiting for a monitor lock.
     * A thread in the blocked state is waiting for a monitor lock
     * to enter a synchronized block/method or
     * reenter a synchronized block/method after calling
     * 阻塞状态的线程在等待一个监视器锁去进入同步代码块/方法，
     * 或者调用后重新进入同步代码块/方法
     */
    BLOCKED,

    /**
     * Thread state for a waiting thread.
     * A thread is in the waiting state due to calling one of the following methods:
     * Object.wait()
     * Thread.join()
     * LockSupport.park()
     * A thread in the waiting state is waiting for another thread to
     * perform a particular action.
     * 由于调用了Object.wait()、Thread.join()、LockSupport.park()其中一个方法，线程进入等待状态
     * 处于等待状态的线程，正在在等待另一个线程执行特定的操作
     */
    WAITING,

    /**
     * Thread state for a waiting thread with a specified waiting time.
     * A thread is in the timed waiting state due to calling one of
     * the following methods with a specified positive waiting time:
     * Thread.sleep()
     * Object.wait()
     * LockSupport.parkNanos()
     * LockSupport.parkUntil()
     * 有指定等待时间的线程等待状态
     * 由于调用了Thread.sleep(long)、Object.wait(long)、Thread.join(long)、
     * LockSupport.parkNanos(long)、LockSupport.parkUntil(long)其中一个方法并传入了正（数）时间参数，
     * 线程处于有时间限制的等待状态
     */
    TIMED_WAITING,

    /**
     * Thread state for a terminated thread.
     * The thread has completed execution.
     * 线程已经执行完成，处于终止状态
     */
    TERMINATED;
}

```

#### 死锁的产生、防止、避免、检测和解除.

https://blog.csdn.net/jgm20475/article/details/81297819