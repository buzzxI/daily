## ::

在 lambda 表达式中，如果有：

```java
x->System.out.println(x);
```

可以简化为：

```java
System.out::println
```

起因：lambda表达式相当于创建匿名内部类，并定义匿名方法，且在匿名方法中有时只调用一个方法就结束，所以引入方法引用（::就是方法引用）

> 比如如果这个匿名方法只会执行打印输出的语句，那么就可以写成上面的形式

一些语法：

* 静态方法引用：classname::methodname
* 对象实例方法引用：instancename::methodname
* 对象父类方法引用：super::methodname
* 类构造器方法引用：classname::new
* 数组构造器方法引用：typename[]::new

举几个例子：

静态方法引用：

```java
import java.util.List;
import java.util.ArrayList;

public class demo{
	public static void main(String[] args) {
		List<String> name = new ArrayList<String>();
		name.add("a");
		name.add("b");
		name.add("c");
		name.forEach(demo :: print);
		//name.forEach(s -> print(s));
	}
	public static void print (String s) {
		System.out.println(s);
	}
}
```

> 调用上方的方法打印输出结果和下面相同

对象实例方法引用：

```java
import java.util.List;
import java.util.ArrayList;

public class demo{
	public static void main(String[] args) {
		List<String> name = new ArrayList<String>();
		name.add("a");
		name.add("b");
		name.add("c");
		name.forEach(new demo() :: print);
		//name.forEach(s -> print(s));
	}
	public void print (String s) {
		System.out.println(s);
	}
}
```

对象父类方法引用：

```java
import java.util.List;
import java.util.ArrayList;

public class demo extends father{
	public static void main(String[] args) {
		demo d = new demo();
		d.test();
	}
	public void test() {
		List<String> name = new ArrayList<String>();
		name.add("a");
		name.add("b");
		name.add("c");
		name.forEach(super::print);
	}
	
}

class father{
	public void print (String s) {
		System.out.println(s);
	}
}

```

> 不能在静态方法中使用这种写法，就像上面在main不能写成this::print同理



# stream

先给出一些常用的函数式接口：

![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/java8%E5%B8%B8%E7%94%A8%E6%8E%A5%E5%8F%A3.png)

* `Supplier`：无中生有，不传入参数，他还能返回一个类型
* `Function`：传入一个参数，返回一个结果；还有一个函数式接口，可以传入两种类型的参数，并返回一个结果：`BiFunction`
* `Consumer`：传入一个参数，但是不返回结果；类似的也有可以接收两种类型的参数，并返回结果：`BiConsumer`

* `UnaryOperator`：传入一个参数，返回一个同类型的参数，这个在原子类`Atomic`中使用的比较多

## 惰性求值和及早求值

```java
    List<Integer> list = new ArrayList<>();
    for(int i = 0; i < 10; i++) {
        list.add(i);
    }
    long rst = list.stream().filter(num -> num > 5).count();
    System.out.println(rst);
```

使用java8lambda表达式的写法与新特性流

rst标识list中大于5的元素的个数

使用filter筛选过滤，使用count计数

```java
	List<Integer> list = new ArrayList<>();
    for(int i = 0; i < 10; i++) {
        list.add(i);
    }
    list.stream().filter(num -> {
        System.out.println(num);
        return num > 5;
    });
```

如果只筛选过滤而不进行计数，则不会打印输出

只是描述了stream，不产生新的集合，称为惰性求值方法

```java
	List<Integer> list = new ArrayList<>();
    for(int i = 0; i < 10; i++) {
        list.add(i);
    }
    Stream<Integer> s = list.stream().filter(num -> {
        System.out.println(num);
        return num > 5;
    });
	System.out.println("<----->")
	s.count();
```

上述筛选后并计数会打印输出，不过会在打印分割线后才打印num

从stream中产生了新的值，称为及早求值方法

![](https://cdn.jsdelivr.net/gh/SunYuanI/img@latest/img/stream_filter_count.png)

判断是惰性求值或及早求值：看返回值

如果返回值是stream，那么为惰性求值；如果返回值是空或其他值，那么为及早求值

一个理想的操作，形成一个惰性求值链，在最后进行及早求值

## 一些常用的操作

## collect（Colloectors.toList()）

```java
	String[] strings = {"a","b","c"};
	int[] nums = new int[] {1,2,3};
	List<Integer> intList = Arrays.stream(nums).boxed().collect(Collectors.toList());
	List<String> stringList = Arrays.stream(strings).collect(Collectors.toList());
    for(int num : intList) {
        System.out.println(num);
    }
    for(String s : stringList) {
        System.out.println(s);
	}
```

![](./img/stream_collect.png)

collect实现数组到list的转化

Arrays调用stream方法获取stream对象，如果是int，double，long基本数据类型，还需要对stream调用boxed（）方法进行封装在collect为集合

> 八种基本数据类型：byte，char，short，int，float，long，double，boolean

显然因为collect方法返回一个list，所以list属于及早求值

## map()

一个将小写字母变为大写字母的举例

> map实现一种映射关系

```java
	String[] strings = {"a", "ab", "abc"};
	List<String> list = Arrays.stream(strings).map(string -> string.toUpperCase()).collect(Collectors.toList());
    for(String s : list) {
        System.out.println(s);
    }
```

![](./img/map操作.png)

对于map函数，接收一个Function接口实例

![](./img/Function接口.png)

Function接口接收一个T类型的参数，返回一个R类型的参数，T,R可相同可不同

## flatMap()

将多个stream连接为一个stream

先写一个前提：

```Java
	List<Integer> list1 = Arrays.stream(new int[]{1,2,3,4,5,6}).boxed().collect(Collectors.toList());
	List<Integer> list2 = Arrays.stream(new int[] {7,8,9,10,11,12}).boxed().collect(Collectors.toList());
	List<List<Integer>> list = new ArrayList<List<Integer>>();
	list.add(list1);
	list.add(list2);
```

因为涉及到多个流，先创建两个list，分别由array创建

然后将这两个list作为元素加入list中

第一次尝试：

```java
	List<Integer> all = list.stream().flatMap(l -> l.stream()).collect(Collectors.toList());
	for(int num : all) {
    	System.out.println(num);
	}
```

将list的元素以流的形式构建all，对于all中每个元素输出结果为：

![](./img/flatMap_1.png)

可见all中存储元素为两个list的总和

第二次尝试：

```java
	List<Integer> all = list.stream().flatMap(l -> l.stream()).map(num -> num + 10).filter(num -> num > 20).collect(Collectors.toList());
    for(int num : all) {
        System.out.println(num);
    }
```

在进行了flatMap将两个流合并后，再进行map操作将流中所有元素自增10，并通过filter操作进行过滤，筛选掉小于等于20的元素

![](./img/flatMap_2.png)

flatMap方法中传入的接口和map相同，都是Function接口，只不过flatMap限定返回值为Stream类型

> 将多个流合并为一个流

```java
List<Stream<Integer>> all = list.stream().map(l -> l.stream()).collect(Collectors.toList());
```

如果将flatMap直接换为map，那么返回值多个流，在collect后收集为List

## filter()

在前面说惰性取值和及早取值的时候提到过

![](./img/filter操作.png)

filter操作会接收一个predicate接口，内含一个test方法，接收一个T类型的参数，返回一个boolean类型参数

![](./img/predicate接口.png)

对于返回值为true的项将被保留在stream中，经过及早求值可以实现将满足条件的项放入list中

## min、max

用来对一个集合，数组找到最小最大值

查找最大最小的元素，需要考虑的是对应的指标，如果是对于字符串类型可能的一个指标是字符串的长度

```java
	int[] nums = new int[] {1,2,3,4,5,6};
	int min = Arrays.stream(nums).boxed().min(Comparator.comparing(num -> num)).get();
	System.out.println(min);//1
```

需要注意的是，对于min，max方法需要传入一个Comparator对象，调用其静态方法comparing

这个方法需要一个Function接口

```java
int min = Arrays.stream(nums).boxed().min(Comparator.comparing(num -> -num)).get();
```

如果指标变为-num，那么上述将返回6作为结果

值得注意的是，如果只是调用了min、max方法，将返回一个Optional对象；所以如果为了获得对应的元素，不要忘了调用Optional对象的get（）方法

```java
Optional<Integer> min = Arrays.stream(nums).boxed().min(Comparator.comparing(num -> num));
```

如果stream为空，那么返回空

```java
	int[] nums = new int[] {};
	Optional<Integer> min = Arrays.stream(nums).boxed().min(Comparator.comparing(num -> num));
	System.out.println(min);
```



![](./img/stream为空返回Optional.png)

如果不为空：

![](./img/stream不为空返回Optional.png)

其实比较神奇的是，min、max方法需要传入一个Comparator对象，调用comparing方法，而这个方法居然只需要提供一个存取方法就行，如果是以前需要比较两个属性的值

比如对于优先队列中，如果存入数组，小顶堆需要这么写：

```java
PriorityQueue<int[]> queue = new PriorityQueue<int[]>((nums1, nums2) -> nums1[0] - nums2[0]);
```

> 以数组中第0位置上元素的大小作为排序的标准

其实如果不用lambda来写，那么将是

```java
	PriorityQueue<int[]> queue = new PriorityQueue<int[]>(new Comparator<int[]>() {
        @Override
        public int compare(int[] o1, int[] o2) {
            // TODO Auto-generated method stub
            return o1[0] - o2[0];
        }
    });
```

也就是传入一个Comparator对象（和上面一样），实现对应的compare方法

## reduce()

reduce()实现了从一组值中返回一个值

举例来说，如果需要取得数组的和，可以使用如下操作：

```java
	int[] nums = new int[] {1,2,3,4,5,6};
	int sum = Arrays.stream(nums).boxed().reduce(0, (a, b) -> a + b);
	System.out.println(sum);//21
```

reduce方法接收两个参数，一个参数为初始值，在该例子中为0，另一个为一个实现了BinaryOperator的对象

BinaryOperator接口有一个apply方法，接收两个相同类型的参数，返回值为一个同类型的参数

如果reduce方法中不填入初始值，那么将返回一个Optional对象

> 这是因为如果不填入初始值，那么在面对stream为空的情况时，返回值为初始值，而初始值未设定
>
> 而如果返回Optional对象就好说了，上面也看到了，如果stream为空，那么打印返回对象为Optional.empty

```java
Optional<Integer> sum = Arrays.stream(nums).boxed().reduce((a, b) -> a + b);
```

## 高阶函数

高阶函数是指接受另外一个函数作为参数， 或返回一个函数的函数。

高阶函数不难辨认： 看函数签名就够了。

如果函数的参数列表里包含函数接口， 或该函数返回一个函数接口， 那么该函数就是高阶函数  

> 上面的好多函数都是高阶函数

# 类库



