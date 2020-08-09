# single instance(单实例)

两种方式创建单实例：

## 基于双重检查锁来创建单实例

```java
public class DoubleCheckedLocking {
    private static volatile Instance instance;
    
    public static Instance getInstance(){
        if(instance == null){
            synchronized(DoubleCheckedLocking.class){
                if(instance == null){
                    instance =  new Instance();
                }    
            }
        }
        return instance;
    }
}
```

## 基于类初始化的解决方案

```java
public class InstanceFactory {
    private static class InstanceHolder {
        public static Instance instance = new Instance();
    }
    public static Instance getInstance(){
        return InstanceHolder.instance;
    }
}
```

JVM在类初始化期间会获取这个初始化锁，并且每个线程至少获取一次锁来确保这个类已经被初始化过了。

