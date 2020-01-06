# final域的内存语义

这里的**域**，应理解为对象内共享变量。

final的java语义，被final声明的变量，必须在构造方法结束之前被初始化一个值，并且只能被初始化一次，并不能再被更改。也就是说final声明变量，只能是有三次被初始化的机会：

1.变量初始化，例如：final int a = 1;

2.构造方法，final int a;  public 构造方法() {a=1;}

3.构造块，final int a; {a=1;}

## final域的重排序规则

我们先来看一个例子：

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

线程A执行writer()，其会创建FinalExample对象并赋值给静态成员变量obj，而线程B执行reader()读取成员变量obj，并获取其内存的obj.i和obj.j，表面上看风平浪静没有事，线程B执行reader()应获取到a=1，b=2，但实际上则会出现i=0的情况，为什么，不是线程A已经调用writer()给i变量已经赋值为1了吗？

因为：**被构造对象的引用赋值给一个引用变量和普通域变量的初始化是两个动作，其不能保证是一个原子操作。**

这就造成了线程B，已经得到了FinalExample对象的引用obj，而线程A调用构造方法的i变量还没有被初始化完的情况，因此线程B获取到obj.i有可能是0的情况；而obj.j就不会出现这样的问题，因为j变量被final修饰了。

**final声明的域变量，其会保证被构造对象的引用赋值给一个引用变量和final域变量的初始化也是两个动作，但是一个原子操作。**

final的重排序规则定义：

1. 在构造方法内对一个final域的写入，与随后把这个被构造对象的引用赋值给一个引用变量，这个两个操作之间不能重排序。
2. 初次读一个包含final域的引用，与随后初次读这个final域，这个两个操作之间不能重新排序。

写final域的重排序规则：可以确保，在对象引用为任意线程可见之前，对象的final域已经被正确初始化过了，而普通域不具有这个保障。

读final域的重排序规则：在一个线程中，初次读对象引用与初次读该对象包含的final域，JMM禁止处理器重排序这个两个操作。

## final域为引用类型

final修饰引用类型，不仅可以保证这个对象的引用，还可以保护这个对象内部的域变量在正确初始化后，才可以被其它线程看到。

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

增加了如下约束：在构造函数内对一个final引用的对象的成员域的写入，与随后在构造函数外把这个被构造对象的引用赋值给一个引用变量，这个两个操作之间不能重排序。

也就是说：

obj = new FinalReferenceExample();

内部分解为：

        intArray = new int[1];
        intArray[0] = 1;
        obj = new FinalReferenceExample();
这三步是一个原子操作，因为intArray;，声明为了final域。



