# volatile

volatile理解为轻量级的synchronized，他在多处理器开发中保证了共享变量的"可见性"。可见性：当前一个线程修改一个变量时，另一个线程马上能读取到这个修改值。volatile修饰变量比synchronized成本低，它还不会引起线程上下文件切换。

底层原理：

通过查看代码的汇编反编译，可以见到有一个lock前缀，例如：

```
0x01a3deld: movb $0*0,0*1104800(%esi);0x01a3de24: lock addl $0*0,(%esp);
```

lock前缀指令在多核CPU下会发生两件事：

1. 将当前处理器缓存行的数据写回到系统内存；
2. 这个写回内的操作会使用其他CPU里缓存了该内存地址的数据无效；

## volatile内存语义

可见性：对一个volatile变量的读，总能看到(任意线程)对这个volatile变量最后的写入。

原子性：对任意单个volatile变量的读/写具有原子性，但类似于volatile++这样复合操作不具有原子性。

volatile写一个volatile变量时，JMM会把该线程对应的本地内存中的共享变量值刷新到主内存中。

当读一个volatile变量时，JMM会把该线程对应的本地内存置为无效。线程接下来将从主内存中读取共享变量。

## volatile声明对象引用

volatile只能保护声明变量引用的可见性，如果volatile声明的数组，不能保证数组元素的可见性，如果需要可以使用AtomicReferenceArray。如果volatile声明成员变量，不能保证成员变量的可见性，如果需要，可以为成员变量也声明为volatile或final。



## volatile可见性测试

### 1.有无volatile声明变量的区别

```java
package test;

public class NoVisibility {
    // private volatile static boolean ready;
    private static boolean ready;
    private static int number;

    private static class ReaderThread extends Thread {
        public void run() {
        	long i=0;
            while (!ready){i++;} // 测试ready内存可见性
            System.out.println(number);
        }
    }

    public static void main(String[] args) throws InterruptedException {
        new ReaderThread().start();
        number = 42;
        Thread.sleep(1000);
        ready = true;
   }
}
```

注意：使用jvm的-server参数启动程序，否则无法测出效果。测试成功环境：jdk1.6_20。

### 1.2 测试操作volatile变量对前后普通变量可见性的影响

```java
package test;

public class TestCAS {
	private volatile static boolean ready; // 声明为volatile变量
	private static int number = 0;
	private static int a = 0;

	private static class ReaderThread extends Thread {
		public void run() {
			long i = 0;
			int flag = 0;
			while (true) {
				i++;
				if (number == 42 && a == 41) {
					flag = 1; // 测试操作volatile变量前普通变量可见性
					break;
				}
				boolean x = ready; // 操作volatile变量，测试对前后普通变量的可见性影响
				if (number == 42 && a == 41) {
					flag = 2; // 测试操作volatile变量后普通变量可见性
					break;
				}
			}
			System.out.println(flag); 
		}
	}

	public static void main(String[] args) throws InterruptedException {
		new ReaderThread().start();
		for (long j = 0; j < 2000000000; j++) { // 保证ReaderThread线程已经运行
			int x = 1;
		} 
		number = 42;
		a = 41;
		ready = true; // 操作volatile变量的写操作，同步刷新前面的所有普通变量到主内存
	}

}

```

这个例子非常好，足足花费了4个小时的时间不断的调试代码。

注意：使用jvm的-server参数启动程序，否则无法测出效果。测试成功环境：jdk1.6_20。

上面的代码会正常结束，而且输出2。

**因为number=42;和a=41;的代码行，在ready=true的前面，而ready是volatile变量，对ready变量的写操作会保证其前面的普通共享变量马上刷新到主内存(JMM模型)。而另一个线程的boolean x = ready代码行，对ready变量的读操会清除线程的本地内存中的共享变量。**

**通过上面的测试，证明如下：**

volatile的写操作会把前面**所有**的普通共享变量的新值刷新到主内存。

volatile的读操作会从本地线程内存清除后面**所有**的普通共享变量，促使下面的代码在使用共享变量的时候从主内存中获取新值。

如果你把a = 41;代码行放在ready = true;的后面，System.out.println(flag); 输出的大多时候是1，因为volatile只能保证前面的普通共享变量**马上**属性到主内存，尽管其也会影响到后面的共享变量刷新到主内存，但没有那么及时了。

### 1.3 JUC的锁内存可见性实现

通过上面的测试，证明如下：

volatile的写操作会把前面**所有**的普通共享变量的新值刷新到主内存。

volatile的读操作会从本地线程内存清除后面**所有**的普通共享变量，促使下面的代码在使用共享变量的时候从主内存中获取新值。

**这也就是juc相关代码包，不使用synchronized，使用java代码就也可以实现共享变量内存可见性的原因。例如：ReentrantLock、FairSync等。**，其使用的就是上面的特性：

我们来看看标准的ReentrantLock的编码，如果保证共享变量a的可见性：

```java

public class Test1 {
	static int a = 0;
	static ReentrantLock lock = new ReentrantLock();
	
	public static void writer(){
		lock.lock();
		try{
			a++;
		}finally{
			lock.unlock();
		}
	}
	
	public static void reader(){
		lock.lock();
		try{
			int i = a;
		}finally{
			lock.unlock();
		}
	}

}

```

lock.lock()方法的核心实现为nonfairTryAcquire方法，如下：

int c = getState()代码执行了state的volatile变量读操作，其会保证其后的执行代码，遇到共享变量会先从主内存中获取。

```java
    private volatile int state;        

    protected final int getState() {
        return state;
    }

        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();  // volatile变量(state)读操作(放在方法开头)
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```

lock.unlock()方法的核心实现为tryRelease方法，如下：

setState(c);代码执行了state的volatile变量写操作，其会保证其前面执行代码，遇到共享变量会刷新输出到主内存中。

```java
        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);  // volatile变量(state)写操作(放在方法结尾)
            return free;
        }
```

}

