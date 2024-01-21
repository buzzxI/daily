以前看过js，不过都只是草草看了一眼

# 引入JS

* 嵌入页面：这个就是在`<script></script>`标签内写js代码
* 引入额外的js文件，就是把js单独写成一个文件，引入的时候使用：`<script src="{地址}"></script>`

> 要注意的是，引入外部资源的时候一定不要写成自闭和的形式：`<script src="地址"/>`
>
> 不能说一定会出错，反正以前这么写的时候出现过问题

有的时候使用`<script>`标签的时候会看到：`<script type ="text/javascript"></script>`，又称脱裤子放屁

# 基本语法

虽然不会报错，但是一定要在每个语句的结尾加上`;`表示结束，问就是规范

注释的方式和java一样都是`//`，代码块的注释是：`/* */`

## 数据类型

* `Number`类型：就是数据，不分整数和浮点数

```javascript
123; // 整数123
0.456; // 浮点数0.456
1.2345e3; // 科学计数法表示1.2345x1000，等同于1234.5
-99; // 负数
NaN; // NaN表示Not a Number，当无法计算结果时用NaN表示
Infinity; // Infinity表示无限大，当数值超过了JavaScript的Number所能表示的最大值时，就表示为Infinity
```

* 字符串类型：这里就有点不一样了，它单引号和双引号是一样的，比如它认为：`'abc'`和`"abc"`是一样的

  我感觉唯一的区别就是比如我们要写标签的时候：

  ```javascript
  let string = '<img src="{一个路径}">'
  ```

* 布尔值：不想说，都一样

* 比较运算符：这里唯一需要说的是等于运算符，在`javascript`中，有两种等于：

  * `==`：先检查两个操作数的类型，如果是相同类型的就比较数值，返回数值的比价结果；**如果类型不相同，他会进行类型转化，然后再进行数值比较**

    如果一个操作数是`null`一个是`undefined`他会返回相等

  * `===`：比较到如果两个操作数的类型不同时，就直接返回`false`

  这里建议的是，如果**要比较两个操作数是否相等时，使用`===`**

  注意操作数`NaN`和任何数都不相等，如果需要判断一个操作数是不是`NaN`不能直接通过操作符进行比较

  ```javascript
  NaN === NaN // 这个会返回false
  isNaN(NaN) // true
  ```

* `null`和`undefined`：

  ```javascript
  var a = 1;
  var b;
  alter(typeof a); //number
  alter(typeof b); //undefined;
  ```

  一个是空，一个是未定义，用`null`就行了

* 创建数值：`var arr = [1,2,3,4];`，在数组中变量的类型可以是不同的！！！

* 创建对象：反正就是键值对的集合

  ```javascript
  var person = {
      id : 1,
      name : 'buzz',
      age : 10
  };
  ```

  写着和`json`一样，想要获取的时候使用`person.name`获取名字

* 变量：`var`和`let`：

  这两个最大的区别是作用域，`var`的作用域比`let`更大

  > `var`是函数作用域，`let`是块作用域

  ```javascript
  function fun() {
      for (let i = 0; i < 10; i++) {
          // 这里i可见
      }
      // 这里i不可见
      for (var j = 0; j < 10; j++) {
          // 这里j可见
      }
      // 这里j居然也是可见的
  }
  let a = 'buzz';
  let a = 'bezz'; //报错
  
  var b = 'buzz';
  var b = 'bezz'; // 这居然不报错
  ```

  行吧，那还是使用`let`吧

  我最接受不了的是：

  ```javascript
  var a = 1;
  a = 'buzz';
  ```

  服了这个写法

  在`es6`中也引入了常量const，一次赋值，不再改变


## 字符串

### 字符串拼接

原来的写法：

```javascript
var a = 'buzz';
var b = 'is';
var c = 'shit';
var d = a + b + c + "确实";
```

现在的写法：

```javascript
var a = 'buzz';
var b = 'is';
var c = 'shit';
var d = `${a} ${b} ${c}确实`;
```

### 对字符串的操作

在js中，字符串本身就是一个字符数组。

```javascript
let a = 'buzz';
let b = a.length;
let c = a[b - 1];
```

js中的字符串是不可变的：

```java
let a = 'buzz';
a[1] = 'e';
```

上述操作在js中并不会改变原来的a，而如果是java的话，那么编译都不会通过

一些函数：

* `toUpperCase()`
* `toLowerCase()`
* `substring()`

这些java里的String类也是有的

## forEach

就是语法的问题：

```javascript
let a = [1, 2, 'buzz', false];
for (let item in a) {
    console.log(a[item]);
}
```

我不能理解的是，为什么他这个循环得到的`item`居然是下标索引。

真正意义上的`forEach`为`for-of`

```javascript
let a = [1, 2, 'buzz', false];
for (let item of a) {
    console.log(item);
}
```

`forEach`还有一个函数的形式：

```javascript
let a = [1, 2, 'buzz', false];
a.forEach(function (element, index, array) {
   // 这个函数的一个参数为forEach遍历的每个元素，第二个参数为元素下标，最后一个参数为原数组本身
   console.log(index + '--' + element); 
});
a.forEach(function(element) {
    // 如果只有一个参数，那么获取到的将是数组的值
    console.log(element);
})
```

## Map和Set

简单的`map`

```javascript
let map = new Map();
// 新增或修改都是set
map.set('buzz', 1);
map.set('bezz', 2);
map.set('bazz', 3);
map.set(1, 'a');
// 查询
console.log(map.get('buzz'));
// 删除
map.delete('buzz');
// 判断是否具有键buzz
if (map.has('buzz')) {
    console.log('lalala');
}
map.forEach(function(key, value, m) {
    // 对于map而言forEach的三个参数分别为key，value，原map
    console.log(key + '----' + value);
});
map.forEach(function(value, key) {
    // 对于map而言，如果只有两个参数，那么第一个参数为value，而第二个参数为key
    console.log(key + "----" + value);
})
```

实际上，就可以把`js`中的`map`当成一个二维数组，对`map`进行`for-of`循环最终获得的居然是一个数组

对于`map`而言，其`forEach`函数的三个参数

简单的`set`

```javascript
let set = new Set();
set.add(1);
set.add(true);
set.add('buzz');
set.delete('1');
set.forEach(function(element, semelent, s) {
    // 对于set而言forEach的前两个参数是相同的都是元素element，最后一个参数为原set
 	console.log(element);
});
```

js还是那么的随意，一个map中可以存储不同的键，一个set中也可以存储不同的值

## 函数

在js中函数是一个对象：

```javascript
function fun() {
    console.log("this is a function");
}

let f = function fun() {
    console.log("this is a function");
};
// 上下的两种写法完全等价
```

在`js`中调用函数时，参数的个数可以和参数列表完全不同，可以比参数列表多也可以少：

```java
fun(1,2,3,'buzz', true, 'lala');
function fun() {
    for (let i = 0; i < arguments.length; i++) {
        console.log(arguments[i]);
    }
}
```

在函数内部有一个关键字`arguments`，这个关键字不是一个数组，但是用法上可以和数组一样

> 为什么不是数组呢，主要是因为如果使用数组的方式调用`forEach`函数的话会报错

如果函数传入的参数太多了，不方便管理：

```javascript
fun(1,2,3,'buzz', true, 'lala');
function fun(a, b, ...rest) {
    console.log(rest);
}
```

这里的`...rest`表示为多个参数，在方法内部，`rest`就是一个数组，是所有其他参数的数组。从写法上`rest`必须写在最后面，写格式为：`...[参数名]`

因为方法名中并没有规范返回参数类型，显然方法的返回值也是多种多样的：

```javascript
fun(1,2,3,'buzz', true, 'lala');
function fun() {
    let args = [];
    for (let i = 0; i < arguments.length; i++) {
        args.push(arguments[i]);
    }
    return args;
}
```

在`js`中不在函数中定义的变量均具有全局作用域，这是因为`js`中有一个默认的全局对象`window`，所有的变量定义都会被绑定到`window`上。而因为前面已经说过了函数本身也是一个对象，所以函数也会被绑定到`window`上

因为所有的变量都会被绑定到`window`对象上，容易出现冲突，可以通过手动将变量绑定到唯一的全局变量中减少冲突，比如：

```javascript
// 唯一的全局变量MYAPP:
var MYAPP = {};

// 其他变量:
MYAPP.name = 'myapp';
MYAPP.version = 1.0;

// 其他函数:
MYAPP.foo = function () {
    return 'foo';
};
```

### 解构赋值

对多个变量同时赋值

```javascript
var [x, y, z] = ['hello', 'JavaScript', 'ES6'];
// 甚至支持嵌套
let [x, [y, z]] = ['hello', ['JavaScript', 'ES6']];
// 甚至某些参数为空也行
let [, , z] = ['hello', 'JavaScript', 'ES6']; 
```

个人感觉比较典型的应用应该是解析`json`：

```java
var person = {
    name: '小明',
    age: 20,
    gender: 'male',
    passport: 'G-12345678',
    school: 'No.4 middle school'
};
var {name, age, passport} = person;
```

# 啥也不会，一无是处

## Math.ceil()

这个函数用来向上取整，主要是针对小数而言的

## setInterval() 和 clearInterval()

[setInterval()|MDN](https://developer.mozilla.org/zh-CN/docs/Web/API/setInterval) 和 [WindowTimers.clearInterval()|MDN](https://developer.mozilla.org/zh-CN/docs/Web/API/clearInterval)

setInterval() 方法用来定时重复执行一个方法，而 clearInterval() 用来终止这个定时任务

比如

```typescript
let times = 0;
const timer = setInterval(() => {
    console.log("每秒打印一次");
    times++;
    if (times == 10) {
        clearInterval(timer);
    }
}, 1000);
```

此外，不仅仅匿名函数，还可以定时调用其他的已知的函数，其一般形式为：setInterval([function_name], [interval], [param_1],...)

## scroll 相关

### [Window.scroll()](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/scroll)

这个函数用来滚动窗口到指定的位置，一般的写法为：Window.scroll([x], [y])

### [Window.scrollY](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/scrollY)

返回文档在 Y 方向上已经滚动的大小

### [Element.scrollTop](https://developer.mozilla.org/zh-CN/docs/Web/API/Element/scrollTop)

返回或设置某个元素内用在垂直方向上的像素数，和 scrollY 最大的区别在于这个函数还可以用来返回某些滚动元素内部滚动的大小

# 回调函数

总是在函数的参数列表中看到 callback，这个其实就是回调函数，在 js 中回调函数其实和异步执行密切相关，说难原理确实很复杂，说简单，其实使用的时候确实不需要那么麻烦

在 stack overflow，总结的回调函数：**A "callback" is any function that is called by another function which takes the first function as a parameter.** 

>   回调函数本身就是一个函数，他特别的地方就是其本身作为了一个参数传入了另一个函数中，并且在另一个函数中对其进行了调用

比如说：

```js
function a(val) {
    console.log(val);
}

function b(val, callback) {
    val++;
    callback(val);
}

b(1, a); // 应该打印输出 2
```

现在我们有一个需求，定义变量 a 为 1，打印初始值，等待 3s 后让 a 自增，并再次打印 a 的取值

```js
let a = 1;

const b = (x) => {
    console.log(x);
}

const c = (delay, val) => {
    setTimeout(() => {
        val++;
    }, delay);
}

b(a);
c(3000, a)
b(a);
```

显然上面的代码是不能解决问题的，因为会连续的打印两次，原因不解释，看上面[异步](./简单的例子)就知道了

这个问题的一个解决思路是在 setTimeout 内部自增之后再进行函数调用，此时就需要回调函数了

```js
let a = 1;

const b = (x) => {
    console.log(x);
}

const c = (delay, val, callback) => {
	setTimeout(() => {
    	val++
        callback(val)
    }, delay)   
}
console.log(a);

c(1000, a, b);
```

当然，像我们这么简单的逻辑其实是可以避免函数声明的，使用匿名函数的形式反而更加简洁

```js
let a = 1;

const c = (delay, val, callback) => {
    setTimeout(() => {
        val++
        callback(val)
    }, delay)   
}
console.log(a);

c(1000, a, (x) => console.log(x));
```

# 异步

其实之前也接触过异步调用，比如以前学过的 ajax 调用(jquery 封装)

而在底层，其实是 js 是通过回调函数实现的异步，比如使用 setTimeout() 方法

使用异步可以避免程序被阻塞，对于那些耗时的操作(比如请求网络中的数据)，可以让其异步执行，当执行完成后，通知当前的程序异步执行的结果

以前的程序通过回调函数实现结果的通知，典型的例子

```js
// 成功的回调函数
function successCallback(result) {
  console.log("调用成功: " + result);
}

// 失败的回调函数
function failureCallback(error) {
  console.log("异步调用出错: " + error);
}
// 一个耗时的异步函数，配置了方法参数、异步调用成功的回调函数和失败的回调函数
timeConsumingRequest(funParams, successCallback, failureCallback)
```

## 简单的例子

```js
function print() {
    console.log("this is async");
}
setTimeout(print, 3000);
console.log("this is sync");
```

程序首先会打印 "this is sync", 3 秒后将打印 "this is async"

这里的方法 setTimeout 就是一个异步方法，它接受了一个方法参数 3000 和一个回调函数，表示在异步等待 3s 后调用回调函数

## Promise

回调函数的写法其实很简单，但问题在于如果出现了多个异步函数的调用链时

比如我希望每隔几秒打印一次，那么回调函数的写法就需要写成

```js
setTimeout(() => {
    console.log("第一次打印");
    setTimeout(() => {
        console.log("第二次打印");
        setTimeout(() => {
            console.log("第三次打印");
        }, 1000);
    }, 1000);
}, 1000);
```

虽然实现了功能，但确实称不上好写，主要是看起来比较繁琐，套了好几层函数

肯定是需要简化的，所以在 js 中提供了 Promise API

>   js 原生的 fetch 函数返回的就是一个 Promise 对象

Promise 可以通过链式调用避免出现回调函数嵌套回调函数的情况

```js
fetch("https://[某个网址]")
	.then((response) => response.json())
	.then((json) => console.log(json));
```

fetch 返回一个 Promise 对象，在这个对象的 then 方法中配置回调函数即可实现回调

而在上面的例子中 response.json() 方法本质上也是一个异步调用，通过链式调用，在 then 后面继续 then 即可为其配置好回调函数

其实上面都是在进行异步调用成功的回调函数调用，注意到最开始提到了异步调用失败是也是具有回调函数的，在 Promise API 中可以通过配置 catch 的方式配置异步调用失败的回调函数

还是上面的例子，为其配置一个异步调用失败的回调函数

```js
fetch("https://[某个网址]")
	.then((response) => response.json())
	.then((json) => console.log(json));
	.catch((error) => console.log(error));
```

因为是链式调用，统一配置 catch 将处理所有可能的错误，当异步调用链中的任意一个异步调用出现了错误，都将直接调用 catch 中的失败的回调函数，而忽略剩下的所有 "then"

这里的 catch 很容易让人联想到 try-catch，毕竟从名字上就很接近，甚至说，在 Promise API，甚至可以配置 finally

```js
fetch("https://[某个网址]")
	.then((response) => response.json())
	.then((json) => console.log(json));
	.catch((error) => console.log(error));
	.finally(() => {
        // 一些释放资源的操作
    })
```

好了这下彻底跟 try-catch-finally 一样了

一个 Promise 对象具有三种状态：Pending(进行中)、Resolved(已完成，有的地方称为 Fulfilled)、Rejected(已失败)

通过 new 可以获得一个 Promise 对象，比如：

```js
const pObj = new Promise((resolve, reject) => {
	// 一些异步操作
   	// 调用 resolve，表示异步调用正常结束
    // 调用 reject，表示异步调用出现了异常
});
```

构造一个 Promise 对象，需要一个参数，一个函数作为参数，这个函数称为起始函数，起始函数本身又具有两个参数 resolve 和 reject，还是举一个例子吧

```js
const p = new Promise((resolve, reject) => {
    // resolve('lala') // 打印 then handle lala
    reject('lala') // 打印 catch handle lala
}).then((rst) => {
    console.log(`then handle ${rst}`);
}).catch((error) => {
    console.log(`catch handle ${error}`);
});
```

其实更多的时候根本不需要手动创建一个 Promise 对象，更多的都是调用某个方法，然后返回了一个 Promise 对象，而真实操作是返回的 Promise 对象

## async

使用 async 标注的函数是一个异步函数，返回一个 Promise 对象

>   然而在写程序的时候不必每次都手动 new 一个对象，直接返回结果就行，写成 return rst，会被编译器翻译成 return Promise.resolve(rst)

async 本质还是 Promise，是 Promise 的一个语法糖，借助 async 和 await 可以使用同步的写法写出异步的程序

>   异步的最高境界就是根本不需要关心异步，直接按照同步的写就行

await 关键字后面必须跟着一个 Promise 对象，在一个 async 方法中，可以等效的认为程序在运行到 await 后停止，直到 Promise 运行结束

```js
async function fun() {
    const rst1 = await func1('第一个参数');
    const rst2 = await func2(rst1);
    const rst3 = await func3(rst2);
    // ...
}
```

>   只要异步调用在逻辑上是串行的，那么就可以写成这种形式，尽管是异步调用，但写法上还是同步的

而一定要注意，如果两个异步函数之间不存异步逻辑的关系，还是不要写成两个 await 了，不然效率就低了

```js
async function fun() {
	const rst1 = await func1('第一个参数');
    const rst2 = await func2('第二个参数');
}
```

注意上面的写法是存在问题的，不要这么写，因为异步第二个异步调用会在第一个异步调用结束后才进行，效率变低了

>   你又不是调用链，没必要等前一个调完再调

这里推荐的写法是利用 Promise.all 进行组合

```js
async function fun() {
	const rst1 = await func1('第一个参数');
    const rst2 = await func2('第二个参数');
	const [r1,r2] = Promise.all([rst1, rst2]);
}
```

>   此时真正的返回值放在了 r1 和 r2 中

对于使用了 async-await 的函数，可以使用普通的 try-catch 进行异常捕获

```js
async function fun() {
    try {
        const rst1 = await func1('第一个参数');
        const rst2 = await func2(rst1);
        const rst3 = await func3(rst2);
        // ...
    } catch(error) {
        console.log(error);
    }
}
```

# 随便看看 vue

按照官网上的教程，一点点来

## 刘姥姥进大观园

### hello 例程

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>vue</title>
</head>
<body>
	<div id="hello-vue">
		{{message}}
	</div>
<!--在线 cdn 的方式导入 js 文件-->
<script src="https://cdn.jsdelivr.net/npm/vue@3.2.37/dist/vue.global.js"></script>
<script type="text/javascript">
	const APP = {
		data() {
			return {
				message: "hello vue"
			}
		}
	}
    <!--绑定了一个 id 为 hello-vue 的组件，这个组件的 message 属性由 APP 对象的data 决定-->
	Vue.createApp(APP).mount("#hello-vue");
</script>
</body>
</html>
```

这其实就是一个声明式渲染，此外通过 v-bind 还可以将标签的指定属性和 vue 实例的 property 绑定

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>vue</title>
</head>
<body>
	<div id="hello-vue">
		<span v-bind:tile="message">
            悬浮一会
        </span>
	</div>
<!--在线 cdn 的方式导入 js 文件-->
<script src="https://cdn.jsdelivr.net/npm/vue@3.2.37/dist/vue.global.js"></script>
<script type="text/javascript">
	const APP = {
		data() {
			return {
                // 加了一个日期，这样页面显示的内容会随着时间改变
				message: "hello vue" + new Date();
			}
		}
	}
    <!--绑定了一个 id 为 hello-vue 的组件，这个组件的 message 属性由 APP 对象的data 决定-->
	Vue.createApp(APP).mount("#hello-vue");
</script>
</body>
</html>
```

### 处理用户输入

前面通过 v-bind 绑定 span 标签(节点) 的属性和实例中的属性

现在通过 v-on 绑定事件和方法，下面是一个点击按钮使得显示文本反转的实例

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>vue</title>
</head>
<body>
	<div id="hello-vue">
		<p>{{message}}</p>
		<button v-on:click="reverseMsg">点我反转上面的文本</button>
	</div>
	
<script src="https://cdn.jsdelivr.net/npm/vue@3.2.37/dist/vue.global.js"></script>
<script type="text/javascript">
	const APP = {
		data() {
			return {
				message: "hello vue"
			}
		},
		methods: {
			reverseMsg() {
				this.message = this.message.split('').reverse().join('')
			}
		}

	}
	Vue.createApp(APP).mount("#hello-vue")
</script>
</body>
</html>
```

通过 v-model 实现表单输入和数据的双向绑定，修改输入框中的数据可以被 APP 中的 message 捕获

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>vue</title>
</head>
<body>
	<div id="hello-vue">
		<p>{{message}}</p>
		<input type="text" v-model="message">
	</div>
	
<script src="https://cdn.jsdelivr.net/npm/vue@3.2.37/dist/vue.global.js"></script>
<script type="text/javascript">
	const APP = {
		data() {
			return {
				message: "hello vue"
			}
		}
	}
	Vue.createApp(APP).mount("#hello-vue")
</script>
</body>
</html>
```

### 分支和循环

vue 必然支持可选展示，通过 v-if 展示部分内容

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>vue</title>
</head>
<body>
	<div id="hello-vue">
		<p v-if="flag">{{message}}</p>
		<button v-on:click="trans">点击查看或隐藏文本内容</button>
	</div>
	
<script src="https://cdn.jsdelivr.net/npm/vue@3.2.37/dist/vue.global.js"></script>
<script type="text/javascript">
	const APP = {
		data() {
			return {
				message: "hello vue",
				flag: true
			}
		},
		methods: {
			trans() {
				this.flag = !this.flag
			}
		}

	}
	Vue.createApp(APP).mount("#hello-vue")
</script>
</body>
</html>
```

有 if 必然也有 for 用来列表展示

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>vue</title>
</head>
<body>
	<div id="hello-vue">
		<ol>
			<li v-for="thing in myList">
				{{thing.msg}}
			</li>
		</ol>
	</div>
	
<script src="https://cdn.jsdelivr.net/npm/vue@3.2.37/dist/vue.global.js"></script>
<script type="text/javascript">
	const APP = {
		data() {
			return {
				myList: [
					{msg: "学点 vue"},
					{msg: "学点 java"},
					{msg: "看会 CSAPP"}
				]
			}
		}
	}
	Vue.createApp(APP).mount("#hello-vue")
</script>
</body>
</html>
```

### 组件

前端的页面很多部分都是复用的，之前一直都是一个组件(父组件)，一个应用，比如上面的例子中，把列表单独拆出来当一个组件

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>vue</title>
</head>
<body>
	<div id="hello-vue">
		<ol>
			<list></list>
		</ol>
	</div>
	
<script src="https://cdn.jsdelivr.net/npm/vue@3.2.37/dist/vue.global.js"></script>
<script type="text/javascript">
	const list = {
		template: '<li>this is my list</li>'
	}

	const APP = {
		components:{
			list
		}
	}
	Vue.createApp(APP).mount("#hello-vue")
</script>
</body>
</html>
```

这看上去总感觉缺了点什么，我希望子组件可以从父组件接受参数

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>vue</title>
</head>
<body>
	<div id="hello-vue">
		<ol>
            <!-- v-bind 绑定 list 组件中的参数 item 为 v-for 循环中的 item-->
			<list v-for="item in myList" v-bind:item="item"></list>
		</ol>
	</div>
	
<script src="https://cdn.jsdelivr.net/npm/vue@3.2.37/dist/vue.global.js"></script>
<script type="text/javascript">
	const list = {
		props:['item'],
		template: '<li>{{item.msg}}</li>'
	}

	const APP = {
		data() {
			return {
				myList: [
					{msg: "学点 vue"},
					{msg: "学点 java"},
					{msg: "看会 CSAPP"}
				]
			}
		}
		components:{
			list
		}
	}
	Vue.createApp(APP).mount("#hello-vue")
</script>
</body>
</html>
```

官方建议的一个前端的基本结构为：

```html
<!--父组件绑定 app-->
<div id="app">
  <!--子组件 app-nav、app-view、app-sidebar、app-content-->
  <app-nav></app-nav>
  <app-view>
    <app-sidebar></app-sidebar>
    <app-content></app-content>
  </app-view>
</div>
```

## 应用 & 组件实例

前面的例程中也已经看到了 vue 应用通过 Vue.createApp() 启动

这个方法需要一个参数，根据前面的经验，这里需要的是最顶级的根组件

而在绑定前端表现的时候，需要调用方法 .mount() 就像实例里那样

>   后面举例的时候都假设挂载在 app 中，假设存在标签 \<div id="app">\</div>

```js
// 这个方法会返回一个实例 app
const app = Vue.createApp({});
// 可以通过链式编程的方式使得 app 绑定其他的组件，mount 一个 DOM 元素中
app
    .component('my-component', {})
    .mount("#app");
```

app.mount() 方法返回一个 ViewModel，下面写为 vm

一些比较好的设计中，都是一个跟组件下面嵌套了若干组件，而组件还可以递归嵌套

```
Root Component
└─ TodoList
   ├─ TodoItem
   │  ├─ DeleteTodoButton
   │  └─ EditTodoButton
   └─ TodoListFooter
      ├─ ClearTodosButton
      └─ TodoListStatistics
```

### 组件实例的 proptery

在前面的实例中看到了一个 Vue 组件具有的 property: data、methods、props，显然 Vue 丰富的 proterty 肯定不止这些

```js
const RootComponent = {
    data() {
        return {
            msg: "this is vue";
        }
    }
};
const app = Vue.createApp(RootComponent).mount("#app");
const vm = app.mount("#app");
console.log(vm.msg);
```

>   还有比如 computed、inject、setup 等

### 声明周期钩子函数

一个组件被创建需要经历一系列初始化过程，在这些过程中需要运行一些钩子函数，比如

```javascript
const RootComponent = {
    data() {
        return {
            msg: "this is vue"
        }
    },
    created() {
        console.log(this.msg);
    }
};
const app = Vue.createApp(RootComponent);
const vm = app.mount("#app");
```

那么在根组件初始化的过程中调用 create() 方法的时候会打印输出"this is vue"

其他的函数比如 mounted()、updated()、unmounted()

一个 vue 的生命周期如下：

![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/vue_lifecycle.svg)

## 模板语法

### 插值

把 vue 组件中的值插入到 html 标签中

#### 简单文本

这个前面看到很多了，一个简单的 {{msg}} 即可将组件中 msg 的值作为文本插入到页面中

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>vue</title>
</head>
<body>
	<div id="app">
		{{msg}}
	</div>
	
<script src="https://cdn.jsdelivr.net/npm/vue@3.2.37/dist/vue.global.js"></script>
<script type="text/javascript">
	const RootComponent = {
    	data() {
        	return {
            	msg: "this is vue"
        	}
    	}
	};
	const app = Vue.createApp(RootComponent);
	const vm = app.mount("#app");
</script>
</body>
</html>
```

如果标签带有 v-once 那么被插入的值不会随着组件 property 的变化而变化

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>vue</title>
</head>
<body>
	<div id="app">
		<span v-once>{{msg}}</span></br>
		<button v-on:click="changeMsg">修改文本</button>
	</div>
<script src="https://cdn.jsdelivr.net/npm/vue@3.2.37/dist/vue.global.js"></script>
<script type="text/javascript">
	const RootComponent = {
    	data() {
        	return {
            	msg: "this is vue"
        	}
    	},
    	methods: {
    		changeMsg() {
    			this.msg="text has been changed"
    		}
    	}
	};
	const app = Vue.createApp(RootComponent);
	const vm = app.mount("#app");
</script>
</body>
</html>
```

>   把 v-once 去掉，单击后修改文本，加上 v-once 文本不会修改

#### Attribute

关键在于[v-bind](https://v3.cn.vuejs.org/api/directives.html#v-bind)

比如[前面的](#hello 例程) title 属性，将标签属性绑定到 vue 实例中的数据中

我在示例中看到一个比较有意思的：

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>vue</title>
</head>
<body>
	<div id="app">
		<img v-bind:src="src">
	</div>
	
<script src="https://cdn.jsdelivr.net/npm/vue@3.2.37/dist/vue.global.js"></script>
<script type="text/javascript">
	const RootComponent = {
    	data() {
        	return {
            	src: "https://cdn.jsdelivr.net/gh/SunYuanI/img/img/vue_lifecycle.svg"
        	}
    	}
	};
	const app = Vue.createApp(RootComponent);
	const vm = app.mount("#app");
</script>
</body>
</html>
```

>   绑定了 img 标签的 src 属性

所以 v-bind 就是用来绑定标签属性的

#### 来点别的

简单的属性插值确实比较单调 vue 中支持插入属性表达式

```html
<div id="app">
    <!--插入属性 number + 1-->
    {{ number + 1 }}
	<!--根据属性 ok 的布尔值选择展示 'YES' 或 'NO' -->
    {{ ok ? 'YES' : 'NO' }}
	<!--直接展示 message 反转后的样式-->
    {{ message.split('').reverse().join('') }}
	<!--字符串拼接构造标签的 id-->
    <div v-bind:id="'list-' + id"></div>
</div>
```

### 指令

就是所有的 v- attribute，比如前面的 v-bind、v-on、v-if、v-for

v-bind 指令的参数是标签的 attribute 值，比如 title、src、id

v-on 指令的参数是监听的事件名

...

#### 动态参数

指令的参数本身可以是动态的比如有一个标签：<a v-bind:[attributeName]="url"> ... \</a>

这里的 `attributeName` 会被作为一个 JavaScript 表达式进行动态求值，如果组件实例有一个 data property `attributeName`，其值为 `"href"`，那么这个标签等价为：\<a v-bind:href="url"> ... \</a>

#### 缩写

vue 给 v-bind 和 v-on 提供了缩写

```html
<!-- 完整语法 -->
<a v-bind:href="url"> ... </a>

<!-- 缩写 -->
<a :href="url"> ... </a>

<!-- 动态参数的缩写 -->
<a :[key]="url"> ... </a>


<!-- 完整语法 -->
<a v-on:click="doSomething"> ... </a>

<!-- 缩写 -->
<a @click="doSomething"> ... </a>

<!-- 动态参数的缩写 -->
<a @[event]="doSomething"> ... </a>
```

总之 'v-bind:' 使用 ':' 替代，而 'v-on:' 使用 '@' 替代

## Data 和 Methods

这两个属性前面已经见过很多次了，一个用来绑定数据，一个用来绑定事件

### Data

组件的 Data 是一个函数，Vue 的组件实例化时会调用这个函数，这个函数会返回一个对象，在 Vue 的实例中通过 $data 引用

```js
const RootComponent = {
    data() {
        return {
            msg: "this is vue",
        }
    }
};
const app = Vue.createApp(RootComponent);
const vm = app.mount("#app");
console.log(vm.msg);	// this is vue
console.log(vm.$data.msg);	// this is vue
console.log(vm.msg === vm.$data.msg);	//true
vm.msg = "change msg";	// 修改的数据会被实时更新到组件实例的 $data 对象中
console.log(vm.$data.msg);	// change msg
```

### methods

在 methods 中 this 为本组件，可以用来获取组件的属性；其他的就不说了，看看后面的计算属性和 methods

## 计算属性(computed) 和侦听器(watch)

### 计算属性

举一个官网上的例子，在 vue 中模板(template) 中的内容时允许表达式计算的，比如：

```html
<div id="computed-basics">
  <p>Has published books:</p>
  <span>{{ author.books.length > 0 ? 'Yes' : 'No' }}</span>
</div>
```

可以根据 author 的 books 数组的长度，选择性的输出 yes 或 no

尽管官方支持这种写法，但是不建议，因为这里把逻辑判断和视图表现混在了一起，有点乱

官方建议使用计算属性达到同样的效果，所谓计算属性，其实就是 getter 和 setter 方法

就按照上面的例子可以写成

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>vue</title>
</head>
<body>
	<div id="app">
		<span>{{bookMsg}}</span>
	</div>
	
<script src="https://cdn.jsdelivr.net/npm/vue@3.2.37/dist/vue.global.js"></script>
<script type="text/javascript">
	const RootComponent = {
    	data() {
        	return {
            	author: {
            		name: "buzz",
            		books: [
            			 'java is the best language ever',
            			 'why IDEA is god',
            			 'maybe vue is not bad'
            		]
            	}
        	}
    	},
    	computed: {
    		bookMsg() {
    			return this.author.books.length > 0 ? 'books left' : 'no book left';
    		}
    	}
	};
    const app = Vue.createApp(RootComponent);
	const vm = app.mount("#app");
</script>
</body>
</html>
```

我们在组件中声明了 computed 中的一个计算属性 bookMsg，这其实就是一个 getter 方法，如果 vm.author.books 发生了改变，这个计算属性也会重新计算

#### computed 和 methods

既然计算属性的作用和 getter 方法相同，那我在 methods 中定义一个 getter 方法不也是一样的吗

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>vue</title>
</head>
<body>
	<div id="app">
		<p>{{ bookMsg() }}</p>
	</div>
	
<script src="https://cdn.jsdelivr.net/npm/vue@3.2.37/dist/vue.global.js"></script>
<script type="text/javascript">
	const RootComponent = {
    	data() {
        	return {
            	msg: "this is vue",
            	author: {
            		name: "buzz",
            		books: [
            			'java is the best language ever',
            			'why IDEA is god',
            			'maybe vue is not bad'
            		]
            	}
        	}
    	},
    	methods: {
    		bookMsg() {
    			return this.author.books.length > 0 ? 'books left' : 'no book left'
    		}
    	}
	};
	const app = Vue.createApp(RootComponent);
	const vm = app.mount("#app");
</script>
</body>
</html>
```

从最终呈现的效果上，二者确实差不多，但是：**计算属性将基于它们的响应依赖关系缓存**。计算属性只会在相关响应式依赖发生改变时重新求值。这就意味着只要 `author.books` 还没有发生改变，多次访问 `bookMsg` 时计算属性会立即返回之前的计算结果，而不必再次执行函数。

#### setter

计算属性默认仅有一个 getter，如果需要的话需要手动配置 setter

```js
computed: {
  fullName: {
    // getter
    get() {
      return this.firstName + ' ' + this.lastName
    },
    // setter
    set(newValue) {
      const names = newValue.split(' ')
      this.firstName = names[0]
      this.lastName = names[names.length - 1]
    }
  }
}
```

更新 fullName 后 vm.firstName 和 vm.lastName 都会得到更新

### 侦听器

计算属性平时已经够用了，但 vue 中可以使用 watch 侦听数据变化，更加通用，经常用在当数据发生变化后，**需要执行异步操作**或其他开销比较大的操作

官方的话就是使用了 axios 进行异步请求的例子：

```html
<div id="watch-example">
  <p>
    Ask a yes/no question:
    <input v-model="question" />
  </p>
  <p>{{ answer }}</p>
</div>
<script src="https://cdn.jsdelivr.net/npm/vue@3.2.37/dist/vue.global.js"></script>
<script src="https://cdn.jsdelivr.net/npm/axios@0.12.0/dist/axios.min.js"></script>
<script>
  const watchExampleVM = Vue.createApp({
    data() {
      return {
        question: '',
        answer: 'Questions usually contain a question mark. ;-)'
      }
    },
    watch: {
      // 每当 question 发生变化时，该函数将会执行
      // 但只有当 quesion 以'?'结尾时才会发起 ajax 请求
      question(newQuestion, oldQuestion) {
        if (newQuestion.indexOf('?') > -1) {
          this.getAnswer()
        }
      }
    },
    methods: {
      getAnswer() {
        this.answer = 'Thinking...'
        axios
          .get('https://yesno.wtf/api')
          .then(response => {
            this.answer = response.data.answer
          })
          .catch(error => {
            this.answer = 'Error! Could not reach the API. ' + error
          })
      }
    }
  }).mount('#watch-example')
</script>
```

## v-if 和 v-show

[v-if 和 v-show](https://v3.cn.vuejs.org/guide/conditional.html#v-show)

使用 v-show 修饰的元素，始终会被渲染，并保留在 DOM 中，只不过在条件不满足的情况下 v-show 会修改 css 中的 display 样式控制

>   应该是在条件不满足的情况下，给 display 设置为 none

所以到底什么时候用 v-if 什么时候使用 v-show：因为 v-if 是条件渲染，元素在满足条件时会被创建，而在不满足条件时会被销毁，所以切换成本更高；而 v-show 不管什么情况都进行渲染，只不过会让 display 为 none，所以初始创建开销更大

如果切换频繁，那么使用 v-show；而如果不怎么切换，可以考虑 v-if(减少初次渲染的开销)

## v-for

v-for 前面用到过，可以通过列表的形式遍历一个数组

比如：

```html
<li v-for="item in items">
    {{ item.message }}
</li>
```

当然这是迭代器的写法，其实还可以获得下标

```html
<li v-for="(item, idx) in items">
    {{idx}} --- {{item.message}}
</li>
```

除了遍历数组还可以遍历对象

```html
<li v-for="(value, name, index) in myObject">
  {{ index }}. {{ name }}: {{ value }}
</li>
```

v-for 语句中的第一个参数为对象中的 val，第二个参数为对象中的 key，最后一个参数为遍历的下标索引

### 维护状态

在 v-for 中为每项提供唯一的 key，将写法修改为：

```html
<div v-for="item in items" :key="item.id">
  <!-- 内容 -->
</div>
```

key 取值必须唯一，且不能是 v-for 中得到的下标 idx

据说可以提升性能，避免数据错乱，原理想不明白，反正是有这么个事

### 实时检测更新

因为 v-for 用来遍历数组，因此数组修改时，需要让 vue 重新修改

数组的方法：push、pop、shift、unshift、splice、sort、reverse；这些都是在原数组上修改，因此 vue 会自动重新渲染

而如果使用的是 filter、concat、slice 的话，因为这些方法会返回一个新数组，所以需要手动将其替换给原数组

```js
vm.items = vm.items.filter(item => item.message.match(/Foo/))
```

## 表单数据绑定

使用 v-model 实现数据双向绑定

简单的文本输入框，绑定数据格式为字符串

单个复选框绑定数据格式为布尔值

多个复选框绑定数据格式为数组(多复选框绑定到同一个 v-model，那个对应的数据格式为数组，数组的取值为复选框的 value)

单选框绑定数据格式取决于单选框的 value

select 选择框(下拉选择框) 可以和 v-for 一起使用

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>vue</title>
</head>
<body>
	<div id="app">
		<select v-model="selected">
			<option v-for="option in options" v-bind:value="option.value">{{option.text}}</option>
		</select>
		{{selected}}
	</div>
<script src="https://cdn.jsdelivr.net/npm/vue@3.2.37/dist/vue.global.js"></script>
<script type="text/javascript">
	const RootComponent = {
    	data() {
        	return {
            	options:[
            		{text: 'One', value:'A'},
            		{text: 'Two', value:'B'},
            		{text: 'Three', value:'C'}
            	],
            	selected: 'A'
        	}
    	}
	};
	const app = Vue.createApp(RootComponent);
	const vm = app.mount("#app");
</script>
</body>
</html>
```

## 使用 vue-cli

配置好 [npm](./配置 win.md#node.js)

需要使用的是 [vue-cli](https://cli.vuejs.org/zh/) 和 [vue-router](https://router.vuejs.org/zh/)，具体的安装命令需要到官网上看，之前看了老教程，有的命令已经不一样了，走了点弯路

图形化展示使用的是 [Element Plus](https://element-plus.gitee.io/zh-CN/)(就是不想写 CSS)

### 起步

首先是创建项目：

```shell
$ vue create [project-name]
```

选 default，自定义的配置我看不懂

等待一会就会，新建好了，之后会告诉你

```shell
# ... 上面还有一些别的东西，就不管了
🎉  Successfully created project [project-name].
👉  Get started with the following commands:

 $ cd [project-name]
 $ npm run serve
```

就是说现在我们直接进入工作目录下就可以直接让运行了

进入工作目录下，可以看到默认已经使用 git 帮我们进行了版本控制了

添加 element-plus 依赖：

```shell 
$ npm install element-plus --save
```

使用 element-plus，这里选择完整导入，修改 main.js

```js
import { createApp } from 'vue'
import ElementPlus from 'element-plus'
import 'element-plus/dist/index.css'
import App from './App.vue'

createApp(App).use(ElementPlus).mount('#app')
```

### element-plus 初体验

#### 简单的导入几个按钮

在 /src/component 下添加新的组件，并将该组件添加到顶级的父组件 App.vue 中，这里以 element-plus 中的第一个按钮例子为例

```vue
<!--Welcome.vue-->
<template>
    <el-row class="mb-4">
    <el-button>Default</el-button>
    <el-button type="primary">Primary</el-button>
    <el-button type="success">Success</el-button>
    <el-button type="info">Info</el-button>
    <el-button type="warning">Warning</el-button>
    <el-button type="danger">Danger</el-button>
    <el-button>中文</el-button>
  </el-row>
</template>

<script>
export default {
    name: 'welcome-page'
}
</script>

<style>
</style>
```

直接从 element-plus 中复制过来就行

```vue
<!--App.vue-->
<template>
  <img alt="Vue logo" src="./assets/logo.png">
  <HelloWorld msg="Welcome to Your Vue.js App"/>
  <Welcome/>
</template>

<script>
import HelloWorld from './components/HelloWorld.vue'
import Welcome from './components/Welcome.vue'

export default {
  name: 'App',
  components: {
    HelloWorld,
    Welcome
}
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
```

在 template 中添加了一行，展示我们添加的按钮

最后 npm run serve 就可以看到展示页面了(这个按钮的样式通过 type 控制，肯定比我写的好看多了)

#### ~~使用 element-plus 的图标~~

==现在 element-plus 的图标已经可以被 [iconfont](#iconfont) 完美替代了==

之前 npm install 的时候已经下载的时候默认已经包括了 element-plus 的图标了，现在需要将其导入，参考官网上全局导入图标的方式：

```js
// main.js
// ... 其他依赖
import * as ElementPlusIconsVue from '@element-plus/icons-vue'

const app = createApp(App)
// 全局导入 element-plus 图标
for (const [key, component] of Object.entries(ElementPlusIconsVue)) {
    app.component(key, component)
}
```

然后就可以随便使用了

### axios

发起 ajax 请求，因为没有后端，为了方便调试，还需要使用 mockjs，伪造后端数据

[mockjs](https://github.com/nuysoft/Mock/wiki/Getting-Started)、[axios](https://github.com/axios/axios)

#### 简单看看 mockjs

先下好

```shell
npm install mockjs
```

为了方便管理在 /src 下新建目录 /mock，并在该文件夹下新建 index.js

按照官网的写法，在 index.js 中添加：

```js
// index.js
// 引入 Mock 对象
var Mock = require('mockjs');
// 定义模板
var template = {
    "name" : "buzz",
    "age" : 10,
    // 这里的 |+1 是 mockjs 中特有的写法，具体的可以进入 wiki 中查询
    "count|+1": 1,
}
// 处理来自 /mock/data.json 的 get 请求
Mock.mock("/mock/data.json", "get", template);
```

然后需要在 main.js 中引入当前文件

```js
// main.js
// ... 前面有若干依赖的配置
require('./mock')
// ... 后面是 app.mount() 之类的 
```

#### 使用 axios 请求一下

还是需要下好 axios

```js
npm install axios
```

还是上面的 welcome 页面，现在仅保留一个按钮，并添加点击事件为发出 ajax 请求

```vue
<!--Welcome.vue-->
<template>
    <el-row class="mb-4">
        <el-button type="primary" @click="fetchData">Primary</el-button>
        <div>{{msg}}</div>    
    </el-row>
</template>

<script>
// 引入 axios
import axios from 'axios'
export default {
    name: 'welcome-page',
    data() {
        return {
            // 先给一个初始值
            msg:1
        }
    },
    methods: {
        fetchData() {
            axios
                .get("/mock/data.json")
                .then(response => this.msg = response.data)
        }
    }
}
</script>
<style>
</style>
```

#### 加点难度

现在需求变了，我希望通过 post 的方式提交表单，表示登录，返回登录成功与否

因为前面引入了 element-plus，现在使用 element-plus 提供的表单，并且返回信息也使用 element-plus 内的反馈组件(对话框)进行展示

要实现这个功能其实很简单，首先创建 Login.vue

整体的页面并不复杂，就一个表单和一个对话框

```vue
<template>
  <el-row :gutter="20">
    <el-col :span="6" :offset="9">
      <!--表单输入，将输入绑定到 form 中-->
      <el-form :model="form" label-width="120px">
        <el-form-item label="Username">
          <el-input v-model="form.name" />
        </el-form-item>
        <!--密码的输入，show-password 添加了展示密码的按钮-->
        <el-form-item label="Password">
          <el-input 
            v-model="form.password" 
            type="password"
            show-password/>
        </el-form-item>
        <el-form-item>
          <!--定义了两个方法，onSubmit 用来进行表单提交，reset 用于清空表单-->
          <el-button type="primary" @click="onSubmit">Login</el-button>
          <el-button type="danger" @click="reset">Reset</el-button>
        </el-form-item>
      </el-form>
      <!--对话框部分，v-model 用来绑定对话框是否出现，draggable 表示表单可以拖拽-->
      <el-dialog 
        v-model="dialogMsg.visable"
        draggable
        width="20%">
          <!--根据输入密码的正确与否，提供了两种对话框的 header-->
          <template #header>
            <strong v-if="dialogMsg.loginMsg">验证通过</strong>
            <strong v-else="dialogMsg.loginMsg">用户名和密码错误</strong>
          </template>
		  <!--根据输入密码的正确与否，提供了两种对话框的 "body"-->
          <span v-if="dialogMsg.loginMsg">正在跳转</span>
          <span v-else="dialogMsg.loginMsg">需要重新输入</span>
		  <!--对话框的 footer 其实就是两个按钮-->
          <template #footer>
            <span class="dialog-footer">
              <el-button @click="confirmMsg">Cancel</el-button>
              <el-button type="primary" @click="confirmMsg">Confirm</el-button>
            </span>
          </template>
      </el-dialog>
    </el-col>
  </el-row>
</template>

<script setup>
import axois from 'axios'
import { reactive } from 'vue'

const form = reactive({
  name: '',
  password: ''
})

const dialogMsg = reactive({
  visable: false,
  loginMsg: false
})

const onSubmit = () => {
  axois.post('/mock/data.json', {
    name: form.name,
    password:form.password
  }).then((response) => {
    console.log(response.data)
    dialogMsg.loginMsg = response.data == 1
    dialogMsg.visable = true
  }).catch((error) => {
    console.log(error)
  })
}

const reset = () => {
  form.name='';
  form.password=''
}

const confirmMsg = () => {
  dialogMsg.visable = false;
  reset();
}

</script>
<style>
.dialog-footer button:first-child {
  margin-right: 10px;
}
</style>
```

为了实现在不同的输入的情况下具有不同的相应，还需要修改 Mock 中的 index.js 部分

```js
var Mock = require('mockjs');
// 这里调用 mock 函数的格式为 Mock.mock(url, type, function(options))，具体的写法可以参考官网
Mock.mock("/mock/data.json", "post", (options) => {
    // 从 http request 的 body 中获取表单数据，并使用 JSON.parse 进行解析
    const obj = JSON.parse(options.body);
    const userName = obj.name;
    const password = obj.password;
    // 这里直接比较，只有特定的用户名和密码才可以登录
    if (userName === 'buzz' && password === "123") return 1;
    return 0;
});
```

到这里其实就已经将功能实现了，但现在存在一些改进空间和疑问

#### 表单例程的疑问

首先这个表单没有表单校验功能，这点可以进行优化，而且这个表单确实称不上好看，需要优化

此外在这个例程中，马马虎虎的用了一些之前没有使用过的东西，比如 script 标签中使用 setup，比如使用了 reactive 包裹了数据，比如我们发现这个组件的格式好像和其他的不太一样，他没有 method，没有 data()，甚至没有 name，这些都留到后面解释吧

#### 表单验证

element-plus 官方说如果希望使用高级一点的表单验证功能，需要参考 [async-validator](https://github.com/yiminghe/async-validator)

我实话实说，并没有看出来官方的表单验证是不是 async-validator 的封装，反正写法很奇怪，就按照官方的来吧

##### 简单的表单验证

就是按照官方的示例来的，虽然并不是都很明白，不过应该是能顺下来吧(大概:sweat_smile:)

```vue
<template>
    <el-row :gutter="20">
        <el-col :span="6" :offset="9">
            <!--表单数据通过 :model 绑定到 formData 中
                表单组件的实例使用 ref 绑定到 formRef 中
                表单的规则通过 :rules 绑定到 formRules 中-->
            <el-form
                size="default"
                label-position="left"
                label-width="auto"
                :model="formData"
                ref="formRef"
                :rules="formRules">
                <!--el-item 的 prop 属性绑定的是 :model 中的键名，即 formData 中的键名-->
                <el-form-item label="Username" prop="name">
                    <el-input v-model="formData.name"/>
                </el-form-item>
                <el-form-item label="Password" prop="password">
                    <el-input
                        type="password"
                        v-model="formData.password"
                        show-password />
                </el-form-item>
                <el-form-item>
                    <el-button type="primary" @click="submit(formRef)">登录</el-button>
                    <el-button type="default" @click="resetForm(formRef)">清空</el-button>
                </el-form-item>
            </el-form>
        </el-col>
    </el-row>
    <el-dialog
        v-model="dialogMsg.visable"
        draggable
        width="20%">
        <template #header>
            <strong v-if="dialogMsg.loginMsg">验证通过</strong>
            <strong v-else>用户名或密码错误</strong>
        </template>
        <span v-if="dialogMsg.loginMsg">正在跳转</span>
        <span v-else>需要重新输入</span>
        <template #footer>
            <el-button type="primary" @click="confirmDialog(formRef)">确认</el-button>
            <el-button type="default" @click="confirmDialog(formRef)">取消</el-button>
        </template>
    </el-dialog>
</template>

<script lang="ts" setup>
import axios from 'axios';
import { ref, reactive } from 'vue';
// element-plus 官方文档中的写法，应该是和下面带有泛型的 ref 和 reactive 有关
import type { FormInstance, FormRules } from 'element-plus';
// 官方的写法，formRef 是 FormInstance 的一个 ref 包装
const formRef = ref<FormInstance>();
// 官方的写法，针对 formData 中的每一项使用一个数组对象进行修饰，表示了验证
// 比如下面这两个验证其实都是验证非空就能通过
const formRules = reactive<FormRules>({
    name: [
        {
            required: true,
            message: 'please input username',
            trigger: 'blur',
        },
    ],
    password: [
        {
            required: true,
            message: 'please input password',
            trigger: 'blur',
        },
    ],
});

const formData = reactive({
    name: '',
    password: '',
});

const dialogMsg = reactive({
    visable: false,
    loginMsg: false,
});
// 在提交的时候进行验证
const submit = (formEl: FormInstance | undefined) => {
    if (!formEl) return;
    // validate 方法有一个参数 callback，为一个回调函数，在 form 组件实例进行参数验证后调用这个 callback
    // 这个 callback 有两个参数，第一个参数 isValid 是 boolean 类型，第二个 invalidFields 是 ValidateFieldsError 类型
    // 第一个参数是必须使用的，它表明了表单校验是否通过
    // 第二个参数是可选的，它表明了那些没有校验通过的 form-item
    formEl.validate((valid, fields) => {
        if (valid) {
            console.log('check finish');
            axios.post('/mock/data.json', {
                name: formData.name,
                password: formData.password,
            }).then((response) => {
                dialogMsg.loginMsg = response.data === 1;
                dialogMsg.visable = true;
            });
        } else {
            console.log('check error', fields);
        }
    });
};

const resetForm = (formEl: FormInstance | undefined) => {
    if (!formEl) return;
    formEl.resetFields();
};

const confirmDialog = (formEl: FormInstance | undefined) => {
    resetForm(formEl);
    dialogMsg.visable = false;
};
</script>
<style>
</style>

```

可以看到，这里面表单校验的核心方法为 validate 方法，这里返回一个 promise 对象，具体的，可以参考 async-validator 的 [validate](https://github.com/yiminghe/async-validator#validate) 方法

所以其实上面的逻辑完全可以修改为：

```typescript
formEl.validate().then(() => {
    console.log('check finish');
    axios.post('/mock/data.json', {
        name: formData.name,
        password: formData.password,
    }).then((response) => {
        dialogMsg.loginMsg = response.data === 1;
        dialogMsg.visable = true;
    });
	// 反正在 async-validator 中使用 catch 是可以捕获 errors 和 fields 的，而在这里就不行了
}).catch(({ errors, fields }) => {
    console.log('check fail');
    console.log(errors);
    console.log(fields);
});
```

##### 带有自定义的表单校验

formRules 对象指定了校验规则，在提交表单时，调用 formRef.validate() 时将根据 formRules 中的规则进行校验

如果需要自定义校验规则，那么应该从 formRules 下手，上例中的规则如下：

```typescript
const formRules = reactive<FormRules>({
    name: [
        {
            required: true,
            message: 'please input username',
            trigger: 'blur',
        },
    ],
    password: [
        {
            required: true,
            message: 'please input password',
            trigger: 'blur',
        },
    ],
});
```

按照官方的写法，就是在 rules 中的添加一个 validator 字段，指向一个自定义的校验函数即可，修改后如下

```typescript
const usernameRule = (rule: any, value: string, callback: any) => {
    if (!value || value === '') {
        callback(new Error('Please intput Username'));
    }
    if (value.length < 3) {
        callback(new Error('用户名太短'));
    }
    callback();
};

const passwordRule = (rule: any, value: string, callback: any) => {
    if (!value || value === '') {
        callback(new Error('please input Password'));
    }
    if (value.length > 3) {
        callback(new Error('密码太长'));
    }
    callback();
};

const formRules = reactive<FormRules>({
    name: [
        {
            required: true,
            validator: usernameRule,
            trigger: 'blur',
        },
    ],
    password: [
        {
            required: true,
            validator: passwordRule,
            trigger: 'blur',
        },
    ],
});
```

这个 validator 的格式也完全是按照官方的建议来的，三个参数(虽然第一个参数一直没用上)

第二个参数 value 就是我们的表单输入，最后一个 callback 不需要管，但一定要有，且官方强调了，不管校验是否成功，都需要调用这个 callback

## 使用 vite

[Vite | 下一代的前端工具链 (vitejs.dev)](https://cn.vitejs.dev/)，(我超，无情)

新的打包工具应该比老的要好吧... 大概

### 新建一个 vite 项目

```shell
$ npm create vite@latest [project-name] --template vue-ts
```

>   创建了一个使用 vite 构建的基于 typescript 的 vue 项目

第一次运行：

```shell
# 添加依赖
$ npm install
# 运行项目(这个可以在 package.json 中改)
$ npm run dev
```

## 如何在 vue 中使用 markdown

因为网上的插件还是挺多的

### v-md-editor

[Introduction | v-md-editor (code-farmer-i.github.io)](https://code-farmer-i.github.io/vue-markdown-editor/)

>   看不懂了再看中文文档[介绍 | v-md-editor (code-farmer-i.github.io)](https://code-farmer-i.github.io/vue-markdown-editor/zh/)
>
>   最好不要一上来就中文文档，他这个文档翻译的不是很好，所以有的地方看起来很奇怪

#### 先试试吧

```shell
# 添加依赖
$ npm i @kangc/v-md-editor@next -S
```

然后在 main.ts 中注册

```typescript
// main.ts
import { createApp } from 'vue'
import './style.css'
// 引入 Base Editor，基本的组件
import VueMarkdownEditor from '@kangc/v-md-editor';
import '@kangc/v-md-editor/lib/style/base-editor.css';
// 使用 VuePress 主题，官方默认也可以使用 github 的主题
import vuepressTheme from '@kangc/v-md-editor/lib/theme/vuepress.js';
import '@kangc/v-md-editor/lib/theme/style/vuepress.css';
import App from './App.vue'
// 引入代码高亮插件
import Prism from 'prismjs';

VueMarkdownEditor.use(vuepressTheme, {
    Prism,
});

const app = createApp(App);
app.use(VueMarkdownEditor);
app.mount('#app');
```

这里报错了，因为 ts 工程中引入的 js modules，报错为引入了类型 implicit any 的 modules

**最好的方法就是把 js 的 module 换成 ts 版本的**，但这招不是通用的，有的包他就是 js 版本的，对于这种情况，官方的解决方法：[TypeScript: Documentation - Modules (typescriptlang.org)](https://www.typescriptlang.org/docs/handbook/modules.html#shorthand-ambient-modules)

在 src 目录下新建文件 js-modules.d.ts

```typescript
// js-module.d.ts
declare module '@kangc/v-md-editor*';
```

而至于 prismjs，本身就提供了 ts 版本的包，直接导入即可：

```shell
$ npm i --save-dev @types/prismjs
```

### mavon-editor

这里选择的 mavon-editor 官方已经支持了 typescript，不过这里还是有坑(至少 22/8/16 的时候是有坑的)

首先安装的时候，注意因为我们使用的是 vue3 的版本，所以安装依赖的时候，应该安装 v3 版本：

```shell
$ npm install mavon-editor@next --save
```

>   显然不仅仅开发的时候需要这个包，实际运行的时候也需要这个包

## 简单看看响应性

为了创建响应式状态，需要使用 reactive 方法

```js
import {reactive} from 'vue'

const state = reactive({
    count: 0
})
```

响应式状态改变时，视图会自动更新，比如上面在表单例程中使用响应式状态 form，form 中具有两个属性 username 和 password 分别绑定了表单中的两个输入框

其实这里和之前的 data() 是类似的，我们在 data() 中的属性绑定到标签上后也会自动更新，

特别的对于独立的一个值(而不是一个对象)，可以使用 ref 使其变为响应式的

```js
import {ref} from 'vue'

const count = ref(0)
```

ref 也会返回一个响应式对象，内部维护一个值

### ref 解包

也是官网上看到的，感觉有病

```js
import {reactive, ref} from 'vue'

const count = ref(0)
// 拿 reactive 进一步将 ref 包裹起来了
const state = reactive({
  count
})
// 访问的时候直接 state.count 就行
console.log(state.count) // 0
// 显然响应式访问也是支持的
state.count = 1
console.log(count.value) // 1
```

上面都是 reactive 包裹了一个 ref 对象的情况，如果 reactive 包裹的是一个 ref 数组(或者 map) 就需要使用 value 继续进行访问了

```js
import {reactive, ref} from 'vue'

const count = ref(0);

const state = reactive([count])

console.log(state[0]); // 这个返回值看不懂

console.log(state[0].value); // 0
```

### 使用 readolny

有的时候可能需要防止某些程序对响应式对象修改，此时可以借助 readonly

```js
import {reactive, ref, readonly} from 'vue'

const count = ref(0);

const state = reactive([count]);

const unchanged = readonly(state);

unchanged.state[0].value++;	// 这条会报错
```

## 快点用上 \<script setup>

之前的 SFC 文件的基本格式是：

```vue
<template></template>
<script>
import ...
export default{
    props: {
        
    }
    data() {
    	return {
    
		}
	},
    methods: {
        
    },
}
</script>
<style></style>
```

现在变为了

```vue
<template></template>
<script setup>
import ref from 'vue'

const val = ref([...]);
                 
</script>
<style></style>
```

需要方法，直接写就行；需要组件，直接引入就行，此外为了得到响应式状态，需要引入 ref 和reactive

使用组件的时候官方是建议和 PascalCase 格式保持一致

```vue
<template>
	<MyComponent/>
	<!--写成 <my-component> 也行-->
</template>
<script setup>
import MyComponent from './MyComponent.vue'
    
</script>
```

对于一个组件来说，使用 props 和 emits 来声明自己的参数和方法，而在 setup 中需要使用 defineProps 和 defineEmits

比如：

```vue
<script setup>
const props = defineProps({
  foo: String
})

const emit = defineEmits(['change', 'delete'])

</script>
```

## 过渡

[进入过渡 & 离开过渡 | Vue.js](https://v3.cn.vuejs.org/guide/transitions-enterleave.html#单元素-组件的过渡)

Vue 本身提供了 transistion 组件，可以为元素或组件提供进入和离开的过渡

一般来说，使用了 v-if(或者 v-show) 使得组件在满足一定条件时才显示，而 transition 就是使得这些条件显示的组件在进入和离开的时候具有过渡效果，官方的那个例子很好

### 过渡 class

官方提供了 6 个 class 样式控制，可以分成两组，其中带有 enter 的表示在组件出现过程中的三个阶段；leave 表示在组件消失过程中的三个阶段

![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/vue_transitions.svg)

enter-from 为过渡开始前的样式(一般我都写成 opacity: 0)

enter-to 定义过渡结束时的状态(一般我都写成 opacity: 1)

enter-active 表示过渡阶段的状态(一般都是 transition:[property] [duration] [timing-function] [delay]，具体的可以参考[transition](./css_傻逼.md#隐式过渡(transition))

要注意的是，上面的三个 class 仅仅表示整个 enter 过程中样式，在 enter 结束后，这三个 class 就不起作用了，后面的样式控制就仅仅取决于组件本身了

leave 也是类似的

按照官方的教程，如果给 transition 指定了 name 属性，那么在定义 class 的时候，就应该定义 [name]-enter-from、[name]-enter-to、[name]-enter-active...

## font awesome

就是一个图标库，官网在 [Font Awesome](https://fontawesome.com/)

使用的时候其实还是很方便的，就是安装的步骤有点多

```shell
# 安装 font awesome 的核心库
npm install --save @fortawesome/fontawesome-svg-core
# 添加 icon 样式，其实只有这三种样式是免费的，而且这三种当中也不都是免费的
npm install --save @fortawesome/free-solid-svg-icons
npm install --save @fortawesome/free-regular-svg-icons
npm install --save @fortawesome/free-brands-svg-icons
# 添加组件
npm install --save @fortawesome/vue-fontawesome@latest-3
```

然后就是修改 main.js 了

```js
import {createApp} from 'vue'
import App from './App.vue'
// 引入 fontawesome core
import {library} from '@fortawesome/fontawesome-svg-core'
// 引入 font awesome icon 组件
import {FontAwesomeIcon} from '@fortawesome/vue-fontawesome'
// 引入 指定类型的 icon，这里我们使用的 solid 类型，即实线
import {faUserSecret} from '@fortawesome/free-solid-svg-icons'

/* add icons to the library */
library.add(faUserSecret)
const app = createApp(App)
// 添加组件 font-awesome-icon
app.component('font-awesome-icon', FontAwesomeIcon)
app.mount("#app")
```

## [iconfont](https://www.iconfont.cn/)

一个阿里的矢量图库，图标个数相比 element-plus 肯定是多太多了

### 图形化界面就是好

首先进入官网，在图形化界面中选择合适的图标，加入购物车，添加到项目中

随后选择将图标下载至本地，注意，如果有彩色图片要求的，需要在项目设置需要在[字体格式]中勾选上[彩色]

下载的时候选择 Symbol 格式的下载(尽管兼容性差，可我也不是前端啊)

### 导入项目

下载好的是一个压缩包，里面有一个说明作用的 demo.html，这个**重点看**

我们需要导入的是除了 demo.html 和 demo.css 的所有文件，因为图标，及其格式，字体都是静态文件，所以这里统一在 assests 目录下新建 iconfont 目录保存管理

>   反正我当前这个版本导入的时候是 6 个文件

随后根据 demo.html 中的说明，我们需要在 main.js 中导入

```js
import '@/assets/iconfont/iconfont';
```

>   注意最后的 iconfont 指代的是 iconfont.js 这里之所以没有写成 iconfont.js 是因为如果添加上后 eslint 总是会报出莫名其妙的异常

因为我这里使用了 eslint 所以导入后报出了一堆异常，都是针对 iconfont.js 的，官方给出的文件，运行肯定不会出大问题，所以这里异常其实都是格式化的要求，这里选择让 eslint 忽略这些异常

>   具体的方法可以看[让 eslint 忽略检查某些文件](#让 eslint 忽略检查某些文件)

### 模块化 icon

在 demo.html 中我们看到了引入了一个图标，需要写那么多标签，写好后还需要样式控制，图标多了之后就很麻烦

我使用 iconfont 就是希望使用图标，而不是繁琐的配置，一个名字就对应了一个图标，所以正常来说，我也只需要提供一个名字，就需要给我显示一个图标

那么这里就图标的配置抽象，进而抽取成 vue 中的一个组件 IconComponent

```vue
<template>
    <svg class="icon" aria-hidden="true">
        <!-- 按照 demo.html 的官方写法，选择一个图标，其中 realName 绑定了图标名 -->
        <use :xlink:href=realName></use>
    </svg>
</template>

<script lang="ts" setup>
import { defineProps, computed, reactive } from 'vue';
    // 在使用这个组件的时候，需要提供一个图标名，作为参数 iconName
    const props = defineProps({
        iconName: {
            type: String,
            require: true,
        },
        // icon 的大小
        size: {
            require: false,
            default: '1em',
        },
    });
    // 因为实际中阿里的图标名为 #icon-xxx，所以需要我们使用计算属性，为传入的参数添加上前缀
    // 这样传递参数的时候，只需要传递有效的图标名即可
    const realName = computed(() => `#icon-${props.iconName}`);
</script>

<style lang="less" scoped>
    // 完全按照阿里官方的写法，一笔没改
    .icon {
        height: 1em;
        width: 1em;
        // 绑定 css 中的 font-size 的大小为参数中的 size 属性
        font-size: v-bind('props.size');
        vertical-align: -0.15em;
        fill: currentColor;
        overflow: hidden;
    }
</style>

```

### 使用

现在我们已经写好一个 component，使用的时候，直接引入，传参即可

现在关键是图标名到底应该怎么写，这个其实可以参考 iconfont.json，这个文件中记录了刚刚我们在购物车中添加的所有图标的信息，当然也包括了图标名

更为具体的，需要参考 json 对象的 font_class 属性

毕竟是 iconfont，本质上修改 font 的属性都可以用来修改 icon，所以如果需要修改图标大小请使用 font-size 属性

### 后记

这里说的是在使用 iconfont 的过程中出现的问题

#### 关于无法改变图像颜色的问题

因为我是用的是 symbol 方式引入图标，即图标可以多色，在使用中，发现我无法通过修改 color(或者是 fill) 属性修改图标的颜色

问题就出现在通过这种方式引入有色图标，那么在 svg 的 path 标签中会添加 fill 属性，即默认赋予 svg 图像一个颜色，此时使用封装好的 IconComponent 的时候是无法正常修改颜色的

解决方法是，==进入到 iconfont.js 中，删除掉对应 path 标签的 fill 属性==

>   由此我们可以看出 iconfont 就是通过 js 代码动态导入 svg 标签实现的图标导入的

但如果这么做了，会让图标变灰掉，即失去了默认的颜色(这个也是很好理解的)，不过此时通过在 IconComponent 中设置 color 就可以改变图标的颜色了

所以现在其实是变得更麻烦了，因为有的图标我们希望颜色不变，而有的图标需要颜色改变，对于那些颜色不变的图标，我们可以保留他们的 fill 属性，而对于那些颜色改变的图标，就需要额外的传入 color 参数表示图标颜色默认颜色和激活状态下的颜色了，现在，封装好的 IconComponent 如下

```vue
<template>
    <svg class="icon" aria-hidden="true">
        <!-- 按照 demo.html 的官方写法，选择一个图标，其中 realName 绑定了图标名 -->
        <use :xlink:href=realName></use>
    </svg>
</template>

<script lang="ts" setup>
import { defineProps, computed } from 'vue';
    // 在使用这个组件的时候，需要提供一个图标名，作为参数 iconName
    const props = defineProps({
        iconName: {
            type: String,
            require: true,
        },
        // 图标的大小
        size: {
            require: false,
            default: '1em',
        },
        // 图标的透明度
        opacity: {
            type: Number,
            require: false,
            default: 1,
        },
        // 图标的颜色
        color: {
            type: String,
            require: false,
            default: 'currentcolor',
        },
    });
    // 因为实际中阿里的图标名为 #icon-xxx，所以需要我们使用计算属性，为传入的参数添加上前缀
    // 这样传递参数的时候，只需要传递有效的图标名即可
    const realName = computed(() => `#icon-${props.iconName}`);
</script>

<style lang="less" scoped>
    // 完全按照阿里官方的写法，一笔没改
    .icon {
        height: 1em;
        width: 1em;
        // 官方推荐的写法，props 中的 size 属性绑定 font-size 用来修改组件大小
        font-size: v-bind('props.size');
        opacity: v-bind('props.opacity');
        fill: v-bind('props.color');
        vertical-align: -0.15em;
        overflow: hidden;
    }
</style>

```

## 添加动画

这里借助了[GreenSock](https://greensock.com/)，一个真正用于制作动画的 API

因为习惯了全局引入，所以这里使用 npm 的方式添加依赖(尽管官方建议使用下载静态资源的方式引入)

```shell
$ npm install gsap --save
```

>   因为实际生产环境下也需要这个依赖，所以使用的是 --save 

这个 API，说简单真的不难，所有的动画属性基本上都可以通过 gsap.to、gsap.fromTo 确定，随便看看文档就行了

### 一个简单的开发

这里仅简单说一下实际开发遇到的需求：为提交按钮添加一个 loading 图标，当尚未接收到相应时，显示 loading 图标，表示加载中

借助 el-button 的 icon slot，很容易添加一个图标，关键是如何让图标动起来

这里将 loading 图标封装为一个 component，在 mount 阶段进行动画的设置

loading 动画主要就是让它转起来，一个 rotate 就可以解决，主要需要设置的动画曲线

官方给了一个专门用户设置动画曲线的图形化工具: [GreenSock Ease Visualizer](https://greensock.com/docs/v3/Eases)，用法其实很简单，随便拉一下点就能设置好动画曲线

然后我们需要将自定义的动画曲线进行配置到静态 svg 图像上，这个参考 [GreenSock | Docs | Eases | CustomEase](https://greensock.com/docs/v3/Eases/CustomEase)

>   我选择的方式是直接复制 code

一个请求打过去不知道什么时候能接收到，所以这里设置图标一直旋转，具体的 API 参考 [GreenSock | Docs | GSAP | Tween | repeat()](https://greensock.com/docs/v3/GSAP/Tween/repeat())

>   其实正常的话，在图标不显示的时候应该让图标停止旋转，具体可以参考：[GreenSock | Docs | GSAP | Tween | paused()](https://greensock.com/docs/v3/GSAP/Tween/paused())，这里也偷懒了，其实就算让他一直转影响也没那么大

封装好的组件如下：

```vue
<template>
    <div>
        <!-- 这里的 icon-component 是一个封装好的 component 用到了阿里的 iconfont -->
        <icon-component :icon-name="iconName" id='loading'/>
    </div>
</template>

<script lang="ts" setup>
import IconComponent from './IconComponent.vue';
import { onMounted, watch, ref } from 'vue';
import gsap from 'gsap';
import CustomEase from 'gsap/CustomEase';
// 这里仅仅是留出了两个接口，其实后面并没有用到，不过为了组件的通用性还是留下了
const props = defineProps({
    // loading 组件的名字
    iconName: {
        require: false,
        type: String,
        default: 'loading',
    },
    // 是否处于 pause(暂停) 状态
    isPause: {
        require: false,
        type: Boolean,
        default: false,
    },
});
// loading 动画实体
const loading_anime = ref();
// 在 mount 阶段进行动画的配置
onMounted(() => {
    // 配置上 CustomEase 插件
    gsap.registerPlugin(CustomEase);
    // 配置上自定义的动画曲线
    CustomEase.create("custom",
    `M0,0 C0,0 0.171,0.019 0.272,0.05 0.358,0.076 0.424,0.126 0.458,
    0.16 0.49,0.192 0.512,0.26 0.522,0.334 0.551,0.549 0.518,0.302 0.562,
    0.598 0.582,0.734 0.641,0.825 0.652,0.836 0.665,0.849 0.684,
    0.882 0.724,0.912 0.76,0.939 0.812,0.959 0.838,0.968 0.912,0.994 1,1 1,1`);
    // 配置好动画实体
    loading_anime.value = gsap.to('#loading', {rotate: 360, duration: 0.5, repeat: -1, repeatDelay: 0, ease: "custom"});
});
// 这里其实已经写好了，只要 isPause 改变后，就让图标停止或旋转
watch(() => props.isPause, () => {
    loading_anime.value.paused(props.isPause);
});
</script>

<style lang="less" scoped>
</style>
```

这里[有一个坑](#坑)，后面说

因为使用的是 el-button 作为按钮，默认的 el-button 提供了 loading slot，直接用就好了

```vue
<template>
	<!-- 其他配置 -->
	<el-button
    class="login_btn"
    type="primary"
    @click.stop="postForm"
    :loading="btnConfig.loading">
    <template #loading>
    	<loading-icon id="loading_icon"/>
	</template>
	{{btnConfig.text}}
	</el-button>
	<!-- 其他配置 -->
</template>

<script lang="ts" setup>
    
const btnConfig = reactive<ButtonConfig>({
    loading: false,
    text: '登录'
});   
const postForm = () => {
    btnConfig.loading = true;
    btnConfig.text = '正在登录';
    postLogin(formData.username, formData.password).then((res) => {
        console.log(res);
        btnConfig.loading = false;
        btnConfig.text = '登录';
    });
};
</script>
```

具体的不用管了，反正就是一个按钮的简单配置

### 坑

注意到这里的 template 中仅有一个 icon-component 组件，那么它外边的 div 应该是没什么用的吧，==**个屁！！！**==

如果不添加这个 div，那么图标就无法正常显示，它出现之后，就留下来了，在接收到响应后，还有图标残留，怎么设置都没用

要么这是 vue 的一个小 bug，要么这是 gsap 的一个小 bug

## vue-router

vue 本身是单页面应用，但网络应用肯定不是只有一个页面的，在 vue 中的解决方案是：路由 + 组件

路由设定了访问路径，形成路径到组件的映射，所以 vue 中使用的跳转标签不是 \<a>，而是 \<router-link>，可以在不重新加载页面的情况下，修改页面的 URL 以及对应的编码

首先肯定是安装，去[官网](https://router.vuejs.org/zh/)看看安装的命令

```shell
$ npm install vue-router@4
```

### 照葫芦画瓢

一般的，在 src 下新建 /router 目录，这个目录下保存路由文件，这里先建立 /src/router/index.js

正常来讲 我们的页面应该放在 /src/views 下，页面本身也是一个组件，在一个页面应该具有若干个组件，而组件放在 /src/components 下，但我们前面的 welcome.vue(vue 自带的) 和 Login.vue 都放在了 /src/components 下，这里仅起到说明作用；好了总之我们知道正经项目中不应该这么随意就行了，现在按照网上的模板，抄一个路由模板

```js
// /src/router/index.js
import { createRouter, createWebHistory } from "vue-router";
// 别管了反正以后应该从 '@/views/...' 下引入页面
import Welcome from '@/components/Welcome.vue'
import Login from '@/components/Login.vue'
// routes 数组，进行访问路径(页面) 和组件的分配，注意这里是 routes 数组，而不是 routers
const routes = [
    {
        /* 
        	path 是路由分配的路径
        	name 是路由指向此页面时显示的名字
        	component 是此路径下加载的组件名
        */
        path: '/',
        name: 'HomePage',
        component: Welcome
    },
    {
        path: '/Login',
        name: 'LoginPage',
        component: Login
    }
]
// router 主体，我们创建了一个 history 模式的路由
const router = createRouter({
    history: createWebHistory(),
    routes
})

export default router;
```

>   这里先简单说一下 history 模式和 hash 模式最大的区别在于浏览器显示的链接地址不同
>
>   比如同样是访问 /home，在 hash 模式下为：http://localhost:8080/#/home；而在 history 模式下为：http://localhost:8080/home

接下来需要在 main.js 中引入刚刚加入的路由

```js
// main.js
import { createApp } from 'vue'
import App from './App.vue'
// 如果在 router 下使用的是 index.js 作为配置，那么最后的路径中就不需要写，否则需要将配置路径写到文件
import router from './router'
const app = createApp(App)
app.use(router)
app.mount("#app");
```

在 src 下有 App.vue，这个特别的组件，是项目的入口，我们的路由跳转应该在这个组件下进行，所以后面需要修改 App.vue

```vue
  <template>
    <div id="nav">
      <router-link to="/">首页</router-link> |
      <router-link to="/login">登录</router-link>
      <router-view />
    </div>
</template>

<script>
export default {
  name: 'App',
  components: {
	}
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
```

### 带有参数的路由

有的时候，我们的访问路径中包含了路由信息，比如用户组件会根据不同的用户 id 查询不同的用户数据，从而展示不同的数据，这就表明我们需要在路由信息中添加参数

比如现在有一个用户组件，访问的时候需要接收一个用户 id 的参数，那么路由信息 routes 数组可以写为：

```js
const routes = [
    {
        path: '/user/:userId',
        component: User
    }
]
```

注意到我们使用了 restful 风格的方式传递了用户信息，现在访问 /user/2，即表明访问第二个用户的信息，在 routes 中路径参数通过 ':' 表示，在 User 组件中，我们可以通过 $route.params.userId 的方式访问参数，比如：

```vue
<template>
	<h1>this is user: {{$route.params.userId}}</h1>
</template>
```

其实，使用正则表达式可以限制访问路径中的参数，比如上面的用户 id，我希望它只包含数字，那么我可以写成：

```js
const routes = [
    {
        path: '/user/:userId(\\d+)',
        component: User
    }
]
```

注意到括号中内容其实是一个正则表达式，其中 '\d' 表示匹配数字，'+' 表示匹配一个或多个

>   而最前面的 '\\'，其实是一个转义字符

既然可以做到正则表达式匹配，那么自然可以进行 404 路由匹配，只要我把正则表达式写成 '.*' 就可以匹配所有路由了

## 简单的状态管理

有的时候并不需要 vuex 这么复杂的状态管理插件，

利用响应式 API 完全可以实现简单的状态管理，具体的按照官方的来：[状态管理](https://staging-cn.vuejs.org/guide/scaling-up/state-management.html#simple-state-management-with-reactivity-api)

首先在项目的根目录下创建文件夹 /store，并在该文件夹下新建 store.js

>   甚至具有和 vuex 一样的目录结构

```js
// store.js
import { reactive } from 'vue'

export const store = reactive({
  count: 0
})
```

我们需要使用的时候，直接在组件中引入即可使用

```vue
<!-- ComponentA.vue -->
<script setup>
import { store } from './store.js'
</script>

<template>From A: {{ store.count }}</template>
```

## ~~干碎 import~~

==干没干碎 import 我不知道，反正我是被干碎了，老老实实用 import 吧==

成天 import 写一堆，烦死了，使用两个插件，解放双手: [unplugin-auto-import](https://github.com/antfu/unplugin-auto-import)、[unplugin-vue-components](https://github.com/antfu/unplugin-vue-components)

这两个就是工具类，自动导入依赖，在编译的时候给我们添加上，所以实际生产环境中并不需要这两个依赖

```shell
$ npm install unplugin-auto-import --save-dev
$ npm install unplugin-vue-components --save-dev
```

进入 vite.config.ts 进行基本的配置：

```typescript
// ... 其他配置
import AutoImport from 'unplugin-auto-import/vite'
import Components from 'unplugin-vue-components/vite'
export default defineConfig({
  plugins: [
    vue(),
    AutoImport({
      // 这里就是简单的 import 了 vue，vue-router，axios，其他的写法参考官网
      imports: [
        'vue',
        'vue-router',
        {
          'axios': [
            ['default', 'axios'],
          ],
        },
      ],
      // 这个.d.ts 文件最好放在 src 下，文件名字最好就是 auto-imports.d.ts 和官方保持一致
      dts: 'src/auto-imports.d.ts',
    }),
    Components({
      // 这个也是类似的，可以默认的所有在 components 下的组件都不需要 import 了
      dts: 'src/components.d.ts',
    })
  ]
})
```

随后在 src 下，所有的 .vue 和 .ts 中的 import 都可以省略了(当然，仅限于我们在上面配置的)

如果按照上方这么配置的话，根本没有必要修改 tsconfig.json 文件，不需要在 include 中添加额外的配置，因为默认已经包括了 /src\/\*\*/*.d.ts，即生成的两个 .d.ts 文件都已经包括在内了

## 博客后台

这个后台是基于 [vue-admin-tempate](https://github.com/PanJiaChen/vue-admin-template) 的，相当于提供了一个基础的模板

这里的配置是：vite + vue3 + typescript

### 起步

```shell
$ npm create vite@latest back-stage
# 后面选择框架为 vue，vue-ts
$ cd back-stage
$ npm install
# 确保新建的项目没有问题，可以运行
$ npm run dev
```

#### typescript 支持

这里参考官方建议：[搭配 TypeScript 使用 Vue | Vue.js (vuejs.org)](https://cn.vuejs.org/guide/typescript/overview.html)

因为需要在 vue3 中使用 typescript，所以需要 vscode 添加两个插件：

*   [Volar](https://marketplace.visualstudio.com/items?itemName=Vue.volar): vscode 对 .vue 文件的高亮

*   [TypeScript Vue Plugin](https://marketplace.visualstudio.com/items?itemName=Vue.vscode-typescript-vue-plugin): 支持在普通的 ts 文件中 import .vue 格式的文件

    >   比如在 main.ts 中 import xxxx.vue

#### ~~eslint~~

~~使用 vite 默认创建出来的项目是不带 eslint 的，需要额外配置，也是按照官方的建议: [工具链 | Vue.js (vuejs.org)](https://cn.vuejs.org/guide/scaling-up/tooling.html#linting)~~

eslint 的配置太麻烦了，tnnd，我自己开发，不需要统一代码格式

#### 一些依赖

*   上来就先[干碎 import](#干碎 import)

*   element-plus(生产环境下需要)

    ```shell
    npm install element-plus --save
    ```

    ~~[按需导入组件](https://element-plus.gitee.io/zh-CN/guide/quickstart.html#按需导入)，在 vite.config.ts 添加配置：~~

    ```typescript
    // ...
    import AutoImport from 'unplugin-auto-import/vite'
    import Components from 'unplugin-vue-components/vite'
    import { ElementPlusResolver } from 'unplugin-vue-components/resolvers'
    export default defineConfig({
      // ...
      plugins: [
        // ...
        AutoImport({
          resolvers: [ElementPlusResolver()],
        }),
        Components({
          resolvers: [ElementPlusResolver()],
        }),
      ],
    })
    ```

    因为按需导入有的时候不太好用，所以还是使用[全局导入](#element-plus 初体验)吧

*   axios(生产环境下需要)

    ```shell
    $ npm install axios --save
    ```

*   mockjs(开发环境下需要)

    ```shell
    $ npm install mockjs --save-dev
    ```

    因为 mockjs 并不是原生支持 ts，但在 npm 中提供了 ts 环境的 declaration

    ```shell
    $ npm install @types/mockjs --save-dev
    ```

    这样就可以创建 /mock/index.ts 了

*   mavon-editor(生产环境下需要，且需要 vue3 的支持)

    ```shell
    npm i mavon-editor@next --save
    ```

*   vue-router(生产环境下需要)

    ```shell
    $ npm install vue-router@4 --save
    ```

*   ~~vuex~~，分明一个 store 就可以解决的问题，为什么要使用 vuex

*   <a id="node"></a>@types/node，需要这个环境具体的原因，可以看<a href="#process_error">下面</a>

    因为是开发环境下需要(可能~~默认生产环境下一定有 node 吧~~，真相了，因为实际运行的 js 文件，最后 ts 不过是多了类型检查)

    ```shell
    $ npm install @types/node --save-dev
    ```

    此外还需要在 tsconfig.json 中把 node 添加到 types 属性中

    ```json
    // tsconfig.json
    {
      "compilerOptions": {
        // ...
        "types": ["node"]
      },
      // ...
    }
    ```
    
*   nprogress：一个用来加载进度条的东西

    ```shell
    $ $ npm install @types/nprogress --save
    ```

    

#### 基础框架

有些东西，以前写过，没必要重写一遍了

##### IconComponent.vue

这是一个通用类，因为图标库使用了 iconfont，所以封装了这么一个组件，还是很好用的

```vue
<template>
    <svg class="icon" aria-hidden="true">
        <!-- 按照 demo.html 的官方写法，选择一个图标，其中 realName 绑定了图标名 -->
        <use :xlink:href=realName></use>
    </svg>
</template>

<script lang="ts" setup>
import { computed } from 'vue';
// 在使用这个组件的时候，需要提供一个图标名，作为参数 iconName
const props = defineProps({
    iconName: {
        type: String,
        require: true,
    },
    // 图标的大小
    size: {
        require: false,
        default: '1em',
    },
    // 图标的透明度
    opacity: {
        type: Number,
        require: false,
        default: 1,
    },
    // 图标的颜色
    color: {
        type: String,
        require: false,
        default: 'currentcolor',
    },
});
// 因为实际中阿里的图标名为 #icon-xxx，所以需要我们使用计算属性，为传入的参数添加上前缀
// 这样传递参数的时候，只需要传递有效的图标名即可
const realName = computed(() => `#icon-${props.iconName}`);
</script>

<style lang="less" scoped>
    // 完全按照阿里官方的写法，一笔没改
    .icon {
        height: 1em;
        width: 1em;
        // 官方推荐的写法，props 中的 size 属性绑定 font-size 用来修改组件大小
        font-size: v-bind('props.size');
        opacity: v-bind('props.opacity');
        fill: v-bind('props.color');
        vertical-align: -0.15em;
        overflow: hidden;
    }
</style>
```

##### axios

因为需要使用 axios 实现异步交互，所以这里涉及了 mockjs 和 axios 的全局配置

首先是 mockjs，习惯上配置保存在 mock/index.ts 中

```js
// mock/index.ts
import Mock from 'mockjs';
// 简单的基础配置，设置了延迟时间为 1s
Mock.setup({
  timeout: 1000,
});
```

不要忘了在 main.ts 中引入 mock

然后是封装 axios，首先在新建 /utils/request.ts 表示一个 axios 封装

这个文件用来封装对 axios 的统一配置，比如 baseUrl，超时时间，request/response 的拦截器...

```typescript
// request.ts
// 抽取出来的 axios 配置，全局使用统一的 request 实例发送 axios 请求
import { store } from '@/store/store';
import axios, { AxiosRequestConfig } from 'axios';
/**
 * 默认的基址为 localhost，超时时间为 0
 * 即默认是本地开发，同时 axios 不限制超时时间
 * 使用 mockjs 可以使得请求在不同时间内返回
 */
let baseUrl = 'http://localhost';
let timeout = 0;
/**
 * 如果检测到环境为生产环境(就是实际应用了)就使用域名作为基址
 * 同时超时时间设置为 5s
 */
if (process.env.NODE_ENV === 'production') {
    baseUrl = 'https://buzzx.icu';
    timeout = 5000;
}

const config = {
    baseUrl,
    timeout,
};
// 创建使用了特定配置的 axios 实例
const request = axios.create(config);
// 设置请求拦截器，发送请求前拦截，在请求头的 Authorization 中添加 token，这个主要是为了方便后面登录鉴权的工作
request.interceptors.request.use((reqConf: AxiosRequestConfig) => {
    if (store.token !== '') {
        reqConf.headers!.Authorization = store.token;
    }
    return reqConf;
}, (error) => Promise.reject(error));

export default request;
```

<a id="process_error"></a>

==从旧项目复制过去出现了报错，在上面的第 16 行，说是无法识别 process==，vscode 友好的给出了建议，就是<a href="#node">配置 node 环境</a>

### 鉴权逻辑

基本可以参考[鉴权](./基础不牢地动山摇.md#token)，使用双 token 的机制，目前看起来没什么大问题，主要是后端需要设置 cookie，返回 JWT，对于前端而言，基本上没什么需要改动的，反正就是设置一下 store

### 加载进度条

[NProgress: slim progress bars in JavaScript (ricostacruz.com)](https://ricostacruz.com/nprogress/)

官网甚至只有 1 页，npm 上一次更新是在 15 年...

>   但用起来很简单

具体的使用手册，还是需要看 [github](https://github.com/rstacruz/nprogress#nprogress)

#### 安装

```shell
# 实际生产环境下需要使用 --save 选项
$ npm install @types/nprogress --save
```

因为本质上就是一个 html，所以引入比较随意，因为需要对 nprogress 进行统一配置，所以这里简单封装了一下

```typescript
// utils/nprogress.ts
import nprogress from "nprogress";
// 引入 nprogress 的样式控制
import 'nprogress/nprogress.css';

nprogress.configure({
    // 最小步进单位
    minimum: 0.1,
    // 去掉旋转的 loading 小圆圈(默认的话这个 loading 在右上角)
    showSpinner: false,
    // 进度条多久更新一次
    // 和上面的 0.1 配合起来的话，就是最多 1s 就能跑到头
    // 所以加载都是假的，都是人为的动画，我说加载多久就加载多久
    trickleSpeed: 100,
});

export default nprogress;
```

#### 使用

nprogress.start() 和 nprogress.done()

>   没了，不然呢

因为实际会在发送 ajax 请求(或者路由跳转)的时候进行加载，所以这里分配将 nprogress 配置到 axios 的拦截器和 router 中

```ts
// request.ts
import axios, { AxiosRequestConfig } from 'axios';
import nprogress from './nprogress';

const config = {
    // ... 若干 axios 的配置
};

// 创建使用了特定配置的 axios 实例
const request = axios.create(config);
// 请求拦截器
request.interceptors.request.use((reqConf: AxiosRequestConfig) => {
    // 发送请求前加载
    nprogress.start();
    // ... 若干其他操作
    return reqConf;
}, (error) => Promise.reject(error));
// 响应拦截器
request.interceptors.response.use((response) => {
    // 接收到请求后结束加载
    nprogress.done();
    // ... 若干其他操作
    return response;   
})

export default request;
```

```typescript
import { createRouter, createWebHistory, RouteRecordRaw } from 'vue-router';
import nprogress from '../utils/nprogress';

const routes: Array<RouteRecordRaw> = [
	// 若干路由配置
];

const router = createRouter({
    history: createWebHistory(),
    routes,
});

router.beforeEach(async (to) => {
    // 路由跳转前加载
    nprogress.start();
    // ... 若干其他操作
    return true;
});

router.afterEach(() => {
    // 路由跳转后结束加载
    nprogress.done();
    // ... 若干其他操作
})

export default router;
```

### 登录页面

#### 基本框架

一个 container 内包含了一个表单标题(h3 标签)和一个登录表单(el-form)

#### 表单部分

h3 标题确实没什么需要解释的，这里重点说一下表单部分

*   从表现上
    *   在两个 input 前增加了 username 和 password 图标
    *   在提交按钮处增加了 loading 图标，loding 的动画逻辑通过 gsap 进行确定
    *   增加 Elmessage，在登录失败后弹出
*   从逻辑上
    *   input 的两个 icon 增加单击逻辑：单击图标后，选中对应的 input
    *   在提交表单前需要在前端进行基础的校验
    *   接收到后端响应后，登录成功后通过 vue-router 跳转到后台主页

##### 自定义的 loading 图标

封装了一个 loading-icon，具体的可以参考[添加动画](#一个简单的开发)

#### 表单校验

这是一个很基本的需求，即密码校验，要求密码必须包括大写、小写、数字三种类型，且密码长度范围从 6-16，且不能有空字符

这么简单的需求，一个正则表达式应该就可以实现，然而就是这简单的正则表达式，就花费了两天时间研究

这里使用到了零宽断言(我真的不喜欢这个名字)

最终实现的正则表达式为: `^(?=.*[a-z])(?=.*[A-Z])(?=.*[0-9])(?!.*\s).{6,16}$`

具体的解释，可以看[零宽断言](./复杂的正则表达式.md#零宽断言)

虽然上面的正则表达式可以实现这个功能，最好也不要这么写，而是将一个正则表达式拆分成多个，比如写成：

```typescript
function validation(val: string): boolean {
    const nospace = /^\s+$/;
    if (nospace.test(val)) {
        console.log('不能有空格');
        return false;
    }
	const lenLimit = /^.{6, 16}$/;
    if (!lenLimit.test(val)) {
        console.log('密码长度必须在 6 到 16 位');
        return false;
    }
    const numLimit = /^\d+$/;
    if (!numLimit.test(val)) {
        console.log('密码中必须包含数字');
        return false;
    }
    const lowerLimit = /^[a-z]+$/;
    if (!lowerLimit.test(val)) {
        console.log('密码中必须包含小写字母');
        return false;
    }
    const upperLimit = /^[A-Z]+$/;
    if (!upperLimit.test(val)) {
        console.log('密码中必须包含大写字母');
        return false;
    }
    return true;
}
```

可以看到，分类的好处是用户的交互性更强，用户知道自己的那个输入是有问题的，而如果全堆在了一起，那么用户只知道自己输入错了，而并不能明确知道到底错在了哪

#### 整体代码

```vue
<template>
    <div class="form_container">
        <h3 class="form_header">
            后台管理
        </h3>
        <!-- 
            :model 绑定两个 input 输入到 formData 对象
            ref 为表单实例
            :rules 为 form 表单绑定校验逻辑
         -->
        <el-form
        class="form_entity"
        :model="formData"
        ref="formRef"
        :rules="rules">
            <!-- 
                表单中的每一项都对应了一个 el-form-item, prop 对应了上面 :model 中绑定的键名
                el-input 中的 v-model、size、ref 不需要解释
                因为自定义图标，所以在插槽 prefix 中添加自定义图标, 注意到图标绑定了单击事件即 focus input 输入
             -->
            <el-form-item prop="username">
                <el-input v-model="formData.username" size="large" ref="username">
                    <template #prefix>
                        <icon-component icon-name="login_username" class="input_icon" @click.stop="username.focus()"/>
                    </template>
                </el-input>
            </el-form-item>
            <!-- 具体的和上面的 username 类似，不需要解释 -->
            <el-form-item prop="password">
                <el-input v-model="formData.password" type="password" :show-password="true" size="large" ref="password">
                    <template #prefix>
                        <icon-component icon-name="login_password" class="input_icon" @click.stop="password.focus()"/>
                    </template>
                </el-input>
            </el-form-item>
            <!-- 
                el-button 为表单提交按钮
                因为表单提交时，会等待加载，所以为 button 绑定了 loading 图标
                :loading 表示当前是否处于 loading 状态
                loading 图标采用了自定义的 loading-icon
                最后 button 本身的内容在 btnConfig.text 中展示
                受限于 element-plus 本身的封装, loading 图标只能在文字前面展示
             -->
            <el-form-item>
                <el-button
                class="login_btn"
                type="primary"
                @click.stop="postForm"
                :loading="btnConfig.loading"
                size="large">
                    <template #loading>
                        <loading-icon id="loading_icon"/>
                    </template>
                    {{btnConfig.text}}
                </el-button>
            </el-form-item>
        </el-form>
    </div>
</template>

<script lang="ts" setup>
import { reactive, ref } from 'vue';
import { FormData, ButtonConfig } from '../types';
import  IconComponent  from '../components/icons/IconComponent.vue';
import { FormInstance, FormRules, ElMessage } from 'element-plus';
import { postLogin } from '../api';
import LoadingIcon from '../components/icons/LoadingIcon.vue';
import router from '../router';

// 表单实体, 如果使用了组合式 API 那么这里的变量名和 form 表单处的 ref 必须一致，所以这里必须变量名必须时 formRef
const formRef = ref<FormInstance>();
// 表单数据
const formData = reactive<FormData>({
    username: '',
    password: '',
})
// username input 输入实体
const username = ref();
// password input 输入实体
const password = ref();

/**
 * 两个表单校验方法
 * 写法完全参照 element-plus 官方
 * 校验方法必须包含三个参数
 * 第二个参数表示当前表单的输入 value
 * 第三个参数为一个回调函数, 无论校验是否成功, 这个回调函数必须被调用
 * 如果校验失败, 就向这个回调函数中传入一个 Error 对象, 其中包含了错误信息(只有这样错误信息才会在校验失败时, 默认在 input 下方展示)
 * 如果校验成功, 直接调用这个函数就行
 */

/**
 * 校验用户名输入
 * 用户名长度为 1 到 10 位
 * 这里用户名可以为空
 */
const validateUserName = (rule: any, value: string, callback: any) => {
    const lenReg = /^.{1,10}$/;
    if (!lenReg.test(value)) {
        callback(new Error('用户名必须为 1 到 10 位'));
    }
    callback();
}

/**
 * 此处为严格的校验
 * 密码长度为 6-16 位
 * 不能包括空字符(回车、换行、空格)
 * 必须包含数字、大写、小写三种情况
 * 虽然一个正则表达式就可以完成校验
 * /^(?=.*[a-z])(?=.*[0-9])(?=.*[A-Z])(?!.*\s).{6,16}$/
 * 不过为了分类，这里选择使用多个正则表达式
 * 区分不同的错误类型
 */
const validatePass = (rule: any, value: string, callback: any) => {
    const spaceReg = /^.*\s+.*$/;
    if (spaceReg.test(value)) {
        callback(new Error('密码不可包含空字符'));
    }
    const lenReg = /^.{6,16}$/;
    if (!lenReg.test(value)) {
        callback(new Error('密码必须为 6 到 16 位'));
    }
    const numReg = /^.*\d+.*$/;
    if (!numReg.test(value)) {
        callback(new Error('密码必须包括数字'));
    }
    const lowerReg = /^.*[a-z]+.*$/;
    if (!lowerReg.test(value)) {
        callback(new Error('密码必须包括小写字母'));
    }
    const upperReg = /^.*[A-Z]+.*$/;
    if (!upperReg.test(value)) {
        callback(new Error('密码必须包括大写字母'));
    }
    callback();
}
/**
 * 校验规则: 具体的写法完全参考 element-plus 官方示例
 * validator 表示了校验方法
 * trigger 表示了何时校验, 这里都是失焦后进行校验
 */
const rules = reactive<FormRules>({
    username: [
        {
            required: true,
            validator: validateUserName,
            trigger: 'blur',
        }
    ], 
    password: [
        {
            required: true,
            validator: validatePass,
            trigger: 'blur',
        }
    ]
});
/**
 * 一个简单的 btn 封装对象, 没什么深层意义, 分开写也行
 */
const btnConfig = reactive<ButtonConfig>({
    loading: false,
    text: '登录'
});
/**
 * button 的单击事件, 表示提交表单
 * 在提交之前需要先进行表单校验, 只有校验成功后才进行提交
 * 校验时调用 element-plus 提供的 validate 函数
 * validate 函数接受一个回调函数或者返回一个 Promise 对象
 * 所以写法上可以写成:
 * formRef.value.validate((isValid) => {
 *      // 这里的 isValid 即为校验结果(boolean 类型)
 * })
 * formRef.value.validate().then((isValid) => {
 *      // 这里的 isValid 同理
 * })
 * 其实都差不多
 * 
 * 如果校验通过后调用 axios 抽取出来的 postLogin 方法(在 ./api/index.ts 中)
 * 这是一个异步方法, 会返回一个 Promise<boolean> 对象
 * 进一步的逻辑就放在了 loginHandle 中了,
 * 因为 postForm 本身的作用就是 post, 至于 post 的结果, 就交给 handle 函数吧
 */
const postForm = () => {
    if (!formRef.value) {
        return;
    }
    formRef.value.validate((isValid) =>{
        if (isValid) {
            btnConfig.loading = true;
            btnConfig.text = '正在登录';
            postLogin(formData.username, formData.password).then((res) => {
                loginHandle(res);
            });
        }
    });
};

/**
 * handle login 的结果
 * 其实无法就是两种, 要么密码输错了,要么就跳转到主页
 * ElMessage 弹出一个 Message, 提醒用户登录结果
 * 注意到 默认 Elmessage 是展示时间为 3s(3000ms)
 * 而登录成功的提示框显然不需要展示这么长时间
 */
const loginHandle = (msg: boolean) => {
    if (!formRef.value) return;
    formRef.value.resetFields();
    btnConfig.loading = false;
    btnConfig.text = '登录';
    if (msg) {
        ElMessage({
            message: '正在跳转',
            type: 'success',
            showClose: true,
            duration: 1000,
        })
        // 登录成功后跳转
         router.push('/home');
    } else {
        ElMessage({
            message: '用户名或密码错误',
            type: 'error',
            showClose: true,
        });    
    }
}
</script>

<style lang="less" scoped>
    .form_container{
        position: relative;
        top: 80px;
        width: 100%;
        display: flex;
        flex-flow: column nowrap;
        .form_header {
            margin-bottom: 30px;
        }
        .form_entity {
            width: 500px;
            margin: auto;
            .input_icon {
                margin-right: 10px;
            }
            .login_btn {
                width: 100%;
                #loading_icon {
                    padding-right: 10px;
                }
            }
        }
    }
</style>
```

### 后台框架

基本结构: sidebar 做侧边栏、header 做头部、有效页面为 main

>   没有 footer，其实也不需要 footer

具体的结构就是下图:

![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/back_stage_layout.png)

这三部分分别对应了 ./layout/BackMain.vue、[./layout/SidebarMenu.vue](#SidebarMenu.vue)、LayoutHeader.vue

而这三个 component 放在在 ./views/HomeView.vue 中整合

这里使用了 flex 布局，最外层的 container flex 方向为横向；而将 header 和 main 放入了内层的 container，内层 container flex 方向为竖向

如果是静态框架的话，设置好长宽高，就好了，这里面的难点在于侧边栏是可收缩的，这意味着不仅仅需要动态修改侧边栏的宽度，还需要同时改变内层 container 的宽度；此外还需要引入动画效果，进行线性的过渡，维护动画的一致性，就比较麻烦了

### SidebarMenu.vue





# 甚至还需要学一点 ts

因为 element-plus 中使用的是 ts 语句，所以这里还需要额外学习一下

环境就不配了，毕竟 vscode 已经很好了，在 sublime 中也只是临时写一下而已

首先全局安装 ts 环境

```shell
$ npm install --location=global typescript
```

然后就可以使用 tsc 进行编译了

```shell
$ tsc hello.ts
$ node hello.js
```

>   其实就是多了一点编译前的检查，防止出现不可估计的错误

## 非空

在 ts 中数据类型可能为 string | undefined，通过额外的判断，可以排除 undefined 的情况

```typescript
const num: number | undefined = undefined;
if (!num) {
	console.log("num is undefined");
}
```

所以判断 undefined 其实不需要 == 只要 if(num) 就行了

## 函数定义

### 简单函数

```typescript
function [function_name](): [return_type] {
    [function_body]
}
```

比如：

```typescript
function print(): string {
    return "this is typescript";
}
```

### 带参函数

```typescript
function [function_name]([param_name]: [param_type]...): [return_type] {
    [function_body]
}
```

比如：

```typescript
function print(msg: string): void {
    console.log(msg);
}
```

>   没错 void 也是返回类型

### 可选参数函数

```typescript
function [function_name]([fix_param]: [fix_param_type], [opt_param]?: [opt_param_type]): [return_type] {
    [function_body]
}
```

比如：

```typescript
function print(msg: string, opt?: any): void {
    if (opt) {
        console.log(msg, opt);
    }
}
```

### 默认参数函数

如果不传入参数就取默认值

```typescript
function [function_name]([param_name]: [param_type] = [default_value]): [return_type] {
    [function_body]
}
```

比如：

```typescript
function print(msg: string = "我是傻逼"): void {
    console.log(msg);
}
```

# 甚至还需要学一点 less

在模仿的 Gblog 中使用了 less 进行样式控制 

>   官网[Less.js](https://less.bootcss.com/)

```shell
$ npm install --location=global less
$ npm install less-loader
```

## 本地看 css

使用 sublime 引入 less

```shell
# 全局下载 less，包含了 lessc 命令，可以用来将 less 文件转化为 css 文件
$ npm install -g less
```

在 html 文件中使用 css 样式控制，首先需要将 less 文件编译为 css 文件

```shell
$ lessc style.less > style.css
```

然后在 \<head> 标签内引入样式

```html
<link rel="stylesheet" type="text/css" href="style.css" />
```

## 变量定义

```less
@size: 10px;

img {
    height: @size;
    weight: @size;
}
```

## 嵌套

在嵌套中 '&' 表示父标签，比如：

```less
a {
    color: blue;
    &:hover {
        color: green;
    }
}
```

等效于：

```css
a {
    color: blue;
}
a:hover {
    color: green;
}
```

## 去除链接的下划线

在 vue 中因为使用 vue-router 进行路由，所以链接都是写成 \<router-link>，默认的话在连接下面有下划线，所以这里使用：

```css
a {
	text-decoration: none;
}
```

# 让 eslint 忽略检查某些文件

这个属于是没办法的办法了

首先在根目录下新建 .eslintignore 文件(这个文件就是没有后缀名)

然后在这个文件中添加需要让其忽略检查的文件(相对路径即可，注意需要带上 /src)

# markdown

在前端渲染 markdown 确实有点费事，目前的策略是前端从后端获取 markdown 文本，在前端通过一些 markdown 解析器，将文本解析为 html 代码，然后在 vue 的 template 中插入 html

目前使用到了 [marked](https://marked.js.org/) 和 [highlight.js](https://highlightjs.org/) 分别进行 markdown 解析和代码高亮

## 依赖导入

```shell
$ npm install --save marked
$ npm install --save @types/marked
$ npm install --save highlight.js
```

## 一个 demo



# MyBlog

整合一下前端的前台和后台部分

## 前置工作

这里的配置是：vite + vue3 + typescript

```shell
$ npm create vite@latest my-blog
# 后面选择框架为 vue，vue-ts
$ cd back-stage
$ npm install
# 确保新建的项目没有问题，可以运行
$ npm run dev
```

### typescript 支持

这里参考官方建议：[搭配 TypeScript 使用 Vue | Vue.js (vuejs.org)](https://cn.vuejs.org/guide/typescript/overview.html)

因为需要在 vue3 中使用 typescript，所以需要 vscode 添加两个插件：

*   [Volar](https://marketplace.visualstudio.com/items?itemName=Vue.volar): vscode 对 .vue 文件的高亮

*   [TypeScript Vue Plugin](https://marketplace.visualstudio.com/items?itemName=Vue.vscode-typescript-vue-plugin): 支持在普通的 ts 文件中 import .vue 格式的文件

    >   比如在 main.ts 中 import xxxx.vue

### eslint

eslint 的配置太麻烦了，tnnd，我自己开发，不需要统一代码格式

### 一些依赖

#### element-plus

```shell
npm install element-plus --save
```

>   生产环境下需要

**因为按需导入有的时候不太好用，所以还是使用全局导入吧**

主要方法就是修改 main.ts 文件

```typescript
// main.ts
import { createApp } from 'vue'
import './style.css'
import App from './App.vue'
import ElementPlus from 'element-plus'
import 'element-plus/dist/index.css'

const app = createApp(App);
app.use(ElementPlus);
app.mount('#app')
```

因为使用 vs code，并且添加了 volar 插件，为了 vscode 可以实现自动补全，这里需要在 tsconfig.json 中添加配置：

```json
// tsconfig.json
{
  "compilerOptions": {
    // ...
    "types": ["element-plus/global"]
  }
}
```

#### less

```shell
$ npm install less-loader --save
```

因为 css 确实需要加强一下，所以添加了 less 解析器

#### nprogress

```shell
$ npm install @types/nprogress --save
```

>   一个用来产生进度条的库，在 axios 请求发出后，可能需要等待响应，此时通过进度条可以优化用户体验

一个进度条也没什么配置的，配置文件放在了 /src/utils/nprogress.ts

```typescript
// /src/utils/nprogress.ts
import nprogress from "nprogress";
import 'nprogress/nprogress.css';

nprogress.configure({
    // 最小步进单位
    minimum: 0.1,
    // 去掉旋转的 loading 小圆圈(默认的话这个 loading 在右上角)
    showSpinner: false,
    // 进度条多久更新一次
    // 和上面的 0.1 配合起来的话，就是最多 1s 就能跑到头
    // 所以加载都是假的，都是人为的动画，我说加载多久就加载多久
    trickleSpeed: 100,
});

export default nprogress;
```

>   是的，在使用 nprogress 的时候完全没有必要在 main.ts 中声明，main.ts 已经够臃肿了

#### axios

```shell
$ npm install axios --save
```

>   生产环境下需要

#### mockjs

```shell
$ npm install @types/mockjs --save-dev
```

#### mavon-edior

```shell
$ npm i mavon-editor@next --save
```

>   生产环境下需要，且需要 vue3 的支持

#### gsap

```shell
$ npm install gsap --save
```

>   这是一个动画库，简化了动画的开发

#### vue-router

```shell
$ npm install vue-router@4 --save
```

>   生产环境下需要

#### vuex

这个不需要，因为最多就存储一个 token，使用全局变量 store 就可以存储，没必要使用 vuex

#### @types/node

<a id="node"></a>@types/node，需要这个环境具体的原因，可以看<a href="#process_error">下面</a>

```shell
$ npm install @types/node --save-dev
```

>   因为是开发环境下需要(可能~~默认生产环境下一定有 node 吧~~，真相了，因为实际运行的 js 文件，最后 ts 不过是多了类型检查)

此外还需要在 tsconfig.json 中把 node 添加到 types 属性中

```json
// tsconfig.json
{
  "compilerOptions": {
    // ...
    "types": ["node"]
  },
  // ...
}
```

### 基础框架

#### 图标

使用 [iconfont](https://www.iconfont.cn/) 进行图标管理，为了方便使用，使用组件进行封装

```vue
<!--/src/components/icon/IconComponent.vue-->
<template>
    <svg class="icon" aria-hidden="true">
        <!-- 按照 demo.html 的官方写法，选择一个图标，其中 realName 绑定了图标名 -->
        <use :xlink:href=realName></use>
    </svg>
</template>

<script lang="ts" setup>
import { computed } from 'vue';
// 在使用这个组件的时候，需要提供一个图标名，作为参数 iconName
const props = defineProps({
    iconName: {
        type: String,
        require: true,
    },
    // 图标的大小
    size: {
        require: false,
        default: '1em',
    },
    // 图标的透明度
    opacity: {
        type: Number,
        require: false,
        default: 1,
    },
    // 图标的颜色
    color: {
        type: String,
        require: false,
        default: 'currentcolor',
    },
});
// 因为实际中阿里的图标名为 #icon-xxx，所以需要我们使用计算属性，为传入的参数添加上前缀
// 这样传递参数的时候，只需要传递有效的图标名即可
const realName = computed(() => `#icon-${props.iconName}`);
</script>

<style lang="less" scoped>
    // 完全按照阿里官方的写法，一笔没改
    .icon {
        height: 1em;
        width: 1em;
        // 官方推荐的写法，props 中的 size 属性绑定 font-size 用来修改组件大小
        font-size: v-bind('props.size');
        opacity: v-bind('props.opacity');
        fill: v-bind('props.color');
        vertical-align: -0.15em;
        overflow: hidden;
    }
</style>
```

从官网下到本地的文件中，除了 demo.html 和 demo.css 之外剩下的都是和自定义的图标相关的

为了统一管理，将所有文件放在了 /src/assets/iconfont 下

>   这些文件包括了 .css .js .json .ttf ...

最后不要忘了在 main.ts 中导入这些静态图标

#### 功能函数

这里的功能函数借助了 ts 的 is 关键词，用来反应变量的类型，比如判断变量是不是 number 类型、string 类型...，主要是为了借助 is 关键字避免编译报错

这里将函数定义在了 /src/utils/index.ts 中

```typescript
// /src/utils/index.ts

/**
 * 判断输入是否为 number 类型
 * 这里使用了 ts 的特性, 即 is 关键字
 * @param val 输入
 * @returns 是否为 number 类型
 */
export function isNumber(val: any): val is number {
    return typeof val === 'number';
}

/**
 * 判断输入是否为 string 类型
 * 这里使用了 ts 的特性，is 关键字
 * @param val 输入
 * @returns 是否为 string 类型
 */
export function isString(val: any): val is string {
    return typeof val === 'string';
}
```

#### 统一的消息处理

因为使用了 element-plus，这里借助其中的 [Message](https://element-plus.gitee.io/zh-CN/component/message.html) 组件进行统一的消息处理

所谓消息处理，不过就是调用一个方法，这里在 /src/utils/index.ts 中定义了两个打印方法：

```typescript
// /src/utils/index.ts

/**
 * 通过 ElMessage 打印成功的消息
 * @param msg 成功的消息
 */
export function pubSuccessMsgByElMessage(msg: string) {
    ElMessage({
        type: 'success',
        duration: 1000,
        showClose: true,
        message: msg,
    });
}

/**
 * 通过 Elmessage 打印失败的消息
 * @param msg 传入的消息
 * 因为失败的消息可能是 Error 类型，或者其他什么奇怪的类型
 * 因此这里只有在 msg 的类型为 string 的时候才使用 ElMessage 打印输出
 * 如果是其他的类型，就退化为 console.log() 打印输出了
 */
export function pubErrorMsgByElMessage(msg: Error) {
    if (isString(msg)) {
        ElMessage({
            message: msg,
            type: 'error',
            duration: 3000,
            showClose: true,
        });
    } else console.log(msg);
}
```

#### 统一的类型管理

因为有的时候需要使用一个对象，限制对象的类型，这里使用 type 限定类型

所有的类型放在了 /src/types/index.ts 中，首先最为典型的两个类型 GlobalState 和 ResponseData

```typescript
// /src/types/index.ts

export type GlobalState = {
    token: string,
}

export type ResponseData = {
    code: number,
    data?: any,
    msg?: string,
}
```

其中 GlobalState 为保存在浏览器中的 token 缓存，刷新后即消失；ResponseData 为所有返回值的类型，这里的类型和后端的类型一一对应，其中 data 字段在后端是以 List 集合的形式存在的，而在前端，就是以数组的方式存在

以后其他的类型也都放在这个文件中

#### 统一的变量管理

因为应用本身就不大，用不上 vuex，所以这里直接使用一个全局的变量，保存在内存中，刷洗就失效

保存路径和 vuex 是类似的，变量保存在 /src/store/index.ts

```typescript
import { GlobalState } from "../types";
import { reactive } from 'vue';

const store = reactive<GlobalState>({
    token: ''
});

export default store;
```

#### axios

用来发出异步请求，因为有一些基础的配置，比如拦截器的配置，超时时间的配置，异常处理...在处理不同的请求的时候，可以通过统一配置的方式，避免重复配置

这里选择在 /src/utils/request.ts 中进行 axios 的配置

```typescript
// /src/utils/request.ts
import axios, { AxiosRequestConfig } from 'axios';
import nprogress from './nprogress';
import store from '../store';
/**
 * 默认的基址为 localhost，超时时间为 1
 * 如果是生产环境的话，需要修改超时时间，少说也要给 5s 吧
 */
axios.defaults.timeout = 1;
axios.defaults.baseURL = 'https://localhost:8081'
// 设置跨域时携带 cookie 信息，同时也保证了跨域响应中的 cookie 信息会被保存
axios.defaults.withCredentials = true
// 创建使用了特定配置的 axios 实例
const request = axios.create();
/**
 * 设置请求拦截器，发送请求前拦截
 * 在请求头的 Authorization 中添加 token(如果有的话)
 * 此外还需要加载进度条
 */
request.interceptors.request.use((reqConf: AxiosRequestConfig) => {
    nprogress.start();
    if (store.token !== '') {
        reqConf.headers!.Authorization = store.token;
    }
    return reqConf;
}, (error) => Promise.reject(error));
/**
 * 只要接收到了请求，不管请求是否得到了正确的处理
 * 反正已经得到响应了，那么这个时候进度条就不要加载了
 */
request.interceptors.response.use((response) => {
    // 正常请求的处理
    nprogress.done();
    return response; 
}, (error) => {
    // 错误请求的处理，比如超时请求，或者网络问题
    nprogress.done();
    return Promise.reject(error);
});

export default request;
```

更进一步的，可以对所有请求进行抽象，将所有的请求放在 /src/api/index.ts 中，这个文件中保存的均为 async 的方法，表示各种异步请求，这样，在其他的 .vue 文件中，所有的请求，直接传入参数，调用方法即可，完全不需要操心 axios 是以那种方式(post/get)请求，如何进行异常处理(async-await/try-catch)，返回值就是需要请求的结果

简单举一个例子吧：

```typescript
import request from "../utils/request";
import { ResponseData } from "../types"
/**
 * 一个 test 方法，基本上所有的 axios 请求都具有如下的格式
 * 注意到这里没有进行特殊的异常处理，这意味着在其他 .vue 文件中发出请求时
 * 需要手动 try-catch 打印异常
 * @param param 参数
 * @returns echo 请求会返回一个 string 类型的响应
 */
export async function echo(param: string): Promise<string> {
    try {
        const response = await request.post('/test/echo',{
            param
        });
        const rst: ResponseData = response.data;
        console.log(rst);
        if (rst.code === 200) {
            return rst.data[0];
        }
        return Promise.reject(rst.msg);
    } catch (error) {
        return Promise.reject(error);
    }
}
```

## 前端的结构

因为是一个前后台的博客系统，从页面上也就分为了前台和后台，此外还有一个登录页，其结构就是一个表单，相对来讲比较独立

### 修改 style.css 文件

默认的 style.css 文件，其显示效果不是很好，主要原因在于 #app 并不是默认充满整个页面的，因此这里强制的修改了 html 标签、 body 标签以及 #app 的属性，使其足够拉伸

```css
html {
  min-height: 100%;
  margin: 0;
  padding: 0;
}

body {
  margin: 0;
  place-items: center;
  min-height: 100vh;
  padding: 0;
}

#app {
  width: 100%;
  height: 100%;
  margin: 0;
  padding: 0;
  text-align: center;
}
```

此外为了统一主题，设置了滑块的样式，并取消了超链接的下划线，设置字体

```css
/* 滚动条的宽度(针对上下的滚动条而言)和滚动条的高度(针对左右的滚动条而言) */
::-webkit-scrollbar {
    width: 4px;
    height: 4px;
}
 /* 滚动条的上下方向键 */
::-webkit-scrollbar-button {
    display: none;
}
 /* 滚动条的滚动滑块 */
::-webkit-scrollbar-thumb {
    border-radius: 5px;
    background-color: #d81e06;
}
 /* 滚动条的轨道 */
::-webkit-scrollbar-track {
    background-color: #ffffff;
}

/* 设置页面统一字体 */
* {
  font-family: miranafont, "Hiragino Sans GB", STXihei, "Microsoft YaHei", SimSun, sans-serif;
}

/* 取消所有超链接的下划线 */
a {
  text-decoration: none;
}
```

### 前台页面

所有的前台页面可以分为三部分：header、body、footer

前端页面默认组件出现和消失都设置了动画，组件出现的动画时间：.2s，组件消失的动画时间为 .5s，默认的动画曲线为 ease

header 通过设置 position: fixed 固定在页面顶端；保证了：当页面较长时，此时滚轮移动，header 消失，而当向上滚动滚轮，此时 header 重新出现

body 和 footer 采用 flex 的方式排列，默认的 flex-flow 为 column nowrap，为了保证 footer 可以一直存在于底部，设置了 justify-content 为 space-between

整个页面如下：

```vue
<!--因为这里只关心样式控制所以这里只有 template 和 stype 没有 script-->
<template>
    <div class="front_main_container">
        <HeaderLayout class="front_main_header"/>
        <BodyLayout class="front_main_body"/>
        <FooterLayout class="front_main_footer"/>
    </div>
</template>

<style lang="less" scoped>
.front_main_container {
  position: relative;
  width: 100%;
  min-height: 100vh;
  height: fit-content;
  display: flex;
  flex-flow: column nowrap;
  justify-content: space-between;
  .front_main_header {
    height: 80px;
    position: fixed;
    top: 0;
    width: 100%;
    z-index: 10;
  }
  .front_main_body {
    min-height: 1000px;
    width: 100%;
  }
  .front_main_footer {
    min-height: 100px;
    height: 100px;
    width: 100%;
    bottom: 0px;
    width: 100%;
  }
}
</style>
```







### 后台页面

### 登录页面





