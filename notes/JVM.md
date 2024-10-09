# 指令集架构

两种：基于栈的指令集架构和基于寄存器的指令集架构

jvm 选择的是基于栈的指令集架构；传统的 x86 系统是基于寄存器的架构

> 个人感觉这两个没什么类比的意义，毕竟 jvm 是虚拟机，明面上是执行字节码指令，但实际上是将字节码指令翻译为机器指令

基于栈的指令集架构比较简单

* 所有的操作不过就是入栈和出栈，不需要考虑分配寄存器的问题
* [零地址指令](#零地址指令)，所以指令集小，但是需要完成同样的操作需要的指令数量更多
* 因为和寄存器无关，天然具有跨平台的特性

基于寄存器的架构：

* 因为依赖寄存器，所以可移植性差
* 因为是一地址指令(反正我看 x86 的汇编码是二地址指令)，所以指令集大，但是完成相同操作的指令数量少

# 零地址指令

> 跨专业的补作业
>
> 参考了[《计算机组成原理》课程笔记（五）——指令的组成与格式 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/357419709)

指令包含了**操作码**和**地址码**两部分；其中操作码表明了当前指令的作用和功能，而地址码指明了操作数的地址或者操作数本身

根据指令中地址码的个数，分为零地址指令、一地址指令、二地址指令、三地址指令、四地址指令

因为 jvm 是基于栈的结构，所以本质上并不需要指明操作数的地址，所有的操作数都在栈顶，所以 jvm 中的指令是零地址指令

四地址指令：格式化表示为：$|op|A_1|A_2|A_3|A_4|$，其中 op 为操作码，$A_1$ 和 $A_2$ 为两个参与运算的操作数的地址(或者操作数本身)，$A_3$ 为结果的保存地址，$A_4$ 为下一条指令的地址

三指令地址：格式化表示为：$|op|A_1|A_2|A_3|$，其中 op 为操作码，$A_1$ 和 $A_2$ 为两个参与运算的操作数的地址(或者操作数本身)，$A_3$ 为结果的保存地址

> 因为没有了下一条指令的地址，所以使用 PC 记录下一条指令的地址，默认 PC 自增

二地址指令：格式化表示为：$|op|A_1|A_2|$，其中 op 为操作码，$A_1$ 和 $A_2$ 为两个参与运算的操作数的地址(或者操作数本身)

> 因为没有了运算结果的存储地址，结果会放在 $A_1$ 或 $A_2$ 中
>
> 反正在之前学汇编的时候，x86 的指令都是这样的，放在 $A_1$中

一地址指令：格式化表示为：$|op|A_1|$，其中 op 为操作码，$A_1$ 为操作数的地址(或者操作数本身)

# 对象创建

*   检查类是否加载, new 对象时, 首先根据符号引用在常量池中检查对应的类是否已经完成了加载 => 加载, 链接, 初始化

*   分配对象内存, 类完成加载后, 其对应需要的内存大小便可以确定, 此时通过: 指针碰撞或空闲列表为对象分配内存

    >   指针碰撞: 一个指针一侧为未分配的内存, 另一侧为已经分配的内存, 为对象分配内存其实就是移动指针
    >
    >   空闲列表: 使用一个列表管理所有空闲的内存, 为对象分配内存时从列表中选择空闲的, 大小满足需求的分配
    >
    >   这两种方式的选择和 GC 是否具有整理功能有关, 如果 GC 可以进行内存整理, 则采用指针碰撞, 否则采用空闲列表 (CMS)

*   初始化零值 

*   设置对象头: mark word (GC, 锁相关), klass word (一个指针, 指向了对应 class 的 meta data)

    >   对于数组类型对象头中还有一个额外的字段存储当前数组的大小 (4 Bytes)
    >
    >   在 32 bit os 中, 对象头大小为 8 Bytes (数组的话是 12 Bytes)
    >
    >   在 64 bit os 中, 对象头大小为 16 Bytes (数组的话是 20 Bytes) => 启用了引用压缩 (compressed reference) 之后 klass word 占用 4 Bytes

*   调用 \<init> (构造方法)

完成 init 的调用后, 对象才完成了创建

对象在内存中, 由三部分组成: header, instance data, padding, 其中 header 主要包括 mark word 与 klass word

在虚拟机栈中, 不考虑逃逸性分析的情况下, 程序都是通过引用的方式访问对象的, 实际的引用可能存在两种情况:

*   句柄引用: 此时对象的引用指向的是句柄池, 而句柄池中保存了对象的实际地址与类信息
*   直接引用: 对象引用指向的就是堆中的对象实例数据

>   使用句柄引用的好处是, 加了一层, 在 GC 进行整理时, 不需要修改句柄引用, 直接修改句柄池就好; 使用直接引用的好处是可以直接定位到对象本身, 减少了一次内存访问的开销

# 类加载子系统

类加载的过程：加载(loading)、链接(linking)、初始化(initialization)，其中链接过程本身又分为验证(verify)、准备(prepare)、解析(resolve)三个过程

类加载子系统，加载字节码文件，class字节码文件，魔数为0xCAFEBABE

> 类加载子系统负责加载，无论字节码文件是否能被执行

加载`class`字节码文件，会在堆区生成对应的`Class`对象，并在方法区，存储类信息，称为`DNA`元数据模板；当`new`对象的时候，就是根据模板实例化对象

> 方法区内还包含有运行时常量池的信息，字面量等

## JVM内存分布

![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/JVM%E5%86%85%E5%AD%98.png)

![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/JVM%E5%86%85%E5%AD%98%EF%BC%88%E5%85%A8%EF%BC%89.png)

## loading

* 通过类的全限定名获取类的二进制字节流
* 将字节流所代表的静态存储结构转化为方法区运行时数据结构（就是`DNA`元数据模板）

* 在内存中生成代表该类的Class对象（堆区）

> 加载 class 文件的方式：
>
> - 从本地系统中直接加载
> - 通过网络获取，典型场景：Web Applet
> - 从 zip 压缩包中读取，成为日后 jar、war 格式的基础
> - 运行时计算生成，使用最多的是：动态代理技术
> - 由其他文件生成，典型场景：JSP 应用
> - 从专有数据库中提取.class 文件，比较少见
> - 从加密文件中获取，典型的防 Class 文件被反编译的保护措施

类的 loading, 由类加载器完成, 具体使用的类加载器通过双亲委派机制确定, 所有被创建的 Class 对象都包含了一个指向对应类加载器的引用 => getClassLoader() 获取

## linking

分为三个过程：验证、准备、解析

* 验证（Verify）：目的在子确保 .class 文件的字节流中包含信息符合当前虚拟机要求，保证被加载类的正确性，不会危害虚拟机自身安全。主要包括:

    *   文件格式验证: 当前字节流需要符合 .class 文件规范, 魔数信息, 版本号符合 jvm 要求; 文件格式验证后, 不会再操作字节码流了
    *   元数据验证: 对当前类的验证, 不允许继承 final 类型的类
    *   字节码验证
    *   符号引用验证: 该过程发生在 linking 的解析阶段 => 将符号引用转化为直接引用; 会验证方法的权限信息

    >   验证阶段可以通过修改 jvm 参数关闭 -Xverify: none

* 准备（Prepare）：

  * 为类变量分配内存并且设置该类变量的默认初始值, 即零值 => 方法区 (meta space)

    > 类变量就是被 `static` 修饰的变量
    >
    > * 类变量在 `prepare` 阶段赋零值，在 `initialization` 阶段赋初值(特别的，对于引用类型赋值为`null`)
    > * `final`修饰的变量在编译时期就已经赋予了零值，在`prepare`阶段会赋予初值
    > * 成员变量在`new`对象的时候赋初值
    >
    > 根据变量的作用域可以将变量分为局部变量和成员变量，成员变量又可以分为类变量(static 修饰)和实例变量；
    >
    > 要注意的是成员变量具有默认初始化操作，类变量是在 linking 阶段中的 prepare 阶段进行的初始化(而在 initialization 阶段进行显式赋值)，而实例变量是在 new 对象的时候进行默认初始化(如果在构造方法中进行显式赋值)；
    >
    > 而局部变量没有默认的初始化操作，在使用前需要手动赋予初值
    
    实际中有些类变量可能在堆区分配, 比如字符串常量
  
* 解析 (Resolve): 将常量池内的符号引用转换为直接引用的过程。

  > 符号引用就是一组符号来描述所引用的目标。符号引用的字面量形式明确定义在《java 虚拟机规范》的 Class 文件格式中。直接引用就是直接指向目标的指针、相对偏移量或一个间接定位到目标的句柄。
  >
  > 解析动作主要针对类或接口、字段、类方法、接口方法、方法类型等。对应常量池中的 CONSTANT_Class_info，CONSTANT_Fieldref_info、CONSTANT_Methodref_info 等。
  >
  > 该过程一般会在`initialization`之后

## initialization

在该阶段 jvm 执行字节码, 调用 `<clinit>` 方法，这里的`<clinit>`和普通的`<init>`方法类似，满足父类的`<clinit>`在子类的`<clinit>`之前，且这个方法也不需要手动定义，这个方法是编译器手动收集类变量的赋值操作和`static`代码块的内容合并得到的

这里特别注意：

* 仅使用了`final`修饰的变量就是常量，他会在编译期赋零值，而在`prepare`阶段赋初值，在`initialization`阶段早就完成了赋值操作

* 仅使用了`static`修饰的变量是类变量，`<clinit>`，就是收集了类变量赋值和`static`代码块部分的内容，比如：

  ```java
  public class InitializationTest {
  	private static int val = 10;
  	static {
  		val = 20;
  	}
  }
  ```

  反编译得到：

  ```shell
  {
    # 这里对应的就是<init>方法
    public com.buzz.loading_system.InitializationTest();
      descriptor: ()V
      flags: ACC_PUBLIC
      Code:
        stack=1, locals=1, args_size=1
           0: aload_0
           1: invokespecial #1                  // Method java/lang/Object."<init>":()V
           4: return
        LineNumberTable:
          line 9: 0
        LocalVariableTable:
          Start  Length  Slot  Name   Signature
              0       5     0  this   Lcom/buzz/loading_system/InitializationTest;
    # 这里对应的就是<clinit>方法
    static {};
      descriptor: ()V
      flags: ACC_STATIC
      Code:
        stack=1, locals=0, args_size=0
           0: bipush        10
           2: putstatic     #2                  // Field val:I
           5: bipush        20
           7: putstatic     #2                  // Field val:I
          10: return
        LineNumberTable:
          line 10: 0
          line 14: 5
          line 15: 10
  }
  ```

* 对于同时使用了`final`和`static`修饰的变量，可以在定义时就进行赋值，此时相当于定义了一个常量；也可以在`static`代码块中赋值，此时表现出类变量的特性，会在`<clinit>`方法中显式赋值，不过要注意，此时也只能赋值一次

  > 所以使用了`final`修饰的变量，可以在`initialization`阶段赋初值

关于类变量，变量的声明可以在`static`代码块之后：

```java
public class InitializationTest {
	static {
		val = 20;
        
        int num = val; // Illegal forward reference
	}
	private static int val = 10;
}
```

此时如果反编译，会发现在`<clinit>`方法中`val`会先被赋值为20，然后被赋值为10；不过要注意，在这种情况下，是不能使用变量`val`的，会报出异常`Illegal forward reference`

`jvm`保证了`<clinit>`方法仅会被调用一次，即正常加锁，就是这里保证了类加载的原子性

> 利用类加载的原子性，实现单例模式，具体的有饿汉式和基于静态内部类的单例

要注意的是`<clinit>`和类变量高度相关，如果一个类中没有类变量，自然也就没有`<clinit>`方法

jvm specification 中规定了 6 中必须执行类初始化的情况:

*   字节码指令: new (初始化一个实例时, 类尚未加载), getstatic (获取类变量), putstatic (设置类变量), invokestatic (调用静态方法)
*   反射有关的调用, Class.forName()
*   加载一个类的时候, 其父类尚未被加载
*   jvm 启动时, 包含了 main 方法的类会被加载

## 卸载

对应了 Class 对象被 GC 回收掉, 要回收 Class 对象需要满足

*   当前类的各个实例对象都被 GC 回收了
*   当前类不能被其他地方引用
*   当前类的 ClassLoader 也被 GC 回收了

## 类加载器

其实严格意义上就分为两类：`bootstrap class loader`和`user-defined class loader`，其中`bootstrap class loader`是由`jvm`提供的，而`user-defined class loader`是抽象类`ClassLoader`的子类

下面的部分来源于`ClassLoader`源码注释：

A class loader is an object that is responsible for loading classes. The class ClassLoader is an abstract class. Given the binary name of a class, a class loader should attempt to locate or generate data that constitutes a definition for the class. A typical strategy is to transform the name into a file name and then read a "class file" of that name from a file system.

> 类加载器是负责加载类的对象(第一句就是废话)。类`ClassLoader`是一个抽象类(第二句也是废话)。
>
> 给定一个类的`binary name`，类加载器会试图定位这个类，并(在方法区)生成定义这个类的数据(DNA元数据模板)。
>
> 这里面注意这个`binary name`，主要在这里定义[Chapter 13. Binary Compatibility](https://docs.oracle.com/javase/specs/jls/se11/html/jls-13.html#jls-13.1)

Every Class object contains a reference to the ClassLoader that defined it.

> 每一个Class对象都保留了对应类加载器的引用

Class objects for array classes are not created by class loaders, but are created automatically as required by the Java runtime. The class loader for an array class, as returned by Class.getClassLoader() is the same as the class loader for its element type; if the element type is a primitive type, then the array class has no class loader.

> 数组类型的Class对象不是由类加载器创建的，而是在运行时自动创建的(这里翻译的有问题)
>
> 调用数组类型的Class对象的getClassLoader()方法获取到的类加载器和数组中每个元素的类加载器相同
>
> 特别的如果数组中的元素为八种基本数据类型，是没有类加载器的
>
> 比如：
>
> ```java
> @Test
> public void test() {
>     int[] nums = new int[10];
>     System.out.println(nums.getClass().getClassLoader()); // null
> }
> ```

Applications implement subclasses of ClassLoader in order to extend the manner in which the Java virtual machine dynamically loads classes.

> 实现了 ClassLoader 子类的可以扩展 jvm 动态加载类的方式

In addition to loading classes, a class loader is also responsible for locating resources. A resource is some data (a ".class" file, configuration data, or an image for example) that is identified with an abstract '/'-separated path name. 

> 除了用于加载类之外，类加载器还用于定位资源。这里的资源可以是一个 .class 文件、配置文件、图片，这些资源通过URI定位。

The ClassLoader class uses a delegation model to search for classes and resources. Each instance of ClassLoader has an associated parent class loader. When requested to find a class or resource, a ClassLoader instance will usually delegate the search for the class or resource to its parent class loader before attempting to find the class or resource itself.

> 类加载器通过 delegation model (这就是我们常说的双亲委派模型)查找类或资源。每个类加载器实例都关联了一个父-类加载器(注意这里是父-类加载器，而不是父类-加载器，因为严格意义上，常见的类型加载器之间并不存在父类和子类的关系)。当请求找一个类或者一个资源的时候，类加载器的实例通常会先委派它的父-类加载器加载对应的类或资源

在 jdk9 引入模块的概念后，三个类加载器发生了变化，主要区别在于 ExtClassLoader 变为了 PlatformClassLoader

这里主要看 [JEP 261: Module System (java.net)](http://openjdk.java.net/jeps/261)(在后面找到 class loaders 部分)

主要是模块化机制，使得原来的 extension 机制被废除了[JEP 220: Modular Run-Time Images (java.net)](http://openjdk.java.net/jeps/220)

* application class loader：现在不再是 URLClassLoader 的一个实现了，而是类 ClassLoaders 的一个内部类，不过功能还是一样，是除了核心类库的默认的类加载器
* platform class loader：也是一个内部类，用来加载一部分 Java SE and JDK 的模块 (大部分还是 bootstrap class loader 加载的)
* bootstrap class loader：还是 jvm 内部实现的，并且调用 getParent() 方法也是获取不到的; 负责加载 java 的基础库, 所有 java.xxx.* 的类都是通过这个类加载器加载的

更为具体的，这三个类加载器的加载范围在上面的链接中有详细的展示，这里不复制了

判断两个 Class 对象是否相同，不仅需要比较两个类的全类名，还需要比较两个类的类加载器是否相同，同时满足时，才可以说明两个类是相同的

class loader 会保存其加载的各个类 (置于字段 classes 中, 在 jdk 17 中是使用 ArrayList 实现的)

如果一个类是通过 user-defined class loader 加载的，那么当这个类被加载到方法区后，会在方法区同时保留指向对应类加载器的引用

> 这也就是为什么 bootstrap class loader 加载的类在 getClassLoader() 时返回值为 null

通过继承 ClassLoader 可以自定义类加载器, 默认的通过修改方法 findClass() 可以实现自定义的类加载器 => jdk 官方文档的例子是通过网络获取二进制流, 自定义的 class loader 加载来自网络的 class

jvm 官方推荐使用双亲委派机制实现类加载, ClassLoader 的 loadClass 方法:

*   先调用 findLoadedClass, 看看当前类是否已经加载过了
*   调用 parent ClassLoader 的 loadClass 方法执行类加载
*   最后调用 findClass 方法执行本 ClassLoader 方法中定义的类加载方法

需要注意的是, parent 可能为 null (PlatformClassLoader), 此时会调用 native 方法 findBootstrapClass 显式调用 BootstrapClassLoader 进行类加载

整个 loadClass 的代码逻辑就是双亲委派, 显然委托父类加载器进行加载, 加载不了再调用当前类加载器的方法进行加载

>   application class loader 并不是 platform class loader 的子类, 二者之间的关系通过子类加载器中的字段 parent 维护

值得庆幸的是方法 loadClass 并没有被 final 修饰, 这意味着这个方法是可以被重写的, 因此自定义的 ClassLoader 可以通过修改 loadClass 方法打破双亲委派机制, 比如 tomcat  就续写了 Class Loader 的委派链, 变得更长了 ...

# 运行时数据区

线程共享的：方法区(meta space)、堆

线程私有的：PC，虚拟机栈、本地方法栈 (所以声明周期和线程保持一致)

> GC 的重点区域为方法区和堆

一个 jvm 实例对应了一个进程，对应内存中的一个运行时数据区，在 java 层面对应了一个 RunTime 对象；jvm 中的线程和操作系统提供的线程具有映射的关系

jvm 自带的后台线程：

* 虚拟机线程：所有线程到达 safe point，此时堆不会发生变化，书面用语为 STW (stop the world) 此时进行垃圾回收、线程栈的收集、偏向锁的撤销等操作
* 周期任务线程
* GC 线程
* 编译线程
* 信号调度线程

在整个运行时数据区中：栈(包括虚拟机栈和本地方法栈)、PC 不存在 GC；且仅有 PC 不会出现 OOM 的情况

## PC (程序计数器)

一种软件实现的逻辑意义上的寄存器，本身是对处理器寄存器的一种模拟；存储的数据为下一条需要执行的字节码指令的地址

占用空间小，在整个运行时数据区中运行速度最快，且在该区域无 GC，也不会出现 OOM

任何时间内，一个线程正在执行的方法称为当前方法，如果当前方法是 java 方法，那么 PC 中存储的是当前方法的字节码指令的地址，而如果当前方法为 native 的方法(c 或 c++)实现，那么此时 PC 中存储的值为 undefined (未定值)

为什么要使用 PC 记录字节码指令的地址：在多线程环境下，处理器需要不断的在线程之间切换，为了保证线程切换回来时可以恢复现场，需要记录各个线程正在执行的字节码指令的地址，也正是这个原因 PC 为定义为线程私有的

## 虚拟机栈

栈中的操作仅有入栈和出栈，所以访问速度也是很快的，仅慢于 PC；虚拟机栈不涉及到 GC

栈中可能出现 OOM 准确来说是：stack over flow 和 out of memory；其中如果设置栈的大小为固定值时，可能出现 stack over flow，而如果设置栈的大小为动态扩展时，可能出现 out of memory

可以在启动参数中加入：-Xss[对应大小]，来指定 jvm 启动后虚拟机栈的大小

> 默认情况下单位为 Byte，当然可以通过 k，m，g 指定单位

栈的基本单位为栈帧：调用方法对应了将栈帧压入操作数栈、方法返回对应了栈帧弹栈

> 有两种方式实现方法返回：
>
> * 正常的方法返回：这个对应了 return，即便是 void 类型的方法也可以具有 return
> * 抛出异常：更精确的说法是抛出的异常并未使用 try-catch 进行捕获

当然栈帧也不是最小的单元，一个栈帧中包含了：操作数栈、局部变量表、动态链接、方法返回地址、一些附加内容

> 一个栈帧中的主要部分为局部变量表和操作数栈，这两部分的大小决定的栈帧的大小

因为虚拟机栈是线程私有的，在一条活跃的线程中，一个时间点上只有一个活动的栈帧，称其为当前栈帧(对应的方法称为当前方法)

> 正是因为虚拟机栈是私有的，所以绝对不允许不同虚拟机栈中的栈帧相互引用
>
> 栈帧中的变量都是局部变量，不需要通过相互引用实现变量共享

### 局部变量表(local variables table)

这部分可以参考 [Chapter 2. The Structure of the Java Virtual Machine (oracle.com)](https://docs.oracle.com/javase/specs/jvms/se17/html/jvms-2.html#jvms-2.6.1)

本身就是一个数组，存储了方法参数，局部变量(方法内部定义的变量)

> 严格意义上除了静态方法，还包含了 this 引用，特别的对于构造方法和非静态方法(实例方法)而言，其局部变量表的下标索引为 0 的地方存储的是 this

这个数组可以存储多种类型的数据，包括：

* 八种基本数据类型
* 对象引用类型
* returnAddress 类型(一个地址，对应了一个字节码指令)

局部变量表中的存储单元为 slot，1 个 32 位的数据会占据一个 slot；这里面要注意的是仅有 long 类型和 double 类型会占据两个 slot，剩下的，包括对象引用类型和方法返回地址，都只占据一个 slot。

jvm 为每个 slot 提供了访问索引，这些索引的先后顺序和变量在方法中的声明顺序有关(当然参数最先，然后是方法中定义的局部变量)，因为 long 类型和 double 类型占据了两个 slot，访问的时候使用小索引访问，比如 long 类型同时占用了 idx 为 1 和 2 的，此时使用 1 引用当前的变量

一个 slot 大小为 32 位，byte 类型的变量(仅占据 8 位)，会被当成 int 类型(占据 32 位)进行存储；其他的比如 short 也是一样的，boolean 类型的也会被认为是 int 类型，取 0 时认为是 false，其他情况都认为是 true

因为局部变量表在虚拟机栈中，保存的是局部变量，所以不存在线程安全问题

> 这里注意，说的是局部变量的安全性，而不是局部变量表中数据的安全性
>
> 方法的参数在方法调用时，默认会占据局部变量表中的位置，比如传递一个对象引用，在方法中进行修改，那么显然这些修改操作并不是线程安全的
>
> 此外还要考虑局部变量是否逃逸，比如方法的返回值为一个引用类型，在方法结束后，多个线程同时操作同一个对象的引用，同样会导致线程安全性问题(不过这种情况下，也不能称为局部变量了)

局部变量表的大小在编译器就可以确定，在整个运行期间，局部变量表的大小是不变的

编译器是很聪明的，它可以实现局部变量表中 slot 的重复利用，在 for 循环中定义的局部变量，在 for 循环外是无意义的，所以编译器会把其对应的 slot 分配给其他的变量：

```java
public void slotReuseTest(int num) {
    for (int i = 0; i < num; i++) {
        long a = 10;
        num += a;
    }
    int b = 10;
}
```

现在考虑这段代码编译得到的字节码中，关于局部变量表的信息，首先可以知道 idx 为 0 的地方存储的是 this；idx 为 1 的地方存储的是 num；idx 为 2 的地方存储的是 i；idx 为 3 的地方存储的 a(且占用了两个 slot，即 slot[3] 和 slot[4] 共同构成了 a)；

如果存在复用，那么局部变量表的最大槽位为 5，且 b 存储在 slot[2]；如果不存在复用，那么 slot[5] 存储的是 b，局部变量表大小为 6

```shell
$ javap -v LocalVariablesTest.class
 LocalVariableTable:
 	Start  Length  Slot  Name   Signature
       11       6     3     a   J
        2      21     2     i   I
        0      27     0  this   Lcom/buzz/vm_stack/LocalVariablesTest;
        0      27     1   num   I
       26       1     2     b   I
```

可以看到 b 和 i 公用了 slot[2]，从 jclasslab 的反汇编的结果上来看，其最大槽位为 5

局部变量表中的变量属于 GC 的根节点(这意味着：被局部变量表直接或间接引用的变量不会被回收)

>   局部变量表可以类比 clox 中的运行时当前栈帧对应的栈信息, 在 clox 中方法调用后, 栈帧被加载, 此时执行栈中默认加载的 slot 就是这里的局部变量表

### 操作数栈(operand stack)

> 称 jvm 的执行引擎是基于栈的，这里说的就是操作数栈

从名字上就可以看出来，栈结构，仅有入栈和出栈两种操作

操作数栈，本身并不是用来存储数据的，仅用来保留计算的中间结果，这片区域用作临时存储

和局部变量表类似，在编译后就可以知道操作数栈的最大深度了，这个在 jclasslab 中也可以看到

此外栈中存储的单元也是 32 bit 的，如果数据大小为 64 bit，那么将占据 2 个深度(还是一样的引用类型和方法返回地址类型都仅占 1 个深度，即 32 位)

需要注意，当调用的方法具有方法返回值时，那么在调用的方法结束后(栈帧弹栈后)，其方法返回值会被压入当前的操作数栈，而无论是否使用变量进行接收，下面举例说明：

首先是具有返回值的方法，这个很简单

```java
public int funWithReturnVal() {
    return 0;
}
```

然后我们需要从一个方法中调用对应的方法，观察其操作数栈的变化，首先使用一个变量 returnVal 接收这个方法返回值：

```java
public void pushReturnValueTest() {
    int returnVal = funWithReturnVal();
}
```

反编译得到的操作数栈的最大深度为 1，其字节码长成这样：

```java
0 aload_0
1 invokevirtual #5 <com/buzz/vm_stack/LocalVariablesTest.funWithReturnVal : ()I>
4 istore_1
5 return
```

这个方法首先 aload_0，把 this 引用放入了操作数栈(从这里也可以看出引用类型仅占 1 个深度，即仅占 32 位)

然后调用方法，注意下面是 istore_1，是将栈顶的元素放在局部变量表中，这里对应了我们定义 returnVal 的部分

如果不使用变量接收返回值，这个返回值也是会被压入操作数栈中的

```java
public void pushReturnValueTest() {
    funWithReturnVal();
}
```

反编译得到操作数栈的最大深度还是 1，其字节码长成这样：

```java
0 aload_0
1 invokevirtual #5 <com/buzz/vm_stack/LocalVariablesTest.funWithReturnVal : ()I>
4 pop
5 return
```

可以看到，最大的区别从 istore_1 变为了 pop，弹出操作数栈的，就是方法返回值

>   clox 并没有区分操作数栈和局部变量表, 函数调用后, 会将 this 关键字与参数列表压入运行时栈 (有点类似操作数栈), 在执行函数的过程中会将局部变量压入操作数栈, 但不弹出
>
>   因此在 clox vm 在进行运算操作时, 每次操作的元素不一定是栈顶元素, 可能是函数入参, 还可能是某些之前的局部变量

#### 栈顶缓存技术

参考博客：[JVM操作数栈之栈顶缓存](https://blog.csdn.net/yangxiaofei_java/article/details/119287202)

因为 jvm 是基于栈的指令集架构，使用零地址指令，完成特定操作需要的指令数目更多，这意味着对内存的读写次数更多，通过将栈顶的操作数缓存到处理器的寄存器中，减少对内存的读写，提高运算速度

如果缓存了 n 个元素，称其为 n-TOS caching；如果缓存带有 n 个状态，称其为 n-state TOS caching

首先考虑简单的 n-TOS caching，假设有一个简单的方法：

```java
public void foo(Object o) {
    Object tmp = o;
} 
```

上述操作将对应两条字节码指令：aload_1 和 astore_2

> 因为 this 占用了局部变量表中的 0 号位，所以参数 o 占据了 1 号位置

* 如果不支持栈顶缓存技术，那么 aload 将执行一次内存读取，一次内存写入的操作；而 astore 将执行一次内存读取，一次内存写入操作；即需要两次内存读写操作
* 如果支持栈顶缓存技术，那么 aload 将执行一次内存读取，一次寄存器写入的操作；而 astore 将执行一次寄存器读取，一次内存写入的操作；即需要一次内存读写和一次寄存器读写

寄存器的读写可比内存读写快多了，而且我们在定义 tmp 变量的时候才进行 astore 操作，如果仅对参数 o 进行运算操作，那么栈顶缓存的优势更加明显

>如果不支持栈顶缓存，运算得到的操作数都将压入操作数栈中，即对操作数栈的读写操作(入栈和出栈)；而如果支持了栈顶缓存技术，运算得到的结果都将缓存到寄存器中，即对寄存器的读写操作

上面的举例中仅缓存了一个操作数，称其为 1-TOS caching，而实际中可能存在多个缓存，这个时候就需要状态的作用了，比如对于 3-TOS caching，需要知道当前已经缓存的元素的个数，以及缓存元素被分配到的寄存器的位置，多个状态可以表示不同的组合

而就算还是 1-TOS caching，也许我们希望通过状态区分不同操作数的类型，HotSpot VM 中使用了 1-TOS caching 和 9-state TOS caching 组合，表示了缓存不同类型的操作数

### 动态链接(dynamic linking)

> 栈帧中的动态链接是一个指针，它指向了运行时常量池中方法的引用(Method references)

![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/dynamic_linking.png)

.java 文件被编译为 .class 文件时，所有的变量和方法引用都作为符号引用保存在常量池中。

实际调用方法时，需要知道对应方法的直接引用(内存地址)，但是并不是所有的方法，在类加载阶段都将符号引用转化为了直接引用。在 java 中调用接口中定义的方法其实最终会变为调用某个实现类中的方法，像这种就属于在编译期不能知道最终调用方法的类型(或者称其为版本)

#### 虚方法和非虚方法

如果在编译器可以确定方法调用的版本(也就是说在运行期不可变)，那么称这种方法为非虚方法；其余的称为虚方法。

典型的非虚方法：

* 在字节码中使用 invokespecial 调用的方法

  > 这里对应了 `<init>`方法、private 的方法和父类方法(使用 super 调用)

* 在字节码中使用 invokestatic 调用的方法

  > 这里对应了静态方法，静态方法不会被继承

* 使用 final 修饰的方法

  > 要注意的是调用 final 修饰的方法，字节码指令为 invokevirtual，但使用 final 修饰的方法有且仅有一个版本，所以并不会被覆盖

剩下的方法都是虚方法，比如：

```java
public class Test {
    public void test (User service) {
        service.add();
    }
}

interface UserService {
    int add();
}

class UserServiceImpl1 implements UserService {
    public void add() {
        return 0;
    }
}

class UserServiceImpl2 implements UserService {
    public void add() {
        return 1;
    }
}
```

在方法 test 中调用了方法 add()，但在编译器不能确定调用方法的具体实现类，所以接口中的方法 add() 是虚方法

可以看到虚方法的调用其实体现了多态性

> 为了实现多态，必须要有：类的继承关系；子类重写父类方法

#### 静态解析和动态链接

字节码文件中存在大量的符号引用，从字节码的反编译的结果可以看到，代码中调用方法其实就是调用符号引用对应的方法；而在实际运用时，会将符号引用转化为直接引用实现方法的调用

静态解析：调用方法在编译期间就可以确定，这类方法会在类加载的解析阶段(或第一次调用时)转化为直接引用，对应的称这种转化的方式为静态解析

动态链接：调用的方法在编译器不能确定(比如调用接口的方法)，每次调用时都需要重新从符号引用变为直接引用，称这中转化的方式为动态链接

> 动态链接体现了多态的特性

#### 方法重写

其实这部分应该是动态链接的表现，不过因为一般的方法都是虚方法，即可以被重写，所以这里将动态链接和方法重写混在了一起

本质上，调用 invokevirtual(invokeinterface) 时发生的过程如下：

* 先找到操作数栈顶的元素对应的类，这里假设为 c
* 在类 c 中(位于方法区)，找到符合的方法名称，并进行权限校验
  * 如果校验不通过直接报出 java.lang.IllegalAccessError
  * 如果通过直接返回对应的方法引用
* 如果类 c 中不存在符合的方法名称，那么会根据继承关系找到 c 对应的父类，然后重复上述过程
* 如果找到顶层父类还是没有找到对应的方法，会抛出 java.1ang.AbstractMethodsrror

#### 虚方法表

根据关于方法重写的描述，我们知道了，执行虚方法(invokevirtual)需要查找类对应的方法，如果该类找不到，就需要找其父类方法，依次类推

现在假设存在若干继承关系，其中最顶层的父类具有方法 test()，当我们调用子类的方法 test()  时本质上调用的还是调用从父类继承得到的方法 test()；如果每次进行方法调用都是上面一套查找逻辑，显然存在多余的查找开销

为了提高性能 jvm 引入了虚方法表，所谓虚方法表，本质上还是一个表(废话)，建立了方法到类的映射关系，这样每次调用某个类的对应方法时，仅需要查表即可

![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/virtual_method_table.png)

类 Cat 是 Object的子类，并重写了方法 toString() 和 finalize()，则对应的虚方法表如上图所示

#### invokedynamic

动态解析调用的方法，并执行，比较典型的例子是 Lambda 表达式

```java
public static void main(String[] args) {
    Thread theard = new Thread(() -> {
        System.out.println("thread");
    });
}

// 字节码
 0 new #2 <java/lang/Thread>
 3 dup
 4 invokedynamic #3 <run, BootstrapMethods #0>
 9 invokespecial #4 <java/lang/Thread.<init> : (Ljava/lang/Runnable;)V>
12 astore_1
13 return
```

这部分可以参考 [Lambda 表达式与invokedynamic](https://www.yuque.com/wanghuaihoho/aw880k/ya8ryy)

>   clox 没有实现动态链接, 因为只有一个保存全局符号的 hash table, 也没有方法区的概念, 默认的在类的声明中, 将类保存在全局符号 hash table 中了 (在程序运行前执行所有类的加载)

### 方法返回地址(return address)

考虑一个简单的场景，即当前虚拟机栈中存在两个栈帧 A 和 B，其中 B 为当前栈帧(即在方法 A 中调用了方法 B)，称 A 为主调方法

在 B 的栈帧中存在一个方法返回地址，这个值就是主调方法 PC 的值

> 所以 return address 本身就是 PC 的值 

方法的结束存在两种方式：正常执行完成退出、出现异常而退出；但只要是方法结束，都需要让程序运行到调用该方法的地方

* 正常执行结束而退出的，可以根据方法返回地址(当初调用方法时 PC 寄存器的值)回到调用方法的位置；此时方法具有返回值(即便是 void 类型的也对应了一条字节码指令 return)，返回值会被压入上一个栈帧中的操作数栈

  返回指令包括了：ireturn(boolean、short、char、int 类型)、lreturn、freturn、dreturn、areturn(引用类型，包括数组类型)、return(void 类型、构造方法(`<init>`方法)、类的初始化(`<clinit>`方法))

* 出现未捕获的异常而退出的，抛出异常的方法不会将方法返回值压入调用该方法的栈帧的操作数栈中

关于两种方法退出的方式，其实是存在疑问的，对于正常退出的情况，方法返回地址保存了主调方法的 PC 中的值，使得当前栈帧出栈后，PC 依旧可以回到主调方法

但是对于异常退出的情况，看到的解释都很模糊，和异常表扯上了关系，说返回地址通过异常表确定，而事实上，异常表中仅包含了那些被捕获的异常，且处理的方式是根据 catch 代码块中的内容确定的，说白了，就是出现异常跳转到执行 catch 块中的部分，但方法可能不会返回；而对于未捕获的异常，则会直接抛出(当前栈帧出栈)，那此时又如何保证程序可以回到主调方法呢

这里唯一可以确定的是，当出现了未捕获的异常时，方法将不会有返回值(不会将返回值压入主调方法的操作数栈中)

#### 异常处理机制

可以参考 [JVM是如何处理异常的](https://www.cnblogs.com/qdhxhz/p/10765839.html)

异常捕获主要针对的就是 try-catch 代码块的内容，这部分会对应字节码文件中异常处理表的部分，这个表可以告诉程序在对应位置出现了异常下一步程序需要如何执行

```java
public void methodA() {
     try {
         methodB();
     }catch (RuntimeException e) {

     }
}

public void methodB() {
	throw new RuntimeException();
}
```

```shell
# 反编译得到方法 A 的异常表
public void methodA();
 descriptor: ()V
 flags: ACC_PUBLIC
 Code:
   stack=1, locals=2, args_size=1
      0: aload_0
      1: invokevirtual #2                  // Method methodB:()V
      4: goto          8
      7: astore_1
      8: return
   Exception table:
      from    to  target type
          0     4     7   Class java/lang/RuntimeException
```

值得注意的是，如果使用了同步代码块，那么编译得到的字节码文件中也会生成异常表

```java
public void methodA() {
     synchronized (this) {
     }
}
```

```shell
public void methodA();
 descriptor: ()V
 flags: ACC_PUBLIC
 Code:
   stack=2, locals=3, args_size=1
      0: aload_0
      1: dup
      2: astore_1
      3: monitorenter
      4: aload_1
      5: monitorexit
      6: goto          14
      9: astore_2
     10: aload_1
     11: monitorexit
     12: aload_2
     13: athrow
     14: return
   Exception table:
      from    to  target type
          4     6     9   any
          9    12     9   any
```

在同步代码块中出现的任何异常都会被正确捕获，并在离开同代码块后抛出(对应字节码指令中 12-13)

## 本地方法栈(native method stack)

本质上和虚拟机栈差不多，这不过这里面的栈帧中的方法是 c / c++ 实现的

所以本地方法栈也是线程私有的，不存在 GC，可能出现 OOM (stack over flow)

和本地方法栈密切相关的：本地方法接口(native method interface)、本地方法库(native method library)

> A native method is a Java method whose implementation is provided by non-java code.
>
> native 的方法是具有方法体的，这是和 abstract 最大的区别

调用 native 方法:会在本地方法栈中进行方法注册，当执行引擎执行到对应方法时，会调用本地方法库中的函数，当执行本地方法时，线程将不再受到 jvm 的限制

要注意的是本地方法可以通过本地方法接口访问 jvm 的运行时数据区

在 HotSpot vm 中虚拟机栈和本地方法栈是一体的

## 堆(heap)

堆是线程共享的，GC 的重点区域，同时可能出现 OOM

虽然堆区是线程共享的，但是每个线程可以在堆区划分线程私有的缓存区(Thread Local Allocation Buffer，TLAB)

堆区是存放对象和数组的空间，但并不是所有的对象(或数组)都存放在堆上(涉及到栈上分配、标量替换...)

堆区中存放的对象，在失去所有的引用后**不会立刻被回收，需要等到发生 GC 时才回收内存空间**

堆空间的大小通过启动参数控制：

* -Xms(-XX:InitialHeapSize)：堆区的初始大小(建议值为 [实际物理内存 / 64])
* -Xmx(-XX:MaxHeapSize)：堆区的最大空间大小(建议值为 [实际物理内存 / 4])

当需要的内存空间超出了堆区最大的内存空间后，将抛出 OOM

jdk 8 之后 jvm 堆结构由三部分组成: 

*   young generation: 包括 eden 区, survivor 0 区, survivor 1 区
*   old generation
*   meta space

默认对象被分配到 eden 区, 在一次 GC 后被放入 survivor 0/1 区 => 如果当前 GC 后被放入了 survivor 0 区, 则下次 GC 后放入 survivor 1 区

每次 GC 后对象的年龄增加, 对象超过一定年龄后被存储到 old generation 中 => 该限最大值为 15, 实际中为动态界限

>   对象在内存中由三部分组成:
>
>   *   header: 包括 mark word 与 klass word (指向对应的 Class 对象), 其中 mark word 使用 4 bit 记录了对象的年龄信息, 因此最大年龄限为 15
>   *   instance data: 对象的实例数据
>   *   padding: 对齐填充 (对齐作用)

基本的分代界限计算思路是, 从小到大计算各个年龄的对象大小之和, 当超出了 survivor 区一半大小后, 此时的年龄为分代年龄

## 方法区

在 jdk 8 之后, 方法区的实现变为了使用 direct memory 的 meta space; 使用了 meta space 后, 解除了 jvm 的内存限制

主要包含了类信息, 类包含的字段, 方法, 常量, 静态变量, 

## 常量池

运行时常量池: .class 文件编译后得到常量池表, jvm 加载运行时将这部分抽象为运行时常量池, 包含了字面量与各种符号引用 => 保存在方法区中

字符串常量池: 主要是各种 string 类型的字面量, 或通过 intern 后得到的 string 类型对象, 保存在堆区中 => 放在堆区而不是 meta space 中, 提高 GC 回收 string 类型的效率

# 垃圾回收

对象首先被分配到堆区的 eden 区中, 如果 eden 空间不足, 则会发起 minor GC; 此时 eden 中幸存的对象会被存放于 survivor 区中 (0/1 交替)

可能出现 survivor 区不够大的情况, 此时对象会直接在 old generation 中分配; 在经过多次 minor GC 后, survivor 中幸存的对象会被分配到 old generation 中

>   将 survivor 中对象移动到 old generation 中的时机是动态调整的, 源码中, 会根据对象的年龄, 从小到大计算对象大小的和, 一旦满足了当前年龄下对象大小大于了 survivor 区的一半大小, 就将该年龄作为门限, 大于这个门限的对象会被移动到 old generation

jvm 中的 gc 分为以下几类:

*   young GC: 只针对 young generation 的 GC => eden, survivor 0/1
*   old GC: 只针对 old generation 的 GC
*   mixed GC: young generation 与一部分 old generation 的 GC
*   full GC: 全部区域, 包括了 meta space

一般而言, 有两种判断对象是否死亡的方法: 引用计数/可达性分析; 一般前者不用, 因为不能解决循环引用的问题

所谓可达性分析, 说的是从 GC root 出发进行标记, 同步标记 GC root 引用的各个对象 (clox 中主要是栈/全局变量表中的对象), jvm 中的 GC root 比较多:

*   栈帧中局部变量表中的引用的对象 (虚拟机栈/本地方法栈)
*   方法区中静态变量引用的对象 (包括常量)
*   被同步锁持有的对象

一般而言不可达的对象就是垃圾了, 可以被回收了, 远古版本中 Object 类中定义了方法 finalize(), 会在对象销毁前调用, 现在这个方法已经被废弃了, 可以认为不可达的对象就是垃圾

垃圾回收算法并没有那么复杂:

*   标记-清除算法 (clox 实现): 最简单的垃圾回收方法, 垃圾直接释放掉就好了, 问题在于可能导致外部碎片问题
*   标记-复制算法: 将所有的内存分为两部分, 每次只用一半, 一次 GC 之后, 将活着的对象复制到另一个区域中; 相比之下进行了对象在内存中占用空间的整理, 显著改善了外部碎片问题, 但是最直接的问题是可用内存大小减半了
*   标记-整理算法: 在标记清除算法的基础上, 会让幸存的对象向一端移动, 同时维护内存边界, 这样边界一侧都是可用对象, 另一次的都是垃圾

没有哪种算法是最好的, 所以 jvm 采用的分代垃圾回收, jvm 的 survivor 的设计显然是参考了标记复制算法; 统计表示, 很多对象其实都活不过 eden, 进入 survivor 的就更少了, 在这种背景下, 采用标记-复制实现 survivor 其实并没有那么慢; 而一旦进入了 old generation, 那么对象轻易就不会死掉了, 对 old generation 采用标记整理算法会更合适

*   在 jvm 中最早的 GC 是 serial GC, 每次工作时会发生 STW, 对 young generation 采用标记-复制算法实现, old generation 采用标记-整理算法实现

*   ParNew 在 serial 的基础上进行了改进, 从名字上能看出来, 针对 young generation 实现了并发的标记-复制算法

*   Parallel Scavenge 针对 old generation 实现了并行的标记-整理算法, 设计目标为最大化吞吐量 (CPU 执行用户程序的时间与 CPU 执行时间的比值)

*   CMS: 首次实现了并发 GC, 整个标记过程分为几个阶段

    *   初始标记: STW, 只标记 GC root
    *   并发标记: 和用户程序同步执行, 从 GC root 出发标记各个对象
    *   重标记: STW, 并发标记, 主要目标为并发标记阶段中用户程序额外创建的对象

    整个垃圾回收阶段也是并发的, 和用户程序同步, 执行标记-清除算法的清除阶段, 显然存在外部碎片问题, 因为清理过程是和用户程序并发进行的, 所以有浮动垃圾

*   G1: 最大的特点是对整个内存区域分为若干个大小相同的 region, 每个 region 都可以是 eden 区, survivor 区, old 区, 这意味着这几个区域不再连续了, 这意味着各个区域的内存标记, 回收可以并发进行 ! young GC 与 mix GC 采用标记-复制算法, full GC 采用标记-整理算法

    *   初始标记 (STW): 仅 GC root, 比较快

    *   并发标记: 与用户程序同步进行, 不考虑耗时

    *   最终标记 (STW): 标记在并发标记阶段变化的对象

    *   清理 (STW): 主要目的是区分出具有包含了存活对象的分区与无存活对象的分区, 不进行对象复制, 很快

    *   复制 (STW): 主要包含了分配新内存与对象复制, 其中瓶颈出现在对象复制阶段, 对象越复杂数量越多耗时越长
    
    G1 相比于 CMS 实现了可预测的停顿时间模型

*   ZGC: 可以看到 G1 的瓶颈出现在了对象复制必须是 STW 的, ZGC 就是想办法实现了并发的对象复制, 主要的技术是指针染色与读屏障

    *   指针染色: ZGC 仅支持 64 bit 的平台, 地址采用 64 bit 使用, 在虚拟地址空间, 仅低 44 bit 作为寻址, 高 4 bit 作为控制信息存在, 最高位的 16 bit 未被使用; 4 个控制 bit 中重要的 3 个 bit: remap bit, m0 bit, m1 bit, 任何对象在任意时刻这三个 bit 只有一个 bit 会被置位, 即每个物理内存上的对象都可能被映射到上述三个虚拟地址上
    *   读屏障: 简单来说就是通过引用获取对象时, ZGC 通过类似 interposition 的技术, 插入控制流, 从而实现修改引用程序访问的引用地址的功能, 比如应用程序访问的引用地址为 0xff 时, 通过读屏障可以将该引用地址修改为 0x3f ...
    
    ZGC 也是在分区的基础上进行垃圾回收的, 向比 G1, ZGC 优化的点就是对象复制, ZGC STW 的对象复制阶段仅复制 GC root, 相比 G1 复制的对象更少, 而被 GC root 引用的对象是与应用程序是并行复制的, 关键点在应用程序访问那些被复制过的对象是, ZGC 通过读屏障修改其原始引用, 定位到最新的地址上
    
    [JVM工作原理与实战(四十)：ZGC原理](https://developer.aliyun.com/article/1502806)

# 字节码

前端编译器：将代码编译为字节码，比较典型的javac

除去javac之外还存在一种常用的前端编译器，即Eclipse中内置的ECJ(Eclipse complier for java)

javac是一种全量编译器，每次会将文件中全部内容进行一次编译

而ECJ是一种增量编译器，每次会将没有编译的部分（新增的）进行编译

> IDEA中使用的javac，所以可能会慢一点点

要注意的是前端编译器不会进行编译优化

> 后端编译器负责优化：JIT
>
> JIT即时编译器会将热点代码翻译成机器指令，比较典型的有循环部分的代码
>
> 与其对应的，解释器：逐条执行字节码指令，并将其翻译为机器指令
>
> AOT（Ahead of time compiler）静态提前编译器，会将字节码直接翻译机器码

一个class类文件对应于一个类或一个接口的定义信息，但字节码不一定是文件的形式，也可以是8字节为单位的二进制字节流

字节码文件没有分隔符，所以字节的顺序，数量是被限制的，具体的那个字节是什么定义都是固定的

字节码文件采用类似结构体的形式存储，其中具有两种数据类型：无符号数和表

* 无符号数：基本数据类型，通过u1表示一个字节，u2表示两个字节，依次类推，还有u4，u8；
* 表：由多个无符号数或者其他表作为数据项构成的复合数据类型，表都是以`_info`结尾，因为表没有固定的大小，为了具体指明大小通常表的前面会有数量说明

## 字节码文件格式

字节码文件结构：

![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/classfile_structure.png)

对应的：

* 魔数：0xcafebabe
* Class文件版本信息：主版本和小版本
* 常量池：计数器用来表示表的大小，后面紧跟个一个表
* 访问标志
* 类索引，父类索引，接口索引集合
* 字段（域）表集合
* 方法表集合
* 属性表集合


<table>
    <tbody>  
        <tr>
            <th></th> 
            <th>类型</th> 
            <th>名称</th> 
            <th>说明</th> 
            <th>长度</th> 
            <th>数量</th> 
       </tr>
       <tr>
            <td>魔数</td>
            <td>u4</td>
            <td>magic</td>
            <td>魔数,识别Class文件格式</td>
            <td>4个字节</td>     
            <td>1</td>
       </tr>
       <tr>
            <td rowspan="2">版本号</td>
            <td>u2</td>
            <td>minor_version</td>
            <td>副版本号(小版本)</td>
            <td>2个字节</td>     
            <td>1</td>
       </tr>
       <tr>
            <td>u2</td>
            <td>major_version</td>
            <td>主版本号(大版本)</td>
            <td>2个字节</td>     
            <td>1</td>
        </tr>	
        <tr>
            <td rowspan="2">常量池集合</td>
            <td>u2</td>
            <td>constant_pool_count</td>
            <td>常量池计数器</td>
            <td>2个字节</td>     
            <td>1</td>
        </tr>
        <tr>
            <td>cp_info</td>
            <td>constant_pool</td>
            <td>常量池表</td>
            <td>n个字节</td>     
            <td>constant_pool_count - 1</td>
        </tr>
        <tr>
            <td>访问标识</td>
            <td>u2</td>
            <td>access_flags</td>
            <td>访问标识</td>
            <td>2个字节</td>     
            <td>1</td>
        </tr>	
        <tr>
            <td rowspan="4">索引集合</td>
            <td>u2</td>
            <td>this_class</td>
            <td>类索引</td>
            <td>2个字节</td>     
            <td>1</td>
        </tr>
        <tr>
            <td>u2</td>
            <td>super_class</td>
            <td>父类索引</td>
            <td>2个字节</td>     
            <td>1</td>
        </tr>
        <tr>
            <td>u2</td>
            <td>interfaces_count</td>
            <td>接口计数器</td>
            <td>2个字节</td>     
            <td>1</td>
        </tr>
        <tr>
            <td>u2</td>
            <td>interfaces</td>
            <td>接口索引集合</td>
            <td>2个字节</td>     
            <td>interfaces_count</td>
        </tr>    
        <tr>
            <td rowspan="2">字段表集合</td>
            <td>u2</td>
            <td>fields_count</td>
            <td>字段计数器</td>
            <td>2个字节</td>     
            <td>1</td>
        </tr>
        <tr>
            <td>field_info</td>
            <td>fields</td>
            <td>字段表</td>
            <td>n个字节</td>     
            <td>fields_count</td>
        </tr>	
        <tr>
            <td rowspan="2">方法表集合</td>
            <td>u2</td>
            <td>methods_count</td>
            <td>方法计数器</td>
            <td>2个字节</td>     
            <td>1</td>
        </tr>
        <tr>
            <td>method_info</td>
            <td>methods</td>
            <td>方法表</td>
            <td>n个字节</td>     
            <td>methods_count</td>
        </tr>
        <tr>
            <td rowspan="2">属性表集合</td>
            <td>u2</td>
            <td>attributes_count</td>
            <td>属性计数器</td>
            <td>2个字节</td>     
            <td>1</td>
        </tr>
        <tr>
            <td>attribute_info</td>
            <td>attributes</td>
            <td>属性表</td>
            <td>n个字节</td>     
            <td>attributes_count</td>
        </tr>	
   <tbody> 
</table>
## 魔数

每一个class文件的开头都是相同的四个字节：`0xcafebabe`，其作用是表示该文件是一个合法的class文件

相比于单纯的基于扩展名的校验文件的方式具有更高的安全性，但其实也就那样

## 版本号

版本号也是四个字节，前面的5，6字节标识编译的副版本号`minor_version`，后面的7，8字节标识编译的主版本号`major_version`

| 主版本（十进制） | 副版本（十进制） | 编译器版本 |
| ---------------- | ---------------- | ---------- |
| 45               | 3                | 1.1        |
| 46               | 0                | 1.2        |
| 47               | 0                | 1.3        |
| 48               | 0                | 1.4        |
| 49               | 0                | 1.5        |
| 50               | 0                | 1.6        |
| 51               | 0                | 1.7        |
| 52               | 0                | 1.8        |
| 53               | 0                | 1.9        |
| 54               | 0                | 1.10       |
| 55               | 0                | 1.11       |

java的主版本号从45开始，编译器为1.1版本的对应于主版本45，还要注意的1.1版本比较特殊，对应于其副版本号为3，其余版本的副版本都是0

内部配置java8

![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/java_-version.png)

随后使用javap命令反编译hello类，得到关于的版本信息：java1.8

![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/javap_version.png)

## 常量池

主要包括常量池大小与保存了符号的数组, 注意到创建的常量池数组大小为 count - 1, 实际中的运行时常量池大小为 count, 区别在于这里保存的常量池不包括运行时常量池下标为 0 处的符号, 常量池中主要保存的是字面量与符号引用

|               类型               | 标志（tag） |          描述          |
| :------------------------------: | :---------: | :--------------------: |
|        CONSTANT_utf8_info        |      1      |   UTF-8 编码的字符串   |
|      CONSTANT_Integer_info       |      3      |       整形字面量       |
|       CONSTANT_Float_info        |      4      |      浮点型字面量      |
|        CONSTANT_Long_info        |      5      |      长整型字面量      |
|       CONSTANT_Double_info       |      6      |   双精度浮点型字面量   |
|       CONSTANT_Class_info        |      7      |   类或接口的符号引用   |
|       CONSTANT_String_info       |      8      |    字符串类型字面量    |
|      CONSTANT_FieldRef_info      |      9      |     字段的符号引用     |
|     CONSTANT_MethodRef_info      |     10      |   类中方法的符号引用   |
| CONSTANT_InterfaceMethodRef_info |     11      |  接口中方法的符号引用  |
|    CONSTANT_NameAndType_info     |     12      |  字段或方法的符号引用  |
|     CONSTANT_MethodType_info     |     16      |      标志方法类型      |
|    CONSTANT_MethodHandle_info    |     15      |      表示方法句柄      |
|   CONSTANT_InvokeDynamic_info    |     18      | 表示一个动态方法调用点 |

常量池表中的每一项大小都不是确定的, 每项的第一个字节通过标志位表示类型

## 访问标识

就是个大小为两个字节的标识位, 比如 ACC_PUBLIC, ACC_SYNCHRONIZED ...



## 一些字节码指令

* `iconst`：就是将`int`类型的常量，压入操作数栈，这里面的常量从`-1`到`5`，如果超过了`5`（或者小于`-1`）就便成了`bipush`

  存入`-1`的时候是`iconst_m1`（minus1?)

  一个典型的例子：`int i = 0`

* `istore`：将一个`int`类型的值存入局部变量表中

  一个典型的例子：`int i = 0`

  变为字节码就是：

  ```shell
   # 将int类型常量0存入操作数栈
   0 iconst_0
   # 将int类型的操作数存入局部变量表中1号位置
   1 istore_1
   # 为什么不存0号位置，因为这个是main方法，0号被args占用了
  ```

* `iload`：将`int`类型的值从局部变量表中取出，放入操作数栈中

  比如`int j = i`（续上面）

  ```shell
   # 从局部变量表中1号位置处取出int类型操作数放入操作数栈
   2 iload_1
   # 将操作数栈中int类型的操作数放入局部变量表中2号位置
   3 istore_2
  ```

* `iinc`：让局部变量表中对应位置的数自增，一个典型的写法：

  `iinc [index],[const]`即让局部变量表中`index`位置处的增加`const`

  我们比较典型的写法有`i++`就对应了`iinc 1 by 1`（注意这里第一个`1`是说`i`存在了局部变量表中下标索引为1的地方，而第二个`1`是让其自增）

  说到这里其实就可以讨论以下[`i++`和`++i`的区别](#`i++`和`++i`)

* `dup`：复制操作数栈顶的值，并压入栈，值得一提的是，在[官方文档](https://docs.oracle.com/javase/specs/jvms/se16/html/jvms-6.html#jvms-6.5.dup)中说明了只有第一类计算结果的值可以通过这个命令得到复制

  所谓第一类计算结果，原文地址[Chapter 2. The Structure of the Java Virtual Machine (oracle.com)](https://docs.oracle.com/javase/specs/jvms/se16/html/jvms-2.html#jvms-2.11.1)

  这里直接复制过来：

  | Actual type     | Computational type | Category |
  | --------------- | ------------------ | -------- |
  | `boolean`       | `int`              | 1        |
  | `byte`          | `int`              | 1        |
  | `char`          | `int`              | 1        |
  | `short`         | `int`              | 1        |
  | `int`           | `int`              | 1        |
  | `float`         | `float`            | 1        |
  | `reference`     | `reference`        | 1        |
  | `returnAddress` | `returnAddress`    | 1        |
  | `long`          | `long`             | 2        |
  | `double`        | `double`           | 2        |

  好吧，其实只要除了`long`和`double`之外的操作数，都可以使用`dup`作为复制

  > 没什么用，我们也不写字节码，这些是用来约束jvm具体实现的

* 

# 四种引用

* 强引用：平时使用的就是强引用:joy:，这些引用属于不可回收的的，当jvm内存空间不足时，这部分引用不会被回收，而是会抛出`OutOfMemeryError`，程序终止

  ```java
  Object object = new Object()
  ```

  上述的`object`就是强引用，如果希望GC可以将其回收，我们可以写成：

  ```java
  object = null;
  ```

  其实就是程序中没有指向堆区中的对象的引用时，对象会被回收

* 软引用：只要内存空间足够，jvm就不会回收这部分空间，在系统将要发生内存溢出之前，将会把这些对象列入回收范围之中进行第二次回收；所以软引用通常作为内存敏感性的缓存

  GC清理软引用时，会将引用放入一个引用队列（reference queue）中

  通过：`SoftReference`可以指定一个软引用

* 弱引用：只要发生了GC，就会回收弱引用，而无论内存空间是否充足；

  和软引用类似的，当弱引用指向的对象被回收时，弱引用回假如指定的引用队列，通过这个引用队列可以跟踪对象回收的情况

  GC进行回收时，针对软引用，需要额外的算法，判断是否回收软引用，而对于弱引用，就直接回收了，所以弱引用的对象显然更容易被回收

  通过`WeakReference`可以指定一个弱引用

* 虚引用：就跟没有一样，调用其get方法也得不到对象，唯一的作用在于跟踪垃圾回收的过程，因为被虚引用的对象在被回收时，可以收到系统通知；

  虚引用在使用的时候必须指定一个引用队列

  可以通过`PhatomReference`指定一个虚引用

# `i++`和`++i`

如果只是单纯的尬写`i++`和`++i`那么他俩从字节码的角度没有任何区别，口说无凭，我们看一下字节码：

首先新建`test.java`

```java
public class test {
	public static void main(String[] args) {
		int i = 0;
		i++;
		++i;
	}
}
```

然后编译java文件后使用javap得到字节码指令

```shell
$ javac test.java
$ javap -c test.class
Compiled from "test.java"
public class test {
  public test();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       # 首先将0存入局部变量表
       0: iconst_0
       1: istore_1
       # 这里对应了i++，简简单单的局部变量表内部的自增
       2: iinc          1, 1
       # 这里对应了++i，和上面完全一样
       5: iinc          1, 1
       8: return
}
```

所以单纯的`i++`和`++i`仅仅对应了`iinc`操作，这一条指令仅仅让操作数栈中的一个位置自增，所以是原子的

然而，这是不是说明单纯的`i++`和`++i`本来也是原子的呢，我的判断是不好说

我们考虑有这样一个工具类：

```java
class MathUtil {
    public static int getInc(int num) {
        return num++;
    }
}
```

你能说他是线程安全的吗，还是一样的，我们通过反编译得到字节码指令：

```shell
$ javac MathUtil.java
$ javap -c MathUtil.class
Compiled from "MathUtil.java"
public class MathUtil {

  public static int getInc(int);
    Code:
       0: iload_0
       1: iinc          0, 1
       4: ireturn
}
```

我们可以看到，我们调用这个函数的操作一共被拆分成了三个指令：

* 先从局部变量表中取出操作数
* 然后将局部变量表中的数自增
* 方法返回

我们可以认为，在局部变量表中自增的这一操作是原子的，但我们不能认为，`++`操作是原子的

同样的，我们在`for`循环中也经常使用`i++`的操作，这里的就可以认为自增是原子的

> 我知道，考虑一个循环中`i++`是不是原子的并没有什么意义，这里就是想说事

```java
public class test {
    public static void fun() {
		for (int i = 0; i < 10; i++);
	}
}
```

```shell
$ javac test.java
$ javap -c test.class
Compiled from "test.java"
public class test {
  public static void fun();
    Code:
       # 0和1就是初始化i
       0: iconst_0
       1: istore_0
       # 从局部变量表中的第0位取出i
       2: iload_0
       # 将10压入操作数栈
       3: bipush        10
       # 比较如果大于等于10跳转到14，即直接返回
       5: if_icmpge     14
       # 自增操作
       8: iinc          0, 1
       # 跳转到2
      11: goto          2
      14: return
}
```

先总结一下：

上面已经证明了，单纯的`i++`和`++i`在编译后变为了`iinc`，`iinc`本身是原子的，而实际引用的时候，我们不会仅仅调用一次`i++`

如果在方法中我们还需要返回，所以自增的方法，不是原子的

然后我们来讨论`i++`和`++i`之间的区别：

```
public class test {
	public static void fun() {
		int i = 0;
		int j = i++;
		int k = ++i;
	}
}
```

```shell
$ javac test.java
$ javap -c test.class
Compiled from "test.java"
public class test {
  public static void fun();
    Code:
       # 0和1是初始化i
       0: iconst_0
       1: istore_0
       # 2-6其实对应的是int j = i++; 首先从局部变量表0的位置取出i
       2: iload_0
       # 然后让局部变量表中0号位也就是i自增
       3: iinc          0, 1
       # 将操作数栈中取出的数，放到局部变量表中的1号位，也就是j
       6: istore_1
       # 7-11对应的是int k = ++i; 首先让局部变量表中0号位自增
       7: iinc          0, 1
       # 然后从局部变量表中的0号位取出放入操作数栈
      10: iload_0
       # 将操作数栈中的操作数放入局部变量表中的2号位
      11: istore_2
      12: return
}
```

综上，我们知道了，为什么`int j = i++`，`j`最后的值和`i`不同，因为我们是先取值，后自增；而`int k = ++i`就不一样了，我们是先自增，后取值



推荐阅读：

* [学习 i++ 和 ++i 的本质区别](https://www.junhaow.com/2018/08/07/035 | 学习 i++ 和 ++i 的本质区别/)
* [iinc指令（jvm官方文档）](https://docs.oracle.com/javase/specs/jvms/se16/html/jvms-6.html#jvms-6.5.iinc)
* [jvm - Is iinc atomic in Java? - Stack Overflow](https://stackoverflow.com/questions/15287419/is-iinc-atomic-in-java/30269266#30269266)



正当我满怀欣喜的认为已经研究透彻了`i++`的本质后，我看到了：[Thread Interference (The Java™ Tutorials > Essential Java Classes > Concurrency) (oracle.com)](https://docs.oracle.com/javase/tutorial/essential/concurrency/interfere.html)

在这里面，他已经明确说了，`i++`这个操作一共分为三步：

* 获取当前的`i`值
* 让获取到的值加一
* 将加一后的值保存到原来的位置`i`

我有点没懂，我已经通过反编译看过字节码指令了，`i++`本质上就是`iinc`，为什么他还要分成三步

我就不服，我把官方的那个写法写下来，然后编译得到一份字节码：

```java
public class test {
	private int num;

	public void inc() {
		num++;
	}
}
```

```shell
$ javac test.java
$ javap -c test.class
Compiled from "test.java"
public class test {
  public void inc();
    Code:
       # 这里从局部变量表的0号位（0号位存入的是this）
       0: aload_0
       # 复制一份
       1: dup
       # 获取到成员变量num
       2: getfield      #7                  // Field num:I
       # 在操作数栈中压入常数1
       5: iconst_1
       # 执行加法操作
       6: iadd
       # 给成员变量（域）赋值
       7: putfield      #7                  // Field num:I
      10: return
}
```

好吧，从字节码的角度考虑，确实他这个方法不是原子的

后来在`StackOverFlow`上乱找，反正就记住：`++`操作不是原子的，不管是实际意义上的，还是说为了和`c`、`c++`保持一致，如果需要同步，就使用`synchronized`关键字

> 但是我现在还是不明白，`iinc`底层上到底是不是原子的，毕竟`jvm`本身也是通过`c`、`c++`实现的
>
> 不过不重要，重要的是，`++`一定不是原子的

# 逃逸分析

在引入了 JIT 后, jvm 会对对象执行逃逸分析, 如果对象没有逃逸到方法外, 则可以采用标量替换的方式实现栈上对象的分配, 此时对象不再保存在堆区中

# JMM

即java内存模型

## 类比一下物理机

现代处理器的运行速度是很快的，有的时候运算的速度的瓶颈其实在于读写速度上

所以为了尽可能匹配处理器的运算速度，再内存和处理器之间引入了高速缓存

> 准确而言，可以借用一下CSAPP的一个图
>
> ![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/%E5%AD%98%E5%82%A8%E5%99%A8%E5%B1%82%E6%AC%A1%E7%BB%93%E6%9E%84.png)

这样处理器从内存中拷贝数据到高速缓存中，进行运算操作，运算结束后再将其写回内存

然而，在多核心的条件下，会出现缓存一致性的问题；处理器的每个核心多是相互独立的，即每个核心具有自己的高速缓存，运算时从各自的缓存中取数据，而每个核心又共享主内存

![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/cpu_cache.jpg)

实际中，当处理器从主内存中存取数据的时候需要遵循统一的协议（一些复杂的缓存一致性协议）

此外处理器还会对指令进行乱序优化，处理器会在计算之后将乱序执行的结果重组，保证该结果与顺序执行的结果是一致的，但并不保证程序中各个语句计算的先后顺序与输入代码中的顺序一致，因此，如果存在一个计算任务依赖另外一个计算任务的中间结果，那么其顺序性并不能靠代码的先后顺序来保证

## jvm的内存模型

Java内存模型的主要目标是定义程序中各个变量的访问规则，即在虚拟机中将变量存储到内存和从内存中取出变量这样的底层细节。

注意这里考虑的变量主要考虑的是多线程访问下，可能存在安全问题的变量，即包括了：实例字段、静态字段和构成数组对象的元素；排除了局部变量和方法参数；

![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/JMM.jpg)

这里的主内存可以认为是物理内存，每个线程工作时认为直接访问的是工作内存

之前学单例模式的时候知道了，jvm可有类似指令乱序执行的毛病，不过这里主要是字节码指令了

### 8种原子操作

jmm模型种规定了8种操作，在jvm具体的实现上，必须保证这八种操作的原子性

- lock（锁定）：作用于主内存的变量，它把一个变量标识为一条线程独占的状态。
- unlock（解锁）：作用于主内存的变量，它把一个处于锁定状态的变量释放出来，释放后的变量才可以被其他线程锁定。
- read（读取）：作用于主内存的变量，它把一个变量的值从主内存传输到线程的工作内存中，以便随后的load动作使用。
- load（载入）：作用于工作内存的变量，它把read操作从主内存中得到的变量值放入工作内存的变量副本中。
- use（使用）：作用于工作内存的变量，它把工作内存中一个变量的值传递给执行引擎，每当虚拟机遇到一个需要使用到变量的值的字节码指令时将会执行这个操作。
- assign（赋值）：作用于工作内存的变量，它把一个从执行引擎接收到的值赋给工作内存的变量，每当虚拟机遇到一个给变量赋值的字节码指令时执行这个操作。
- store（存储）：作用于工作内存的变量，它把工作内存中一个变量的值传送到主内存中，以便随后的write操作使用。
- write（写入）：作用于主内存的变量，它把store操作从工作内存中得到的变量的值放入主内存的变量中。

举个简单的例子，如果线程需要从主内存中读取变量，需要进行`read`和`load`两个操作；类似的如果需要将工作内存中的变量保存到主内存中，也需要进行`store`和`write`两个操作

此外，在`jvm`进行上述操作时还需要满足一定的规则：

- 不允许read和load、store和write操作之一单独出现，即不允许一个变量从主内存读取了但工作内存不接受，或者从工作内存发起回写了但主内存不接受的情况出现。
- 不允许一个线程丢弃它的最近的assign操作，即变量在工作内存中改变了之后必须把该变化同步回主内存。
- 不允许一个线程无原因地（没有发生过任何assign操作）把数据从线程的工作内存同步回主内存中。
- 不允许在工作内存中直接使用一个未被初始化（load或assign）的变量。换句话说，就是对一个变量实施use、store操作之前，必须先执行过了assign和load操作。
- 一个变量在同一个时刻只允许一条线程对其进行lock操作，但lock操作可以被同一条线程重复执行多次，多次执行lock后，只有执行相同次数的unlock操作，变量才会被解锁。
- 如果对一个变量执行lock操作，那将会清空其他线程的工作内存中此变量的值，在执行引擎使用这个变量前，需要重新执行load或assign操作初始化变量的值。
- 如果一个变量事先没有被lock操作锁定，那就不允许对它执行unlock操作，也不允许去unlock一个被其他线程锁定住的变量。
- 对一个变量执行unlock操作之前，必须先把此变量同步回主内存中（执行store和write操作）
