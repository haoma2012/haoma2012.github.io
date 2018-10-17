---
title: '前端基础知识二:CSS&JavaScript'
date: 2018-04-18 20:43:51
categories:
- 微信小程序
tags: 
- 前端
- 微信小程序
---

学习前端CSS样式&JavaScript基础语法学习；

<!-- more -->


## 前端基础CSS样式

- [菜鸟教程](http://www.runoob.com/html/html-tutorial.html)
- [CSS教程](http://www.runoob.com/css/css-tutorial.html)


> 通过使用 CSS 我们可以大大提升网页开发的工作效率！使用 CSS可以 同时控制多重网页的样式和布局。

**id & class选择器**

- HTML元素以id属性来设置id选择器,CSS 中 id 选择器以 "#" 来定义。
- class 选择器在HTML中以class属性表示, 在 CSS 中，类选择器以一个点"."号显示；

插入样式表的方法有三种:

- 外部样式表(External style sheet)：每个页面使用link标签链接到样式表。 link 标签在（文档的）头部
```
<head>
<link rel="stylesheet" type="text/css" href="mystyle.css">
</head>
```
- 内部样式表(Internal style sheet):使用style标签在文档头部定义内部样式表
- 内联样式(Inline style)

CSS 属性定义背景效果:

- background-color:背景色
- background-image：背景图片
- background-repeat：图像平铺
- background-attachment：背景图像是否固定或者随着页面的其余部分滚动。
- background-position：设置定位

[CSS属性大全：查找&使用](http://www.runoob.com/cssref/css-reference.html)

**CSS 文本格式 基本属性**

- 文本颜色：Color
- 文本的对齐方式:text-align
- 文本修饰:
- 文本缩进:
```
h1 {text-decoration:overline;}
h2 {text-decoration:line-through;}
h3 {text-decoration:underline;}

p {text-indent:50px;}
对齐方式：
vertical-align:text-top;
字符间隔：
letter-spacing:2px;
行间距：
line-height:70%;
```
- CSS字体属性定义字体，加粗，大小，文字样式。

```
font-family 属性设置文本的字体系列。
p{font-family:"Times New Roman", Times, serif;}

字体样式：
正常 - 正常显示文本
斜体 - 以斜体字显示的文字
倾斜的文字 - 文字向一边倾斜（和斜体非常类似，但不太支持）
p.normal {font-style:normal;}
p.italic {font-style:italic;}
p.oblique {font-style:oblique;}
设置字体大小像素：font-size:40px;
用em来设置字体大小，1em的默认大小是16px。可以通过公式将像素转换为em：px/16=em
```
所有CSS字体属性：

Property | 描述 
------|---
font | 在一个声明中设置所有的字体属性
font-family | 指定文本的字体系列
font-size | 指定文本的字体大小
font-style | 指定文本的字体样式
font-variant | 以小型大写字体或者正常字体显示文本。
font-weight | 指定字体的粗细。

**链接样式**

- a:link - 正常，未访问过的链接
- a:visited - 用户已访问过的链接
- a:hover - 当用户鼠标放在链接上时
- a:active - 链接被点击的那一刻

```
a:link {color:#000000;}      /* 未访问链接*/
a:visited {color:#00FF00;}  /* 已访问链接 */
a:hover {color:#FF00FF;}  /* 鼠标移动到链接上 */
a:active {color:#0000FF;}  /* 鼠标点击时 */
```

**CSS 盒子模型(Box Model)**
![image](http://www.runoob.com/images/box-model.gif)

- Margin(外边距) - 清除边框外的区域，外边距是透明的。
- Border(边框) - 围绕在内边距和内容外的边框。
- Padding(内边距) - 清除内容周围的区域，内边距是透明的。
- Content(内容) - 盒子的内容，显示文本和图像。

**CSS 边框**

CSS边框属性允许你指定一个元素边框的样式和颜色。border-style属性用来定义边框的样式

- none: 默认无边框
- dotted: 定义一个点线边框
- dashed: 定义一个虚线边框
- solid: 定义实线边框
- double: 定义两个边框。 两个边框的宽度和 border-width 的值相同
- groove: 定义3D沟槽边框。效果取决于边框的颜色值
- ridge: 定义3D脊边框。效果取决于边框的颜色值
- inset:定义一个3D的嵌入边框。效果取决于边框的颜色值
- outset: 定义一个3D突出边框。 效果取决于边框的颜色值

**CSS 分组 和 嵌套 选择器**


- Grouping Selectors在样式表中有很多具有相同样式的元素,为了尽量减少代码，你可以使用分组选择器。每个选择器用逗号分隔.

```
h1,h2,p
{
color:green;
}
```
- 嵌套选择器,它可能适用于选择器内部的选择器的样式。


```
1.为所有p元素指定一个样式
2.为所有class="marked"的元素指定一个样式
3.为所有class="marked"元素内的p元素指定一个样式

p
{
color:blue;
text-align:center;
}
.marked
{
background-color:red;
}
.marked p
{
color:white;
}
```
- 隐藏元素 - display:none或visibility:hidden;visibility:hidden可以隐藏某个元素，但隐藏的元素仍需占用与未隐藏之前一样的空间。也就是说，该元素虽然被隐藏了，但仍然会影响布局。

**CSS Float(浮动)**

CSS 的 Float（浮动），会使元素向左或向右移动，其周围的元素也会重新排列。
Float（浮动），往往是用于图像，但它在布局时一样非常有用。使用 float 属性：


**CSS 导航栏**

熟练使用导航栏，对于任何网站都非常重要。使用CSS你可以转换成好看的导航栏而不是枯燥的HTML菜单。

```
<ul>
<li><a href="#home">主页</a></li>
<li><a href="#news">新闻</a></li>
<li><a href="#contact">联系</a></li>
<li><a href="#about">关于</a></li>
</ul>

样式：
ul
{
	list-style-type:none;
	margin:0;
	padding:0;
}
a:link,a:visited
{
	display:block;
	font-weight:bold;
	color:#FFFFFF;
	background-color:#98bf21;
	width:120px;
	text-align:center;
	padding:4px;
	text-decoration:none;
	text-transform:uppercase;
}
a:hover,a:active
{
	background-color:#7A991A;
}

```

下拉菜单：

```
我们可以使用任何的 HTML 元素来打开下拉菜单，如：<span>, 或 a <button> 元素。

使用容器元素 (如： <div>) 来创建下拉菜单的内容，并放在任何你想放的位置上。

使用 <div> 元素来包裹这些元素，并使用 CSS 来设置下拉内容的样式。
```


## JavaScript教程

- [JavaScript教程](https://www.liaoxuefeng.com/wiki/001434446689867b27157e896e74d51a89c25cc8b43bdb3000)
- [JavaScript 菜鸟教程](http://www.runoob.com/js)

> JavaScript是一种运行在浏览器中的解释型的编程语言。新兴的Node.js把JavaScript引入到了服务器端，JavaScript已经变成了全能型选手。

**JavaScript快速入门**

通过可编程的对象模型，JavaScript 获得了足够的能力来创建动态的 HTML。

- JavaScript 能够改变页面中的所有 HTML 元素
- JavaScript 能够改变页面中的所有 HTML 属性
- JavaScript 能够改变页面中的所有 CSS 样式
- JavaScript 能够对页面中的所有事件做出反应


```
HTML 定义了网页的内容
CSS 描述了网页的布局
JavaScript 网页的行为

```
**JavaScript 显示数据**

- 使用 window.alert() 弹出警告框。
- 使用 document.write() 方法将内容写到 HTML 文档中。
- 使用 innerHTML 写入到 HTML 元素。
- 使用 console.log() 写入到浏览器的控制台。
- 


[Html dom 事件汇总](http://www.runoob.com/jsref/dom-obj-event.html)
```
1.通过id找元素；
var x=document.getElementById("intro");
getElementsByTagName("p");通过标签名查找 HTML 元素

2.
修改 HTML 内容的最简单的方法时使用 innerHTML 属性。
document.getElementById("p1").innerHTML="新文本!";

3.改变 HTML 样式
document.getElementById(id).style.property=新样式

4.HTML DOM 允许我们通过触发事件来执行代码。
向 button 元素分配 onclick 事件：
<button onclick="displayDate()">点这里</button>

onload 和 onunload 事件:会在用户进入或离开页面时被触发。
onload 和 onunload 事件可用于处理 cookie。

onchange 事件常结合对输入字段的验证来使用。

onmouseover 和 onmouseout 事件可用于在用户的鼠标移至 HTML 元素上方或移出元素时触发函数。

addEventListener() 方法用于向指定元素添加事件句柄。
```

在 JavaScript 中有 5 种不同的数据类型：

- string
- number
- boolean
- object
- function
3 种对象类型：

- Object
- Date
- Array
2 个不包含任何值的数据类型：

- null
- undefined

**函数定义**


```
function abs(x) {
    if (x >= 0) {
        return x;
    } else {
        return -x;
    }
}
```

- function指出这是一个函数定义；
- abs是函数的名称；
- (x)括号内列出函数的参数，多个参数以,分隔；
- { ... }之间的代码是函数体，可以包含若干语句，甚至可以没有任何语句。

**高阶函数**

- Map()：我们有一个函数f(x)=x2，要把这个函数作用在一个数组[1, 2, 3, 4, 5, 6, 7, 8, 9]上，就可以用map

```
function pow(x) {
    return x * x;
}

var arr = [1, 2, 3, 4, 5, 6, 7, 8, 9];
var results = arr.map(pow); // [1, 4, 9, 16, 25, 36, 49, 64, 81]
console.log(results);
```

- reduce():Array的reduce()把一个函数作用在这个Array的[x1, x2, x3...]上，这个函数必须接收两个参数，reduce()把结果继续和序列的下一个元素做累积计算

```
[x1, x2, x3, x4].reduce(f) = f(f(f(x1, x2), x3), x4)
var arr = [1, 3, 5, 7, 9];
arr.reduce(function (x, y) {
    return x + y;
}); // 25
```

> 在JavaScript的世界里，一切都是对象。为了区分对象的类型，我们用typeof操作符获取对象的类型，它总是返回一个字符串

```
typeof 123; // 'number'
typeof NaN; // 'number'
typeof 'str'; // 'string'
typeof true; // 'boolean'
typeof undefined; // 'undefined'
typeof Math.abs; // 'function'
typeof null; // 'object'
typeof []; // 'object'
typeof {}; // 'object'
```

**浏览器对象**

- 由于HTML文档被浏览器解析后就是一棵DOM树，要改变HTML的结构，就需要通过JavaScript来操作DOM。始终记住DOM是一个树形结构。
- window对象不但充当全局作用域，而且表示浏览器窗口。innerWidth和innerHeight属性，可以获取浏览器窗口的内部宽度和高度。内部宽高是指除去菜单栏、工具栏、边框等占位元素后，用于显示网页的净宽高。

```
navigator对象表示浏览器的信息;
navigator.appName：浏览器名称；
navigator.appVersion：浏览器版本；
navigator.language：浏览器设置的语言；
navigator.platform：操作系统类型；
navigator.userAgent：浏览器设定的User-Agent字符串。

screen对象表示屏幕的信息;
screen.width：屏幕宽度，以像素为单位；
screen.height：屏幕高度，以像素为单位；
screen.colorDepth：返回颜色位数，如8、16、24。

location对象表示当前页面的URL信息;
比如：http://www.example.com:8080/path/index.html?a=1&b=2#TOP
location.protocol; // 'http'
location.host; // 'www.example.com'
location.port; // '8080'
location.pathname; // '/path/index.html'
location.search; // '?a=1&b=2'
location.hash; // 'TOP'

history对象保存了浏览器的历史记录，JavaScript可以
调用history对象的back()或forward()，相当于用户点击了
浏览器的“后退”或“前进”按钮。
```

**Ajax:JavaScript执行异步网络请求。**

> 这就是Web的运作原理：一次HTTP请求对应一个页面。AJAX请求是异步执行的，也就是说，要通过回调函数获得响应。

**JavaScript 库**

- jQuery:它使用 CSS 选择器来访问和操作网页上的 HTML 元素（DOM 对象）
- Prototype 是一种库，提供用于执行常见 web 任务的简单 API。
- MooTools 也是一个框架，提供了可使常见的 JavaScript 编程更为简单的 API。
