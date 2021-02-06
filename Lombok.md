# Lombok

## lombok插件安装

### eclipse

下载插件

https://projectlombok.org/download

双击运行lombok.jar

弹出界面，点击Specify location. 选择要安装eclipse.exe文件。选择后点击"Install/Update"按钮，执行安装。

安装成功后，会在eclipse目录下，多出lombok.jar文件，并且eclipse.ini文件末尾会增加一行：-javaagent:I:\eclipse202009\eclipse\lombok.jar，重启eclipse。

## lombok使用

### pom.xml

如果你使用了 Spring Boot，可以不用带版本号，在 Spring Boot `spring-boot-dependencies.pom` 这个配置文件里面定义了 Lombok 依赖。

```xml
		<dependency>
		    <groupId>org.projectlombok</groupId>
		    <artifactId>lombok</artifactId>
		    <scope>provided</scope>
		</dependency>
```

### 使用注解简化代码

```
@Getter and @Setter
@FieldNameConstants
@ToString
@EqualsAndHashCode
@AllArgsConstructor, @RequiredArgsConstructor and @NoArgsConstructor
@Log, @Log4j, @Log4j2, @Slf4j, @XSlf4j, @CommonsLog, @JBossLog, @Flogger
@Data
@Builder
@Singular
@Delegate
@Value
@Accessors
@Wither
@SneakyThrows
@Cleanup
@Synchronized
from Intellij 14.1 @val
from Intellij 15.0.2 @var
from Intellij 14.1 @var
from Intellij 2016.2 @UtilityClass
Lombok config system
Code inspections
Refactoring actions (lombok and delombok)
```

也可以去 Lombok 对应的包里面看所有支持的注解。

现在挑几个讲一下它们的用法吧！

**@Getter 和 @Setter**

```java
@Getter
@Setter
public class User {

  private String name;

  private int age;

  ...

  // 无需生成 get/set 方法

}
```

添加 `@Getter` 和 `@Setter` 注解用在 Java Bean 类上面，无需生成 get/ set 方法，会自动生成所有的 get/ set 方法及一个默认的构造方法。

**@ToString**

使用在类上，默认生成所有非静态字段以下面的格式输出，如：

```java
public String toString(){
    return "Person(userName=" + getUserName() + ", id=" + getId() + ", age=" + getAge() + ", address=" + getAddress() + ", memo=" + getMemo() + ")";
}
```

里面也有很多参数，用来自定义输出格式。

@ToString源注释属性说明

```java
@ToString(
        includeFieldNames = true, //是否使用字段名
        exclude = {"name"}, //排除某些字段
        of = {"age"}, //只使用某些字段
        callSuper = true //是否让父类字段也参与 默认false
) 
```

**@NoArgsConstructor**

用在类上，用来生成一个默认的无参构造方法。

**@RequiredArgsConstructor**

用在类上，使用类中所有带有 `@NonNull` 注解和 `final` 类型的字段生成对应的构造方法。

**@AllArgsConstructor**

用在类上，生成一个所有参数的构造方法，默认不提供无参构造方法。

**@Data**

用在类上，等同于下面这几个注解合集。

- @Getter
- @Setter
- @RequiredArgsConstructor
- @ToString
- @EqualsAndHashCode

**@Value**

@Value注解用于修饰类，相当于是@Data的不可变形式，因为字段都被修饰为private和final，默认的情况下不会生成settter。还有一点更狠的，默认类本身也是final的，不能被继承。

用在类上，等同于下面这几个注解合集。

- @Getter
- @FieldDefaults(makeFinal=true, level=AccessLevel.PRIVATE)
- @AllArgsConstructor
- @ToString 
- @EqualsAndHashCode

**@NonNull**

用在属性上，用于字段的非空检查，如果传入到 set 方法中的值为空，则抛出空指针异常，该注解也会生成一个默认的构造方法。

还有很多，这里不再撰述。

**@Slf4j**

```java
@Slf4j
public class SyncDataScheduler {
    public void task() {
        log.debug("task execute.");
    }
}
```

