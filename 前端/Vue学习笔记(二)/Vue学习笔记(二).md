# Vue学习笔记(二)  相识篇

> 阅读本文需要基本掌握HTML、CSS和JavaScript的中级知识，还有《Vue学习笔记(二)  初遇篇》。本篇在某种程度上并不是承接上一篇，更像是初遇篇的重构。

[TOC]



## 前言

去思否翻了翻自己第一篇写Vue相关的文章《Vue 学习笔记(一)初遇》, 发表的日期是2019年12月14号，也就是这一年出现了疫情，到现在大概有三年了，不胜唏嘘。今年据说是“寒意凛然”，为了提升自己的性价比，增强自己的竞争力，我决定重新学习一下前端。这是开玩笑的话，其实是为了做一个开源项目做准备。让我们来回顾一下《Vue 学习笔记(一)初遇》讲的东西，在这篇内容中我们主要讲了JavaScript的大致发展历史，早期的浏览器对JavaScript适配不完全，相同的JavaScript可能在不同的浏览器上没有一致的行为，而且在写控制DOM结点方面十分繁琐，于是有人推出了JQuery，化繁为简，但是随着网页在越来越复杂，由于JQuery直接操纵DOM结点，DOM的节点的改变就会带来网页的重绘，于是后面的前端框架的思路就是推出虚拟DOM，运用diff算法来计算出真正需要更新的节点，最大限度地减少DOM操作以及DOM操作带来的排版与重绘损耗，从而显著提高性能。 等等，JQuery操纵结点需要重绘，你的Vue也需要重绘啊？ 那么你的Vue究竟好在哪里呢？ 好在，会合并运算，将多次更改合并为一次，举一个例子，在JQuery中假设需要操纵三次Dom，由三个方法触发，但其实最终不需要更新那么多次结点，我们将DOM理解为一个数字，每次操作像是对这个数字进行加减，那么三次DOM操作可能就像是 5+ 3 - 5 , 最终只需要加3渲染一次就行了，这是Vue的做法。但是对于JQuery来说就是运算三次，带来三次页面重绘。这个性能节省就在这里。 那么对于一个复杂的页面，你这个diff算法的计算不需要时间吗？当然也需要时间, 其实我们Vue主要还是在提升开发效率、代码复用性。 这也就是Angular、Vue、React的思路，但是这些前端框架所带来的并不仅仅是虚拟DOM，还带来了响应式、MVVM。但Vue并不具备侵入性，是渐进式的，所谓渐进式也就是说，如果你现在就有一个应用，那么你可以选择将Vue嵌入到你的应用中，来享受Vue提供的便利，而不会对全局进行改变。等一等，你说Vue提供的便利，你说到这个我就不困了，能举一个例子来说明下Vue的便利吗？  好的，下面是一个最终效果图: 

![最终效果图](http://tvax1.sinaimg.cn/large/006e5UvNgy1h7m69g5qfaj30kl04ojrl.jpg)

对html有一点熟悉的同学可能一下子就看出来了，这是四个无序列表标签，列表标签后面是一个json。如果这些json是从服务端传递过来的，那么用JQuery实现上面的思路就是首先使用Ajax获取前端数据，然后拼接成对应的html字符串，通过JQuery选择器选中对应的元素，获取到JQuery对象之后，将对应的html字符串传给对象的append方法，此方法会将我们拼接的字符串渲染为真实的DOM结点。

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
  <div id = "app">

  </div>  
</body>
 
<script src="https://cdn.bootcdn.net/ajax/libs/jquery/3.6.1/jquery.min.js">
    // 这个标签用于引入JQuery
</script>
<script>
   $(document).ready(function(){
        // dom加载结点之后触发getJSON,获取的数据会传给result
		$.getJSON("http://localhost:8080/hello",function(result){
            var products  = result.products;
            console.log(products);
            var targetStr = "";    
            // 通过foreach方法拼接成最终的字符串
			$.each(products, function(i, field){
                field = JSON.stringify(field);
                targetStr  = targetStr + "<li>" + field + "</li>"
            });
            // 通过append方法创建结点
            $("#app").append("<ul>"+targetStr+"</ul>");
		});
});
</script>
</html>
```



最终的结果如下图所示:

![JQuery最终结果](http://tva4.sinaimg.cn/large/006e5UvNgy1h7m86dvweoj30ph036t8v.jpg)

```json
// getJson返回的数据格式如下
{
  "products": [
    {
      "id": 1,
      "quantity": 1,
      "name": "红焖羊肉"
    },
    {
      "id": 2,
      "quantity": 0,
      "name": "猪肉韭菜饺子"
    },
    {
      "id": 3,
      "quantity": 4,
      "name": "热干面"
    },
    {
      "id": 4,
      "quantity": 5,
      "name": "烩面"
    }
  ]
}
```

下面Vue的写法：

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
<div id = "app">
    <ul v-for = "product in products" > 
        <li>
             {{product}} 
        </li> 
    </ul>
</div>
<div>
</div>
<script src="https://cdn.jsdelivr.net/npm/vue@2/dist/vue.js">
    // 这里代表引入Vue
</script>
<script>
    // 代表创建一个Vue实例
    const app = new Vue({
        el:'#app', // el 代表element 后面跟的是需要Vue接管的元素
        data:{
            products:[]
        },
        // 页面加载的时候触发created函数
        created(){
            // 请求这个地址
           fetch('http://localhost:8080/hello').then(response=> response.json()).then(json => {
             // 将拿到的值赋给products    
            this.products = json.products;
           })     
        }
    })
</script>
</body>
</html>
```

上面的v-for 就是循环products，最终的效果与JQuery一样，但是Vue的更为自然一点，如果我们要再加要求，比如计算出还剩多少份饭，渲染到指定结点上，剩余份数为0就显示饭已经卖光了，如果用JQuery来做，还是用选择器选择对应的元素，然后写入到对应的元素中，这里不再给出JQuery的写法，我们直接上Vue的写法:  

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
<div id = "app">
    <ul v-for = "product in products" > 
        <li>
             {{product.name}}  {{product.quantity}} 
             <span v-if = "product.quantity === 0">
                已经售空
            </span>
            <button @click = "product.quantity += 1">
                添加
           </button> 
        </li> 
    </ul>
    <h2> 总库存 {{ totalProducts}}</h2>
</div>
<script src="https://cdn.jsdelivr.net/npm/vue@2/dist/vue.js">
    // 这里代表引入Vue
</script>
<script>
    // 代表创建一个Vue实例
    const app = new Vue({
        el:'#app', // el 代表element 后面跟的是需要Vue接管的元素
        data:{
            products:[]
        },
        // 页面加载的时候触发created函数
        created(){
            // 请求这个地址
           fetch('http://localhost:8080/hello').then(response=> response.json()).then(json => {
             // 将拿到的值赋给products    
            this.products = json.products;
           })     
        },
        // computed 是计算属性
        computed:{
            totalProducts(){
                // reduce是js提供, 进行累加   
                return this.products.reduce((sum,product) => {
                    return sum + product.quantity;
                },0);
            }
        },
    })
</script>
</body>
</html>
```

最终的效果如下图所示: 

![Vue的最终效果](http://tvax4.sinaimg.cn/large/006e5UvNgy1h7m9t9dtz4j30nx061q31.jpg)

当我们点击猪肉韭菜饺子的添加按钮，你会发现已经售空会消失，同时总库存会响应的增加。这也就是Vue中响应式的真义: 数据在发生变动的时候,Vue会帮你更新所有网页中用到它的地方。上面的例子中我么首先碰到的就是{{变量名}}， 这种语法被用来获取Vue实例中data对象的属性，这种语法在Vue中我们称之为模板语法，这在Vue中被称之为声明式渲染，Vue 基于标准 HTML 拓展了一套模板语法，使得我们可以声明式地描述最终输出的 HTML 和 JavaScript 状态之间的关系。 声明式和响应式是Vue的两个核心功能。

## MVVM模式

做过服务端开发大多都对MVC模式不会模式:

![MVC模式](http://tvax2.sinaimg.cn/large/006e5UvNgy1h7u0qdvdx5j30um0ctmxf.jpg)



view负责图形显示，是提供给用户的操作界面，是程序的外壳。Controller负责转发用户请求，将请求分发给对应的model，由model对提取用户所需要的数据。那MVVM模式呢？下面是一张介绍Vue中mvvm模式的一张经典图片:

![mvvm](http://tvax3.sinaimg.cn/large/006e5UvNgy1h7u0zr5prdj318g0nmadh.jpg)

页面中的一个常规操作是我们用JavaScript对DOM节点的某些事件进行监听，事件发生触发我们的监听函数，监听函数更新DOM。在Vue中引入了v-model指令来对上面的常规操作进行封装，我们无需在写监听事件，借助v-model指令，页面的数据发生变化时候自动到达我们的函数，函数来更新对应的节点，这一切仅仅需要一个简单v-model指令,让我们来看下面一个例子:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>
<div id = "app">
    <p>Message is: {{ message }}</p>
    <input v-model="message" placeholder="edit me" />
</div>

<body>
<script src="https://cdn.jsdelivr.net/npm/vue@2/dist/vue.js"></script>   
<script>
 var vue = new Vue({
    el:'#app',
    data:{
       'message': 'Hello Vue.js'     
    }
 })
</script>
</body>
</html>
```

当我们在输入框中输入值，输入值会自动替换p标签的{{message}}, 你看看这是不是简单很多了， 这里我们先让v-model漏个面，简单的先认识下，忘了说，这种现象我们也称之为双向绑定。

## Vue 简介

什么是 Vue？Vue2文档的回答是:

>  Vue (读音 /vjuː/，类似于 **view**) 是一套用于构建用户界面的**渐进式框架**。与其它大型框架不同的是，Vue 被设计为可以自底向上逐层应用。

Vue2文档的回答最终只突出渐进式框架, 我觉得这个回答是不够直观的。那我们接着来看Vue 3是如何回答这个问题: 

> Vue (发音为 /vjuː/，类似 **view**) 是一款用于构建用户界面的 JavaScript 框架。它基于标准 HTML、CSS 和 JavaScript 构建，并提供了一套声明式的、组件化的编程模型，帮助你高效地开发用户界面。无论是简单还是复杂的界面，Vue 都可以胜任。

我们从这个回答中看到的就比较多了，并且更加直观，相对于Vue2的回答较新的词是声明式、组件化，声明式在上面我们已经介绍过了，那什么是组件化？ 或者什么是组件？ 组件事实上是一个含义很大的概念，一般是指软件系统的一部分，承担了特定的职责，可以独立于整个系统进行开发和测试，一个良好设计的组件应该可以在不同的软件系统中被使用(可复用)。例如V8引擎是Chrome浏览器的一部分，负责运行JavaScript代码，这里V8引擎就可以被视为是一个组件。V8引擎同时也是Node.js的JavaScript解释器，这体现了组件的可复用性。Vue组件也是如此，承担了特定的职责，具备可复用性。举一个例子，我们司空见惯的下拉框，一个按钮事实上分成三块，第一个是骨架也就是html，第二个是css也就是样式，第三个是触发按钮事件(点击)之后的逻辑(JavaScript)。在Vue中我们就可以将这个按当做一个组件来看待,如下图所示: 

```vue
<script>
export default {
  data() {
    return {
      count: 0
    }
  }
}
</script>

<template>
  <button @click="count++">Count is: {{ count }}</button>
</template>

<style scoped>
button {
  font-weight: bold;
}
</style>
```

我们将上面的代码放入一个叫button.vue文件中，如果我们想在别的地方使用这个组件，在需要用到的地方使用即可，我们可以将button.vue称之为单文件组件，用英文名为Single-File Component, 缩写为SFC。单文件是组件是Vue的标志性功能。基于此我们就可以将一个页面切分成若干个组件，组件系统是 Vue 的另一个重要概念，因为它是一种抽象，允许我们使用小型、独立和通常可复用的组件构建大型应用。仔细想想，几乎任意类型的应用界面都可以抽象为一个组件树：

![组件树](http://tvax1.sinaimg.cn/large/006e5UvNgy1h7ucnl2c7wj31320f4dh6.jpg)

这个图片来自Vue官网。Vue2的文档在回答何为Vue的时候，突出了渐进式，但是在Vue 3的第一回答中没有渐进式，但这并不意味着Vue具备侵入性，它仍然是渐进式的，渐进式出现了Vue的特点介绍中。在上面的例子中，我们就是没有借助Vue的构建工具，而是将Vue当作类似JQuery一样的存在，渐进式的增强静态的html。事实上Vue的使用方式是多种多样的，你可以这样使用Vue:

- 无需构建步骤，渐进式增强静态的 HTML
- 在任何页面中作为 Web Components 嵌入
- 单页应用 (SPA)
- 全栈 / 服务端渲染 (SSR)
- Jamstack / 静态站点生成 (SSG)
- 开发桌面端、移动端、WebGL，甚至是命令行终端中的界面

## Vue 实例

在上面的例子中，我们的应用总是从new 一个Vue对象开始，再看一下我们的第一个Vue示例，这个示例中我们用到了Vue实例中的data和el属性、created方法，el属性用来指明Vue实例的作用范围，在Vue实例的作用范围中，在html中可以用双大括号可以取出data对象的值，当一个 Vue 实例被创建时，它将 `data` 对象中的所有的 property 加入到 Vue 的**响应式系统**中。当这些 property 的值发生改变时，视图将会产生“响应”，即匹配更新为新的值。Vue的构造函数会为Vue的成员变量赋值，注意在JavaScript函数也能当做一个变量处理，所以我们也可以选择传入一个函数。那Vue实例有哪些成员变量呢，我们来看以下Vue的API文档:

![Vue](http://tvax3.sinaimg.cn/large/006e5UvNgy1h7nd5es4jwj318o0l4di6.jpg)

我们目前用过的成员变量有data这个属于数据，computed属于计算属性，created方法在Vue的官方文档中被称为钩子，每个 Vue 实例在被创建时都要经过一系列的初始化过程——例如，需要设置数据监听、编译模板、将实例挂载到 DOM 并在数据变化时更新 DOM 等。同时在这个过程中也会运行一些叫做**生命周期钩子**的函数，这给了用户在不同阶段添加自己的代码的机会。还有很多陌生的，不要着急，我们会做懒加载。

## 模板语法

我们上面使用的是Vue模板语法中最常见的一种，被称为”Mustache“语法(双大括号取值)的文本取值，如果你想取出来的是html，可以通过下面的指令告诉Vue：

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
<div id = "app">
    <span v-html = "product"></span>
</div>
<script src="https://cdn.jsdelivr.net/npm/vue@2/dist/vue.js">
    // 这里代表引入Vue
</script>
<script>
    // 代表创建一个Vue实例
    const app = new Vue({
        el:'#app', // el 代表element 后面跟的是需要Vue接管的DOM元素ID
        data:{
            product: "<h1>这是被Vue渲染出来的</h1>"
        },
    })
</script>
</body>
</html>
```

最终效果: 

![Vue渲染出来的](http://tva1.sinaimg.cn/large/006e5UvNgy1h7n4vdsq8vj309g029mx4.jpg)

动态渲染的任意HTML可能会非常危险，因为它很容易导致XSS攻击，请谨慎使用动态渲染。那这种模板语法能否作用dom元素的属性呢，在Vue是不能的，但是Vue中提供了另一种曲线救国的方式：v-bind指令。v-bind指令可以帮我们将dom元素的属性和Vue对象的data实例绑定在一起。下面是v-bind指令的一个例子：

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
<div id = "app">
    <span v-html = "product"></span>
    <div v-bind:id = "dynamicId" >
    <button v-bind:disabled="isButtonDisabled">Button</button>
    </div>
</div>
<script src="https://cdn.jsdelivr.net/npm/vue@2/dist/vue.js">
    // 这里代表引入Vue
</script>
<script>
    // 代表创建一个Vue实例
    const app = new Vue({
        el:'#app', // el 代表element 后面跟的是需要Vue接管的DOM元素ID
        data:{
            product: "<h1>这是被Vue渲染出来的</h1>",
            dynamicId: "dynamicId",
            isButtonDisabled: true // 对于布尔类型的属性, false、null、undefined,对应的属性不会出现在渲染出来的元素中
    })
</script>
</body>
</html>
```

v-bind的语法为: v-bind:绑定属性。 

## 类与样式绑定

### 操纵元素的class

操纵元素的class列表和内联样式是数据绑定的一个常见需要，它们都是attribute，所以我们可以用v-bind指令来处理，但是拼接字符串往往是个麻烦的事情，因此在Vue 中专门对class和style 做了增强，表达式的结果除了字符串之外，还可以是对象和数组。让我们来看下面的例子:

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
<div id = "app">
    <div v-bind:class="{ active: isActive }" id = "app01">测试</div>    
    <div v-bind:class="{  active: isActive, 'text-danger': hasError }" id = "app02">这代表对象语法</div>    
    <div v-bind:class="[activeClass, errorClass]"> 这是数组语法</div>
</div>
</body>
<script src="https://cdn.jsdelivr.net/npm/vue@2/dist/vue.js"></script>   
<script>
 var vue = new Vue({
    el:'#app', // active这个样式的存在与否取决于isActive的true、false
    data: {
        isActive: true, // #app02 是否具备active和text-danger取决于isActive和hasError的true和false
        hasError: true,
        activeClass: 'active',
        errorClass: 'text-danger'
    }
 })
 </script>
</html>  
```

### 绑定内联样式

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
    <div id = "app">
        <div v-bind:style="styleObject"> 对象语法</div>
        <div v-bind:style="[styleObject, overridingStyles]"> 数组语法,data中声明的overridingStyles、overridingStyles在style中</div>
    </div>
</body>

<script src="https://cdn.jsdelivr.net/npm/vue@2/dist/vue.js"></script>   
<script>
    var vue = new Vue({
       el:'#app', 
       data: {
        styleObject: {
            color: 'red',
           fontSize: '13px'
        },
        overridingStyles: {
           'background-color': 'black',
            'width':'500px'
        }
       }})
    </script>
</html>
```

## 计算属性

模板里面可以允许出现一些表达式，如下面的代码所示: 

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
    <div id = "app">
         {{number  + 1}}
         {{message}}
         {{ message.split('').reverse().join('') }}
         {{tips}}
         {{reversedMessage}}
        这里直接调用方法{{reversedMessagePlus()}}
    </div>
</body>
<script src="https://cdn.jsdelivr.net/npm/vue@2/dist/vue.js">
    // 这里代表引入Vue
</script>
<script>
    var vue  = new Vue({
        el: '#app',
        data:{
            number: 0,
            message: '护维以难,法写种这荐推不',
            tips: 'Vue 为我们准备了计算属性来应对这样的场景'
        },
        //  下面是用计算属性完成的
        computed:{
            // reversedMessage 关联上函数
            // 这个函数将被用作reversedMessage的getter 函数
            // 当data中的message发生改变, reversedMessage会自动跟着改变
            // 验证这一点只需要打开浏览器的控制台, 为vue.messsage 赋值即可
            reversedMessage:function(){
                return this.message.split('').reverse().join('')
            }
        },
        // 注意methods 和 computed中的函数不能同名
        methods: {
            reversedMessagePlus:function () {
                 return this.message.split('').reverse().join('')
            }
       }   
    })
</script>
</html>
```

在computed函数中的我们称之为计算属性，在methods中我们称之为普通的方法，两种方式的最终计算结果是完全相同的。不同的在于计算属性是基于它们的响应式依赖进行缓存的。只有相关响应式依赖发生改变时它们才会重新求值。这就意味着只要message还没有发生改变，多次访问reversedMessage会立刻返回之前的计算结果，而不必再执行函数。

下面的计算属性不会再页面上更新，原因在于Date.now并不是响应式依赖，不依赖于data对象的值。

```vue
computed: {
  now: function () {
    return Date.now()
  }
}
```

相比之下，每次触发重新渲染，调用方法将再次执行methods中的函数。计算属性的缓存在于，假如我们需要有一个开销很大的计算属性A，它需要遍历一个巨大的数组并作大量的计算。然后我们可能其他的计算属性依赖于A，如果没有缓存，那么这个计算就会被执行多次。

## 条件渲染与列表渲染

所谓条件渲染我们可以这么理解，满足一定条件，对应的dom元素才会被渲染出来，就像我们上面用到的v-if。有v-if 就有v-else。值得注意的时v-else元素必须紧跟v-if或者`v-else-if` 的元素的后面，否则它将不会被识别。v-else-if 于Vue在2.10版本推出，充当 `v-if` 的“else-if 块”，可以连续使用。值得注意的时，Vue会尽可能高效地渲染元素，通常会复用已有元素而不是从头开始渲染。这么做除了使 Vue 变得非常快之外，还有其它一些好处。例如，如果你允许用户在不同的登录方式之间切换：

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
    <div id = "app">
        <template v-if="loginType === 'username'">
            <label>Username</label>
            <input placeholder="Enter your username">
          </template>
          <template v-else>
            <label>Email</label>
            <input placeholder="Enter your email address">
          </template>
          <button @click = "toggle">切换登陆方式</button>
    </div>
    
</body>
<script src="https://cdn.jsdelivr.net/npm/vue@2/dist/vue.js">
    // 这里代表引入Vue
</script>
<script>
      var vue  = new Vue({
        el: '#app',
        data: {
            loginType: 'username'
        },
        // 在这个例子中,用户输入,点击切换方式,并不会清空用户的输入。  
        methods:{
            toggle:function(){
                this.loginType = this.loginType === 'username' ? 'email': 'username';
            }
        }    
      });
</script>
</html>
```

   如果想点击切换登陆方式之后切换,给每个input的key属性一个唯一的key就行，像下面这样:

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
    <div id = "app">
        <template v-if="loginType === 'username'">
            <label>Username</label>
            <input placeholder="Enter your username" key="username-input">
          </template>
          <template v-else>
            <label>Email</label>
            <input placeholder="Enter your email address" key="email-input">
          </template>
          <button @click = "toggle">切换登陆方式</button>
    </div>
    
</body>
<script src="https://cdn.jsdelivr.net/npm/vue@2/dist/vue.js">
    // 这里代表引入Vue
</script>
<script>
      var vue  = new Vue({
        el: '#app',
        data: {
            loginType: 'username'
        },
        methods:{
            toggle:function(){
                this.loginType = this.loginType === 'username' ? 'email': 'username';
            }
        }    
      });
</script>
</html>
```

另一个用于根据条件展示元素的选项是 `v-show` 指令。用法大致一样：

```html
<h1 v-show="ok">Hello!</h1>
```

不同的是带有 `v-show` 的元素始终会被渲染并保留在 DOM 中。`v-show` 只是简单地切换元素的 CSS property `display`。也就是隐藏。

v-for就是列表渲染，遍历data中的元素，这里不再做介绍。

## 总结一下

重新学习了一下JavaScript内框架与库的发展历史，从原生JavaScript到JQuery，再到现代前端框架。现代前端框架更加着眼于代码复用、性能、开发效率。我们上面介绍了Vue的几大重要特性: MVVM，双向绑定、响应式。MVVM是一种理念，某种程度上双向绑定可以算的上是MVVM在Vue的实现，所谓双向绑定，就是我们的数据对象和DOM建立关系之后，数据会自动的从DOM结点流向我们的数据对象，在Vue实例中我们如果对数据对象再进行操作，不需要我们直接更新DOM结点，会自动的挂载到对应的数据结点。网页开发中我们一个常见的需求是根据数据渲染结点，在Vue中我们可以用模板语法、条件渲染、列表渲染来完成。如果你想要操纵样式，可以通过V-bind指令来完成。

## 参考资料

- 虚拟DOM(Virtual DOM)的优势在哪里？  https://juejin.cn/post/7106340182974005255#heading-2

- 为什么说JS的DOM操作很耗性能  https://zhuanlan.zhihu.com/p/86153264
- 怎样精确区分这些名词：库、插件、组件、控件、扩展？ https://www.zhihu.com/question/49536781/answer/117606933
- MVC  https://developer.mozilla.org/zh-CN/docs/Glossary/MVC