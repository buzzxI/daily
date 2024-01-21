稍微看一下设计模式
# singleton pattern
只有一个实例对象的类

有的时候会看见单例的类被 `final` 修饰，这是为了防止子类破坏父类的单例，因为按理来说，父类是单例，子类应该也是单例。

> 但这种 `final` 的做法确实不够优雅

## 饿汉式

将构造器私有化, 然后创建一个类的实例化对象, 对外暴露一个静态方法用于获取该实例化对象
```java
public class SingleTon {
	// private constructor
	private SingleTon() {}
    
    // initialize with static block
	private static final SingleTon singleTon;
	static {
		singleTon = new SingleTon();
    }
    
    // get instance with static method
	public static SingleTon getInstance() {
		return singleTon;
	}
}
```
饿汉式使用静态常量, 将实例化对象写死为 final 类型, 并通过静态代码块初始化对象, 在编译期间就确定对象

但是这种单例模式的问题在于, 其对象加载不是 lazy-loading, 单例对象在类加载的时候就已经进行了实例化, 不过好处是因为使用了类加载的机制，导致不会出现线程安全的问题，不需要额外加锁，效率比较高

>   从写法上也是最容易实现的一种单例模式

## 懒汉式

实现 lazy-loading, 只有调用 getInstance() 时才进行对象的实例化

### thread-unsafe

既然在 static block 中初始化不能 lazy-loading, 那么在 getInstance() 中初始化就好了吧

```java
public class SingleTon {
	private SingleTon() {}

	private static SingleTon singleTon;
    
	public static SingleTon getInstance() {
		if (singleTon == null) {
			singleTon = new SingleTon();
		}
		return singleTon;
	}
}
```
此时实例对象 singleTon 不使用 final 修饰, 也不再编译期间实例化对象, 转而在运行期初始化, 这看起来很好, 但存在线程竞争问题

考虑多个线程进入 getInstance 方法, 并通过了 if 判断, 从而实例化多个对象

### thread-safe
既然上面的方法线程不安全, 加一个锁不就线程安全了吗

```java
public class SingleTon {
	private SingleTon() {}

	private static SingleTon singleTon;
	public synchronized static SingleTon getInstance() {
		if (singleTon == null) {
			singleTon = new SingleTon();
		}
		return singleTon;
	}
}
```
注意到使用 synchronization 对方法 getInstance 加锁, 确实实现了线程安全, 但这种写法同时引入了效率问题, 由于 getInstance 方法是静态方法, 使用 synchronization 加锁, 锁住的对象是 `SingleTon.class`, 不仅 getInstance 在线程之间会阻塞, 所有调用 static 方法的线程之间还会相互阻塞

本身 getInstance 方法的效率应该很高, 只有在对象为 null 时候需要把对象 new 出来; 其余的时候直接把对象返回就好了; 因此一种想法是使用同步代码块提高效率

```java
public class SingleTon {
	private SingleTon() {}

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
看起来加了, 但又没加上, 还是可能存在实例化多个对象导致单例被破坏的情况

>   考虑两个线程同时进入 if 的情况

## DCL

>   double checked-locking pattern

```java
public class SingleTon {
	private SingleTon() {}

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
尽管第一个 if 是线程不安全的, 但第二个 if 是线程安全的 => double check

然而上面的写法还是存在问题的主要原因在于:

*   内存可见性: new 对象的在运行时不一定会立刻同步到内存中, 在多级缓存的环境中, 运算结果可能只是保存在了快速缓存中;
*   指令重排: jvm 为了优化运行速度, 可能对指令进行重新排序

在 jdk 5 使用 jvm 提供了 `volatile` 关键字, 保证对变量的修改是内存可见的, 同时不允许进行指令重排序, 因此在 DCL 实现单例的场景中需要对变量添加 `volatile` 修饰:

```java
public class SingleTon {
	private SingleTon() {}

	private static volatile SingleTon singleTon;
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

## 静态内部类

```java
public class SingleTon {
	private SingleTon() {}

	private static class SingleTonHolder{
		private static final SingleTon singleTon = new SingleTon();
	}

	public static SingleTon newInstance() {
		return SingleTonHolder.singleTon;
	}
}
```
>   借助 chatgpt 的说法: This implementation is efficient, thread-safe, and follows the Initialization-on-demand holder idiom, ensuring lazy initialization of the Singleton instance and high performance

当外部类加载的时候不会同时加载静态内部类，只有当第一次调用的方法 newInstance() 的时候才会执行类的加载 (lazy loading)

lazy-loading 这个好理解，现在关键点在于如何保证线程的安全的

JVM 对类进行初始化的场景 ([Chapter 5. Loading, Linking, and Initializing (oracle.com)](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-5.html#jvms-5.5))
* 字节码指令为 `new, getsatic, setstatic, invokestatic` 时：
    
    >   分别对应于: new 一个实例化对象; 读取一个静态字段; 设置一个静态字段; 调用静态方法
* 使用 `java.lang.reflect` 包下的方法对类进行反射调用
* 考虑一个类具有一个子类, 当初始化这个类的子类时, 其本身也会被初始化
* 考虑一个声明了 `non-abstract`, `non-static` 的接口, 初始化一个实现了这个接口的类, 同时会初始化这个接口 (本质上也是类)
* JVM 启动的时候，会默认初始化用户执行的主类，即含有 main 方法的类
* The first invocation of a `java.lang.invoke.MethodHandle` instance which was the result of method handle resolution ([§5.4.3.5](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-5.html#jvms-5.4.3.5)) for a method handle of kind 2 (`REF_getStatic`), 4 (`REF_putStatic`), 6 (`REF_invokeStatic`), or 8 (`REF_newInvokeSpecial`).
> 最后一点看不懂，就直接粘过来了

如果我们查看编译后的字节码文件，可以看到，`newInstance()`方法长这样：
```byte code
 0: getstatic     #2 // Field com/buzz/SingleTon$SingleTonHolder.singleTon:Lcom/buzz/SingleTon;
 3: areturn
```
调用 `SingleTonHoler.singleTon` 相当于读取一个静态字段，此时类 `SingleTonHolder` 被初始化, jvm 保证类的初始化一定是线程安全的

>   类初始化时调用 `<clinit>`, jvm 保证这个方法在多线程环境中可以被正常的加锁, 同步

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
* 使用带有双重检查的懒汉式 (注意一定要带上 volatile 关键字)
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

# visitor pattern

visitor pattern 可以让 OOP 的语言具有类似函数式编程的特性

简单来说, 抽象类用来定义行为, 使用多个子类继承该抽象类, 实现不同的方法; 后续对抽象类更新, 增加其他的行为时, 那么此时需要修改所有继承了该抽象类的子类, 并在各个子类中实现新的方法

借助 visitor pattern, 可以使得行为 (方法) 针对不同的类型, wan'cheng方法的实现

