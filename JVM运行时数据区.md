# 一、JVM运行时数据区

## 概念

运行时数据区：java运行时JVM会将其所管理的内存划分为若干个区域，统称为运行时数据区。

其中，一些线程间共享的区域，随着JVM的启动而创建，JVM的退出而销毁；另一些线程私有的区域，则随着线程的开始而创建，线程的结束而销毁



当编写的.java文件进行运行，先通过javac编译成.class文件，在通过类加载器进入内存管理

![运行时时区.png](/upload/运行时时区.png)

# 二、解析JVM运行时数据区

## 1.方法区

- 方法区是所有线程的共享的内存区域，在JVM启动时创建，用于存储已被java虚拟机加载的类信息，常量，静态变量，即时编译后的代码等数据
- 有个别命名为Non-Heap（非堆），当方法区无法满足内存分配时，抛出`OutOfMemoryError`异常
- 对于HotSpot虚拟机而言，在JDK1.8之前，方法区实现为“永久代”，属于堆的逻辑组成部分，`-XX:PermSize `设定初始容量`-XX:MaxPermSize`设定最大容量，在1.8后**类的元信息数据被迁移到“元空间”的新区域**，而静态变量，常量规则存储于堆中，元空间没有使用堆内存，而是分配在本地内存中，**默认情况下其容量只受可用的本地内存的大小限制，**HotShot虚拟机也提供了两个参数来调节其大小，`-XX:MetaspaceSize`用于设定初始容量，`-XX:MaxMetaspaceSize`用于设定最大容量

### 运行时常量池

属于方法区的一部分，`class`文件被加载到内存后，其中的常量池信息（包括符号引用和编译期可知的字面值常量）就被存储于此。 这些信息可通过`javap`解析`class`文件查看，class常量池信息中包含了`MethodAreaRtcp`类所有的符号引用和编译期可知的字面值常量

**常量值**：

1. **字符串面值**：编译器在编译 java 代码时，会将字符串字面值添加到常量池中
2. **用**`final`**修饰的基础类型成员变量的字面值**：注意，必须是用`final`修饰的，而且必须是类的成员变量的字面值才会被编译器添加到常量池中
3. **由字面值常量相加得到的结果**：因为编译器做了优化，多个字面值常量相加后得到的结果，也会被添加到常量池中

*程序执行阶段也会有新的常量加入运行时常量池中*

这部分不属于Class常量池，所以无法通过javap解析的class文件中查看，

- `String.intern()`**方法的返回值会加入运行时常量池中**。`String.intern()`方法调用时，如果常量池中已经存在与其相等的`String`对象（使用`equals`比较时返回`true`），则返回该字符串；否则将该字符串添加到常量池中，然后将其返回
- **基础类型的包装类也用到了常量池技术**。Java的部分基础类型的包装类（`Character`、`Byte`、`Short`、`Integer`、`Long`、`Boolean`）也用到了常量池技术，当使用数值字面值给它们赋值时，它们就会被存储到运行时常量池中。此外，位于运行时常量池中的包装类相加得到的结果，也会被存储在常量池中。值得注意的是，只有在 [-127, 127] 的范围内，包装类才会使用到常量池技术，超过该范围的还是会在堆中存储

## 2.java堆（java Heap）

- 随着JVM的启动而创建，堆是所有内存共享的，是运行时数据区最大的一块区域
- 绝大部分对象（包括类和实例）都在上面存储（我们通过`new`创建出来的对象都分配于此，而且无需主动释放内存，统一由垃圾回收器来进行管理和销毁，堆也是垃圾回收器的主要区域，因此被称为“GC堆”）

### 堆的分代管理

内存回收角度来看JVM对堆进行了分代管理，分成**新生代**和**老年代，**新生代又分为`Eden Space`、`From Survivor Space`、`To Survivor Space`三个区域

分区的主要目的是方便垃圾管理器对对象进行管理。

新生代：所有对象诞生的地方，这个区域上的对象有生命周期短的特点，每次都会被大量回收。

老年代：位于新生代的对象经过几轮垃圾回收机制都存活下来就会转到老年代，另外一些大对象创建时如果新生代没有足够的空闲空间JVM也会将其分配到老年代。（生命周期长或者占用较大空间的对象）

## 3.java虚拟机栈

- JVM会给每个线程都分配一个私有的内存空间称为java虚拟机栈，java虚拟机栈是线程私有的，他的生命周期和线程相同

- 虚拟机栈描述的是java方法执行的内存模型，每个方法执行的同时都会创建一个栈帧，用于存储局部变量表，操作数栈，动态链表，方法出口等信息，JVM只会对其执行两种操作：**栈帧**的**入栈**和**出栈**（java虚拟机是存储栈帧的后进先出的队列）

- 每个方法的执行过程都是栈帧的创建、入栈和出栈。栈帧是用来存储局部数据和部分过程结果的数据结构，每个虚拟机栈中是有单位的，单位就是**栈帧**，一个方法一个**栈帧**。一个**栈帧**主要包含局部变量，操作数栈，动态链接，出口等

  - 局部变量表（LVT）：是一个索引以0开始的字节数组，存储一个方法的所有入参和局部变量，包括8个基本数据类型（`byte`、`char`、`short`、`int`、`long`、`float`、`double`、`boolean`）、对象引用地址、returnAddress类型。（returnAddress中保存的是return后要执行的字节码的指令地址。）

    - 第0个Slot（槽位）固定存储指向方法所属对象的this对象
    - `long`和`double`的栈两个槽位，其余类型占一个
    - LVT按照变量的声明顺序进行存储

  - 操作数栈（OS）：用于在方法运算过程存储其中间的运算结果、方法入参和返回结果，它是一个后进先出的队列。`load`是入栈，`store`是出栈，将运行结果会放入局部变量表中

  - 动态链接：所做的就是根据符号引用所表示名字，转换成对方法或变量的实际引用，从而实现**运行时绑定**（每个栈帧内都包含一个指向当前方法所属类的运行时常量池引用，也称为**符号引用**（Symbolic Reference），用于在类加载阶段对代码进行**动态链接**）

  - 出口：return或者是异常报错

    

## 4.本地方法栈

- 与java虚拟机栈类似，区别在于本地为native服务，java虚拟机栈为java方法服务
- 它是虚拟机栈为虚拟机执行java方法（字节码）的服务
- 本地方法栈可以被固定大小，也可以动态实现扩展和收缩，所以在特定情况会抛出`StackOverflowError`异常和`OutOfMemoryError`异常

## 5.程序计数器

- 程序运行前，JVM会将程序编译后的字节码加载到内存中；程序运行时，字节码解析器会读取内存中的字节码，按照顺序将字节码的指令解析成固定的操作，这个过程中**程序计数器保存当前线程正在执行的字节码指令地址**。
- 对于单线程模型下可有可无，但实际程序都是多线程协作完成（CPU会为每个线程分配一定的时间片，一个线程在其时间片耗尽之后会挂起，直到它再次获得时间片后才会重新运行。为了确保程序正确运行，线程必须从挂起的地方重新执行），程序计数器就可以保证在设计上下文切换的情况下，程序依然能够正确无误的运行下去
- 程序计数器是线程私有的，避免了线程之间的相互影响。JVM会为每个线程都分配一块非常小的内存空间用作程序计数器，这也是唯一一个Java虚拟机规范没有规定`OutOfMemoryError`的运行时区域
- 如果一个线程执行的是native本地方法，它的程序计数器的值为**undefined**。因为JVM在执行native本地方法时，是通过JNI调用本地的其他语言来实现的，而非字节码。

其余文章：

[初始JVM](https://onelibero.love/archives/jvm0)

[JVM各类加载机制](https://onelibero.love/archives/jvm2)

[JVM内存管理和垃圾回收](https://onelibero.love/archives/jvm3)