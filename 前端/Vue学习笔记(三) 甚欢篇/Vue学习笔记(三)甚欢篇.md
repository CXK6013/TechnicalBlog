# Vue学习笔记(三) 甚欢篇

[TOC]

## 事件处理

我们在初遇篇已经有意无意的用了监听事件这一概念，我们来回忆一下那个例子:

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

我们在button 标签中写了一个@click =  product.quantity += 1， 这个的语义代表的是当我们在网页点击按钮的时候将product.quantity 加1。对JavaScript有一定了解的同学可能会有以为，有点击事件不假，但是这种写法在JavaScript是没有的。是的，这种写法是Vue带来的, 之所以没有在相识篇展开来介绍，是因为与事件有相关的还有事件的冒泡与捕获，不想都放在一篇文章去介绍，都放在一篇会让一篇的内容篇幅偏大。在完整的介绍Vue的事件与监听之前，我们这里来介绍一下事件冒泡与捕获、JavaScript的模块化。之所以介绍这些呢，原因在于我把这些忘记了。

### 事件系统简介

什么是事件？ 现实中的某某事件一般是某某在某地发生了什么事情。在JavaScript中，事件就是网页中发生的一些特定的交互瞬间。对于Web应用来说，有下面这些代表性事件:  某个元素被点击、关闭弹窗、将鼠标移动至某个元素上方等等。JavaScript是以事件驱动为核心的一门语言。JavaScript与HTML之间的交互是通过事件系统来实现的。一个事件系统有以下三个特点: 

- 事件触发 即某事件发生了
- 通知监听该事件的监听方
- 监听方处理事件

在JavaScript中，我们只需要在元素上生命需要监听的事件和对应的处理函数，事件发生时候会自动的触发我们的处理函数。我们的html的结点像是一颗DOM树，在浏览器中事件并非只和事件对应的元素有关，这也就是事件的传播，事件的传播分成三个阶段: 

- 事件捕获阶段: 事件从祖先元素往子元素查找 , 直到捕获到事件目标。这个过程中，默认情况下，事件对应的监听函数是不会触发的。
- 事件目标: 当到达目标元素之后，执行目标元素该事件相应的处理函数。
- 事件冒泡阶段：事件从事件目标开始，从子元素往祖先元素冒泡，直到页面上的最上一级标签。

![事件冒泡与事件捕获](http://tva4.sinaimg.cn/large/006e5UvNgy1h7vaat5bj7j30gs0d1jru.jpg)



下面是代码示例: 

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
    <div id = "box1" style = "background-color: grey;width: 800px; height: 600px;">
        <div id = "box2"  style = "background-color: darkcyan;width: 400px; height: 300px;">
              <div id = "box3" style = "background-color: bisque;width: 200px; height: 150px;">

              </div>  
        </div>
    </div>
</body>
<script>
    var box1 = document.getElementById("box1");
    var box2 = document.getElementById("box2");
    var box3 = document.getElementById("box3");

    box3.onclick = function () {
        alert("box3 冒泡");
    }

    box2.onclick = function () {
        alert("box2 冒泡");
    }

    box1.onclick = function () {
        alert("box1 冒泡");
    }

    document.onclick = function () {
        alert("document 冒泡");
    }

    box1.addEventListener("click", function () {
        alert("box1 捕获");
    }, true);
    box2.addEventListener("click", function () {
        alert("box2 捕获");
    }, true);

    box3.addEventListener("click", function () {
        alert("box3 捕获");
    }, true);
    window.addEventListener("click", function () {
        alert("捕获 window");
    }, true);

    document.addEventListener("click", function () {
        alert("捕获 document");
    }, true);

    document.documentElement.addEventListener("click", function () {
        alert("捕获 html");
    }, true);

    document.body.addEventListener("click", function () {
        alert("捕获 body");
    }, true);

</script>
</html>
```

当某事件发生于某元素，事件进入捕获阶段，也就是这些事件会先从页面最顶级的标签到发生了事件的标签依次进行通知。事件捕获阶段结束之后，事件通知系统进入事件冒泡阶段，也就是某元素身上发生的某事件会从自身到自己的父亲，再到祖父，依次通知一遍。事件捕获阶段先于时间冒泡阶段发生。

### 监听事件

@click是Vue中监听事件的写法，事实上这是一种简化的写法，完整的写法是v-on:click=methodName 或@click=handler。methodName是跟Vue中声明的方法，handler表示可以直接写JavaScript.  事件处理的值可以是:

1. **内联事件处理器**：事件被触发时执行的内联 JavaScript 语句 (与 `onclick` 类似)。
2. **方法事件处理器**：一个指向组件上定义的方法的属性名或是路径。

有的时候我们希望在内联处理器上访问原生DOM事件，你可以向该处理器传入一个特殊的$event变量，或者使用箭头函数。如下方实例所示:

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
        <!-- 使用特殊的 $event 变量 -->
        <button @click="warn('Form cannot be submitted yet.', $event)">
            Submit
        </button>
         <!-- 使用内联箭头函数 -->
         <button @click="(event) => warn('Form cannot be submitted yet.', event)">
            Submit
        </button>
    </div>
</body>
<script src="https://cdn.jsdelivr.net/npm/vue@2/dist/vue.js"></script>   
<script>
  var vue  = new Vue({
        el: '#app',
        methods:{
            warn:function(message, event){
                 // 这里可以访问原生事件
             if (event) {
                 event.preventDefault();
             }
 			   alert(message);
            }
        }    
      });

</script>
</html>
```

上面我们提到了事件冒泡与事件捕获，那如果我们想阻止冒泡与事件捕获呢，顺带阻止标签的默认行为呢(a标签代表跳转链接, 如果我们想点击a标签的时候，不跳转呢？ 我们就可以监听标签a的点击事件，取消其默认行为，也就是event.preventDefault()方法)，我们可以在拿到事件对象之后，调用stopPropagation()方法，但在Vue中方法最好只有纯粹的数据逻辑，而不是去处理DOM事件细节。为了解决这个问题，Vue.js 为v-on提供了事件修饰符，修饰符是由点开头的指令后缀来表示的:

- .stop  阻止单击事件继续转播
- .prevent 提交事件不再重载页面
- .capture 添加事件监听器时使用事件捕获模式

写法如下图所示: 

```html
<!-- 阻止单击事件继续传播 -->
<a v-on:click.stop="doThis"></a>

<!-- 提交事件不再重载页面 -->
<form v-on:submit.prevent="onSubmit"></form>
```

### 模块化

### 模块化的好处

在不断变大的代码库中，可扩缩性、可读性和整体代码质量通常会随着时间的推移而降低。这是因为代码库在不断变大，而其维护者未采取积极措施来保持易于维护的结构。模块化是一种行之有效的代码库构建方法，可帮助改善可维护性并避免此类问题。那什么是模块化? 

> 模块化是将代码库组织为多个松散耦合的独立部分的做法。每个部分都是一个模块。每个模块都是独立的，并且都有明确的用途。通过将问题划分为更小、更易于解决的子问题，您可以降低设计和维护大型系统的复杂性。

![模块化](http://tvax1.sinaimg.cn/large/006e5UvNgy1h7vgsz204nj30o20ck3zd.jpg)

示例多模块代码库的依赖关系图。

模块化具有众多优势，但核心都是提高代码库的可维护性和整体质量。下表总结了模块化的主要优势。

| 优势           | 摘要                                                         |
| :------------- | :----------------------------------------------------------- |
| 可重用性       | 模块化可支持在同一基础上共享代码并构建多个应用。模块实际上就是构建块。应用应为其各项功能的总和，而这些功能按单独的模块进行划分。特定模块提供的功能不一定会在特定应用中启用。例如，`:feature:news` 可以是完整版本变种和 Wear 应用的一部分，但不是演示版本变种的一部分。 |
| 严格控制可见性 | 借助模块，您可以轻松控制向代码库的其他部分公开哪些内容。您可以将除公共接口以外的所有内容标记为 `internal` 或 `private`，以防止在模块外部使用这些内容。 |
| 自定义分发     | Play Feature Delivery使用了 app bundle 的多种高级功能，让您可以按条件或按需分发应用的某些功能。 |

上面内容摘录与google开发者文档中对模块化的讨论，链接如下: https://developer.android.com/topic/modularization?hl=zh-cn#common-pitfalls

### JavaScript的模块化发展历史

但JavaScript长期没有语言级别的模块化，这也是我在学习JavaScript想到的有个问题，如果我想重用方法该怎么办，没有语言级别的模块化的可能原因在于的脚本又小又简单，所以没必要将其模块化。但随着网页越来越复杂，模块化变的越来越有必要，语言级别没有，JavaScript社区就自己引入，这里列举一些JavaScript社区推出的模块化系统:

- AMD 最古老的模块系统之一 , 最初由require.js库实现
- CommonJS: 为 Node.js 服务器创建的模块系统。
- UMD: 另外一个模块系统，建议作为通用的模块系统，它与 AMD 和 CommonJS 都兼容。

现在，它们都在慢慢的成为历史的一部分，但我们仍然可以在旧的脚本中找到它们。语言级的模块系统在 2015 年的时候出现在了标准（ES6）中，此后逐渐发展，现在已经得到了所有主流浏览器和 Node.js 的支持。因此，我们将从现在开始学习现代 JavaScript 模块（module）。

### JavaScript 模块化简介

一个模块（module）就是一个文件。一个脚本就是一个模块。就这么简单。

模块可以相互加载，并可以使用特殊的指令 `export` 和 `import` 来交换功能，从另一个模块调用一个模块的函数：

- `export` 关键字标记了可以从当前模块外部访问的变量和函数。
- `import` 关键字允许从其他模块导入功能。

例如，我们有一个 `sayHi.js` 文件导出了一个函数：

```javascript
export function sayHi(user) {
  alert(`Hello, ${user}!`);
}
```

……然后另一个文件可能导入并使用了这个函数：

```javascript
import { sayHi } from './sayHi.js';

alert(sayHi); // function...
sayHi('John'); // Hello, John!
```

`import` 指令通过相对于当前文件的路径 `./sayHi.js` 加载模块，并将导入的函数 `sayHi` 分配（assign）给相应的变量。

让我们在浏览器中运行一下这个示例。

由于模块支持特殊的关键字和功能，因此我们必须通过使用 `<script type="module">` 特性（attribute）来告诉浏览器，此脚本应该被当作模块（module）来对待。

```javascript
<!doctype html>
<script type="module">
  import {sayHi} from './say.js';
  document.body.innerHTML = sayHi('John');
</script>
```

## 表单的输入与绑定

在html中表单被用来收集用户信息，html表单是网页的一个区域，会有对应的输入框、下拉框，点击提交，将表单的信息传递给WEB服务器。在相识篇，我们用v-model来和Vue实例中的data元素建立双向绑定，还记得那个例子吗？ 我们这里再来回忆一下:

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
    el:'#app', // 表示id为app的元素将会被Vue接管
    data:{
       'message': 'Hello Vue.js' // 设置{{message}}的初始值 
    }
 })
</script>
</body>
</html>
```

我们通过v-model指令就将data中的message 和  html中的插值表达式{{message}} 关联了起来，当我们在输入框中进行输入，这个值会首先到达我们的Vue实例中，然后自动刷新标签p插值表达式中，所谓双向绑定是也。看起来很神奇是吗？ 但v-model的本质不过是语法糖，本质上仍然是监听用户的输入事件以及更新数据，并对一些极端场景进行一些特殊处理。我们可以用v-model指令在表单input、textarea、select元素上创建双向绑定，它会根据控件类型自动选取正确的方法来更新元素，但需要注意v-model会忽略所有表单元素value、checked、selected的初始值，而总是会将Vue实例的数据作为数据来源。所以我们应该通过JavaScript在组件的data选项中声明初始值。 

另外值得注意的是Vue 3的官方文档比Vue 2的官方文档要直观一些，我们来看同一章节，Vue 2是如下介绍的: 

![Vue2 的表单输入与绑定](http://tva3.sinaimg.cn/large/006e5UvNgy1h7vh8apt5gj30sl0md0vi.jpg)

Vue 3是如是介绍的:

![Vue3的表单输入与绑定](http://tvax3.sinaimg.cn/large/006e5UvNgy1h7vh984j6lj30o40lrac5.jpg)

我个人觉得Vue 3的文档倒是更为自然，从事件监听麻烦引出v-model指令。值得注意也是，我们来看下两者对值得注意的描述，首先是Vue2:

> 对于需要使输入法(如中文、日文、韩文等) 的语言，你会发现 `v-model` 不会在输入法组合文字过程中得到更新。如果你也想处理这个过程，请使用 `input` 事件。

老实说，上面这段话让我看的一脸懵，什么是组合文字，我们来看下Vue 3的文档是怎么说明的:

> 对于需要使用 IME的语言 (中文，日文和韩文等)，你会发现 `v-model` 不会在 IME 输入还在拼字阶段时触发更新。如果你的确想在拼字阶段也触发更新，请直接使用自己的 `input` 事件监听器和 `value` 绑定而不要使用 `v-model`。

拼字阶段，我一下子就听懂了。上面我们演示了单个输入框用v-model实现双向绑定，在表单中我们常用的元素还有多行文本、复选框、单选按钮、选择器。下面是示例: 

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=
    , initial-scale=1.0">
    <title>Document</title>
</head>
<body>
<div id = "app">
    <span>多行文本:</span>
    <p style="white-space: pre-line;">{{ message }}</p>
    <textarea v-model = "message" placeholder="差值表达式无效,textarea不支持插值表达式,可以用v-model来实现曲线救国"></textarea>
    <textarea v-model="message" placeholder="这里是多行文本"></textarea>
    单一的复选框，绑定布尔类型值：
    <input type="checkbox" id="checkbox" v-model="checked" />
    <label for="checkbox">{{ checked }}</label>
    多个复选框绑定到同一个数组或集合的值：
    <input type="checkbox" id="jack" value="Jack" v-model="checkedNames">
    <label for="jack">Jack</label>
    <input type="checkbox" id="john" value="John" v-model="checkedNames">
    <label for="john">John</label>
    <input type="checkbox" id="mike" value="Mike" v-model="checkedNames">
    <label for="mike">Mike</label>
    <br>
    <span>Checked names: {{ checkedNames }}</span>
    下面是多个单行文本绑定对象:
    <input v-model="student.age" placeholder="请输入年龄">
    <input v-model="student.name" placeholder="请输入姓名">
    <p>Message is: {{ student }}</p>
    下面是单选按钮:
    <div>Picked: {{ picked }}</div>

    <input type="radio" id="one" value="One" v-model="picked" />
    <label for="one">One</label>

    <input type="radio" id="two" value="Two" v-model="picked" />
    <label for="two">Two</label>
    下面是下拉框单选的时候绑定到一个值:
    <div>Selected: {{ selected }}</div>
    <select v-model="selected">
        <option disabled value="">Please select one</option>
        <option>A</option>
        <option>B</option>
        <option>C</option>
    </select>
     下面是下拉框多选的时候绑定到一个数组:
     <select v-model="mutiSelected" multiple style="width: 50px;">
        <option>A</option>
        <option>B</option>
        <option>C</option>
      </select>
      <br>
    <span>Selected: {{ mutiSelected }}</span>
</div>
</body>
<script src = "https://cdn.jsdelivr.net/npm/vue@2/dist/vue.js"></script>
<script>
    var vue = new Vue({
        el:'#app',
        data:{
            message: 'hello world',
            checked: true , 
            checkedNames:[],
            student:{
                age:20,
                name:'张三'
            },
            picked:'默认值',
            selected: '' ,
            mutiSelected: []
        }
    })
</script>
</html>
```

这里要说一下单选下拉框，值得注意的是下拉框选中的哪个选项是和select标签中的option标签的value值做匹配，如果v-model绑定对象的初始值和任意一个option标签的value值不匹配，那么下拉框将会渲染成一个“未选择”的状态，在ios上，这将导致用户无法选择下拉框的选项，因为ios在这种情况下不会触发一个change事件，因此，推荐为下拉框提供一个默认匹配的空值禁用选项。

## 侦听器

有些时候我们希望当data中绑定的属性发生变化的时候通知到我们，这也就是侦听器:

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
  <div id="watch-example">
        <p>
          Ask a yes/no question:
          <input v-model="question">
        </p>
        <p>{{ answer }}</p>
   </div>
</body>
<script src = "https://cdn.jsdelivr.net/npm/vue@2/dist/vue.js"></script>
<script>
    var watchExampleVM = new Vue({
  el: '#watch-example',
  data: {
    question: '',
  },
  watch: {
    // 如果 `question` 发生改变，这个函数就会运行
    question: function (newQuestion, oldQuestion) {
        console.log("old question",oldQuestion);
        console.log("new question",newQuestion);
    
    }
  },
})
</script>
</html>
```

再这个例子中我们用v-model指令再data中的question属性和input标签的输入值建立了双向绑定，也就是当我们再input标签，也就是输入框中给输入值，会自动到达data的question属性，watch被称之为侦听器，我们在其中声明了我们要监听变化的属性，当输入框的值与data的初始值不同时，会回调qusetion对应的函数。

## 组件

前面的文章我们已经有意无意的提到了组件这个概念，但是没有细讲，这里我们来细致的介绍一下组件，这次让我们换一种方式，我们先看例子，再来介绍概念。

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
    <div id="components-demo">
        <button-counter></button-counter> 
        <button-counter></button-counter>
        <button-counter></button-counter>
        <button-counter></button-counter>
      </div>
</body>
<script src = "https://cdn.jsdelivr.net/npm/vue@2/dist/vue.js"></script>
<script>
  // button-counter 是组件名称 注册组件
  Vue.component('button-counter', {
  data: function () {
    return {
      count: 0
    }
  },
  // 模板属性
  template: '<button v-on:click="count++">You clicked me {{ count }} times.</button>'
})
// 声明一个Vue实例,接管id=components-demo的html元素及其子元素
new Vue({ el: '#components-demo' })
</script>
</html>
```

浏览器的结果如下:

![实际效果](http://tvax4.sinaimg.cn/large/006e5UvNgy1h8287lnfi2j30ne035t8r.jpg)

我们来分析一下这个实际效果，button-counter标签是我们组件的名字，html标准并无这个标签，button-counter标签的效果恰好是template属性中的声明内容，也就是Vue帮我们做了翻译，也就是说我们在html文件中将写<组件名>，Vue会自动的寻找该组件中声明的template属性的值替换<组件名>，能找到的前提是你已经通过Vue.component进行注册，代码复用的真义就在于此。

这样我们的一个页面就被分割成了由若干个组件来构成，每个组件又可以引用其他组件，为了能在模板中使用，这些组件必须先注册以便Vue能够识别，Vue.component这种注册方式我们称之为全局注册，第一个参数是组件名，Vue推荐遵循W3C规范中的自定义组件名(字母全小写且必须包含一个连字符)，这回这会帮助你避免和当前以及未来的 HTML 元素相冲突。第二个参数是普通Vue实例中的属性，你可以在里面定义compute属性、侦听器、methods方法。通过全局注册的组件可以用在其被注册之后的任何(通过new Vue)新创建的Vue根实例，也包含其组件树中的所有子组件的模板中。

## 总结一下

有同学看到这里可能就会说了，目前只有一个按钮这样简单的看起来是这样，一旦我想构件的元素复杂，这个template还是会很难看，而且没有高亮，语法错误都得在运行时才能看出来，除此之外，这个组件可以像上面讲的JavaScript函数一样import和export？ 在Vue中第一个答案的问题叫单文件组件, 单文件组件的意思时将上面我们注册的组件单独放到一个文件中，文件扩展名(后缀名)为.vue，单文件组件的英文为 **single-file components**。第二个问题的答案是对，Vue组件可以被import、export。我们完全没可能在一篇文章中将Vue单文件组件的知识点介绍完毕，这会让这篇文章的篇幅过大。另一个在这里不介绍的理由是本篇基本参照Vue 的官方文档来写的，在讲到单文件组件、组件注册、props让我感到有点难懂，也在想用更通俗易懂的方式来介绍这些概念。

下面我们来捋一捋Vue目前带给我们的东西:

Vue 实例在和页面元素通过v-model建立双向绑定关系之后，用户操纵产生的值会自动到达绑定的属性。到达属性之后，如果你需要对属性做一点简单的计算，可以通过compute属性来做，最终通过插值表达式将计算属性取出，这是一个丝滑的过程，如果你想在数据发生变化之后被通知，再触发一些行为，那么可以选择watch函数。如果你想在元素上监听事件，在Vue中可以使用v-on指令，如果想根据值来动态的渲染元素，也就是说循环创建html元素，同时满足条件才出现对应的元素，Vue中有v-for、v-if指令来供我们使用。如果你想在页面加载的时候或者说建立绑定关系的时候，初始化一些值，我们可以用created函数，这在Vue中被称为**生命周期钩子**的函数。

## 参考资料

- 千古一号 https://github.com/qianguyihao/Web/blob/master/04-JavaScript%E5%9F%BA%E7%A1%80/35-%E4%BA%8B%E4%BB%B6%E7%AE%80%E4%BB%8B.md
- preventDefault与stopPropagation的作用  https://blog.csdn.net/u010565037/article/details/102640003
- 现代 JavaScript 教程  https://zh.javascript.info/
- Android 应用模块化指南  https://developer.android.com/topic/modularization?hl=zh-cn