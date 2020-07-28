# Concurrent

## 基础理论

### JMM

所有的实例域、静态域和数组元素都存储在堆内存中，堆内存在线程之间共享。局部变量、方法定义参数、异常处理器参数存放在线程变量中，其不会在线程之间共享，它们也不会有内存可见性问题，也不受内存模型限制。

线程之间的共享变量存储在**主内存**中，每个线程都有一个私有的本地内存，本地内存中存储了该线程读/写共享变量的副本。

JMM通过主内存与每个线程的本地内存之间交互，来为java程序提供内存可见性保证。



## 并发编程的挑战

### 1.1 上下文切换(Context Switch)

#### 1.1.1 上下文切换的原因

因为现代的操作系统要多个任务并行执行，即使在单CPU的情况下也支持多线程执行，CPU通过给每个线程分配CPU时间片来实现这个机制。时间片是CPU分配给每个线程的时间，因为时间片非常短，所以CPU通过不停的切换线程执行，让我们感觉是在同时运行，每个线程被分配的执行时间也就是几十毫米，然后有切换到其它线程上了。

对于java编程来说，多线程竞争**锁**时，会引起上下文切换。有些时候锁是无法避免的，但如果滥用锁，就好造成无用的切换。当你使用jstack命令查看线程状态的时候，你会发现大量的WAITTING状态线程，那就是CPU切换的原因，说明大量的线程闲的没事做，在WAITTING和RUNNING之前来回切换着玩，不干正事。

#### 1.1.2 查看切换的次数

当我们在linux下执行vmstat -Sm1监控系统运行情况的时候，你会发现--system--内的cs值在不断的变化，这个值就是上下文切换次数，次数越多说明切换的越多。

#### 1.1.3 如果减少上下文切换

无锁并发编程、CAS算法、使用少量的线程、使用协程。

确实在java中，CAS是无锁并发编程的基础，两者可以归类在一起。

使用少量的线程：这个要看具体的业务和硬件情况了，如果你服务器CPU就12个CPU，而且都是大CPU运算，而你非要启动500个线程，哪除了竞争锁之外CPU也没干什么正事。

协程：这个绝对是好东西，但JAVA目前是不原生支持的，java线程运行片段不能人为干预。这样做好处就是安全了，坏处就是你无法控制java线程hold cpu的时间。只能像netty那样，使用高端的无锁编程来劲量hold cpu。说到协程就提一句：lua编程语言，其原生支持协程，你可以手工编程来控制cpu的释放，这也就是为什么openresty的cpu利用率非常高，但上下文切换低的原因，其只有在等待IO的情况下才会让出cpu。压力测试openresty你会发现CPU都会被打满，这也就是为什么openresty都被应用在网关的原因，openresty绝对的好东西，但就是市场太小，推广不够。如果使用openresty+redis+mysql我相信在高并发的情况下，使用机器的数量会比java少很多。

### 1.2 死锁(deal lock)

A需要B的资源，B也需要A的资源，两者相互竞争，都无法获取到。

样例代码：

```java
private static Object A = new Object();
private static Object B = new Object();

Thread t1 = new Thread(new Runnable()){
    public void run(){
        synchronzied(A) {
            Thread.currenthread().sleep(2000);
        	synchronzied(B) {
            	System.out.println("1");
	        }            
        }
    }
}
Thread t2 = new Thread(new Runnable()){
    public void run(){
        synchronzied(B) {
          synchronzied(A) {
            System.out.println("2");
          }
        }
    }
}

t1.start();
Thread.currenthread().sleep(500);
t2.start();
```

分析问题：

t1先执行t2后执行：t1线程进入到A同步块，持有A锁，t2线程进入到B同步块，持有B锁。t2线程在执行B锁后，马上要获取A锁，但A锁已经被t1占用，无法获取只能等待。而t1线程2000ms后要获取B锁，但B锁已经被t2线程占用，两者都要获取对方的锁，这就产生的死锁。

解决问题：

先使用dump工具查看线程状态，你会发现locked字样，而且你还有看到两个线程相互等待。

避免同一个线程同时获取多个锁，特别是锁的嵌套。

使用tryLock(timeout)，绝对的好习惯，你应该预估你的锁时间，保证在单位时间内释放锁。

不要在锁内搞一些涉及到IO和其它资源的代码，因为一旦这些不可控制资源发生问题，你的整个锁就不会被释放。举个简单而又实战的例子：在锁内操作数据库，一旦你的SQL是慢SQL或者是网络IO、数据库任何一个地方出现问题，都会造成锁的不释放。而由于越积越多，直到雪崩。如果非要这样，请用tryLock(timeout)，保证在单位时间内一定释放锁。

java.util.concurrent包的实现理论基础：

首先，声明共享变量为volatile；

然后，使用CAS的原子条件更新来实现线程之间的同步；

同时，配合volatile的读/写和CAS所具有的volatile读和写的内存语义来实现线程之间的通信。

经过测试，可以参见"volatile.md"文章介绍：

volatile的写操作会把前面**所有**的普通共享变量的新值刷新到主内存。

volatile的读操作会从本地线程内存清除后面**所有**的普通共享变量，促使下面的代码在使用共享变量的时候从主内存中获取新值。

## volatile

### 基本理论

volatile可以在多个线程上保证共享变量的“可见性”，例如：声明了一个类属性变量volatile flag;，A线程修改flag=true，B线程读取boolean x=flag; 值马上可见。

### volatile对前后普通共享变量的影响

仔细观察如下的例子，重点在变量a上，a变量是一个普通共享变量，且没有声明volatile，如果A线程执行完writer()方法，B线程执行reader()方法，a变量是0还是1？

```java
class VolatileExample {
    int a = 0;
    volatile boolean flag = false;
   
    public void writer(){
        a = 1;
        flag = true;
    }
    
    public void reader(){
        if(flag){
            int i = a;
        }
    }
}
```

答案是a=1，你可能会问a没有volatile修饰，怎么B线程马上可见到A线程修改后的值呢？答案就在flag变量的读写对前后共享变量是有影响的，这就是JDK1.5版本后的语义加强，**当写一个volatile变量时，JMM会把该线程对应的本地内存中的所有共享变量值都刷新到主内存。当前读取一个volatile变量时，JMM会把该线程对应的本地内存置为无效，线程接下来将从主内存中读取共享变量。**这个理论也是这个java.util.concurrent包内lock实现的基石，我们看一下ReentrantLock的代码，就能明白为什么其没用java原生的synchronized也可以实现锁(lock)语义了，代码如下：

```java
	private volatile int state;
	
    // lock()方法内的核心代码
	final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();//读取volatile修饰的state变量值,触发JMM语义:会把该线程对应的本地内存置为无效，线程接下来将从主内存中读取共享变量。
        if (c == 0) { 
            if (compareAndSetState(0, acquires)) { 
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        ...
    }

    // unlock()方法内核心代码
    protected final boolean tryRelease(int releases) {
        int c = getState() - releases; 
        if (Thread.currentThread() != getExclusiveOwnerThread()) 
            throw new IllegalMonitorStateException(); 
        boolean free = false;
        if (c == 0) { 
            free = true;
            setExclusiveOwnerThread(null);
        }
        setState(c);//修改volatile修饰的state变量值, 触发JMM语义:把该线程对应的本地内存中的所有共享变量值都刷新到主内存。
        return free;
    }
```

观察上面的代码lock()的时候，getState()方法写到方法的最前面保证下面读取的所有共享变量都从主内存中获取，unlock()的时候，setState(c)写在方法的最后面保证上面所有对共享变量的修改都写刷新到主内存中。



## synchronized

### 加锁对象

synchronzied实现基础：Java中每一个对象都可以作为锁。具体表现：

1. 普通同步方法 ：锁是当前实例对象；
2. 静态同步方法：锁是当前类的class对象；
3. 同步方法块：锁是synchronzied括号里的对象；

当前一个线程视图访问同步代码块时，它首先必须得到锁，退出或异常时必须释放锁。

### 四种类型锁

Java SE 1.6中为了减少获取和释放锁带来的性能销毁而引入的偏向锁、轻量级锁：

**偏向锁**

研究发现大多数情况下，锁不存在多线程竞争，而且总是由一个线程多次获取，为了让线程获取锁的代价更低而引入了"偏向锁"。

偏向锁使用一种等到竞争才释放锁的机制，所有当前其他线程尝试竞争锁时，持有偏向锁的线程才会释放锁。

**轻量级锁**

如果成功，当前线程获取锁，如果失败，表示其他线程竞争锁，当前线程使用CAS自旋来获取锁。



在Java SE1.6中，锁一共有4种状态，级别从低到高：无锁状态、偏向锁状态、轻量级锁状态和重量级锁状态。

|    锁    |                             优点                             |                      缺点                      |            适用场景            |
| :------: | :----------------------------------------------------------: | :--------------------------------------------: | :----------------------------: |
|  偏向锁  | 加锁和解锁不需要额外的消耗，和执行非同步方法相比仅纳秒级差距 | 如果线程间存在锁竞争，会带来额外的锁撤销的消耗 |   只有一个线程访问同步块场景   |
| 轻量级锁 | 竞争的线程不会阻塞，提高程序的响应速度，使用CAS来竞争锁标志位 |    如果始终得不到锁的线程，使用CAS会消耗CPU    | 追求响应时间，同步块执行速度快 |
| 重量级锁 |                线程竞争不使用CAS，不会消耗CPU                |      线程阻塞，响应时间缓慢，上下文件切换      | 追求吞吐量，同步块执行时间较长 |
|          |                                                              |                                                |                                |

### synchronized内存语义

```java
class MonitorExample {
    int a=0;
    public synchronized void writer(){
        a++;
    }
    public synchronized void reader(){
        int i = a;
        ...
    }
 }
```

有两个线程，线程A执行writer()，随后线程B执行reader()方法。线程A在释放锁之前所有可见变量(第5步)，在线程B获取**同一个锁**(本类为this)之后(第7步)，将立即对B线程可见。

当线程释放锁时，JMM会把该线程对应的本地内存中的共享变量刷新到主内存中。上面这句话是“Java并发编程的艺术书”中的一句话，但我的理解这句话不准确，**准确的定义：synchronized(lock){ 涉及到共享变量 }，大括号结束后，都会刷新到主内存中。**

当线程获取锁时，JMM会把该线程对应的本地内存置为无效。从而使得被监视器保护的临界区代码必须从主内存中读取共享变量。上面这句话是“Java并发编程的艺术书”中的一句话，但我的理解这句话不准确，**准确的定义：JMM会把该线程执行到synchronized(lock){涉及到共享变量}对应的本地内存置为无效。从而使得被监视器保护的临界区代码必须从主内存中读取共享变量。**

## atomic

### 基本理论

原子(atomc)本意是不能被进一步分割的最小粒子，而原子操作，不可被中断的一个或一系列操作。

### CAS

典型的CAS(Compare And Swap)：CAS操作需要输入两个数值，一个旧值（期望操作前的值）和一个新值，在操作期间先比较旧值，如果没有发现变化，才交换为新值，发生了变化则不交换。

java实现CAS的核心方法：

```java
sun.misc.Unsafe类

public final native boolean compareAndSwapInt(Object o, long offset, int expected int x);
```



### 自旋

```java
private AtomicInteger atomicI = new AtomicInteger(0);

/** 自旋 */
private void safeCount() {
   for(;;){
     int i = atomicI.get();
     boolean suc = atomicI.compareAndSet(i,++i);
     if(suc){
       break;
     }
   }
}
```

## final

### 基本理论

这里的**域**，应理解为对象内共享变量。

final的java语义，被final声明的变量，必须在构造方法结束之前被初始化一个值，并且只能被初始化一次，并不能再被更改。也就是说final声明变量，只能是有三次被初始化的机会：

1.变量初始化，例如：final int a = 1;

2.构造方法，final int a;  public 构造方法() {a=1;}

3.构造块，final int a; {a=1;}

### final普遍变量多线程访问

例如：

```java
public class FinalExample {
    int i; // 普通域变量
    final int j; // final域变量 
    static FinalExample obj;
    public FinalExample(){
        i = 1;
        j = 2;
    }
    public static void writer(){
        obj = new FinalExample();
    }
    public static void reader(){
        int a = obj.i;
        int b = obj.j;
    }

}
```

从上面的例子，我们来分析，会出现什么问题？

线程A执行writer()，其会创建FinalExample对象并赋值给静态成员变量obj，而线程B执行reader()读取成员变量obj，并获取其内存的obj.i和obj.j，表面上看风平浪静没有事，线程B执行reader()应获取到a=1，b=2，但实际上则会出现a=0的情况，为什么，不是线程A已经调用writer()的new FinalExample()构造方法，已经给域变量i已经赋值为1了吗？

因为：**被构造对象的引用赋值给一个引用变量和普通域变量的初始化是两个动作，其不能保证是一个原子操作。**

这就造成了线程B，已经得到了FinalExample对象的引用obj，而线程A调用构造方法的i变量还没有被初始化完的情况，因此线程B获取到obj.i有可能是0的情况；而obj.j就不会出现这样的问题，因为j变量被final修饰了。

**final声明的域变量，其会保证被构造对象的引用赋值给一个引用变量和final域变量的初始化也是两个动作，但是一个原子操作。**

### final修饰的引用变量

final修饰的引用变量，和其内的域变量或者数组元素初始化都会受到final的语义保护，例如：

```java
public class FinalReferenceExample {
    final int[] intArray;
    static FinalReferenceExample obj;
    
    public FinalReferenceExample(){
        intArray = new int[1];
        intArray[0] = 1;
    }
    
    public static void writerOne(){
        obj = new FinalReferenceExample();
    }
    
    public static void writeTwo(){
        obj.intArray[0] = 2;
    }
    
    public static void reader(){
        if(obj != null){
            int tempI = obj.intArray[0];
        }
    }
}
```

例如上面的obj = new FinalReferenceExample();语句，引用赋值给一个引用变量obj和final域变量的初始化intArray = new int[1]; intArray[0] = 1;三个步骤是一个原子操作。



## happens-before

这是一个抽象的概念，java使用其来解释多线程执行的操作对共享变量的可见性问题：

例如：A happens-before B，那么解释为A先行发生于B，A操作的结果B会马上看到(可见性)。

JSR-133使用happens-before的概念来阐述操作之间的内存可见性。在JMM中，如果一个操作执行的结果需要对另一个操作可见，那么这两个操作之间必须要存在happens-before关系。这里提到的两个操作既可以是在一个线程内，也可以是不同线程之间。

规则如下：

**程序顺序规则：一个线程中的每个操作，happens-before于该线程中的任何后续操作。**

**监视器锁(synchronized)规则：对一个锁的解锁，happens-before于随后对这个锁的加锁。**

**volatile变量规则：对一个volatile域的写，happens-before于任意后续对这个volatile域的读。**

**传递性：如果A happens-before B，且B happens-before C，那么A happens-before C。**

**start()规则：如果线程A执行操作ThreadB.start()（启动线程B），那么A线程的ThreadB.start()操作happens-before于线程B的任意操作。**

**join()规则：如果线程A执行操作ThreadB.join()并成功返回，那么线程B中的任意操作happens-before于线程A从ThreadB.join()操作成功返回。**

注意：两个操作之间happens-before关系，并不意味这前一个操作必须要在后一个操作之前执行。happens-before仅仅要求前一个操作的执行结果，对后一个操作**可见**。

### 监视器(synchronized)规则

![](D:/github/zhangdberic/java/trunk/images/happens-before-synchronized1.jpg)

### volatile变量规则

![](D:/github/zhangdberic/java/trunk/images/happens-before-volatile1.jpg)



### start()规则

**线程A执行ThreadB.start()之前对共享变量的所做的修改，接下来线程B开始执行后都将确保对线程B可见。**

![](D:/github/zhangdberic/java/trunk/images/happens-before-threadstart1.jpg)

#### 解释servlet的线程可见性

**结合上图的线程A和线程B来分析**

tomcat主线程(线程A)初始化Servlet(调用servlet.init()方法)，其可以在init()方法中初始化共享成员变量。当http请求达到，tomcat分配http处理线程(线程B)，http处理线程运行service()方法，可以正确的获取主线程servlet.init()初始化的成员变量。

分析Spring的Controller线程可见性？

tomcat主线程(线程A)启动，调用DispatcherServlet.init()方法初始化了各种spring 单例Bean。当http请求达到，tomcat分配http处理线程(线程B)，调用Dispathcer.service()，其会实例化请求映射对应的Controller，同时Controller可以正确的使用DispatcherServlet.init()初始化的各种spring单例Bean。

### join规则()

 **线程A执行操作ThreadB.join()并成功返回后，线程B中的任意操作都将对线程A可见。**

![](D:/github/zhangdberic/java/trunk/images/happens-before-join1.jpg)

## 延时初始化和单实例

### 基于volatile实现

```java
public class SafeDoubleCheckedLocking {
   private volatile static Instance instance;
   if(instance==null){
       synchronized(SafeDoubleCheckedLocking.class){
           if(instance==null){
               instance = new Instance();
           }
       }
   }
}
```

### 基于类初始化解决方案

JVM在类的初始化阶段（即在Class被加载后，且被线程使用前），会执行类的初始化。在执行初始化期间，JVM会去获取一个锁。这个锁可以同步多个线程对同一个类的初始化。

```java
public class InstanceFactory {
    private static class InstanceHolder {
        public static Instance instance = new Instance();
    }
    public static Instance getInstance() {
        return InstanceHolder.instance();
    }
}
```

## thread

### 基础理论

在一个进程里可以创建多个线程，这些线程都拥有各自的技数器、堆栈和局部变量等属性，并且能够访问共享的内存变量。处理器在这些线程上高速切换，让使用者感觉到这些线程在同时执行。

### ~~线程优先级~~

Java线程中，优先级范围1-10，默认为5。java的优先级设置是无用的，底层的操作系统不会java设置优先级来为线程多分配执行时间。

### 线程状态

Java线程在运行的生命周期中可能处于以下几种状态：

NEW 初始化状态，线程被构建( new Thread() )，但还没有调用start()方法；

RUNNABLE 运行状态，Java线程将操作系统中的就绪和运行两种状态统称为"运行中"；

BLOCKED 阻塞状态，线程阻塞于锁；    例如：线程等待获取锁synchronzied(xxx){}，但现在锁被其它线程占用；

WAITING 等待状态，表示线程进入等待状态，等待其它线程唤醒或中断； 例如：object.wait();

TIME_WAITING 超时等待，区别于WAITING，其可在指定的时间自动返回； 例如：Thread.sleep(xxxx);

TERMINATED 终止状态，表示当前线程已经执行完毕；

我们可以使用jstack工具查看JVM内线程的状态；

**线程状态变迁**

通过下面的图，我们了解到，不同的方法调用(操作)，可以使线程转化到不同的状态；

线程创建之后，调用start()方法开始运行(RUNNING)。当线程执行到wait()方法之后线程进入等待状态(WAITING)。进入到等待状态的线程需要依靠其他线程的通知才能返回到运行状态(RUNNING)，而超时等待状态(TIMED_WAITING)的基础上增加了超时限制，也就是超时时间达到时会返回到运行状态(RUNNING)。当前线程调用同步方法时，在没有获取到锁的情况下，线程会进入到阻塞状态(BLOCKED)。线程在执行Runnable的run()方法之后将会进入到终止状态(TERMINATED)。

![](D:/github/zhangdberic/java/trunk/images/thread_status1.jpg)

### Daemon(守护)线程

可以通过Thread.setDaemon(true)线程设置为守护(Daemon)线程；为了区别守护线程，我们把没有设置为Daemon的线程称为普通线程。当前JVM中不存在普通线程的时候，不管当前是否还有守护线程在运行，JVM将会退出。

Daemon线程随时会中断，不能依靠finally块中的内容来确保执行关闭或清理资源逻辑。

###  线程中断

中断可以理解为线程的标示位属性，其它线程对该线程打招呼，其它线程通过调用该线程的interrupt()方法通知你可以停止你的线程了，至于是否真的中断只能由运行线程自己来决定。

线程通过方法isInterrupted()来进行判断是否该线程已经被其它线程设置为中断。

许多声明抛出InterruptedException的方法（例如 Thread.sleep(long mills)方法）这些方法在抛出InterruptedException之前，Java虚拟机会先将该线程中断标识位清除，然后再抛出InterruptedException，此时调用isInterrupted()返回false。

### 安全的终止线程

如果要安全的终止线程，最好是通过**标志位+isInterrupted()**判断来安全的退出检查。

```java
private static class Runner implements Runnable {
    private volatile boolean on = true;
    @Override
    public void run(){
        while(this.on && Thread.currentThread().isInterrupted()){
            i++;
        }
    }
    public void cannel(){
        this.on = false;
    }
}


```

这样外部线程既可以通过调用cannel()来设置标志终止线程，也可以使用interrupt()来中断线程。

### 线程通信

**通过使用synchronized和volatile关键字**

关键字volatile：可以用来修饰成员变量，就是告知程序任何对该变量的访问均需要从内存中获取，而对的改变也会同步刷新回主内存，它能保证线程对变量的可见性。

关键字synchronzied：可以修饰方法和同步块，它主要确保多个线程在同一个时刻，只能有一个线程处于方法或者同步块中，它保证了线程变量访问的可见性和排他性。

### 等待(wait)/通知(notify)

**经典范式**

等待方遵循如下原则：

1) 获取对象的锁。

2) 如果条件不满足，那么调用对象的wait()方法等待，被通知( notify() )后仍要检查条件。

3) 条件满足则执行对应的逻辑。

```java
synchronized(对象) {
    while(条件不满足) {
        对象.wait();
    }
    对应的处理逻辑代码;
}
```

通知方遵循如下原则：

1) 获取对象的锁。

2) 改变条件。

3) 通知所等待在对象上的线程。

```java
synchronized(对象) {
    改变条件;
    对象.notifyAll();或 对象.notify();   
}
```

wait()和notify()以及notifyAll()是需要注意：

1) 使用wait()、notify()和notifyAll()是需要先对调用的对象加锁。

2) 调用wait()方法后，线程状态由RUNNING变为WAITING，并将当期线程放置到对象的等待队列。

3) notify()或notifyAll()方法调用后，等待线程依旧不会从wait()返回，需要调用notify()和notifyAll()的线程释放锁之后，等待线程才有机会从wait()返回。

4) notify()方法将等待队列中的一个等待线程从等待队列中移动同步队列中，而notifyAll()方法则将等待队列中所有的线程全部移动同步队列，被移动的线程状态由WAITING变为BLOCKED。

5) 从wait()方法返回的前提是获得了调用对象的锁。

### thread.join()

调用线程等待被调用线程的执行完成，例如：A线程调用了b.start()开启运行B线程，A线程需要等待B线程执行完，这里A线程需要调用b.join()阻塞等待B线程运行完成。经典的模型就是多线程并行计算，计算后主线程累计多个线程的计算结果，这里主线程就要join计算子线程。

### 线程变量(ThreadLocal)

ThreadLocal线程变量，用于同一个线程不同方法之间的值传递。

#### 使用

实例化：可以声明为静态类变量和对象实例变量，这个无所谓。

```
private static final ThreadLocal<String> testThreadLocal = new ThreadLocal<String>();
```

设置值：

```
testThreadLocal.set("123");
```

取值：

```
testThreadLocal.get("123");
```

但用过后一定要清除（最好放在finally中）：

```
testThreadLocal.remove();
```

#### 原理说明

```
ThreadLocal<T> tl = new ThreadLocal<T>();
```

每个线程(**Thread**)对象内维护了一个**ThreadLocal.ThreadLocalMap threadLocals**属性，其存放上面创建的线程变量实例，这个ThreadLocalMap对象类似于实现了Map，其key为**ThreadLocal对象(例如：上面的tl)**的哈希值，其value为执行set方法的value参数。因此你new了多少个ThreadLocal();就有多少个线程变量，如下示例：

ThreadLocal threadLocal1 = new ThreadLocal();

threadLocal1.set("123");

ThreadLocal threadLocal2 = new ThreadLocal();

threadLocal2.set("456");

ThreadLocal threadLocal3 = new ThreadLocal();

threadLocal1.set("789");

thread--->ThreadLocalMap

 key=threadLocal1对象的哈希值(例如：1)，value="123"

 key=threadLocal2对象的哈希值(例如：2)，value="456"

 key=threadLocal3对象的哈希值(例如：3)，value="789"

threadLocal1.get();还是通过ThreadLocal内的哈希算法，得到当前对象“threadLocal1对象的哈希值(例如：1)”，然后获取对应的值就可以了。

#### 注意

因为当前线程池的大量使用，线程使用后会被复用，因此线程变量(ThreadLocal)使用后，应在线程业务结束的位置，清除的线程变量(threadLocal.remove())，以免错误和内存泄露。

**InheritableThreadLocal**

前面提到了ThreadLocal是线程变量，其只是当前线程的线程变量，如果当前线程要创建子线程，那么就要使用InheritableThreadLocal把当前线程的线程变量传递到子线程中。

```java
public class Test {

    public static ThreadLocal<Integer> threadLocal = new InheritableThreadLocal<Integer>();

    public static void main(String args[]) {
        threadLocal.set(new Integer(456));
        Thread thread = new MyThread();
        thread.start();
        System.out.println("main = " + threadLocal.get());
    }

    static class MyThread extends Thread {
        @Override
        public void run() {
            System.out.println("MyThread = " + threadLocal.get());
        }
    }
}
```

MyThread线程可以正确的获取到456，而如果你把InheritableThreadLocal修改为ThreadLocal，那么MyThread线程不能获取到线程变量。



## Lock

Lock接口和synchronized关键字的区别

1.尝试非阻塞的获取锁，当前线程尝试获取锁，如果获取不到则直接返回false。

2.获取锁的过程能被中断，与synchronzied不同，获取锁的过程能过响应中断。

3.超时获取锁，在指定的时间内获取锁，获取不到则返回。

### Lock接口

lock太常用了，建议把下面的方法都背下来

| 方法名称                                                     | 描述                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| void lock()                                                  | 获取锁                                                       |
| void lockInterruptibly() throws InterruptedException         | 获取锁的过程可中断                                           |
| boolean tryLock();                                           | 尝试获取锁，调用该方法后立即返回，能获取到则返回true,不能返回false。 |
| boolean tryLock(long timeout,TimeUnit unit) throws InterruptedException; | 尝试在timeout时间内获取锁，如果不能获取则当前线程会被阻塞timeout时间，阻塞的时间内可以响应中断。 |
| void unlock();                                               | 释放锁，应在finally{unlock();}中执行，保证锁被释放。         |
| Condition newCondition();                                    | 创建一个等待通知组件，同线程的wait()和notify()，但使用更简单。 |
|                                                              |                                                              |
|                                                              |                                                              |

### ReentrantLock(可重入锁)

注意：lock()方法，应该在try的上面，unlock()方法在finally中包装锁任何情况下释放锁。如果你使用ReentrantLock同synchronized语义一样(阻塞等待)，建议还是使用synchronized，因为synchronized优化的空间更大(jdk团队对synchronzied在进行持续的优化)。

```java
class ReentrantLockExample {
    int a = 0;
    final ReentrantLock lock = new ReentrantLock();
    
    public void write(){
        lock.lock();
        try{
            a++;
        }finally{
            lock.unlock();
        }
    }
    
    public void reader(){
        lock.lock();
        try{
            int i=a;
        }finally{
            lock.unlock();
        }

    }
}
```

#### 可重入

线程A已经调用lock()方法获取到了锁，如果线程A再次调用lock()则**非阻塞**直接进入锁，代码：

其内部有一个计数器，重入一次加1，退出一次减1，计数器为0，则释放当前线程持有的锁。

```java
    final ReentrantLock lock = new ReentrantLock();
    
    public void write(){
        lock.lock(); // 线程A与其他线程竞争获取锁
        try{
            a++;
            lock.lock(); // 线程A直接进入(可重入),不会发生阻塞
            try{
                a++;
            }finally{
                lock.unlock();
            }
        }finally{
            lock.unlock();
        }
    }
```

#### 公平锁和非公平锁

公平锁：在等待时间内最长的线程最优先获取锁。

非公平锁：线程争夺获取到锁，谁能获取到谁获取。默认情况

公平锁没有非公平说效率高。但是，并不是任何场景都以TPS为唯一指标，公平锁能减少”饥饿“发生的概率，等待越久的线程越能够优先获取到锁。

### ReentrantReadWriteLock(读写锁)

读写锁在同一个时刻可以允许多个读线程访问(共享锁)，但是写线程访问(独占锁)时所有的读线程和其他写线程均被阻塞。

读写锁维护了一对锁，一个读锁，一个写锁，通过分离读锁和写锁，提升并发性。

```java
public class Cache {
    static Map<String,Object> map = new HashMap<String,Object>();
    static ReentrantReadWriteLock rw1 = new ReentrantReadWriteLock();
    static Lock r = rw1.readLock();
    static Lock w = rw1.writeLock();
    
    public static final Object get(String key){  // 读线程同时访问(共享锁)
        r.lock();
        try{
            return map.get(key);
        }finally{
            r.unlock();
        }
    }
    
    public static final void set(String key,Object value){ // 写线程独占(独占锁)
        w.lock();
        try{
            map.put(key,value);
        }finally{
            w.unlock();
        }
    }
    
    public static final void remove(String key){ // 写线程独占(独占锁)
        w.lock();
        try{
            map.remove(key);
        }finally{
            w.unlock();
        }
    }
}
```

### 等待/通知

基于ReentrantLock和Condition实现，等待/通知，和Java原生的wait()和notify()使用上差不多。

注意：只要是由wait语义的地方，就应该使用while来循环条件，不能使用if。

```java
final Lock lock = new ReentrantLock();
Condition condition1 = lock.condition();

public void await(){
    lock.lock();
    try{
        while(条件不满足) {
           condition1.await();
        }
    }finally{
        lock.unlock();
    }
}

public void signal(){
    lock.lock();
    try{
        condition1.signal(); // 通知，去唤醒等待的某个线程
        // condition1.signalAll(); // 通知，去唤醒等待的所有线程
    }finally{
        lock.unlock();
    }   
}
```



## 调用超时(call timeout)

下面是一个简单的模式，基于单线程，如果使用比较频繁的情况下，可以使用线程池来实现。

```java
public class WaitTimeout {

	public <T> T run(Callable<T> callable,final long timeout) {
		ExecutorService executor = Executors.newSingleThreadExecutor();
		Future<T> future = executor.submit(callable);
		try {
			System.out.println(Thread.currentThread()+"timeout:"+timeout);
			return future.get(timeout, TimeUnit.MILLISECONDS);
		} catch (InterruptedException ex) {
			future.cancel(true);
			throw new RuntimeException(ex);
		} catch (ExecutionException ex) {
			future.cancel(true);
			throw new RuntimeException(ex);
		} catch (TimeoutException ex) {
			future.cancel(true);
			throw new RuntimeException(ex);
		} finally {
			executor.shutdownNow();
		}
	}

	public static void main(String[] args) {
		System.out.println("execute success.");
		new WaitTimeout().run(new Callable<Boolean>() {
			public Boolean call() throws Exception {
				Thread.sleep(1000);
				return true;
			}
		},1200);
		
		System.out.println("execute failure.");
		boolean result = new WaitTimeout().run(new Callable<Boolean>() {
			public Boolean call() throws Exception {
				System.out.println(Thread.currentThread());

				Thread.sleep(1000);
				return true;
			}
		},900);
		System.out.println(result);
		
	}

}
```

## ConcurrentHashMap

ConcurrentHashMap是线程安全且高效的HashMap。

新的jdk1.8的ConcurrentHashMap实现，相对于jdk1.6的ConcurrentHashMap有很大的改进，例如：

Jdk1.6的Segment已经不再被使用，取而代之的是更高效的Node级别锁，操作那个Node(元素)，就对那个元素加锁。

HashEntry不存在了，取而代之的是Node。

哈希碰撞的时候，不在单纯的使用链表结构来存储相同hash值的元素了，而是使用链表或红黑树，一旦链表长度大于等于8马上转换为红黑树结构，链表用于小数据查询，红黑树用于大数据查询。

大量的使用volatile和sun.misc.Unsafe来操作table(数组)内的Node数据。

不在使用ReentrantLock而是使用synchronized来对Node加锁，synchronized优化力度更大，锁定Node锁定范围更小。

### hash定位

计算key存放到数组中的位置。

下面的代码摘取之ConcurrentHashMap的putVal方法，

n为数组(table)长度；

hash为spread(int h)方法计算后的hash值(下面会讲解)；

i为计算后数组的索引位置；

很明显通过&运算(两个操作数中，二进制位都为1，结果才为1，否则结果为0)，这样能保证计算后的值一定在(n-1)的数值范围内。

```java
i = (n - 1) & hash
```



### hash计算(再散列)

实现方式如下：

```
static final int spread(int h) {
    return (h ^ (h >>> 16)) & HASH_BITS;
}
```

为了避免不太好的Key的hashCode值(散列不够)，它通过如上方法重新计算得到Key的最终哈希值。不同的是，Java 8的ConcurrentHashMap作者认为引入红黑树后，即使哈希冲突比较严重，寻址效率也足够高，所以作者并未在哈希值的计算上做过多设计，只是将Key的hashCode值与其高16位作异或并保证最高位为0（从而保证最终结果为正整数）。

**解释一下上面的代码**

核心思想，作者认为，**绝大多数情况下哈希表长度(n)一般都小于2^16即小于65536，如果小于65536那么n的高16位一定是零**,执行上面的**hash定位[i = (n - 1) & hash]**后不管你hash值是什么，通过&运算后高16位还是0，hash值的高16位被浪费了，其没有参与到hash计算，上面的**再哈希**方法，目的就是让hash的高16位也参与到计算来，尽量减少哈希碰撞。

h >>> 16，无符号右移16位，前面补零，意味着丢弃h的低16位，只保留h的高16位。

例如：h等于12345678，二进制(0000 0000 1011 1100 0110 0001 0100 1110)，无符号右移16位，前面补零后值为188，二进制(0000 0000 0000 0000 0000 0000 1011 1100)。

12345678：0000 0000 1011 1100 0110 0001 0100 1110

​           188：0000 0000 0000 0000 0000 0000 1011 1100

h ^ (h >>> 16)，h和自己高16位异或(相等位值则为0，不等位为1)，这样高16后也参与到了计算，例如：h等于12345678。

12345678：0000 0000 1011 1100 0110 0001 0100 1110

​           188：0000 0000 0000 0000 0000 0000 1011 1100

12345842：0000 0000 1011 1100 0110 0001 1111 0010  异或后的值

(h ^ (h >>> 16)) & HASH_BITS，这里的HASH_BITS为0x7fffffff，计算后为无符号正整数，因为第1位为0，**与运算**后也一定是0，第1位为零就是无符号正整数。

### 碰撞后是链表存储还是红黑树存储

 链表内的元素个数大于等于8时，转换结构到红黑树。

### 锁

put、replace操作对操作的Node加锁(synchronized)，get、constant操作无锁，remove操作对操作的Node加锁(synchronized)。

### size()和isEmpty()

isEmpty()实现基于size()相对于size()>0没有任何优势，size()在jdk1.8中已经是已经没有锁了，统计很快。



## Queue

### BlockingQueue(阻塞队列)

阻塞队列常用于生产者和消费者的场景，生产者是向队列里添加元素的线程，消费者是从队列里取元素的线程。阻塞队列就是生产者用来存放元素、消费者用来获取元素的容器。

当队列满或者队列为空时，下表是队列操作方法，处理方式： 

| 方法\处理方式 | 抛出异常  | 返回特殊值 | 一直阻塞 | 超时退出           |
| ------------- | --------- | ---------- | -------- | ------------------ |
| 插入方法      | add(e)    | offer(e)   | put(e)   | offer(e,time,unit) |
| 移除方法      | remove()  | poll()     | take()   | poll(time,unit)    |
| 检查方法      | element() | peek()     | 不可用   | 不可用             |

- 抛出异常：是指当阻塞队列满时候，再往队列里插入元素，会抛出IllegalStateException(“Queue full”)异常。当队列为空时，从队列里获取元素时会抛出NoSuchElementException异常 。
- 返回特殊值：插入方法会返回是否成功，成功则返回true。移除方法，则是从队列取出一个元素，如果没有则返回null
- 一直阻塞：当阻塞队列满时，如果生产者线程往队列里put元素，队列会一直阻塞生产者线程，直到队列可用或者响应中断退出。当队列空时，如果消费者线程试图从队列里take元素，队列也会阻塞消费者线程，直到队列可用。
- 超时退出：当阻塞队列满时，队列会阻塞生产者线程一段时间，如果超过指定的时间，生产者线程就会退出。

**ArrayBlockingQueue(数组实现有界阻塞队列)**

ArrayBlockQueue是一个用数组实现的有界阻塞队列。此队列按照先进先出(FIFO)的原则对元素进行排序。

默认是线程非公平性访问，如果要创建一个公平性访问队列，如下：

```java
ArrayBlockingQueue fairQueue = new ArrayBlockingQueue(1000,true);
```

**LinkedBlockingQueue(链表实现有界阻塞队列)**

LinkedBlockingQueue是一个用链表实现的有界阻塞队列。此队列的默认和最大长度为Integer.MAX_VALUE。此队列按照先进先出(FIFO)的原则对元素进行排序。

**PriorityBlockingQueue(优先级的无界阻塞队列)**

存储队列的元素采用自然升序排序，也可以使用自定义的compareTo()方法指定元素排序规则，或者在初始化PriorityBlockingQueue时，指定构造参数Comparator来进行排序。需要注意：不能保证同有效级元素的顺序。这个PriorityBlockingQueue实现了多个生产者线程并发存放无序的元素，但进入队列后就变成有序的了，消费者可以顺序的从队列中获取元素。

**DelayQueue(延时的无界阻塞队列)**

队列中的元素必须实现Delayed接口，其实现了getDelay()方法，该方法返回当前元素还需要延时多长时间，单位是纳秒。还是需要实现compareTo(Delayed other)方法来指定元素的顺序，例如：让延时最长的排在队列的末尾。

其典型的用法：

1.延时处理程序，生产者存放实现Delayed接口元素到队列，消费者阻塞等待延时到期后取出元素执行内部方法；

2.缓存过期处理，创建缓存对象时，包装缓存对象实现Delayed接口并存放到队列，缓存销毁消费者阻塞等待过期的缓存对象；

**SynchronousQueue(不存储元素的阻塞队列)**

每一个put操作必须等待一个take操作，否则不能继续添加元素。中间没有介质来存放队列数据。

**LinkedTransferQueue(适用于传送数据的队列)**

多了两个方法：

transfer方法判断消费者是否可用如果可用则直接把元素传送到给消费者，否则把元素存放到队列介质中阻塞等待消费者有空来消费。

tryTransfer方法区别于transfer方法无论消费者是否接收，方法立即返回，而transfer方法是必须等待到消费者消费才返回。

**LinkedBlockingDeque(双向链表阻塞队列)**

可以从队列的两端插入和移除元素，对比上面的阻塞队列其多了一个操作队列入口，在多线程同时入队的时候，也就减少了一半的竞争。

如果要实现双向队列，最好使用xxxFirst，xxxLast字样方法操作，其实现了BlockingQueue接口的所有语义，例如：

addFirst、addLast，offerFirst、offerLast，peekFirst、peekLast等方法，以first结尾的方法，表示插入、获取或者移除在队列的头部操作，而反之last结尾的方法在队列的尾部操作。

典型用法：

正常情况下生产者put到元素到队列，消费者从队列中take元素，都是按照时间顺序完成。但突然间来了一个重要的元素，需要优先处理，则可以使用addLast放到队列尾部，这个元素就会优先被take出并处理。





