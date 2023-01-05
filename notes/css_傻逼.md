# 语法

css 的基本结构：

```css
[选择器] {
    [属性]:[值];
}
```

# 选择器

*   id 选择器，写法为 #[id]，比如：

    ```html
    <p id="lala">我是废物</p>
    <style>
        #id {
            color:red;
        }
    </style>
    ```

*   类选择器，写法为 .[类]，比如：

    ```html
    <p class="lala">我是废物</p>
    <style>
        .lala {
            color:red;
        }
    </style>
    ```

*   元素选择器，写法为 [元素]，比如：

    ```html
    <p>我是废物</p>
    <style>
        p {
            color:red;
        }
    </style>
    ```

*   指定某个元素下的某个类，写法为[元素].[类]，比如：

    ```html
    <p class="lala">我是废物</p>
    <p>我不是废物</p>
    <style>
        p.lala{
            color:red;
        }
    </style>
    ```

*   伪类选择器，写法为：[某种选择器]:[伪类]，比如：<a id="pseudo-class-example">

    ```html
    <p class="lala">我是废物</p>
    <style>
        p.lala:hover {
            color: red;
        }
        p.lala {
            color: green;
        }
    </style>
    ```

    看不懂伪类的，看[伪类](#伪类)

*   伪元素选择器，写法为：[某种选择器]::[伪元素]，比如：<a id='pseudo-element-example'>

    ```html
    <p class="lala">我是废物</p>
    <style>
        p.lala:hover {
            color: red;
        }
        p.lala::select {
            color: green;
        }
    </style>
    ```

    看不懂伪元素的，看[伪元素](#伪元素)
    
*   后代选择器：[后代选择器|MDN](https://developer.mozilla.org/zh-CN/docs/Web/CSS/Descendant_combinator)

# 伪类

添加到一般的选择器上，**指定被选择元素的特殊状态**，比如：':hover'，指定了用户鼠标悬浮到对应元素上的样式，具体的可以看<a href="#pseudo-class-example">举例</a>

一个比较全面的[伪类索引](https://developer.mozilla.org/zh-CN/docs/Web/CSS/Pseudo-classes#标准伪类索引)

# 伪元素

添加到一般的选择器上，**指定被选择元素的特定部分的样式**，比如：'::select'，指定了文本被选中部分的样式，具体的可以看<a href="#pseudo-element-example">举例</a>

一个比较全面的[伪元素索引](https://developer.mozilla.org/zh-CN/docs/Web/CSS/Pseudo-elements#标准伪元素索引)

# 伪元素和伪类的区别

按照网上大佬的说明：

**伪类**选择元素基于的是当前元素处于的状态，或者说元素当前所具有的特性，而不是元素的id、class、属性等静态的标志。由于状态是动态变化的，所以一个元素达到一个特定状态时，它可能得到一个伪类的样式；当状态改变时，它又会失去这个样式。

由此可以看出，它的功能和class有些类似，但它是基于文档之外的抽象，所以叫伪类。

>   抱歉，我真的看不出相似

**伪元素**是对元素中的特定内容进行操作，它所操作的层次比伪类更深了一层，也因此它的动态性比伪类要低得多。实际上，设计伪元素的目的就是去选取诸如元素内容第一个字（母）、第一行，选取某些内容前面或后面这种普通的选择器无法完成的工作。

它控制的内容实际上和元素是相同的，但是它本身只是基于元素的抽象，并不存在于文档中，所以叫伪元素

>   这个其实我能理解

# 盒子模型

![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/box_model_css.gif)

如果将一个元素抽象为一个盒子，那么一个盒子可以分为下面几部分：

*   Content(内容)：盒子中的内容，比如图片，文本，表单
*   Border(边框)：分割了内容和内容以外的东西，这个边框一般是不可见的，如果在元素外面绕了一个边框，不好看
*   Padding(内边距)：内容到边框的区域，透明的，不可见

*   Margin(外边距)：边框外边的区域，主要是用来隔离盒子和盒子的，透明的，不可见

# 定位(Position)

[Position|菜鸟教程](https://www.runoob.com/css/css-positioning.html)、[position|MDN](https://developer.mozilla.org/zh-CN/docs/Web/CSS/position)

>   position 属性

css 中，通过 position 和 top、bottom、left、right 可以指定元素的位置布局

之前写的时候都是认为是文件流的顺序布局，我写到哪里，元素就放在哪里，而现在通过 position 可以进一步的规划位置

position 具有 5 种取值：

*   static：默认就是这个，此时 top、bottom... 都无效，就是按照文件流的方式布局

*   relative：相对布局，元素根据 top、bottom... 的取值相对原来的位置(没有 position 时的位置)排列

*   absolate：绝对布局，元素根据 top、bottom... 的取值相对于父标签的位置进行排列，最顶级的父标签为 \<body>，此时就是相对于整个页面的相对位置了

    ==22.8.10 更正，absolute 布局会根据第一个布局不是 static 的父组件进行相对排列，而默认父组件的布局方式是 static，所以如果要使用 absolute 布局，记得设置父组件的布局为 relateive==

*   fixed：也是相对布局，不过会随着网页的上下滑动而改变位置，对于是"相对的绝对布局"，相对于整个网页，绝对于浏览器窗口

*   sticky：没用过，以后再说

如果 position 不是 static，那么此时元素的顺序可能出现堆叠，此时使用 z-index 可以指定元素的顺序(哪个元素放在上层)；z-index 越大，越上层

# Flex 布局

[flex 布局|MDN](https://developer.mozilla.org/zh-CN/docs/Web/CSS/CSS_Flexible_Box_Layout/Basic_Concepts_of_Flexbox)、[Flex 布局|菜鸟教程](https://www.runoob.com/w3cnote/flex-grammar.html)

flexbox 模型是一种一维的布局，处理的要么是一行，要么是一列

>   个人认为这种一维的特点特别时候做 header、footer、侧边栏这些

一般的写法：

```less
.container {
    display: flex;
 	display: --webkit-flex; // 如果是 webkit 内核的浏览器，需要加上 webkit 前缀  
}
```

>   点名批评 Safari

## 主轴和交叉轴

flex 中比较关键的是两个轴线：主轴和交叉轴，这两个轴相互垂直；其中主轴通过属性 flex-direction 指定

flex-direction 有四个取值：row、row-reverse、column、column-reverse

从命名上也能看出来，如果带 row 的，主轴就是水平方向的，带 column 的主轴就是垂直方向的

## 方向

按照我们书写文本的逻辑，一定是先行后列，写满一行写下一行，中国人的书写习惯是从左向右写；而在 flex 中这些方向都是可以自定义的

因为前面已经说过了 flex 是一维的布局，一个 flex 容器中的子容器，要么是一行排列的，要么是一列排列的(当然有点例外)

如果设置 flex-direction 为 row，那么容器中的子容器按照 "→" 方向排列；如果是 row-reverse，就按照 "←" 方向排列

>   如果是 column 就是 "↓"；如果是 column-reverse："↑"

```html
<div class="container">
    <div class="sub">a</div>
    <div class="sub">b</div>
    <div class="sub">c</div>
    <div class="sub">d</div>
</div>
```

```less
.container {
    display: flex;
    // 可以加入其他值试试效果
    flex-direction: row;
}
```

## 例外

尽管 flex 是一维的布局，但实际中子容器并不一定都是按照行或列排列；如果子容器太大了，一行放不下，那么按照逻辑，应该是要把放不下的子容器放到下一行

```less
.container {
    flex-wrap: wrap; // 设置这个就可以让 flex 自动换行了
}
```

默认的 flex 是不允许换行的，如果超出的就溢出了，不显示了，所以其实 flex-wrap 默认值为 nowrap

## flex-flow

上面又写了 flex-direction 又写了 flex-wrap 这些都是父容器的属性，提供了一个更简洁(但其实并不简单)的写法

```less
.container {
    flex-flow: row nowrap;
    // 等效于：
    // flex-direction: row;
    // flex-wrap: nowrap;
}
```

## 子容器的属性

上面这些都是父容器的属性，决定了子容器编排上的布局

子容器并不一定会占据父容器的全部空间，通过子容器的属性可以更好的利用剩下的可用空间

### flex-basis

这个决定了子容器的大小(只要有这个值，当 flex-direction 为 row 时，子容器的 width 就没用了；同理 height 在 column 的时候也没用了)

默认的这个值为 auto，即根据子容器的内容自动调整子容器本身的大小

### flex-grow

这个属性是用来扩展的，即如何让子容器充分利用剩下的可用空间

这个属性的有效值为一个正整数，而默认为 0，说明默认不会扩展子容器的大小

这个属性本身的大小不重要，重要的是多个子组件之间的相对大小

```html
<div class="container">
    <div class="a">a</div>
    <div class="b">b</div>
</div>
```

```less
div {
	border: solid black 1px;
}
.container {
	padding: 100px;
	display: flex;
    flex-flow: row nowrap;
    .a {
        flex-basis: 100px;
		border: solid black 1px;
        flex-grow: 2
    }
    .b {
        flex-basis: 100px;
        flex-grow: 1
    }
}
```

如果这样设置，那么可用空间中 $\frac{2}{3}$ 将分配给 a，而 $\frac{1}{3}$ 将分配给 b

所以他其实是一个比例的关系，但要注意这一切有是在有空闲空间的基础上

### flex-shrink

上面说了如果有剩下的可用空间，那么可以通过 flex-grow 分配，但如果剩下的空间不够了，就需要通过 flex-shrink 使得子容器的空间降低到 flex-basis 以下

具体的如果降低，规则可以看[控制 Flex 子元素在主轴上的比例](https://developer.mozilla.org/zh-CN/docs/Web/CSS/CSS_Flexible_Box_Layout/Controlling_Ratios_of_Flex_Items_Along_the_Main_Ax)

### flex

如果每个子容器都写三行，还是有点麻烦的，这里也提供了间接的写法：flex: [flex-grow] [flex-shrink] [flex-basis]

比较傻逼的是，居然还觉得这样也不够间接，还有写成：

*   flex: initial 等效于 flex: 0 1 auto
*   flex: auto 等效于 flex: 1 1 auto
*   flex: none 等效于 flex: 0 0 auto

>   傻逼写法，记都不记

## 对齐规则

*   align-items：这个对应了交叉轴的对齐方向
*   justify-content：主轴对齐方向

具体的看 [flex 元素间的对齐和空间分配](https://developer.mozilla.org/zh-CN/docs/Web/CSS/CSS_Flexible_Box_Layout/Basic_Concepts_of_Flexbox#元素间的对齐和空间分配)，没必要记住

MDN 上给出的属性不是很多，更多还可以参考[Flex 布局语法教程|菜鸟教程](https://www.runoob.com/w3cnote/flex-grammar.html)

# 旋转跳跃(我闭着眼，transform)

设置 transform 属性可以使得元素按照一定规则旋转、放缩、移动、倾斜...

[transform-function|MDN](https://developer.mozilla.org/zh-CN/docs/Web/CSS/transform-function)

## transform-origin

[transform-origin|MDN](https://developer.mozilla.org/zh-CN/docs/Web/CSS/transform-origin)

决定了图形变化的原点，比如希望旋转的中心(所有点都是围绕中心旋转)

默认的旋转中心为元素的中央 center

## rotate

参数为角度，表示旋转的度数，如果是正数，就是顺时针旋转、如果是负数就是逆时针旋转

```less
.container {
    // 元素中心为圆形，旋转 30°
    transform: rotate(30deg);
}
```

使用 [transfrom-origin](#transform-origin) 可以决定旋转中心

然而最逆天的是这个旋转属性还有三维的效果

### rotate3d

要说明的是，二维中围绕着一个点旋转，那么三维中就是围绕一个轴旋转

这个函数默认有 4 个参数，其中前三个参数 x y z 和旋转中心两点连成了一条线，旋转轴就是这条线，第四个参数表示旋转角度

### rotateX、rotateY、rotateZ

逆天，分别表示绕着 x y z 轴旋转

因为我们的图像默认就是二维的，所以绕着 x 旋转看起来就像上下边界收窄，而绕着 y 旋转看起来就像左右边界收窄，最后绕着 z 旋转就和不同的 rotate 类似

>   忘了说这三个都只有一个参数，表示旋转角度

## [scale](https://developer.mozilla.org/zh-CN/docs/Web/CSS/transform-function#scale)

可以用来改变元素本身的大小

如果参数的绝对值大于 1 将起到放大作用，如果绝对值小于 1 将起到缩小作用

如果参数是负数的话，那么同时还起到了翻转的作用

```less
.container {
    transform: scale(2, 2);
}
```

上面这种写法将元素 x 方向和 y 方向大小均翻倍

使用 [transfrom-origin](#transform-origin) 可以决定放缩中心

很巧这个属性也有三维的表示

### scale3d

三个参数，不想多说了分别表示 x 方向、y 方向、z 方向的放缩

### scaleX、scaleY、scaleZ

逆天，不多说了

## skew

看不懂，[transform-function - skew|MDN](https://developer.mozilla.org/zh-CN/docs/Web/CSS/transform-function#skew)

## translate

平移，平移量通过参数表示，因为是平面，所以通过两个参数表示

```less
.container {
    transform: translate(20px, 20px);
}
```

这个函数不受 transform-origin 的影响(其实一想也是，都是平移了，总不会有平移中心吧)

### translate3d

呵呵

### translateX、translateY、translateZ

字面意思，都只有一个参数了

## matrix

上面这些变化都是弟弟，他们都是 matrix 的某种特殊情况

考虑二维空间中的某个点，可以通过坐标(x, y)表示，那不管是放缩，旋转、平移，其实都是元素中所有的点进行一定的操作，在 css 中就是进行矩阵乘法

为了向上兼容，这里使用的是三维矩阵表示，一个 matrix 函数中具有 6 个参数 a b c d e f，其对应的矩阵为：$\begin{bmatrix} a & b & c \\ d & e & f \\ 0 & 0 & 1 \end{bmatrix}$

这是一个三维的矩阵，所以我们也需要一个三维的坐标何其进行运算，而因为浏览器一般展示的是二维画面，所以 z 方向维度补充 1(这里补充 1 应该是约定俗成的，不然我个人感觉应该是 0)，总之这里假设原始坐标为 (x, y, 1)

那么 matrix 函数就是进行了一个矩阵乘法运算：$\begin{bmatrix} a & b & c \\ d & e & f \\ 0 & 0 & 1 \end{bmatrix} \times \begin{bmatrix} x \\ y \\ 1 \end{bmatrix}$

我们上面所有的操作都可以通过这 6 个参数的矩阵乘法刻画，比如 scale(xScale, yScale)，其实等效矩阵为$\begin{bmatrix} \text{xScale} & 0 & 0 \\ \text{yScale} & 0 & 0 \\ 0 & 0 & 1 \end{bmatrix}$

>   剩下的就不列举了，反正本质上都是矩阵乘法

# 视口

[视口概念|MDN](https://developer.mozilla.org/zh-CN/docs/Web/CSS/Viewport_concepts)

视口在 web 中指代的通常是浏览器的窗口(不包括收藏栏，URL...)

一般而言在 PC 上只要没有 <kbd>F11</kbd>，就都不是全屏的，此时视口就是浏览器窗口的大小；而在移动设备上，默认就是全屏的，所以视口就是整个屏幕的大小

简单来说，视口就是当前文档的可见部分

在 web 中具体来说包括布局视口和视觉视口两部分，其中视觉视口特指当前浏览器中的可见部分，他会随着上下滑动，放缩页面发生变化；而布局视口指的的文档部分，不会变化

可以认为布局视口和文档相关，而可是视口就是浏览器窗口的大小

为什么要反复强调这些呢，因为默认情况下，都是根据布局视口编程的，即我希望放大的时候，就是放大，文字变化，图片变大，可能放大之后原来能看到的东西，就看不到了，所以需要左右滑动或者上下滑动

但有的时候我希望有的组件的位置可以相对固定，就算放大后，大小和位置都不发生变化

此时这个组件的尺寸单位(width、height)，位置单位(top、left、bottom、right)，都应该和可视视口相关，即单位为 vw(可是窗口的宽度)、vh(可视窗口的高度)

更多的单位可以参考 [length|MDN](https://developer.mozilla.org/zh-CN/docs/Web/CSS/length)

如果 width = 1vw，说明这个组件的宽度占据了可视窗口宽度的 1%

举个例子：

```html
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<title>vue</title>
<link rel="stylesheet" type="text/css" href="style.css" />
</head>
<body>
	<p>
		这是一段对比性的文字，没有什么实际意义
	</p>
	<div class="container">
	</div>
</body>
</html>
```

```less
.container {
	height: 5vw;
	width: 20vw;
	position: absolute;
	bottom: 20vw;
	left: 20vw;
	border: solid 0.1vw black;
}
```

此时不管怎么放大，container 的框都在同一个位置，且大小不变

# background

所谓背景的一些设置 [background|MDN](https://developer.mozilla.org/zh-CN/docs/Web/CSS/background)

常用设置：

*   [background-color|MDN](https://developer.mozilla.org/zh-CN/docs/Web/CSS/background-color)：设置背景颜色，其实还是很有用的，尤其是设计小组件的样式的时候
*   [background-image|MDN](https://developer.mozilla.org/zh-CN/docs/Web/CSS/background-image#examples)：设置背景图片，有的时候背景可能不只有一个图，所以背景图是可以在 z 方向上堆叠的
*   [background-position|MDN](https://developer.mozilla.org/zh-CN/docs/Web/CSS/background-position)，设置背景的位置，默认位置是从左上开始，按照一定的重复规则向下、向右重复
*   [background-repeat|MDN](https://developer.mozilla.org/zh-CN/docs/Web/CSS/background-repeat)：设置重复规则，如果背景图太小了，不足够覆盖整个背景，可能需要设置重复规则
*   [background-size|MDN](https://developer.mozilla.org/zh-CN/docs/Web/CSS/background-size)：设置背景的大小，因为有的时候可能容器大小和背景图片大小并不匹配，但我们又不希望通过重复背景图片实现容器的覆盖，此时可以设置背景图大小，实现覆盖整个容器
*   [background-attachment|MDN](https://developer.mozilla.org/zh-CN/docs/Web/CSS/background-attachment)：可以决定背景图片是否在视口内固定，如果设置相对固定的话，那么上下滚动看到的背景图是不会变化的

# [object-fit | MDN](https://developer.mozilla.org/zh-CN/docs/Web/CSS/object-fit)

目前的话，仅仅用到了使用 object-fit 让图片使用大小，对图片设置 object-fit 属性可以使得图以不同的规则填充容器

这里的参数取值和 [background-size | MDN](https://developer.mozilla.org/zh-CN/docs/Web/CSS/background-size) 类似：

*   contain：内容会被放缩以适应容器大小，图片的长宽比会得到保持，且不会对原图片进行剪裁，所以可能出现黑边
*   cover：内容会被放缩以适应容器大小，图片的长宽比会得到保持，但会对原来的图片进行剪裁，避免出现黑边
*   fill：内容会被放缩以适应容器大小，图片的长宽比会被破坏
*   none：默认行为不说了
*   scale-down：是 none 和 contain 中的一个，具体的会取决于这两个中尺寸小的那个

# [filter | MDN](https://developer.mozilla.org/zh-CN/docs/Web/CSS/filter)

可以用来：对元素进行模糊、颜色偏移、透明度处理

*   [blur()](https://developer.mozilla.org/zh-CN/docs/Web/CSS/filter-function/blur)：用来进行高斯模糊
*   [brightness()](https://developer.mozilla.org/zh-CN/docs/Web/CSS/filter-function/brightness)、[contrast()](https://developer.mozilla.org/zh-CN/docs/Web/CSS/filter-function/contrast)、[grayscale()](https://developer.mozilla.org/zh-CN/docs/Web/CSS/filter-function/grayscale)、[saturate()](https://developer.mozilla.org/en-US/docs/Web/CSS/filter-function/saturate)：分别用来调整亮度、对比度、灰度、饱和度
*   [drop-shadow()](https://developer.mozilla.org/zh-CN/docs/Web/CSS/filter-function/drop-shadow)：用来提供阴影效果，其效果和参数和 [box-shadow](#添加阴影效果(box-shadow)) 差不多，使用这个 filter 的有点在于有的浏览器会使用硬件加速以提高更好的性能
*   [opacity()](https://developer.mozilla.org/zh-CN/docs/Web/CSS/filter-function/opacity)：和 [opacity](#透明度(opacity | MDN)) 差不多，也是可能有硬件加速效果

# 一些看起来有用的东西

## 文本修饰(超链接去下划线)

对于链接，使用 text-decoration:none 实现删除链接的下划线

```html
<style>
    a {
        text-decoration: none;
    }
</style>
```

## 添加阴影效果([box-shadow | MDN](https://developer.mozilla.org/zh-CN/docs/Web/CSS/box-shadow))

使用属性 box-shadow 添加阴影效果，其可选的参数值包括 阴影的 x 轴偏移量、y 轴偏移量、模糊半径、扩散半径、阴影颜色

>   其他的值，直接到 MDN 上查吧

```html
<div class="container">
    <div class="box">
    	一个盒子
	</div>
</div>
```

```less
.container {
    display: flex;
    flex-flow: row nowrap;
    justify-content: center;
    .box {
        border: solid black 1px;
        height: 100px;
        weight: 200px;
        box-shadow: 0 0 20px 10px red;
    }
}
```

其中比较麻烦的是模糊半径和扩散半径

首先，阴影效果本身是纯色的，如果没有渐变，阴影看起来没有那么"阴影"，此时看起来就像是在一个 box 外边套了一个外壳，这个"外壳"的大小是扩散半径决定的

显然我们希望阴影具有一定的渐变效果，此时就需要使用到模糊半径了，其相当于在扩散半径外侧，再套一个壳子，但其颜色是渐变的

>   个人认为使得盒子具有阴影效果的最关键的因素在于模糊半径

## 隐式过渡(transition)

[使用 CSS transitions|MDN](https://developer.mozilla.org/zh-CN/docs/Web/CSS/CSS_Transitions/Using_CSS_transitions)

有的时候需要属性随着状态的变化发生变化，

比如有可能一个盒子前一秒还是白色的，下一秒就是红色的了，但我们希望这个过程不是瞬间完成的，而是逐渐过渡的

这种过渡看起来就像动画一样，在 css 中使用 transaction 属性刻画

transaction 的一些属性：

*   [transition-property](https://developer.mozilla.org/zh-CN/docs/Web/CSS/transition-property)：决定那些属性进行过渡，不包括在这个集合中的属性将瞬间变化

    >   默认为 all

*   [transition-duration](https://developer.mozilla.org/zh-CN/docs/Web/CSS/transition-duration)：指定过渡的时长，时间短，过渡动画看起来就快

    >   默认为 0

*   [transition-timing-function](https://developer.mozilla.org/zh-CN/docs/Web/CSS/transition-timing-function)：定义属性值如何发生变化

    *   linear：线性过渡，即从开始到结束动画的速度是一样的
    *   ease：加速过渡，最开始慢，后来快，最后慢速结束
    *   ...

    >   默认值为 ease

*   [transition-delay](https://developer.mozilla.org/zh-CN/docs/Web/CSS/transition-delay)：动画开始的延迟

    >   默认为 0

通常，我们不会写四个属性，而使用更加简洁的写法：

```less
div {
    transition: [property] [duration] [timing-function] [delay];
}
```

## 透明度([opacity | MDN](https://developer.mozilla.org/zh-CN/docs/Web/CSS/opacity))

盒子的透明度，合法值范围为 [0, 1]，默认为 1 即不透明，当盒子完全透明时，将展示盒子后面的背景

要注意的是如果出现的嵌套，就算当前盒子和其子元素的 opacity 具有不同的取值，但都以当前盒子的取值为准

## 小三角

通过修改 border 的颜色，很容易实现小三角的效果

当一个盒子的大小为 0，而 border 的大小不是 0，此时通过设置设置四个方向上的 border 为不同的颜色，即可得到四个方向上的不同的小三角

```less
.container {
	@boxSize: 0px;
	@borderSize: 100px;
	width: @boxSize;
	height: @boxSize;
	border: solid @borderSize;
	border-color: #893615 #E76B56 #A72310 #0C1F22;
}
```

因为几个方向上的 border 会相互重叠，相互叠加后的效果就是小三角

但上述效果构成的小三角可能并不是我们希望的那样，毕竟他还是一个矩形，此时可以通过设置其他边的颜色为 transparent，而保留剩下的边

比如下方就仅仅保留了下方的小三角

```less
.container {
	@boxSize: 0px;
	@borderSize: 100px;
	width: @boxSize;
	height: @boxSize;
	border: solid @borderSize transparent;
	border-bottom: solid @borderSize #893615;
}
```

 ## 鼠标特效

有的时候我们希望鼠标悬浮到某处时可以显示一个 :point_up_2:，从而引导用户点击

此时可以通过 cursor 属性进行样式控制

```less
cursor: pointer // 显示类似超链接的点击效果
cursor: wait // 鼠标转圈，类似正在加载的效果
cursor: crosshair // 十字交叉，类似截图的时候的图标
cusor: not-allowed // 类似禁止的效果
```

更多的可以参考 [cursor|MDN](https://developer.mozilla.org/zh-CN/docs/Web/CSS/cursor)

## padding 导致的容器大小变化

[box-sizing|MDN](https://developer.mozilla.org/zh-CN/docs/Web/CSS/box-sizing)，这个值设置的是盒子模型的宽度的基准，默认的话是 content-box，即默认的我们设置的 width 和 height 都是针对内容设置的

所以一个容器实际的尺寸需要同时考虑：content 的大小、padding 的大小、border 的大小 

>   注意不包括 margin，因为 margin 是外边距，所以已经不算是盒子尺寸了

此时如果设置了 width 后又设置了 padding，那么容器的大小会比 width 更大

之前还真的以为 width 就是容器大小呢

如果设置 box-sizing 为 border-box 那么的 width 和 height 就是针对整个盒子而言的，此时盒子就同时包括了 content、padding、border，width 和 height 就是盒子的大小







