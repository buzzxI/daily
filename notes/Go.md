终于还是来到了 Go

# 配置环境

肯定还是参考的官网 [The Go Programming Language (google.cn)](https://golang.google.cn/)

下好 tar 包后，解压到 /usr/local 下即可

```shell
$ tar -C /usr/local -xvz -f go1.19.2.linux-amd64.tar.gz
```

注意：**如果以后要升级 go 版本的话，直接 rm -rf /usr/local/go 然后重新解压就行，不要替换**

然后就是配置环境变量 PATH：

```/etc/profile
# /etc/profile


# ...



export GO_HOME=/usr/local/go
export PATH=$PATH:$GO_HOME/bin
```

配置好了之后不要忘了重新读取一下配置文件，然后再验证版本

```shell
$ source /etc/profile
$ go version
go version go1.19.2 linux/amd64
```

如果配置好了话，是能看到 go 的版本的

>   不过这里建议的话，还是在 ~/.zshrc 中再写一遍 

就像配置 maven 的镜像一样，go 也需要配置 module 的镜像(这里其实是代理)

```shell
# ~/.zshrc

export GO_HOME=/usr/local/go
export PATH=$PATH:$GO_HOME/bin
# 这里说不明白，跟这写就行了
export GO111MODULE=on
# 这个代理用的人比较多，应该挺好用的，没必要用阿里的那个
export GOPROXY=https://proxy.golang.com.cn,direct
```

# 语法部分

## hello world

这里没什么好说的，不管什么语言，入门的时候都是一个 hello world

```go
package main

import "fmt"

func main() {
    fmt.Println("hello word!")
}
```

尽管目前 vscode 会报错，但编译的还是很顺利的

```shell
# 编译得到一个 hello 可执行文件
$ go build hello.go
# 没有可执行文件，而是直接 run 起来
$ go run hello.go
```

## 变量

从上面的第一个 go 例程也能看出来 go 也是不需要写分号的，在 go 里面定义变量的时候直接一个 var 就好了

```go
package main

import "fmt"

func main() {
	var str = "abc"
	fmt.Print(str + "\n")
}
```

```shell
$ go run hello.go
abc
```

>   确实很简单

当然，这里的变量也可以手动限制类型：

```go
package main

import "fmt"

func main() {
	var num int = 1
	fmt.Println(num)
}
```

不过真正有意思的 go 提供了一个 := 的语法

```go
package main

import "fmt"

func main() {
	num := 1
	fmt.Println(num)
}
```

好吧：var num = 1 和 num := 1 是等价的，也就是说 **:=** 是 go 定义变量的一种特殊写法

## 常量

类似 c 里面对常量的定义

```go
package main

import (
	"fmt"
	"math"
)

const N = 1e5 + 10

func main() {
	fmt.Println(N) // 100010
	const M = 1e9 + 7
	fmt.Println(M) // 1.000000007e+09
	fmt.Println(math.Max(N, M)) // 1.000000007e+09
}
```

这里面导了两个包，一个 fmt 用来进行输入输出，一个 math 用来进行常用的数学运算

## 循环

经典 for 循环

```go
package main

import (
	"fmt"
)

func main() {
	for j := 0; j < 10; j++ {
		fmt.Println(j)
	}
}
```

最显著的特点就是没有括号了...

但大括号还是需要的

## 分支

既然 for 都没有括号了 if 当然也没有了

```go
package main

import (
	"fmt"
)

func main() {
	j := 10
	if j > 20 {
		fmt.Println("no way")
	} else {
		fmt.Println("always")
	}
}
```

在 go 里面大括号是必须的，就算只有一行的 if 也不能省略 

```go
if j > 20 fmt.Println("no way")
else fmt.Println("always")
```

向上面这种就不能编译通过了...

因为 go 里面没有三元运算符，所以只能 if 了

## 数组

定义数组的时候就只能使用 var 了...

```go
package main

import "fmt"

func main() {
	var a [5]int
	fmt.Println(a) // [0 0 0 0 0]
}
```

当然二维数组也是可以定义的

```go
package main

import "fmt"

func main() {
	var a [5][2]int
	fmt.Println(a) // [[0 0] [0 0] [0 0] [0 0] [0 0]]
}
```

在 go 里数组的引用表示了整个数组，而不是一个指向了数组中第一个元素的指针，这一点从上面的输出就能看出来

## 切片

具体的可以看一下[官方博客](https://go.dev/blog/slices-intro)

个人感觉类似可变数组：`Go’s slice type provides a convenient and efficient means of working with sequences of typed data`

在 go 里面声明一个数组，可以采用类似数组的方式，不过最大的区别在于不需要再声明大小了

```go
package main

import "fmt"

func main() {
	a := []int{}
	a = append(a, 1)
	fmt.Println(a) // [1]
}
```

注意到 a 是一个 slice，默认为空，而通过方法 append, 可以向数组中添加新的元素 1

此外通过 go 自带的方法 make 也可以创建一个 slice

```go
package main

import "fmt"

func main() {
    // 创建一个大小为 2 的 string 类型的 slice
	b := make([]string, 2)
	b[0] = "a"
	b[1] = "b"
    // 超过索引的部分，需要通过 append 访问
	b = append(b, "c")
	fmt.Println(b)
}
```

>   通过 make 创建 slice 的时候可以选择参数 capacity 和 size
>
>   其中 size 表示了这个切片的初始化大小；而 capacity 表示了这个 slice 底层数组的大小
>
>   具体的 size 和 capacity 的区别，看一下后面的[深入切片](#深入切片)就明白了

既然叫做 slice，自然支持一些切片性的行为，一个切片可以从现有的切片或数组中通过"切片"的方式获取

```go
package main

import "fmt"

func main() {
	a := []int{1, 2, 3, 4, 5}
    // b 为 a 切片的下标索引从 1 到 3 的部分(不包括 3)
	b := a[1:3]
	fmt.Println(b) // [2, 3]
	b[0]++
	fmt.Println(a) // [1,3,3,4,5]
}
```

一个从其他数组或其他切片中得到的切片不是独立的，这一点从上面的输出也能看出来

切片的起点和终点都是可选的，如果省略起点的话，那么默认的起点为切面的开头；如果省略终点的话，那么默认的终点就是切片的结尾

当然，可以来一个有意思的：

```go
package main

import "fmt"

func main() {
	a := []int{1, 2, 3, 4, 5}
	b := a[:]
	fmt.Println(b) // [1 2 3 4 5]
	b[0]++
	fmt.Println(a) // [2 2 3 4 5]
}
```

### 深入切片

在 java 中可变数组底层就是数组，只不过通过其他的机制(比如自动扩容)，让其看起来不是一个固定大小的数组，切片的底层也是类似的，这里完全是对[官方博客](https://go.dev/blog/slices-intro)的总结

一个 slice 具有如下的结构：

![](https://cdn.jsdelivr.net/gh/SunYuanI/img@latest/img/go-slice-struct.png)

可以看到一个 slice 由一个指针 ptr, 两个变量 len 和 cap 组成；其中 ptr 指向了底层的数组，而 len 表示当前切片的大小，cap 表示了底层数组的大小

>   slice 本身大小也就 16 个字节(如果考虑对其的话，可能会大点)，但可以表示一个可变的数组

所以操作 slice 其实是操作数组，这也就是为什么对同一个数组切出来的一个 slice 修改，另一个 slice 也会修改

后面举一个例子说明一下对 slice 的操作

首先创建通过 make 创建一个大小为 5 的切片

```go
s := make([]byte, 5)
```

>   注意到这里并没有声明 capacity 的大小，这是因为默认的话 capacity 和 size 是一样的
>
>   手动声明时，要保证 capacity >= size

![](https://cdn.jsdelivr.net/gh/SunYuanI/img@latest/img/go-slice-1.png)

随后对这个 slice 进行切片操作：
```go
s = s[2:4]
```

![](https://cdn.jsdelivr.net/gh/SunYuanI/img@latest/img/go-slice-2.png)

一次切片不会创建一个数组，不过是创建了一个新的 slice 结构，正因如此，一个 slice 可以在 $O(1)$ 的时间内完成，效率很高；但同时修改了 slice 也会修改原始的数组

这里经过一次切片后 s 的 size 变为了 2(这是由切片行为决定的)，而 capacity 变为了 3(这是由底层数组的大小限制的)

现在如果希望让 s 扩容到撑起整个 capacity，只需要：

```go
s = s[:cap(s)]
```

![](https://cdn.jsdelivr.net/gh/SunYuanI/img@latest/img/go-slice-3.png)

这也是 s 可以扩容的最大大小，如果终点比 cap(s) 更大，那么程序会直接报错

不过既然 slice 是可变数组，那么这其实也就暗示着，只要内存足够，理论上我是可以不断向 slice 中添加数据的

上面已经看到了可以通过 append 向 slice 中添加元素，如果一个 slice 的存储已经达到了底层数组的上限(capacity)，此时继续 append，就只能创建一个新数组，此时新数组和原数组除了取值和大小，不再具有任何关系

```go
package main

import "fmt"

func main() {
	a := make([]int, 2, 2)
	b := append(a, 1)
	b[0]++
	printSlice(a) // slice: [0 0], len: 2, capacity: 2
	printSlice(b) // slice: [1 0 1], len: 3, capacity: 4
}

func printSlice(s []int) {
	fmt.Printf("slice: %v, len: %d, capacity: %d\n", s, len(s), cap(s))
}
```

从打印结果上来看 slice b 指向的数组和 slice a 指向的数组没有任何关系

扩容、复制的操作完全是 go 内部完成的，实际使用切片的时候只要 append 就好了

append 方法更强大的地方是，他可以进行两个 slice 的拼接

```go
package main

import "fmt"

func main() {
	a := make([]int, 2, 2)
	b := make([]int, 1)
	c := append(a, b...)
}
```

这中写法官方解释为：To append one slice to another, use `...` to expand the second argument to a list of arguments.

>   别翻译成中文了，这个英文说的挺好的

尽管 slice 很方便，但使用不当还是会存在问题的

一个 slice 本身不大，但其指向的底层数据结构类型为一个数组，如果这个数组很大，程序还是很占空间的

只要存在一个指向大数组的引用(可能是 slice 还可能就是数组本身)，那么 go gc 是不会对数组进行内存回收的；这点看起来还是比较符合逻辑的，毕竟还保留了引用，gc 就是不应该把他回收掉

但如果考虑使用 slice 的时候，如果底层是一个大数组，而在 slice 中只使用了大数组中的一小部分，此时整个大数组会占用大量内存，而 slice 本身可以使用的空间却很小，官方给出了一个示例：

```go
var digitRegexp = regexp.MustCompile("[0-9]+")

func FindDigits(filename string) []byte {
    b, _ := ioutil.ReadFile(filename)
    return digitRegexp.Find(b)
}
```

方法 FindDigits，将一个文件读取到内存，并对文件内容通过正则表达式的方式进行匹配，然后返回一个切片

看起来好像没什么问题，但返回的切片底层是一个包含了整个文件的数组，可能本身程序关心的是文件中的数字，而不是整个文件，但现在整个文件都在内存中不能释放

因此一个正确的写法可能如下：

```go
func CopyDigits(filename string) []byte {
    b, _ := ioutil.ReadFile(filename)
    b = digitRegexp.Find(b)
    c := make([]byte, len(b))
    copy(c, b)
    return c
}
```

## map

通过 make 可以创建一个 map

```go
package main

import "fmt"

func main() {
	m := make(map[string]int)
	m["a"] = 1
	m["b"] = 2
	fmt.Println(m) // map[a:1 b:2]
	n := map[string]int{
		"a": 2,
		"b": 1,
	}
	fmt.Println(n) // map[a:2 b:1]
}
```

通过 make(map[string]int) 可以创建一个以 string 类型为 key，int 类型为 value 的 map

>   注意到 map 的 key 不可以为 slice 类型

此外还可以通过字面量的形式初始化一个 map

访问 map(存储或获取) 类似于访问数组，告别了 put 方法

当访问不存在的 key 时，go 不会抛出异常(java 你跟人家学学)，它会返回一个 "零值"

这个零值为各种 val 类型的零值，如果 val 为 int 类型，则返回 0，如果 val 为字符串类型，则返回 ""

go 内建方法 len 和 delete 分别用来获取 map 大小和删除 map 中的对应键

```go
package main

import "fmt"

func main() {
	m := make(map[string]int)
	m["a"] = 1
	m["b"] = 2
	fmt.Println(len(m)) // 2
	delete(m, "a")
	fmt.Println(len(m)) // 1
}
```

同样的，如果删除了 map 中不存在的键 go 可以不会抛出异常的(java 你跟人家学学)

go 中可以借助 map 实现 set，比如我希望建立一个 string 类型的 set，只需要新建一个 string -> bool 类型的 map

```go
package main

import "fmt"

func main() {
	m := make(map[string]bool)
	m["abc"] = true
	m["def"] = true
	fmt.Println(m["buzz"]) // false
}
```

存储时，将对应 key 的 value 设置为 true，访问到不存在的 key 时，会返回 false(bool 类型的 0 值)

而有的时候 "零值" 是具有特殊意义的，因此区分零值和 key 不存在的情况还是有必要的

```go
package main

import "fmt"

func main() {
	m := make(map[string]int)
	m["abc"] = 1
	m["def"] = 0
	val, rst := m["buzz"]
	fmt.Printf("val is %d, rst is %t\n", val, rst) // val is 0, rst is false
	val, rst = m["def"]
	fmt.Printf("val is %d, rst is %t\n", val, rst) // val is 0, rst is true
}
```

这里主动存储了一个 val 为 0 的 key，同时访问了一个不存在的 key，返回的 val 都是 0，但 rst 一个为 true，一个为 false

这也就是说，在 go 中对 map 的 key 的一个访问，会返回两个值，一个为 val，一个为当前 key 是否存在

go 提供了 comma ok 语法，方便代码编写

```go
package main

import "fmt"

func main() {
	m := make(map[string]int)
	m["abc"] = 1
	m["def"] = 0
	if val, rst := m["abc"]; rst {
		fmt.Println(val)
	} else {
		fmt.Println("no key")
	}
}
```

在 if 判断中先进行赋值，然后进行判断，根据键是否存在进入不同的分支

有的时候可能并不是真的需要获取 m 中的 val，而仅仅是希望知道 map 中是否包含了对应 key，此时可以使用空占位符占取 val 的位置

```go
package main

import "fmt"

func main() {
	m := make(map[string]int)
	m["abc"] = 1
	m["def"] = 0
	if _, rst := m["abc"]; rst {
		fmt.Println("key")
	} else {
		fmt.Println("no key")
	}
}
```

## range 循环

>   有点类似 go 里的 foreach 循环

举一个简单的例子：

```go
package main

import "fmt"

func main() {
    // 开一个大小为 10 的 slice
	arr := make([]int, 10, 10)
    // 向 slice 中存入等差数列
	for i := 0; i < 10; i++ {
		arr[i] = 10 + i
	}
    // 打印 i 和 num
	for i, num := range arr {
		fmt.Printf("index is %d, num is %d\n", i, num)
	}
}
```

这里能看出来 range 循环的第一个参数表示当前位置的索引，第二个参数表示当前位置的元素

如果迭代的时候只想看看元素的大小，而不是元素的索引，此时可以使用前面提到的空白标识符

```go
package main

import (
	"fmt"
)

func main() {
	m := make(map[string]int, 10)
	for i := 0; i < 10; i++ {
		m[string(rune('a'+i))] = i + 10
	}

	for key, val := range m {
		fmt.Printf("key is %s, val is %d\n", key, val)
	}
}
```

range 循环也可以遍历 map，默认第一个值为 map 的 key，第二个值为 map 的 val

>   显然，也是可以通过空白标识符 "_" 仅遍历 val

然而，这并不是 range 循环最强大的地方：

```go
package main

import (
	"fmt"
)

func main() {
	m := make(map[string]int, 10)
	for i := 0; i < 10; i++ {
		m[string(rune('a'+i))] = i + 10
	}

	for key, val := range m {
        // 这里删除掉 m 中特定的 key
		delete(m, key)
		fmt.Printf("key is %s, val is %d\n", key, val)
	}
	fmt.Println(len(m)) // 0
}
```

虽然 range 循环看起来和 foreach 循环类似，但 foreach 毕竟就是迭代器的一个语法糖，这使其不能在循环内部进行元素删除；而 range 循环弥补了这个缺点，在 range 循环中，我可以随便删除

还没完

```go
package main

import (
	"fmt"
)

func main() {

	m := make(map[string]int, 10)
	for i := 0; i < 10; i++ {
		m[string(rune('a'+i))] = i + 10
	}
	for key, val := range m {
		delete(m, key)
		m[key] = val + 20
		fmt.Printf("key is %s, val is %d\n", key, val)
	}
	fmt.Println(len(m)) // 10
	for key, val := range m {
		fmt.Printf("key is %s, val is %d\n", key, val)
	}
}
```

在遍历的时候，居然还可以动态添加，本来我以为这么写会导致无限迭代

>   这里的 range m 就好像一个快照，一旦获取，都只是在遍历这个快照，对原来的 m 进行修改不会影响快照的状态
>
>   暂时还不知道 range 循环是怎么实现的

string 类型也可以使用 range 遍历，一个 string 类型其实可以看成一个 byte 类型的 slice

# 第一个 Go

紧跟官方教程 [Tutorial: Get started with Go - The Go Programming Language (google.cn)](https://golang.google.cn/doc/tutorial/getting-started)

通过 go.mod 文件进行依赖管理，通过命令：

```shell
$ go mod init [module name] 
```

创建一个 go.mod 文件，参数就是模块名

然后创建一个 hello.go 程序：

```go
package main

import "fmt"

func main() {
    fmt.Println("Hello, World!")
}
```

然后直接运行：

```shell
$ go run ./hello.go
```

>   没有什么中间文件，直接就运行了

## 引用别人的代码

造轮子还挺没意思的，不如直接调用别人写好的程序

类似于 maven，在 go 中可以在 [Go Packages](https://pkg.go.dev/) 添加需要的依赖，在官方的示例中添加了一个"rsc.io/quote"依赖，这样可以将上面的简单代码修改一下：

```go
package main

import (
	"fmt"

	"rsc.io/quote"
)

func main() {
	fmt.Println(quote.Hello())
}
```

刚写完之后肯定是一堆报错，毕竟 "rsc.io/quote" 还不在本地存在，所以需要添加依赖

```shell
$ go mod tidy
```

**以后每次添加依赖都需要 `go mod tidy` 更新一下依赖**

等待一会，提示下载好依赖后，就可以再次运行了

```shell
$ go run hello.go
```

>   quote.Hello() 输出了一个封装后的语句

## Go module

一个 module 中包含了多个 package，而一个 package 中包含了多个函数，使用别人的依赖的时候，也是引入 module 为单位引入的

现在新启用一个目录：
```shell
$ cd ./go_project
$ mkdir greetings
$ cd ./greetings
# 创建一个 go.mod 文件(其实就是创建一个 module)
$ go mod init github.com/buzzx/greetings
```

新创建得到的 go.mod 中仅仅包含了 module 的名字和 module 支持的 go 的版本

后面每次添加依赖后，go mod tidy，go 会帮我们修改好 go.mod 文件，添加上正确的依赖







