# lambda表达式

## lambda写法

多参数和多语句情况写法

```java
(parameters) -> {statements;}
```

单个参数和单语句写法

```java
parameter -> statement
```

无参数和单语句写入

```java
() -> statement
```

无参数和多语句写入

```java
() -> {statements;}
```



## lambda注意

1. 不需要声明参数类型，编译器可以识别参数值；

2. 单个参数和语句下，圆括弧和大括弧可以省略；

   ```java
   forEach(nationality -> System.out.println("Found: " + nationality))
   ```

   表达式是一个闭包，定义了行内执行的方法类型接口；

4. 只能引用标记了final的外层局部变量，不能在表达式内部修改定义在域外的局部变量，否则会编译错误；

   ```java
   int num = 1;  
   Converter<Integer, String> s = (param) -> System.out.println(String.valueOf(param + num));
   s.convert(2);
   num = 5;  
   //报错信息：Local variable num defined in an enclosing scope must be final or effectively final
   ```

   

5. 表达式当中不允许声明一个与局部变量同名的参数或者局部变量；

   ```java
   String first = "";  
   Comparator<String> comparator = (first, second) -> Integer.compare(first.length(), second.length());  //编译会出错 
   ```

   

6. 如果主体只有一个表达式返回值则编译器会自动返回值，大括号需要指定明表达式返回了一个值；

   ```java
   (int a, int b) -> { return a * b; };
   ```

   

## lambda例子

### lambda简单例子

```java
// 1. 不需要参数,返回值为 5  
() -> 5  
  
// 2. 接收一个参数(数字类型),返回其2倍的值  
x -> 2 * x  
  
// 3. 接受2个参数(数字),并返回他们的差值  
(x, y) -> x – y  
  
// 4. 接收2个int型整数,返回他们的和  
(int x, int y) -> x + y  
  
// 5. 接受一个 string 对象,并在控制台打印,不返回任何值(看起来像是返回void)  
(String s) -> System.out.print(s)
```

### lambda定义函数

**要求**

接口有且仅有一个抽象方法

允许定义静态方法

允许定义默认方法

允许java.lang.Object中的public方法

该注解不是必须的，如果一个接口符合"函数式接口"定义，那么加不加该注解都没有影响。加上该注解能够更好地让编译器进行检查。如果编写的不是函数式接口，但是加上了@FunctionInterface，那么编译器会报错

```java
	@FunctionalInterface
    interface HelloWorld {
        void print(String str);
        // 默认(default)方法，只能通过接口的实现类来调用，不需要实现方法，也就是说接口的实现类可以继承或者重写默认方法。
        default void print2(String message) {
            System.out.println(message);
        }
        // 静态(static)方法只能通过接口名调用，不可以通过实现类的类名或者实现类的对象调用。就跟普通的静态类方法一样，通过方法所在的类直接调用。
        static void print3(String message) {
            System.out.println(message);
        }
    }

```

调用

```java
public class Test1 {

	public static void main(String[] args) {
		HelloWorld helloworld = (param) -> {System.out.println(param);};
		helloworld.print("abstract method.");
		helloworld.print2("default method");
		HelloWorld.print3("static method");
	}
	
	@FunctionalInterface
    interface HelloWorld {
        void print(String str);
        // 默认(default)方法，只能通过接口的实现类来调用，不需要实现方法，也就是说接口的实现类可以继承或者重写默认方法。
        default void print2(String message) {
            System.out.println(message);
        }
        // 静态(static)方法只能通过接口名调用，不可以通过实现类的类名或者实现类的对象调用。就跟普通的静态类方法一样，通过方法所在的类直接调用。
        static void print3(String message) {
            System.out.println(message);
        }
    }
}

```



# stream

## 例子

### 打印集合中的值

```java
players.forEach(System.out::println);  
```

### 循环处理集合中的值

nations.stream()得到流式处理；

filter((s) -> s.startsWith("A"))进行过滤处理，例如：字符串开通必须是字符A的；

forEach(nationality -> System.out.println("Found: " + nationality));循环处理集合中的值；

```java
List<String> nations = Arrays.asList("A", "B", "C", "A1");
		nations.stream().filter((s) -> s.startsWith("A"))
				.forEach(nationality -> System.out.println("Found: " + nationality));
```

### 提取集合bean中的某个属性值到List

```java
List<String> names = peoples.stream().map(People::getName).collect(Collectors.toList());
```

### 提取集合bean中的某个属性值到数组

```
String[] names = peoples.stream().map(People::getName).toArray(String[]::new);
```

### 提取集合bean中的某两个属性值到Map

allProjects.stream()得到流式处理；

filter(project -> project.getId() > 1)进行过滤处理，例子：id值大于1；

collect(Collectors.toMap(Project::getId, Project::getName));收集结果到map；Project::getId和Project::getName对应Project类的getId方法和getName方法。这个相比反射获取属性，其支持编译验证(语法级别验证)。

```java
Map<Integer, String> map = allProjects.stream().filter(project -> project.getId() > 1).collect(Collectors.toMap(Project::getId, Project::getName));
```

### 取集合中的最大数值

```java
allProjects.stream().mapToInt(Project::getId).reduce(Math::max).getAsInt();
```

### 求和集合中的属性值

```java
allProjects.stream().mapToInt(k -> k.getId()).reduce((sum,id) -> sum+id).getAsInt();
```

### 求集合中属性值得平均数

```java
allProjects.stream().collect(Collectors.averagingInt(p -> p.getId())).intValue();
```

### 合并集合为分组字符串

```java
allProjects.stream().map(Project::getName).collect(Collectors.joining(",", "[", "]")).toString();
```

### 多线程处理集合

```java
allProjects.parallelStream().filter((p) -> p.getId() > 1).forEach((it) -> System.out.println("Have a beer " + it.getName() + "   " + Thread.currentThread()));
```

注意：因为parallel()或parallelStream()会开多个线程，如果要把操作结果合并为一个集合，则应该注意线程安全问题，例如：正确使用collect(Collectors.toConcurrentMap(keyMapper, valueMapper));

```java
//并发收集
Stream.of(studentA,studentB,studentC)
      .parallel()
      .collect(Collectors.toConcurrentMap(Student::getId,Student::getName));
```

### 集合排序

```java
allProjects.sort((Project o1, Project o2) -> o1.getId() - o2.getId());
或者
allProjects.sort(Comparator.comparing(Project::getId));    
```

### 集合倒序

```java
allProjects.sort(Comparator.comparing(Project::getId).reversed());
```

### 去重

大数据量可以用上面的多线程处理集合

```java
acgs.stream().distinct().collect(Collectors.toList());
```

### 在集合中的每个值上附加上字符串

```java
List<String> collect = list.stream().map(s -> s + "字母").collect(Collectors.toList());
```

### 统计符合条件集合的个数

```java
nums.stream().filter(num -> num != null).count();
```



# 好文章

https://www.jianshu.com/p/6ee7e4cd5314    Collectors收集器

https://www.cnblogs.com/freshchen/p/12313999.html Stream 好文章