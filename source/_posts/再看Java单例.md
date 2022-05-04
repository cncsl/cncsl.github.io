---
title: 再看Java单例
date: 2021-03-12 21:54:46
categories: 设计模式
tags: [Java, 设计模式]
---

此前面试遇到了单例问题，本以为已经背的滚瓜烂熟，没想到被问单例如何避免被反射和序列化破坏，虽然后来还是等到了通知，但还是复习一下单例的实现方式，并学习防止反射和序列化破坏的手段。

<!--more-->

# 基本实现方式

其他相关资料中，最多的能数出八种单例实现方式，而实际上其中有些实现并不具备实际意义，在文中出现也仅是为了指出存在的问题便于引出下文。本文仅介绍有实际意义的单例实现模式。为了缩减篇幅，先给出一个后续出现代码的模板的类图：

{% mermaid classDiagram %}
class Singleton {
  -Logger log$
  -Singleton instance$
  +getInstance()$ Singleton
  +loadClass()$ void
  +function() void
  -Singleton()
}
{% endmermaid %}

单例类 Singleton 模板：后文中介绍具体实现方式仅给出 `Singleton#instance` 引用和 `Singleton#getInstance` 方法的内容，其他内容无变化。

```java
public class Singleton {

    private static final Logger log = LogManager.getLogger(Singleton.class);
    //单例引用，不同实现方式有所不同
    private static Singleton instance;

    /**
     * 获取单例的函数，不同实现方式有所不同
     *
     * @return 单例
     */
    public static Singleton getInstance() {
        //some code
    }

    /**
     * 静态方法，用于触发虚拟机类加载，仅有一行日志用于观察类加载时间
     */
    public static void load() {
        log.debug("{} loaded", Singleton.class);
    }

    /**
     * 单例类的功能函数，仅有一行日志
     */
    public void function() {
        log.debug("Singleton's instance using");
    }

    /**
     * 私有的构造函数，仅有一行日志用于观察构造时间
     */
    private Singleton() {
        log.debug("Singleton's instance instantiated");
    }
}
```

调用单例类的 Main 类：

```java
public class Main {

    public static void main(String[] args) throws Exception{
        //先触发类加载
        Singleton.load();
        //等待一定时间
        TimeUnit.SECONDS.sleep(3);
        //执行单例的功能函数
        Singleton.getInstance().function();
    }

}
```

## 饿汉式

饿汉式具有线程安全和非 Lazy 初始化的特点，实现难度最简单。由于 JVM 的类加载是单线程的，且已加载过的类不会重复加载，所以饿汉式天生具有线程安全的特点。

- 由于是类加载即初始化，单例引用可添加 `final` 修饰。
  
  ```java
  public static final Singleton instance = new Singleton();
  
  //下面写法效果相同
  /*
  public static final Singleton instance;
  
  static {
      instance = new Singleton();
  }
  */
  ```

- 获取单例函数：
  
  ```java
  public static Singleton getInstance() {
      return instance;
  }
  ```

执行结果：

```
13:19:32.565 - Singleton's instance instantiated
13:19:32.569 - class cncsl.github.io.Singleton loaded
13:19:35.572 - Singleton's instance using
```

从日志可以看出，单例类刚加载时就调用构造函数完成了单实例的初始化。

## 双锁检查式

双锁检查是经常出现于面试题中的实现方式，具有线程安全和 Lazy 初始化的特点。需要自行实现线程安全的单例初始化，且要避免指令重排序导致的安全问题，实现难度较高。

- 为了避免指令重排序导致的线程安全问题需要给单例引用添加 `volatile` 修饰：
  
  ```java
  private volatile static Singleton instance;
  ```

- 双锁检查式最难的部分就是在获取单例的函数中进行两次非 null 判断和加锁后再初始化的过程：
  
  ```java
  public static Singleton getInstance() {
      if (instance == null) {
          synchronized (Singleton.class) {
              if (instance == null) {
                  instance = new Singleton();
              }
          }
      }
      return instance;
  }
  ```

执行结果：

```
13:30:07.524 - class cncsl.github.io.Singleton loaded
13:30:10.541 - Singleton's instance instantiated
13:30:10.541 - Singleton's instance using
```

从日志可以看出，类加载之后并没有立即初始化，实际需要调用到单例的功能函数前才进行了初始化。

### volatile 关键字的作用

`volatile` 修饰的变量有可见性和禁止指令重排序优化两个特点：

- 可见性：一个线程修改了 `volatile` 变量，其他线程立即可知。
- 禁止指令重排序优化：处理器为了提高运算单元的利用率，会对指令进行乱序执行优化，处理器能够保证经过乱序的指令和原始顺序的指令执行结果一致。

在多线程环境中，指令重排序优化可能导致访问共享数据出错。这也是双锁检查式单例的单例变量用 volatile 修饰的原因。

Java 源码中 `instance = new Singleton();` 一句对应的字节码指令为：

```
NEW Singleton
DUP
INVOKESPECIAL Singleton.<init> ()V
PUTSTATIC Singleton.instance : LSingleton;
```

总共分为四个步骤：

1. `NEW` 指令创建对象（此时还没有执行构造函数，对象的所有成员变量都是默认的“零值”），将对象引用压入操作数栈。
2. `DUP` 指令将当前操作数栈顶的值拷贝一份（此时操作数栈顶的两个元素是两个相同的、值为前一步创建的对象的引用）。
3. `INVOKESPECIAL` 指令会调用当前操作数栈顶的对象的 `<init>` 方法，该方法是根据空参的构造函数生成的。
4. `PUTSTATIC` 指令将操作数栈顶的对象引用赋值给 Singleton 类的 instance 引用。

根据指令重排序的原则，三、四两步之间乱序执行不会影响结果，如果未加 `volatile` 修饰、发生了指令重排，且两个线程恰好按照如下步骤执行就会发生异常情况（字节码指令和处理器指令没有对应关系，所以这个例子并不是很严谨，但理解当前的上下文够用了）：

{% mermaid sequenceDiagram %}
    threadOne ->> threadOne : NEW Singleton
    threadOne ->> threadOne : DUP
    threadOne ->> Singleton.class : PUTSTATIC Singleton.instance : LSingleton;

    threadTwo ->> Singleton.class : 调用 getInstance() 函数
    Singleton.class ->> threadTwo : Singleton.instance 引用不为 null，返回引用值
    
    threadTwo ->> threadTwo : 使用单例对象
    Note over threadTwo : 此时单例对象还未完全初始化，发生异常
    
    threadOne ->> threadOne : INVOKESPECIAL
{% endmermaid %}

而如果 `instance` 使用 `volatile` 修饰，在指令 `INVOKESPECIAL` 和 `PUTSTATIC` 之间就不会发生指令重排，确保了上述问题不会发生。另外请注意，Java 中双锁检查式方式实现的单例在 Java 5 之后才能完全保证可用，此前版本依然会出现问题。

## 静态内部类式

静态内部类式也用到了 JVM 类加载器的特性，既保证线程安全的情况下实现了 Lazy 加载。

- 添加一个静态内部类持有单例引用：
  
  ```java
  private static class InstanceHolder {
      private static final Singleton INSTANCE = new Singleton();
  }
  ```

- 获取单例的函数调用时才会加载静态内部类，进而触发单实例的初始化：
  
  ```java
  public static Singleton getInstance() {
      return InstanceHolder.INSTANCE;
  }
  ```

执行效果与双锁检查式相同，不在赘述。

## 枚举式

枚举式的实现是将单例类写成一个枚举，枚举值仅包含一个单例引用，再加上与业务逻辑相关的功能函数即可。由于枚举的特点，这种实现方式具有线程安全、非 Lazy 加载和防止反射、序列化破坏单例等特点。

由于改动较大附上全部代码：

```java
public enum Singleton {

    /**
     * 单例枚举值
     */
    INSTANCE;

    /**
     * 获取单例的函数
     */
    public static Singleton getInstance() {
        return INSTANCE;
    }

    /**
     * 静态方法，用于触发虚拟机类加载，仅有一行日志用于观察类加载时间
     */
    public static void load() {
        System.out.printf("%s - %s loaded%n", LocalTime.now().toString(), Singleton.class);
    }

    /**
     * 单例类的功能函数，仅有一行日志
     */
    public void function() {
        System.out.printf("%s - Singleton's instance using%n", LocalTime.now().toString());
    }

    /**
     * 私有的构造函数，仅有一行日志用于观察构造时间
     */
    Singleton() {
        System.out.printf("%s - Singleton's instance instantiated%n", LocalTime.now().toString());
    }
}
```

执行结果：

```
13:57:27.142 - Singleton's instance instantiated
13:57:27.154 - class cncsl.github.io.Singleton loaded
13:57:30.156 - Singleton's instance using
```

可以看出，枚举类加载之后立即初始化了单例对象，而三秒后执行了单例类的功能函数。

# 防反射和序列化破坏单例

在 Java 中，通过序列化也能创建新的对象实例，而反射能突破构造函数 `private` 的限制，下面介绍一下如何避免这些情况的发生。枚举式单例天生避免了这些问题，下方内容都是针对其他三种实现方式而言的。

另外，请明白一个前提，设计模式是一种设计的方式，既不是某种语言的语法约束，除了枚举方式以外、其他实现方式在有人恶意破坏的情况都无法完全确保单例。在这种情况下，需要考虑的不是如何改进现有的设计，而是找出企图通过这些手段破坏单例的人。所以下面的知识一般用于面试：当遇到如何确保单例的问题时，首先说枚举式设计方式、然后才是下面的内容。

## 反射手段

下方是通过反射方式破坏单例的过程：

```java
public static void main(String[] args) {
    try {
        //通过getInstance()获取
        Singleton one = Singleton.getInstance();
        log.debug(one.hashCode());
        //反射调用构造函数
        Constructor<Singleton> constructor = Singleton.class.getDeclaredConstructor();
        constructor.setAccessible(true);
        Singleton two = constructor.newInstance();
        log.debug(two.hashCode());
        log.debug(one == two);
    } catch (Exception e) {
        log.error("Exception: ", e);
    }
}
```

执行结果：

```
15:10:48.237 - Singleton's instance instantiated
15:10:48.240 - 1159114532
15:10:48.240 - Singleton's instance instantiated
15:10:48.240 - 1832580921
15:10:48.240 - false
```

可以看出目前程序中以存在两个 Singleton 类的实例，单例已经被破坏。

解决方案为在单例类的构造函数中进行检查，如果单例引用不为 null 就抛出异常：

```java
private Singleton() {
    if (instance != null) {
        throw new UnsupportedOperationException("不允许重复创建实例");
    }
    log.debug("Singleton's instance instantiated");
}
```

再次执行结果：

```
15:17:36.834 - Singleton's instance instantiated
15:17:36.836 - 1159114532
15:17:36.836 - Exception: 
java.lang.reflect.InvocationTargetException: null
    at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method) ~[?:1.8.0_261]
    at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62) ~[?:1.8.0_261]
    at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45) ~[?:1.8.0_261]
    at java.lang.reflect.Constructor.newInstance(Constructor.java:423) ~[?:1.8.0_261]
    at cncsl.github.io.Main.main(Main.java:18) [classes/:?]
Caused by: java.lang.UnsupportedOperationException: 不允许重复创建实例
    at cncsl.github.io.Singleton.<init>(Singleton.java:32) ~[classes/:?]
    ... 5 more
```

当然，攻击者可以在外部先记录一份 `instance` 引用，通过反射修改 `instance` 引用后再创建对象，这样程序中会存在两个 `Singleton` 实例。

## 序列化手段

下方是通过序列化手段破坏单例的过程：

```java
public static void main(String[] args) {
    try (ObjectOutputStream output = new ObjectOutputStream(new FileOutputStream("Singleton.temp"));
         ObjectInputStream input = new ObjectInputStream(new FileInputStream("Singleton.temp"))) {

        Singleton one = Singleton.getInstance();
        log.debug(one.hashCode());

        output.writeObject(one);
        Singleton two = (Singleton) input.readObject();
        log.debug(two.hashCode());

        log.debug(one == two);
    } catch (Exception e) {
        log.error("Exception: ", e);
    }
}
```

执行结果如下：

```
21:15:36.605 - Singleton's instance instantiated
21:15:36.610 - 22756955
21:15:36.619 - 1582785598
21:15:36.619 - false
```

可以看出，序列化读取到的对象已经是一个新的对象，单例已被破坏。

解决方案是为单例类添加如下函数：

```java
private Object readResolve() {
  return instance;
}
```

再次执行后可以发现已经反序列化时得到的仍然是原单例对象：

```
21:28:34.954 - Singleton's instance instantiated
21:28:34.956 - 22756955
21:28:34.964 - 22756955
21:28:34.964 - true
```

当前序列化有个前提是实现 `Serializable` 接口，私以为这种情况是一个错误的设计：单例类一般和业务逻辑相关、而序列化一般和封装数据用的实体对象有关，二者不应该出现在同一个类里。