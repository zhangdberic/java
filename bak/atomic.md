# atomic

# 1.基本原理

原子（Atomic）本意 ”不能被进一步分割的最小粒子”，而且原子操作（atomic operation）意为“不可以被中断的一个或一系列操作”。

CPU指令集，就提供了atomic的相关指令，来保证一些操作的原子性，最典型的就是CAS（Compare and Swap）即比较和交换，CAS操作需要输入两个值，一个旧值（期望操作前的值），和一个新值，在操作前先比较旧值有没有变化，如果没有变化，才交换为新值，发生变化则不交换，注意这个比较和交换的过程的原子的。

java伪代码：

```java
private int value;

public synchronized boolean cas(int currentValue,int newValue){
    if(value==currentValue){
        value=newValue;
        return true;
    }else{
        return false;
    }
}
```

上面实现了CAS的语义并且能保证线程安全，但执行效率和消耗上和java原生的AtomicInteger没法比。

JAVA中的Atomic相关类内部基本都声明了一个volatile的变量，底层的CAS操作都是通过sun包下Unsafe类实现，而Unsafe类中的方法都是native方法，其是基于CPU的CAS指令来完成的。

## 2. Java如何实现原子操作

例如：加1操作（尽管AtomicInteger提供了原生的加1操作，下面就是为了举例子）

```java
AtomicInteger count = new AtomicInteger();

private void safeCount(){
    for(;;){
        int i = count.get();
        boolean suc = atomic.compareAndSet(i,++i);
        if(suc){
            break;
        }
    }
}

```

以上的代码实现了原子加1操作，其是线程安全的，如果产生竞争只会消耗CPU。

如果两个线程同时进入到safeCount的for循环，线程1执行count.get()获取到0，线程2执行count.get()也获取到0，线程1执行atomic.compareAndSet(0,1);成功退出循环，实现了加1操作，线程2执行atomic.compareAndSet(0,1);失败（因为当前值已经是1了），则需要重新循环。

这篇文章，对java的atomic相关类，解释的比较全：https://segmentfault.com/a/1190000021155407

## 3. CAS带来的问题

#### 3.1 ABA问题

因为CAS需要在操作的时候，检查值有没有发生变化，如果没有发生变化则更新，但如果一个值原来是A，变成了B，又变成了A，那么使用CAS进行检查时会发现它的值没有发生变化，但实际上却发生了变化。

以我对对ABA的理解，ABA对CAS本身来说是没有伤害的，但对ABA的过程中，依赖于CAS的相应操作记录是有影响的。

例如：boolean suc = atomic.compareAndSet(A,B);，依赖于suc是否true来判断是A是否有变化。你感觉是没有变化，可以正确的修改为B，说明当前值时A。但实际上其它的线程已经完成A-C-A的过程。

可以使用*AtomicStampedReference*其自动了版本号，例如：A1->B2->A3。尽管又回到了A，但A的版本号是3。

#### 3.2 自旋时间长CPU消耗大

上面的safeCount()方法就是典型的自旋，自旋：需要循环、count.get()、atomic.compareAndSet(i,++i)、break。

循环：保证不达目的不罢休；

count.get()：获取最新的值；

atomic.compareAndSet(i,++i)：CAS操作；

break：操作成功则退出。

如果CAS长期不成功，会给CPU带来非常大的执行开销。例如：上面的safeCount()方法，线程不可能一次操作成功，有可能需要循环多次才能成功。但即使这样也比synchronzied要强。

#### 3.3 只能保证一个共享变量的原子操作

目前java提供的所有Atomic操作都是基于一个变量的，如果需要多个变量原子保证只能使用synchronized。



