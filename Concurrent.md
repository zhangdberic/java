# Concurrent

java.util.concurrent包的实现理论基础：

首先，声明共享变量为volatile；

然后，使用CAS的原子条件更新来实现线程之间的同步；

同时，配合volatile的读/写和CAS所具有的volatile读和写的内存语义来实现线程之间的通信。

经过测试，可以参见"volatile.md"文章介绍：

volatile的写操作会把前面**所有**的普通共享变量的新值刷新到主内存。

volatile的读操作会从本地线程内存清除后面**所有**的普通共享变量，促使下面的代码在使用共享变量的时候从主内存中获取新值。

## 1.Lock

Lock接口和synchronized关键字的区别

1.尝试非阻塞的获取锁，当前线程尝试获取锁，如果获取不到则直接返回false。

2.获取锁的过程能被中断，与synchronzied不同，获取锁的过程能过响应中断。

3.超时获取锁，在指定的时间内获取锁，获取不到则返回。

### 1.1 Lock接口

lock太常用了，建议把下面的方法都背下来

| 方法名称                                                     | 描述                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| void lock()                                                  | 获取锁                                                       |
| void lockInterruptibly() throws InterruptedException         | 获取锁的过程可中断                                           |
| boolean tryLock();                                           | 尝试获取锁，调用该方法后立即返回，能获取到则返回true,不能返回false。 |
| boolean tryLock(long timeout,TimeUnit unit) throws InterruptedException; | 尝试在timeout时间内获取锁，如果不能获取则当前线程会被阻塞timeout时间，阻塞的时间内可以响应中断。 |
| void unlock();                                               | 释放锁，一般在finally{unlock();}中执行，保证锁被释放。       |
| Condition newCondition();                                    | 创建一个等待通知组件，同线程的wait()和notify()，但使用更简单。 |
|                                                              |                                                              |
|                                                              |                                                              |

### 1.1.1 ReentrantLock(可重入锁)

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

#### 1.1.1.1 可重入

线程A已经调用lock()方法获取到了锁，如果线程A再次调用lock()则非阻塞直接进入锁，代码：

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

#### 1.1.1.2 公平锁和非公平锁

公平锁：在等待时间内最长的线程最优先获取锁。

非公平锁：线程争夺获取到锁，谁能获取到谁获取。默认情况

公平锁没有非公平说效率高。但是，并不是任何场景都以TPS为唯一指标，公平锁能减少”饥饿“发生的概率，等待越久的线程越能够优先获取到锁。

### 1.1.3 ReentrantReadWriteLock(读写锁)

读写锁在同一个时刻可以允许多个读线程访问，但是写线程访问时，所有的读线程和其他写线程均被阻塞。

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

#### 1.1.4 等待/通知

基于ReentrantLock和Condition实现，等待/通知，其比Java自带的等待/通知实现简单多了。

```java
final Lock lock = new ReentrantLock();
Condition condition1 = lock.condition();

public void await(){
    lock.lock();
    try{
        System.out.println("wait start.")
        condition1.await(); // 进入阻塞等待，待通知唤醒
        System.out.println("wait end.")
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



## 3.Concurrent的应用

### 3.1 等待超时模式

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

