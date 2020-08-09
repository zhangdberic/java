# ReentrantLock

示例代码：

```java
class ReentrantLockExample {
    int a = 0;
    ReentrantLock lock = new ReentrantLock();
    
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



如上例，不使用java原生的synchronized，而使用ReentrantLock，而ReentrantLock是使用java编码实现的锁，其如果实现a变量的可见性？

