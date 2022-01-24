---
title: Java运行时数据区和内存模型
date: 2021-03-11 22:20:46
categories: JVM
tags: [JVM, 设计模式]
---

运行时数据区是指对 JVM 运行过程中涉及到的内存根据功能、目的进行的划分，而内存模型可以理解为对内存进行存取操作的过程定义。总是有人望文生义的将前者描述为 “Java 内存模型”，最近在阅读《深入理解 Java 虚拟机》之后对二者加深了部分理解，于是写一篇相关内容的学习总结。

<!--more-->

# 运行时数据区

《Java 虚拟机规范》定义中，由 JVM 管理的内存区域分为以下几个运行时数据区域：

{% mermaid flowchart LR %}
subgraph 运行时数据区
    subgraph 线程私有
        虚拟机栈
        本地方法栈
        程序计数器
    end
    subgraph 线程共享
        方法区
        javaHeap[Java 堆]
    end
end
{% endmermaid %}

## 程序计数器

程序计数器（Program Counter Register）的生命周期和 Java 线程一致，仅能被相关线程访问。用于记录当前线程所执行的字节码的行号，代码中的分支、循环、跳转、异常处理和多线程发生切换时的线程恢复等基础功能都需要依赖程序计数器完成。当执行 Java 代码时，程序计数器存储正在执行的虚拟机字节码指令地址，而执行本地方法时，程序计数器应该为空。

《Java 虚拟机规范》中规定程序计数器不会发生 `OutOfMemoryError` 异常情况。

## 虚拟机栈和本地方法栈

虚拟机栈（Java Virtual Machine Stack）的生命周期和 Java 线程一致，仅能被相关线程访问。用于描述 Java 方法执行的线程内存模型：每次进入一个新的方法时，JVM 都会同步创建一个**栈帧**，栈帧中存储了局部变量表、操作数栈、动态连接和方法出口等信息。每次执行结束一个方法时，对应栈帧就会出栈。

虚拟机栈可能发生两类异常情况：

- `StackOverflowError`：线程请求的栈深度大于虚拟机允许的深度
- `OutOfMemoryError`：如果 JVM 栈容量可以动态拓展，当需要拓展时 JVM 无法申请到足够的内存就会抛出该异常。而在 HotSpot 这种不允许动态拓展的虚拟机中，如果创建时就失败依然也会抛出该异常。

本地方法栈（Native Method Stack）与虚拟机栈基本类似，当执行本地方法时使用本地方法栈，同样会抛出 `StackOverflowError` 和 `OutOfMemoryError` 。

《Java 虚拟机规范》对本地方法栈的实现方式没有任何强制规范，故 HotSpot 虚拟机中虚拟机栈和本地方法栈直接合二为一。

虚拟机栈的大小通过 `java` 命令的参数 `-Xss` 设定。

## Java 堆

Java 堆（Java Heap）在虚拟机启动时就创建，虚拟机关闭时销毁，被所有 Java 线程共享。用于存放所有的对象实例。

Java 堆受垃圾回收器管理，由于现代主流的垃圾回收器都是基于分代收集理论设计，Java 堆中经常会出现*“新生代”、“老年代”、“永久代”、“Eden空间”、“From Survivor空间”、“To Survivor空间”* 等名词。这些名词对于 Java 堆的划分，是指一部分垃圾回收器的设计风格，而不是 JVM 具体实现的固有内存布局，更不是 《Java 虚拟机规范》里对 Java 堆的进一步细致划分。并且近年来新出现的垃圾回收器也有不采用分代设计的，再用这些名词划分 Java 堆空间也已经不正确了。

Java 堆可能发生 `OutOfMemoryError`异常：分配内存给新的对象实例失败、且堆无法再拓展时抛出该异常。

Java 堆的大小通过 `java` 命令的参数 `-Xmx` 和 `-Xms` 设定。

## 方法区

方法区（Method Area）和 Java 堆一样在虚拟机启动时就创建，虚拟机关闭时销毁，被所有 Java 线程共享。用于存储已被虚拟机加载的类型信息、常量、静态变量和即时编译器编译后的代码缓存。

在 HotSpot 虚拟机中，方法区的实现经历过两次大规模改动：

0. JDK 6 及以前：为了方便使用分代垃圾收集器管理方法区内存，使用永久代（Permanent Generation）实现方法区。
1. JDK 7 时期：字符串常量池和静态变量转移到本地内存中（Native Memory），其他数据依然存放在由永久代实现的方法区中。
2. JDK 8 及此后：方法区中的全部数据都存放在被称为元空间（Meta-space）的本地内存中。

如果方法区无法满足新的分配内存需求时会抛出 `OutOfMemoryError` 异常。

# 内存模型

内存模型：在特定的缓存一致性协议约束下，对**特定的内存**或**高速缓存**进行读写访问的过程抽象。内存模型主要用于在共享内存的多核系统中解决缓存一致性的问题。不同架构的物理机通常具有不一样的内存模型，而Java虚拟机也有自己的内存模型。

在硬件层面上，内存模型针对**高速缓存**。如图所示，处理器运算速度和主内存的存取速度天差地别，必须加入高速缓存来作为内存和处理器之间的缓冲。如果两个处理器执行的指令都涉及到主内存同一区域，就会发生各自的缓存数据不一致。这种情况下需要通过缓存一致性协议规定处理器的读写行为。

{% mermaid flowchart LR %}

CpuOne(处理器一) <--> CacheOne(高速缓存一)
CacheOne <--> Protocol[缓存一致性协议]

CpuTwo(处理器二) <--> CacheTwo(高速缓存二)
CacheTwo <--> Protocol

Protocol <--> Memory((主内存))
{% endmermaid %}

在 JVM 中，内存模型针对各线程的**工作内存**。如图所示，其中关键名词解释如下：

- 主内存：与上方硬件层面的主内存概念一致，实际是操作系统为 JVM 分配的物理内存空间。
- 工作内存：每个线程各自的工作内存，可与上方硬件层面的高速缓存类比。

工作内存中保存了该线程使用到的变量的主内存副本（并不是百分之百的完全副本，例如两条线程同时访问一个 10MB 的对象，并不会在各自的工作内存中都创建一个完全相同的对象），对每个变量的读、写操作都必须在工作内存中进行，不能直接访问主内存（`volatile` 变量通过读写屏障实现，也受这条规则约束，并不是直接访问主内存）。

主内存、工作内存的划分方式与前一章中阐述的堆、栈、方法区等划分方式是截然不同的概念，不能类比或对应。

{% mermaid flowchart LR %}

ThreadOne(线程一) <--> WorkingMemoryOne(工作内存一)
WorkingMemoryOne <--> SaveAndLoad[Save和Load操作]

ThreadTwo(线程二) <--> WorkingMemoryTwo(工作内存二)
WorkingMemoryTwo <--> SaveAndLoad

SaveAndLoad <--> Memory((主内存))
{% endmermaid %}

## volatile 变量

`volatile` 变量有可见性和禁止指令重排序优化两个特点。

### 可见性

一个线程修改了 `volatile` 变量，其他线程立即可知。在 JVM 中的实现方式为，线程对 `volatile` 变量的读取时，需要先将主内存中的当前值拷贝到工作内存中；线程对 `volatile` 变量写入时，写入工作内存后立即同步到主内存中。注意可见性和原子性是不同的概念，`volatile` 关键字无法保证对变量操作的原子性。

### 禁止指令重排序优化

指令重排序优化是指处理器为了提高运算单元的利用率，会对指令进行乱序执行优化，处理器能够保证经过乱序的指令和原始顺序的指令执行结果一致。禁止指令重排序的方式是添加内存屏障，重排序时不能把后面的指令重排序到内存屏障之前的位置。

如果在多线程环境中，指令重排序优化可能导致访问共享资源出错，一个常见的例子就是单例模式的双锁检查式实现方式需要将实例引用添加 `volatile` 修饰。

```java
public class Singleton {
    //如果不添加 volatile 修饰可能发生异常
    private volatile static Singleton instance;

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

    private Singleton() {};
}
```

其中 `instance = new Singleton();` 一句对应的字节码指令为：

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

根据指令重排序的原则，三、四两步之间乱序执行不会影响结果，那么如果发生了指令重排、且两个线程恰好按照如下步骤执行就会发生异常情况（篇幅所限不给出能发生指令重排序的代码，其实就是上方代码删去 `volatile` ）：

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

而如果 `instance` 使用 `volatile` 修饰，由于内存屏障的存在指令 `INVOKESPECIAL` 和 `PUTSTATIC` 之间就不会发生指令重排，确保了上述问题不会发生。另外请注意，Java 中双锁检查式方式实现的单例在 Java 5 之后才能完全保证可用，此前版本依然会出现问题。