# ThreadLocal

ThreadLocal线程变量，用于同一个线程不同方法之间的值传递。



## 使用

实例化：可以声明为静态类变量和对象实例变量，这个无所谓。

```java
ThreadLocal<String> testThreadLocal = new ThreadLocal<String>();
```

设置值：

```java
testThreadLocal.set("123");
```

取值：

```java
testThreadLocal.get("123");
```

但用过后一定要清除（最好放在finally中）：

```java
testThreadLocal.remove();
```

## 原理说明

```java
ThreadLocal<T> tl = new ThreadLocal<T>();
```

每个线程(**Thread**)对象内维护了一个**ThreadLocal.ThreadLocalMap threadLocals**属性，其存放上面创建的线程变量实例，这个ThreadLocalMap对象类似于实现了Map，其key为**ThreadLocal对象(例如：上面的tl)**的哈希值，其value为执行set方法的value参数。因此你new了多少个ThreadLocal<T>();就有多少个线程变量，如下示例：

ThreadLocal<T> threadLocal1 = new ThreadLocal<T>();

threadLocal1.set("123");

ThreadLocal<T> threadLocal2 = new ThreadLocal<T>();

threadLocal2.set("456");

ThreadLocal<T> threadLocal3 = new ThreadLocal<T>();

threadLocal1.set("789");



thread--->ThreadLocalMap

​                                               key=threadLocal1对象的哈希值(例如：1)，value="123"

​                                               key=threadLocal2对象的哈希值(例如：2)，value="456"                                      

​                                               key=threadLocal3对象的哈希值(例如：3)，value="789"                              



threadLocal1.get();还是通过ThreadLocal内的哈希算法，得到当前对象“threadLocal1对象的哈希值(例如：1)”，然后获取对应的值就可以了。

## 注意

因为当前线程池的大量使用，线程使用后会被复用，因此线程变量(ThreadLocal)使用后，应在线程业务结束的位置，清除的线程变量(threadLocal.remove())，以免错误和内存泄露。

