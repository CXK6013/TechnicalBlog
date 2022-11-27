# CSS学习笔记(一) 盒子模型

[TOC]

> 本来按照正常的学习顺序, 应该是HTML、CSS、JavaScript，再到Vue。但是到我这里顺序就反过来了一样，先Vue 、再JavaScript，然后CSS。原因在于我之前就学习过这些，虽然当初学的不扎实，但是勉强能用，需要用到什么，然后忘记了就重新再学习一下，这看起来像是逆生长一样，但后面会将这个系列的文章补齐。

## 前言

在上海地铁是大多数人的通勤方式，有的时候地铁人多，就会很拥挤，那拥挤表达的意思是什么呢？ 我想是人与人之间的距离很小吧。如果将人视做html的元素，那么人与人间的距离在在CSS中称为margin，也被称为外边距。但是人和HTML的标签还是有些区别，HTML的元素分块级元素和行内元素、

### 块级元素(Block-level elements)

那什么是块级元素？ 简单的说就是块级元素占满一行，如下图所示:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>
<body>
<p>This</p>
</body>
<style>
p { 
        background-color: #8ABB55;
 }
</style>
</html>
```

效果图: 

![块级元素](http://tva1.sinaimg.cn/large/006e5UvNgy1h8idoa30vqj31hf02imx7.jpg)



虽然标签p的内容只有一个this，但是当我们为它添加上背景色，你会发现整个一行都出现了绿色。细心的朋友运行代码之后可能会说也没占满一行啊，旁边还有两个空白啊。那是因为我们写的p标签在body标签里面，body标签是一个更大的盒子: 

![body标签](http://tvax1.sinaimg.cn/large/006e5UvNgy1h8idv5xcc0j30ig0cbwej.jpg)

浏览器给body默认的margin大小是8个像素, 我们可以通过浏览器的工具来验证我们的论断:

![p标签](http://tvax3.sinaimg.cn/large/006e5UvNgy1h8ie2a5xnfj31h90k040u.jpg)

 

![body标签占8px](http://tvax3.sinaimg.cn/large/006e5UvNgy1h8ie3jukrpj31h80nwq5e.jpg)

然后我们将body标签的margin清空就会发现p标签完全占一整行:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>
<body>
<p>This</p>
</body>
<style>
p { 
        background-color: #8ABB55;
        margin: 0px;
 }
body{
    margin: 0px;
} 
</style>
</html>
```

所以块级元素占据其父元素的整个水平空间，高度等于其内容高度，像是一块一样，所以被称为块级元素。所以两个块级元素不能位于一行，默认情况下在块级元素之后，再写块级元素，块级元素会新起一行。以下是HTML中所有的块级元素列表:

- address article aside  blockquote  dd div
- dl fieldset  figcaption  figure  footer  h1-h6  form
- header  hgroup  hr  ol  p  pre  section table  ul

### 行内元素

那我们可以类比块级元素对行内元素下定义，块级元素占据一行，那行内元素就是只占据它对应标签的边框所包含的空间:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>
<body>
    <p>This <span>span</span> is an inline element; its background has been colored to display both the beginning and end of the inline element's influence</p>
</body>
<style>
    span { background-color: #8ABB55; }
</style>
</html>
```

![行内元素效果](http://tva4.sinaimg.cn/large/006e5UvNgy1h8iezfykvfj31gl0ok0uy.jpg)

一般情况下，行内元素只能包含数据和其他行内元素。而块级元素可以包含行内元素和其他块级元素。默认情况下，行内元素不会以新行开始，而块级元素会新起一行。下面的元素都是行内元素:

- b, big，i，samll，tt。
- abbr，acronym，cite，code，dfn，em，kbd，strong，samp，var。
- a，bdp，br，img，object，q，script，span，sub，sup
- button，input，label，select，textarea。

## 盒子模型

但内容之间通常有距离还往往不够用，就像一幅画一样，我们为了保护这幅画通常会买一个画框，通常画框会比要装的画大一些，起到留白的效果:

![盒子模型](http://tva2.sinaimg.cn/large/006e5UvNgy1h8igiuyridj312n0llq8p.jpg)

当然也有padding(padding为0)的，浏览器的渲染引擎会根据标准之一的CSS基础盒模型(CSS basic box model),将所有的元素表示为一个个矩形的盒子(box)。根据网页中声明的CSS，来决定这些盒子的大小、位置、以及属性(例如颜色、背景、边框尺寸)。每个盒子都由四个部分来组成，如上面的图片所示，每个盒子有四个组成区域

- 内容区域 content area：容纳着元素的“真实内容”，例如文本、图像。它的尺寸为内容宽度和内容高度。通常含有一个背景颜色或背景图像。

> 如果box-sizing 这个属性为默认属性即content-box，则内容区域可以通过明确的width、min-width、max-width、height、min-height，和max-height控制。

- 内边距区域 padding area:  扩展内容区域，间距的大小可以通过明确的padding-top、padding-right、padding-bottom、padding-left和简写属性padding来控制。
- 边框区域 border area:  扩展内边距区域，是容纳边框的区域，边框的粗细由border-width和简写的border属性控制，如果boder-sizing属性被设置为border-box。那border区域的大小可以明确通过width、min-width, max-width、height、min-height，和 max-height 属性控制。假如框盒上设有（background-color 或 background-image），背景将会一直延伸至边框的外沿（默认为在边框下层延伸，边框会盖在背景上）。此默认表现可通过 CSS 属性 background-clip 来改变。
- 外边距区域 margin  area: 即盒子与盒子的距离，外边距的大小由 margin-top、margin-right、margin-bottom、margin-left，和简写属性 margin 控制。在发生外边距合并的情况下，由于盒之间共享外边距，外边距不容易弄清楚。

![盒子模型-检查](http://tvax3.sinaimg.cn/large/006e5UvNgy1h8ihbpzge8j31gw0o0wh5.jpg)



上面我们提到的就属于盒子模型，一般我们称之为标准盒模型。在标准盒模型中，如果你给盒子本身设置width和height，实际上设置的是content box，即内容的长宽高。padding和border再加上设置的宽高一起决定盒子的大小。

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>
<body>
<p>This</p>
</body>
<style>
p { 
        background-color: #8ABB55;
        margin: 10px;
        width: 200px;
        padding: 10px;
        border-width:10px;
        border-style: solid;
        border-color: blue;
 }
body{
    margin: 0px;
} 
</style>
</html>
```

实际效果:

![标准盒子模型示例](http://tvax4.sinaimg.cn/large/006e5UvNgy1h8iluzbhhqj31ga0nstae.jpg)



### IE盒子模型

某些时候标准盒子模型也让我们产生困扰，难道我们我们不能直接设置整个盒子的宽高吗？ 还要加上border和padding，这很麻烦。IE模型中，内容宽度是设置的宽度减去边框和填充部分。一般情况下浏览器采用的都是标准模型，如果需要使用IE模型，可以通过设置box-sizing: border-box来实现。

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>
<body>
<p>This</p>
</body>
<style>
p {     
        box-sizing: border-box;
        background-color: #8ABB55;
        margin: 10px;
        width: 200px;
        padding: 10px;
        border-width:10px;
        border-style: solid;
        border-color: blue;
 }
body{
    margin: 0px;
} 
</style>
</html>
```

![IE盒子模型](http://tva3.sinaimg.cn/large/006e5UvNgy1h8im5dz3svj31f50n840l.jpg)



如果你希望所有元素都使用IE盒子模型，那我们可以将box-sizing在html元素上，然后设置所有元素继承该属性，如下面代码所示:

```css
html {
  box-sizing: border-box;
}
*, *::before, *::after {
  box-sizing: inherit;
}
```

IE浏览器默认使用IE盒子模型，不支持切换标准盒子模型（IE8+ 支持使用box-sizing进行切换）

## 块级盒子和内联盒子

HTML中元素分为块级元素和行级元素，在CSS中盒子也分为块级盒子和内联盒子。块级盒子由块级元素产生，内联盒子由行内元素产生。块级元素会表现出以下行为:

- 盒子会在内联方向扩展并占据父容器在该方向上所有可用空间，在绝大多数情况下意味着盒子会和父容器一样宽。
- 块级盒子不能并列，每个盒子独占一行。
- width和height属性可以发挥作用
- 内边距(padding), 外边距(margin)和边框(border)会将其他元素从当前盒子周围推开。

除非特殊指定，注入标题h1等和段落在默认情况下都是块级的盒子。我们可以通过display属性来将当前元素设置为块级元素或内联元素。display常见的属性有:

- block: 将当前盒子设置为块级盒子
- inline: 将当前盒子设置为内联盒子
- inline-block:  我们有的时候会希望元素具备宽度和高度属性，但是有具备同行特性。在这种情况下就要可以使用inline-box。

如果一个盒子为内联盒子，那么这个盒子具备以下性质:

- 盒子不会换行
- width和height属性不起作用
- 垂直方向的内边距(padding)、外边距(margin)以及边框(border)会被应用但是不会把其他处于 `inline` 状态的盒子推开。
- 水平方向的内边距、外边距以及边框会被应用且会把其他处于 `inline` 状态的盒子推开。

用做链接的 `<a>` 元素、 `<span>`、 `<em>` 以及 `<strong>` 都是默认处于 `inline` 状态的。

我们可以通过display属性的设置，比如inline或者block，来控制盒子的外部显示类型。那么有外部显示类型，就会有内部显示类型。外部显示类型决定盒子是块级还是内联，而内部显示类型则决定盒子内部元素是如何布局的。默认情况下和其他块元素以及内联元素一样。但是我们也可以通过使用flex的display属性值来更改内部显示类型。如果设置display：flex，在一个元素上，外部显示类类型是block，但是内部显示类型修改为flex。该盒子的所有直接子元素都会变成flex元素，会根据弹性盒子flexbox 规则进行布局，我们会在后文介绍这些内容。

块级和内联布局是web上的默认行为，有的时候，他们也被称为正常文档流。

## 外边距

外边距是盒子周围一圈看不到的空间。它会把其他元素从盒子旁边推开。外边距属性值可以为正，也可以为负。设置负值会导致和其他内容重叠。无论使用标准模型还是替代模型，外边距总是在计算可见部分后额外添加。我们可以使用margin属性一次控制一个元素的所有边距，或者每边单独使用等价的普通属性控制:

- margin-top
- margin-right
- margin-bottom
- margin-left

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>
<body>
    <div class="container">
        <div class="box">Change my margin.</div>
      </div>
          
</body>
<style>
.box {
  margin-top: -40px;
  margin-right: 30px;
  margin-bottom: 40px;
  margin-left: 4em;
  border: 5px solid rebeccapurple;
  background-color: lightgray;
  padding: 10px;
   height: 150px;
}
.container {
    border: 5px solid blue;
    margin: 40px;
}
</style>
</html>
```

![效果图](http://tva1.sinaimg.cn/large/006e5UvNgy1h8iniyd0ugj31gd09tdga.jpg)

尝试更改外边距的值，感受一下在外边距设置为正时是如何推开周边元素，以及设置为负时是如歌重叠的。

### 外边距折叠

理解外边距的一个关键是外边距折叠概念，如歌你有两个外边距相接的元素，这些外边距将合并为一个外边距，即最大的单个外边距的大小。在下面的例子中，我们有个两个p标签，第一个标签的margin-bottom为50px。第二段的margin-top为30px。因为外边距折叠的概念，所以框之间的实际外边距是50px，而不是两个外边距的总和。

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>
<body>
    <div class="container">
        <p class="one">I am paragraph one.</p>
        <p class="two">I am paragraph two.</p>
      </div>
</body>
<style>
.container {
    border: 5px solid blue;
    margin: 100px;
}
p {
    border: 5px solid rebeccapurple;
    background-color: lightgray;
    padding: 10px;
}  
.one {
  margin-bottom: 50px;
}

.two {
  margin-top: 30px;
}
</style>
</html>
```

可以尝试修改一下.two的margin-top，只要小于50px，两者的距离不会发生变化，当大于五十的时候按.two的折叠。

![外边距折叠](http://tvax1.sinaimg.cn/large/006e5UvNgy1h8iohgz9k1j31f209n3zd.jpg)

## 内边距

内边距位于边框和内容区域之间。与外边距不同，不可以设置负值。应用于元素的任何背景色都将显示在内边距中。我们可以使用padding简写属性控制元素所有边，或者每边单独使用等价的普通属性:

- padding-top
- padding-right
- padding-bottom
- padding-left

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=, initial-scale=1.0">
    <title>Document</title>
</head>
<body>
    <div class="container">
        <div class="box"> 改内边距</div>
    </div>
</body>
<style>
.container {
    border: 5px solid blue;
    margin: 40px;
    padding: 20px;
}
.box {
    padding-top: 40px;
    padding-right: 20px;
    padding-bottom: 40px;
    padding-left: 4em;
    border: 5px solid rebeccapurple;
    background-color: lightgray;
}
</style>
</html>
```

![内边距](http://tva3.sinaimg.cn/large/006e5UvNgy1h8ioiasvvij31gw06pgm7.jpg)

## 边框

相对于内边距和外边距，边框的属性很多，可以设置边框的宽度、边框颜色，边框样式(边框是虚线 还是直线)。上面的例子我们已经有意无意的用过了：

```css
 border: 5px solid blue;
```

这是为四个边框统一设置宽度、样式、颜色。我们也可以分别设置四个边框的宽度、样式、颜色:

- border-top
- border-right
- border-bottom
- border-left

设置所有边的宽度、样式、颜色也可以这么声明:

- border-width
- border-style
- border-color

我们也可以单独设置每个边的宽度、样式、颜色:

- border-top-width
- border-top-style
- border-top-color
- border-right-width
- border-right-style
- border-right-color
- border-bottom-width
- border-bottom-style
- border-bottom-color
- border-left-width
- border-left-style
- border-left-color

## 总结一下

盒子模型类似于搬家之后对东西进行重新摆放，这也被称为布局，如果房子小，margin就会小就会显得很拥挤。一个良好的布局会让居住者赏心悦目。本篇基本参考了《千古前端图文教程》，《MDN web docs》。这里用我自己的方式将这些内容整合了一下。注意JavaFX我们前面写的文章也会有这样的布局问题，两者是通用的。

## 参考资料

- 千古前端图文教程  https://web.qianguyihao.com/02-CSS%E5%9F%BA%E7%A1%80/06-CSS%E7%9B%92%E6%A8%A1%E5%9E%8B%E8%AF%A6%E8%A7%A3.html#%E7%9B%92%E5%AD%90%E6%A8%A1%E5%9E%8B

- mdn web docs https://developer.mozilla.org/zh-CN/docs/Learn/CSS/Building_blocks/The_box_model
- CSS 中的众多盒子概念怎么区分？ https://www.zhihu.com/question/330596315
