# 



# 泛型

声明中具有一个或者多个类型参数(type parameter)的类或者接口，就是泛型(generic)类或者接口。

每一种泛型都定义一个原生态(raw type)， 即不带任何实际类型参数的泛型名称。

泛型是对原生类型的增强，创造一个更具体的类型，例如：**把List增强为List<String>**，List是集合，List<String>是字符串集合，原生的List内部可以存放任何类型的对象，而List<String>只能存放字符串类型对象。泛型是对某种类型的更具体化，特别为容器(例如:集合)类型，提供了一个可以**更具体的化的集合类型**。

## 尽量不要使用原生态类型

原生态类型是非常不安全的，是为了之前的遗留代码的兼容，因此不要使用。

## 原生List和List&#60;Object&#62;区别

前者逃避了泛型检查，**后者则明确告知编译器，它能持有任何类的对象**，以下这些都是合法的。

原生List没有任何限制，可以存在任何类型，其实不安全的，而List<Object>具体化后读作Object类型列表对象，是你自己明确声明的，不会出现不安全的现象。

```java
List<Object> genericList = new ArrayList<>();
genericList.add("123");
genericList.add(new Integer(1));
genericList.add(new Object());
```

## List&#60;Object&#62;和List<? extends Object>的区别

```java
List<Object> o = new ArrayList<String>(); // 编译错误，因为List<Object>表示集合内元素类型必须是Object类型
List<? extends Object> o1 = new ArrayList<String>();  // 编译正确，因为List<? extends Object>表示集合内元素类型必须是Object和Object子类型
```

## 泛型子类化规则

```
List<String>是原生态List的一个子类型，但不是List<Object>的子类型,例如下面的类型转化：
```

正确的转换：

```java
List<String> stringList = new ArrayList<>();
List rawList = stringList;
```

错误的转换：Type mismatch: cannot convert from List<String> to List<Object>，如果编译不报错那才是不安全。

```java
List<String> stringList = new ArrayList<>();
List<Object> objectList = stringList;
```

## 原生Set和无限通配符Set<?>区别

```
通配符Set<?>是安全的，原生态类型Set则是不安全的。

原生态Set可以将任何元素放进集合中，很容易破坏集合类型约束条件；

通配符Set<?>不能将任何任何元素(除了null之外)放到Set<?>中；Set<?>理解为某种类型对象的只读Set；
```

## 类型(class)判断必须使用原生态类型

```
Class c = List.class是正确的，Class c = List<String>.class是不正确的。
c instanceof List是正确的，c instanceof List<String>是不正取得。
```

## Set&#60;Object&#62;、Set<?>、Set区别

```
Set<Object>是一个参数类型，表示可以包含任何对象类型；
Set<?>则是一个通配符类型，表示只能包含某种未知对象类型的一个集合；
Set则是原生态类型，它脱了了泛型系统。
```

**前两种是安全的，最后一种则不安全。**

## <? extends T>和<? super T>通配符

好文章：https://www.cnblogs.com/chenxibobo/p/9655236.html

<? extends T>和<? super T>可以参数化任何类型，但比较常用的集合，下面使用参数化集合来说明。

### <? extends T>

使用<? extends T>参数化的集合，创建了一个新的集合类型，其内容只能是T类型或T子类型。

例如：

```java
Collection<? extends T> c;   // 这个声明会，创建一个新的集合类型，这个新的集合类型是内部元素必须是T或者T子类型集合。
```

例如：

```java
public class A {
}
public class B extends A {
}
public class C extends B {
}
```

```java
List<? extends B> lista = new ArrayList<A>(); // 编译错误
List<? extends B> listb = new ArrayList<B>(); // 正确
List<? extends B> listc = new ArrayList<C>(); // 正确
```

<? extends T>声明的集合对象，不能使用add和put等方法直接存数据，只能使用get和iterator等取数据。通常修饰一个集合，表示集合内允许某种类型的上界。

举例，这里假设：

```java
List<? extends B> z = new ArrayList<>();
Object o = z.get(1);  // 编译正确
A a = z.get(1);  // 编译正确
B b = z.get(1);  // 编译正确
C c = z.get(1);  // 编译错误

z.add(new Object());  // 编译错误
z.add(new A());  // 编译错误
z.add(new B());  // 编译错误
z.add(new C());  // 编译错误
```

常用的用法：

```java
List<? extends B> list1 = getList();  // getList()方法返回一个B类型的子类型集合
```



### <? super T>

使用<? super T>参数化的集合，创建了一个新的集合类型，其内容只能是T类型或T父类型。

例如：

```java
Collection<? super T> c;   // 这个声明会，创建一个新的集合类型，这个新的集合类型是内部元素必须是T或者T父类型集合。
```

例如：

```java
public class A {
}
public class B extends A {
}
public class C extends B {
}
```

```java
List<? extends B> lista = new ArrayList<A>(); // 正确
List<? extends B> listb = new ArrayList<B>(); // 正确
List<? extends B> listc = new ArrayList<C>(); // 编译错误
```

<? super T>声明的对象，只能存放T类型的子类型数据，读取到的数据只能赋值给Object类型变量。

```java
List<? super B> z = new ArrayList<>();
A a = z.get(1);  // 编译错误
B b = z.get(1);  // 编译错误
C c = z.get(1);  // 编译错误
Object o = z.get(1); // 编译正确
		
z.add(new A());  // 编译错误
z.add(new B());  // 编译正确
z.add(new C());	 // 编译正确	
```

最实用的用法：

```java
void popAll(List<? super B> dst){
   dst.add(new B());
   dst.add(new C());
}
```

其多用于消费者，消费者会调用上面的方法，传递一个消费者参数给某个对象，这个对象会使用内部数据来填充(写入)消费者参数。

### 生产者和消费者

下面的代码来自于：java.util.Collections类，其在参数上使用了生产者和消费者。dest为消费者(数据获取方)，src为生产者(数据来源方)。

```java
    public static <T> void copy(List<? super T> dest, List<? extends T> src) {
        int srcSize = src.size();
        if (srcSize > dest.size())
            throw new IndexOutOfBoundsException("Source does not fit in dest");

        if (srcSize < COPY_THRESHOLD ||
            (src instanceof RandomAccess && dest instanceof RandomAccess)) {
            for (int i=0; i<srcSize; i++)
                dest.set(i, src.get(i));
        } else {
            ListIterator<? super T> di=dest.listIterator();
            ListIterator<? extends T> si=src.listIterator();
            for (int i=0; i<srcSize; i++) {
                di.next();
                di.set(si.next());
            }
        }
    }
```



### <? extends T>和&lt;E extends T&gt;区别

<? extends T>是无限通配符，&lt;E extends T&gt;是有限通配符，在使用上没有区别，用法上有区别，&lt;E extends T&gt;多用于在类(Class)上声明一个类型E，后续的操作必须使用E类型来操作，其使用上受到的限制和上面文章说明的<? extends T>没有区别。

### &lt; E super T&gt;不能编译(没有意义)

## 列表优于数组

数组是协变的，是不安全的，下面举例来说明，E[]和List<E>的区别吧：

```java
	public static <E extends B> void array(E[] earray,List<E> elist) {
		Object[] a = earray;  // 编译正确，问题出在这里，数组协变的特性：子类型的数组可以直接赋值父类型的数组，数组类型是可以多态类型的，是不安全的。
		a[0] = 123;  // 编译正确
		a[1] = "abc"; // 编译正确
		List<Object> b = elist;  // 编译错误,区别于数组协变，List<Object>只表示Object元素列表类型。
        List<? extends Object> bo = elist;  // 编译正确,人为操作，实现了同数组一样的协变特性，但其是安全编程。
	}
```

因此建议，如果在需要使用泛型的地方，最好只使用集合，不使用数组。

# 异常

1.异常应该只用于异常的情况下：永远不应该用于正常的控制流。

2.设置良好的API不应该强迫它的客户端为了正常的控制流而使用异常。

3.对可恢复的情况使用受检异常(检查型异常)，对编程错误使用运行时异常。

4.决定使用受检异常和未受检异常原则：如果期望调用者能够适当地恢复，使用受检异常。通过抛出受检异常强迫调用者在一个catch子句处理该异常，或者传播下去。如果不能确认是否能恢复，则使用运行时异常。

5.优先使用标准的异常，如果没有必要，不要自己编写异常类：

IllegalArgumentException，参数值不正确;

IllegalStateException，对象状态不正确;

NullPointerException，null异常;

IndexOutOfBoundsException，下标越界;

ConcurrentModificationException，并发修改异常(不支持并发修改情况);

UnSupportedOperationException，不支持的操作异常(通过用在接口部分方法没有被实现);

不要直接抛出Exception、RuntimeException、Throwable和Error;                                                                                                                                                      

6.异常转换，抛出和抽象对应的异常，把底层的异常转换为高层的异常，高层的实现应该捕获低层的异常，同时抛出可以按照高层意思进行解释的异常。例如：服务层调用jdbc操作数据库，如果jdbc抛出异常，则应该抛出服务异常，服务异常内详细说明原因，并包含jdbc异常原因(cause)。

7.每个受检异常都应该文档注释说明原因@throw XxxException desc.

8.抛出的异常尽量包含细节信息，包含失败原因，这样便于发现和解决问题。例如：抛出IndexOutOfBoundsException，则这个异常类应该包含beginIndex、endIndex、currentIndex等属性。

9.努力使失败保持原子性，一般而言，失败的方法调用应该使对象保持在被调用之前的状态：

不可变的对象，没有原子性的问题；

可变量的对象，应该在改变之前进行必要的有效性检查；或者在对象被改变前先建立一份拷贝，失败的时候，使用拷贝覆盖被修改的对象。

10.不要忽略任何异常。













