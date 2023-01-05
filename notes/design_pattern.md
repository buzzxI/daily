稍微看一下设计模式
# 单例模式
一个类就一个示例

有的时候会看见单例的类被`final`修饰，这是为了防止子类破坏父类的单例，因为按理来说，父类是单例，子类应该也是单例。

> 但这这种`final`的做法确实不够优雅

## 饿汉式

将构造器私有化，然后创建一个类的实例化对象
对外暴露一个静态方法用于获取该实例化对象
```java
public class SingleTon {
	//私有化构造器
	private SingleTon(){
	}
    //创建类的实例化对象
	private static SingleTon singleTon;
	static {
		singleTon = new SingleTon();
    }
    //该类仅暴露一个静态方法用户获取实例化对象
	public static SingleTon getInstance() {
		return singleTon;
	}
}
```
使用饿汉式可以：
* 使用静态常量，将实例化对象写死为final类型，在编译期间就确定对象
* 使用静态代码块，在类加载的时候实例化对象
特点：
* 不是lazyLoading：在类加载的时候就已经进行了实例化，比较浪费内存
* 但正是因为使用了类加载的机制，导致不会出现线程安全的问题，不需要额外加锁，效率比较高
* 实现比较简单

## 懒汉式
反正都已经懒了，肯定是lazyLoading的，只有调用`newInstance()`方法的时候才进行对象的实例化

### 线程不安全的
这个最好想了，就是在调用`newInstance()`方法再进行对象的实例化就行，显然线程不安全
```java
public class SingleTon {
	private SingleTon(){
	}

	private static SingleTon singleTon;
	public static SingleTon getInstance() {
		if (singleTon == null) {
			singleTon = new SingleTon();
		}
		return singleTon;
	}
}
```
特点：
* 实现了lazyLoading，并且实现难度也低
* 但是就是不能多线程工作

### 线程安全的
其实没什么区别，既然上面的方法线程不安全，现在我加一个synchronized关键字，同步一下不就好了吗

```java
public class SingleTon {
	private SingleTon(){
	}

	private static SingleTon singleTon;
	public synchronized static SingleTon getInstance() {
		if (singleTon == null) {
			singleTon = new SingleTon();
		}
		return singleTon;
	}
}
```
特点：
* lazyLoading，并且实现简单
* 实现了线程同步，但是效率太低了，实际工作中多线程环境下，并不会频繁调用`newInstance`方法

上面通过同步方法实现了多线程安全，但效率就变低了
我们假想的最理想的情况是，在对象为null的时候把对象new出来，其余的时候我们不需要保证多线程的同步，直接把对象返回就好了
一个很正常的想法是使用同步代码块提高效率
```java
public class SingleTon {
	private SingleTon(){
	}

	private static SingleTon singleTon;
	public static SingleTon getInstance() {
		if (singleTon == null) {
			synchronized (SingleTon.class) {
				singleTon = new SingleTon();
			}
		}
		return singleTon;
	}
}
```
但是很遗憾，上面的代码并不理想，因为有可能出现一个线程进入if判断后让出控制权
然后第二个线程进入if判断，此时因为对象没有实例化，所以两次if判断均成立
随后一个线程进入同步代码块，实例化对象，随后让出锁对象，即Class对象
而另外要给线程拿到锁对象后会再次实例化对象，这样对象就会被实例化多次，就不是单例模式了

这个怎么办？

## 双重检查(DCL)
即Double checked-locking pattern
我们将上方的代码改写：
```java
public class SingleTon {
	private SingleTon(){
	}

	private static SingleTon singleTon;
	public static SingleTon getInstance() {
		if (singleTon == null) {
			synchronized (SingleTon.class) {
				if (singleTon == null) {
					singleTon = new SingleTon();
				}
			}
		}
		return singleTon;
	}
}
```
可以看到我们对对象是不是null进行了两次判断，第一次因为没有加锁，所以多个线程可能会同时进入
但是我们在第一次判断内部添加了锁，从而第二次判断时线程是同步的
通过这种形式，就算多个线程同时进入第一个if判断，最终也只会有一个线程对对象进行实例化
两次if，两次检查，即double check

然而上面的写法还是存在问题的，这里主要参考：
* [如何正确的写出单例模式](http://wuchong.me/blog/2014/08/28/how-to-correctly-write-singleton-pattern/) 
* [单例模式](https://www.cnblogs.com/blknemo/p/13254763.html)

问题在于`sigleTon = new SingleTon()`，这句话乍一看没什么问题，就是new一个对象而已
但是在java中new对象并不是一个原子的操作，以上面这个new对象为例它其实一共做了三件事：
* 在内存中开辟一段空间，为即将new出来的对象分配内存（个人感觉主要是在堆区中）
* 调用SingleTon的构造方法初始化对象
* 将singleTon引用指向内存中新开辟的空间

在完成了最后一步，即将引用指向内存中的一个地址后，引用就不是null了
然而在实际中JVM存在指令重排序的优化，即上面的第二条和第三条的顺序是不确定的
执行顺序可能是：`1-2-3`，也可能是`1-3-2`
如果是后者，那么假设在singleTon引用此时已经不是null了，但是对象还没有初始化
如果此时有一个新的线程进入这个方法，并进行外边的第一个检查，会直接返回这个引用
如果我们调用singleTon的方法，会报错
> 一个很奇怪的情况，就是引用已经不是null了，但是对象还没有初始化
> 此时调用方法，肯定会报错

所以改进的方法是在对象上面加上`volatile`进行修饰

添加了这个关键字的变量，当其值发生变化的时候，会将修改值立刻更新到主存。
但这里对关键的地方在于如果变量使用了`volatile`进行修饰，相当于告诉JVM禁止指令重排序优化

引用参考博客中的说法：
**在 volatile 变量的赋值操作后面会有一个内存屏障（生成的汇编代码上），读操作不会被重排序到内存屏障之前。
比如上面的例子，取操作必须在执行完 1-2-3 之后或者 1-3-2 之后，不存在执行到 1-3 然后取到值的情况。**

然而还有一个小问题，就是在jdk5之前的版本就算使用了`volatile`也不能保证指令的重排序

> 额外的提一下，这里是另一个博客中提到的：
> 当比较两个对象是否相等的时候，通常会有两个方法`hashCode()`和`equals()`方法。
> 从实现的复杂程度上，`equals()`方法的效率较低，而`hashCode()`因为仅计算一个hash值，效率肯定更高。
> 一般情况下，我们比较对象是否相等使用`hashCode()`方法就足够了，但是实际中也存在`hashCode()`一致，但对象不同的情况
> 所以，为了兼顾效率和正确性，我们一般的写法是：
> 先调用`hashCode()`方法，如果二者的`hash`值不同，直接返回不相同就好了，
> 而如果二者的`hash`值相同，就需要调用`equals()`方法了
> 这里面的`hashCode()`和`equals()`方法就可以类比于我们的double check：
> 因为对象仅new一次就好了，所以一般singleTon引用不是null，所以不需要同步，
> 而一旦涉及到实例化对象，为了保证正确性，又需要做同步

## 静态内部类

```java
public class SingleTon {
	private SingleTon(){
	}

	private static class SingleTonHolder{
		private static SingleTon singleTon;
		static {
			singleTon = new SingleTon();
		}
	}

	public static SingleTon newInstance() {
		return SingleTonHolder.singleTon;
	}
}
```
当外部类加载的时候不会同时加载静态内部类，只有当第一次调用的方法`newInstance()`的时候才会执行类的加载。
这种做法实现了lazy-loading，同时在多线程下保证了线程安全，实例对象也是唯一的。

lazy-loading这个好理解，现在关键点在于如何保证线程的安全的，
参考：[静态内部类 单例模式](https://blog.csdn.net/mnb65482/article/details/80458571)

JVM对类进行初始化的场景：（一共就5个）
* 字节码指令为`new,getsatic,setstatic,invokestatic`时，这四个指令分别对应于：
    * new一个实例化对象
    * 读取一个静态字段
    * 设置一个静态字段
    * 调用类的静态方法
> 注意，这里的静态字段不能使用final修饰，不然就成常量了，那就不是在运行时确定的的了，而是在编译时期就已经放到常量池中了
* 使用`java.lang.reflect`包下的方法对类进行反射调用
* 初始化一个类，而其父类还没有初始化
* JVM启动的时候，会默认初始化用户执行的主类，即含有main方法的类
* 当使用JDK 1.7等动态语言支持时，如果一个java.lang.invoke.MethodHandle实例最后的解析结果REF_getStatic、REF_putStatic、REF_invokeStatic的方法句柄，并且这个方法句柄所对应的类没有进行过初始化，则需要先触发其初始化。
> 最后一点看不懂，就直接粘过来了

如果我们查看编译后的字节码文件，可以看到，`newInstance()`方法长这样：
```byte code
 0: getstatic     #2 // Field com/buzz/SingleTon$SingleTonHolder.singleTon:Lcom/buzz/SingleTon;
 3: areturn
```
也即我们调用`SingleTonHoler.singleTon`相当于读取一个静态字段，此时类`SingleTonHolder`被初始化。

那么现在问题在于初始化就可以保证线程安全性了吗，确实可以，这是JVM保证的
类的初始化时调用方法，`<clinit>`，JVM保证这个方法在多线程环境中可以被正常的加锁，同步。
当多个线程同时初始化一个类，最终也只会有一个线程执行该类的`<clinit>`方法，而其他的线程会进入阻塞状态，直到活动线程执行完`<clinit>`。
如果一个类的`<clinit>`方法中具有许多耗时的工作，那么**多个线程将进入阻塞**

> JVM的特性保证了同一个类加载器下，同一个类仅会被加载一次

## 枚举

这里主要参考：[深入理解java枚举类型](https://blog.csdn.net/javazejian/article/details/71333103)

```java
public enum SingleTon {

	/**
	 * 单例对象
	 */
	INSTANCE;
	SingleTon() {
        System.out.println("初始化");
    }
}
```

枚举类中定义一个对象，从而实现单例

将上面的进行反编译得到：

```shell
Classfile /home/buzz/java_projects/simple_java/test/SingleTon.class
  Last modified May 14, 2022; size 898 bytes
  MD5 checksum 8bb5cd0ecad79c11318c540bf9e0c2cc
  Compiled from "SingleTon.java"
  # 这里可以看到关键字enum其实包含了一层继承关系，即继承了Enum类
public final class SingleTon extends java.lang.Enum<SingleTon>
  minor version: 0
  major version: 55
  flags: (0x4031) ACC_PUBLIC, ACC_FINAL, ACC_SUPER, ACC_ENUM
  this_class: #4                          // SingleTon
  super_class: #13                        // java/lang/Enum
  interfaces: 0, fields: 2, methods: 4, attributes: 2
Constant pool:
   #1 = Fieldref           #4.#32         // SingleTon.$VALUES:[LSingleTon;
   #2 = Methodref          #33.#34        // "[LSingleTon;".clone:()Ljava/lang/Object;
   #3 = Class              #17            // "[LSingleTon;"
   #4 = Class              #35            // SingleTon
   #5 = Methodref          #13.#36        // java/lang/Enum.valueOf:(Ljava/lang/Class;Ljava/lang/String;)Ljava/lang/Enum;
   #6 = Methodref          #13.#37        // java/lang/Enum."<init>":(Ljava/lang/String;I)V
   #7 = Fieldref           #38.#39        // java/lang/System.out:Ljava/io/PrintStream;
   #8 = String             #40            // 初始化操作
   #9 = Methodref          #41.#42        // java/io/PrintStream.println:(Ljava/lang/String;)V
  #10 = String             #14            // INSTANCE
  #11 = Methodref          #4.#37         // SingleTon."<init>":(Ljava/lang/String;I)V
  #12 = Fieldref           #4.#43         // SingleTon.INSTANCE:LSingleTon;
  #13 = Class              #44            // java/lang/Enum
  #14 = Utf8               INSTANCE
  #15 = Utf8               LSingleTon;
  #16 = Utf8               $VALUES
  #17 = Utf8               [LSingleTon;
  #18 = Utf8               values
  #19 = Utf8               ()[LSingleTon;
  #20 = Utf8               Code
  #21 = Utf8               LineNumberTable
  #22 = Utf8               valueOf
  #23 = Utf8               (Ljava/lang/String;)LSingleTon;
  #24 = Utf8               <init>
  #25 = Utf8               (Ljava/lang/String;I)V
  #26 = Utf8               Signature
  #27 = Utf8               ()V
  #28 = Utf8               <clinit>
  #29 = Utf8               Ljava/lang/Enum<LSingleTon;>;
  #30 = Utf8               SourceFile
  #31 = Utf8               SingleTon.java
  #32 = NameAndType        #16:#17        // $VALUES:[LSingleTon;
  #33 = Class              #17            // "[LSingleTon;"
  #34 = NameAndType        #45:#46        // clone:()Ljava/lang/Object;
  #35 = Utf8               SingleTon
  #36 = NameAndType        #22:#47        // valueOf:(Ljava/lang/Class;Ljava/lang/String;)Ljava/lang/Enum;
  #37 = NameAndType        #24:#25        // "<init>":(Ljava/lang/String;I)V
  #38 = Class              #48            // java/lang/System
  #39 = NameAndType        #49:#50        // out:Ljava/io/PrintStream;
  #40 = Utf8               初始化操作
  #41 = Class              #51            // java/io/PrintStream
  #42 = NameAndType        #52:#53        // println:(Ljava/lang/String;)V
  #43 = NameAndType        #14:#15        // INSTANCE:LSingleTon;
  #44 = Utf8               java/lang/Enum
  #45 = Utf8               clone
  #46 = Utf8               ()Ljava/lang/Object;
  #47 = Utf8               (Ljava/lang/Class;Ljava/lang/String;)Ljava/lang/Enum;
  #48 = Utf8               java/lang/System
  #49 = Utf8               out
  #50 = Utf8               Ljava/io/PrintStream;
  #51 = Utf8               java/io/PrintStream
  #52 = Utf8               println
  #53 = Utf8               (Ljava/lang/String;)V
{
# 这里可以看到定义的枚举类对象其实是本身就是该类的一个成员变量，使用枚举实现单例b
  public static final SingleTon INSTANCE;
    descriptor: LSingleTon;
    flags: (0x4019) ACC_PUBLIC, ACC_STATIC, ACC_FINAL, ACC_ENUM

  public static SingleTon[] values();
    descriptor: ()[LSingleTon;
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=1, locals=0, args_size=0
         0: getstatic     #1                  // Field $VALUES:[LSingleTon;
         3: invokevirtual #2                  // Method "[LSingleTon;".clone:()Ljava/lang/Object;
         6: checkcast     #3                  // class "[LSingleTon;"
         9: areturn
      LineNumberTable:
        line 5: 0

  public static SingleTon valueOf(java.lang.String);
    descriptor: (Ljava/lang/String;)LSingleTon;
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=1, args_size=1
         0: ldc           #4                  // class SingleTon
         2: aload_0
         3: invokestatic  #5                  // Method java/lang/Enum.valueOf:(Ljava/lang/Class;Ljava/lang/String;)Ljava/lang/Enum;
         6: checkcast     #4                  // class SingleTon
         9: areturn
      LineNumberTable:
        line 5: 0

  static {};
    descriptor: ()V
    flags: (0x0008) ACC_STATIC
    Code:
      stack=4, locals=0, args_size=0
         0: new           #4                  // class SingleTon
         3: dup
         4: ldc           #10                 // String INSTANCE
         6: iconst_0
         7: invokespecial #11                 // Method "<init>":(Ljava/lang/String;I)V
        10: putstatic     #12                 // Field INSTANCE:LSingleTon;
        13: iconst_1
        14: anewarray     #4                  // class SingleTon
        17: dup
        18: iconst_0
        19: getstatic     #12                 // Field INSTANCE:LSingleTon;
        22: aastore
        23: putstatic     #1                  // Field $VALUES:[LSingleTon;
        26: return
      LineNumberTable:
        line 6: 0
        line 5: 13
}
Signature: #29                          // Ljava/lang/Enum<LSingleTon;>;
SourceFile: "SingleTon.java"
```

## 破坏单例模式

综上我们知道了，为了实现单例模式，我们有以下几种写法：
* 使用饿汉式
* 使用带有双重检查的懒汉式（注意一定要带上volatile关键字）
* 使用静态内部类
* 使用枚举

然而，反射和序列化有可能破坏单例

要注意的是使用枚举实现单例可以避免出现反射或序列化导致单例被破坏

### 反射

```java

import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationTargetException;

public class Main {
  public static void main(String[] args) {
    Class<SingleTon> singleTonClass = SingleTon.class;
    SingleTon singleTon = SingleTon.newInstance();
    SingleTon reflectSingleTon = null;
    try {
      Constructor<SingleTon> declaredConstructor = singleTonClass.getDeclaredConstructor();
      declaredConstructor.setAccessible(true);
      reflectSingleTon = declaredConstructor.newInstance();
    } catch (Exception e) {
      e.printStackTrace();
    }
    System.out.println(singleTon == reflectSingleTon);//false
  }
}
```
这是因为我们强行调用了私有的构造方法，强行new了一个对象，所以显然，这破坏了单例

解决办法是在构造器中进行条件判断，如果出现多次调用构造器，就抛出异常
```java
public class SingleTon {
	private static boolean flag = true;

	private SingleTon(){
		//注意这里必须加锁
		synchronized (SingleTon.class) {
			if (flag) {
				flag = false;
			}else {
				throw new RuntimeException("more than one instance");
			}
		}
	}

	private static class SingleTonHolder{
		private static SingleTon singleTon;
		static {
			singleTon = new SingleTon();
		}
	}

	public static SingleTon newInstance() {
		return SingleTonHolder.singleTon;
	}
}
```
但其实上面的这种做法存在一个问题，即对于双重加锁的懒汉式，和静态内部类，他们实现了懒加载。
那如果我先通过反射获取实例化对象，此后再调用`newInstance()`获取对象反而会报错，错误的获取对象不报错，但调用正确方法的却报错

> 饿汉式没有这个问题，是因为饿汉式在类加载的时候就进行了初始化，类加载是原子的，肯定不会出现上述的小bug。
> 但我认为其实是这样也没什么问题，毕竟我们还是实现了，只要出现多个实例化对象（破坏单例）就报错

关于构造器中加锁，我是这么考虑的：
如果不加，我们假设一个情景，多个线程同时通过反射创建实例，这样一个线程执行构造方法的时候，先进行条件判断，条件满足进入if，随后在更改flag标志前让出控制权，此时另一个线程也会进入if条件，这样这两个线程都可以通过反射实例化对象，在不报错的情况下，破坏了单例

### 序列化

```java
package com.buzz;

import java.io.*;

public class Main {
	public static void main(String[] args) {
		try {
			ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream("classpath:singleTon"));
			out.writeObject(SingleTon.newInstance());

			ObjectInputStream in = new ObjectInputStream(new FileInputStream("classpath:singleTon"));
			SingleTon singleTon = (SingleTon) in.readObject();
			System.out.println(singleTon == SingleTon.newInstance());//false
		} catch (Exception e) {
			e.printStackTrace();
		}
	}
}
```
问题就出现在使用ObjectInputStream调用了方法：`readObject()`

这里参考了：[单例模式的破坏](https://www.jianshu.com/p/57613ff96aa5)

又到了痛苦的看源码环节，首先进入`readObject()`方法：
```java
public class ObjectInputStream{
	/*
        这里是一些没用的方法
	 */
    public final Object readObject()
            throws IOException, ClassNotFoundException
    {
      if (enableOverride) {
        return readObjectOverride();
      }
  
      // if nested read, passHandle contains handle of enclosing object
      int outerHandle = passHandle;
      try {
		  //重点是它调用了一个readObject0()的方法
		  Object obj = readObject0(false);
        /*
            这里不关键
         */
        return obj;
      } finally {
        passHandle = outerHandle;
        if (closed && depth == 0) {
          clear();
        }
      }
    }

  private Object readObject0(boolean unshared) throws IOException {
    /*
        这里都不关键
     */
    try {
		//下面就是判返回的类型
      switch (tc) {
		  //null
        case TC_NULL:
          return readNull();
        //引用类型
        case TC_REFERENCE:
          return readHandle(unshared);
		  //class类型
        case TC_CLASS:
          return readClass(unshared);
		  //代理类型
        case TC_CLASSDESC:
        case TC_PROXYCLASSDESC:
          return readClassDesc(unshared);
		  //string
        case TC_STRING:
        case TC_LONGSTRING:
          return checkResolve(readString(unshared));
            //数组
        case TC_ARRAY:
          return checkResolve(readArray(unshared));
            //枚举
        case TC_ENUM:
          return checkResolve(readEnum(unshared));
            //Object，对象类型，我们会进入这个方法，所以关键在于readOrdinaryObject方法
        case TC_OBJECT:
          return checkResolve(readOrdinaryObject(unshared));
            //异常
        case TC_EXCEPTION:
          IOException ex = readFatalException();
          throw new WriteAbortedException("writing aborted", ex);
		  /*
              后面的不认识了
		   */
      }
    } finally {
      depth--;
      bin.setBlockDataMode(oldMode);
    }
  }

  private Object readOrdinaryObject(boolean unshared)
          throws IOException
  {
    /*
        不关键
     */
    
    /*
        判断当前对象是否可以实例化，如果可以调用newInstance方法进行实例化，否则为null
        这里其实是反射，其实是有疑问的，
        如果是通过反射获取到的对象，那么它应该也是通过构造器实例化对象的，但是我们已经在单例的类中设置了flag位
        在main程序中，一次通过反射获取对象，一次通过真是调用getInstance实例化对象，按理说肯定会有问题
        不过如果实际debug的话会发现，它这个newInstance方法，其实并不是直接的反射，而是选择了一个构造器进行对象的创建
        Object newInstance()
        throws InstantiationException, InvocationTargetException,
               UnsupportedOperationException
        {
        requireInitialized();
        if (cons != null) {
            try {
                if (domains == null || domains.length == 0) {
                    return cons.newInstance();
                    ...
        这里面cons是一个构造器，不过它的类型是Object类型
        所以到这里其实才真正知道了，原来它是通过Object类的构造器实例化对象的
        不过最后序列话的对象确实是SingleTon类型
        这里其实还是有疑问的，那就是你既然是通过Object的构造器产生的对象，那么你又是怎么转变为SingleTon类型的呢
        ？？？
     */
    Object obj;
    try {
      obj = desc.isInstantiable() ? desc.newInstance() : null;
    } catch (Exception ex) {
      throw (IOException) new InvalidClassException(
              desc.forClass().getName(),
              "unable to create instance").initCause(ex);
    }

    /*
        不关键
     */
    
    /*
        这里再次是一个关键区域，它首先判断当前这个序列化的对象是否具有readResolve()方法
        如果有的话，就调用这个readResolve方法，返回一个新的对象，代替当前这个序列化对象返回
        所以我们就需要从这个位置下手，阻止序列化破坏单例
     */
    if (obj != null &&
            handles.lookupException(passHandle) == null &&
            desc.hasReadResolveMethod())
    {
      Object rep = desc.invokeReadResolve(obj);
      if (unshared && rep.getClass().isArray()) {
        rep = cloneArray(rep);
      }
      if (rep != obj) {
        // Filter the replacement object
        if (rep != null) {
          if (rep.getClass().isArray()) {
            filterCheck(rep.getClass(), Array.getLength(rep));
          } else {
            filterCheck(rep.getClass(), -1);
          }
        }
        handles.setObject(passHandle, obj = rep);
      }
    }

    return obj;
  }
}
```

通过上面源码的分析，我们知道了，如果希望阻止序列化导致单例被破坏，需要让我们的SingleTon实现一个readResolve()方法
> 这个结论如果直接给出的话，就一脸蒙，但是看了源码后也许会好一点吧

注意，这个方法的返回值类型必须是`Object`类型：
```java
public class SingleTon{
    private Object readResolve() {
      return newInstance();
    }	
}
```
> 一旦方法返回值变为`Object`类型了，IDEA就有高亮了，好奇怪
