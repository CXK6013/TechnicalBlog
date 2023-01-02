# Vue学习笔记(四) 久处不厌

> 本来打算写JavaFX的，但是比较令人悲伤的是，那一篇博客没有同步到GitHub上，还在公司的电脑上。这篇还在。



[TOC]

## 前言

学习本篇需要有一定的Vue、Node、HTML、JavaScript、CSS基础，需要一些预置概念。

《Vue学习笔记(一) 初遇篇》介绍Vue的源起以及Vue的基本示例, 《Vue学习笔记(二) 相识篇》对《Vue学习笔记(一) 初遇篇》进行重构，重新组织了《Vue学习笔记(一)初遇篇》的内容: 随着网页越来越复杂，开发者们在追求越来越快的开发速度和性能，这也就是现代框架的缘起，相对于JQuery这样直接操纵DOM结点的库，导致页面重绘影响页面的性能。现代前端框架推出了虚拟DOM不再让开发者直接操纵DOM结点，同时引入了响应式、模板语法进一步加快开发前端页面的速度，但是仅仅如此还不够，现代前端框架开始追求可复用性，这也就是组件，JavaScript、CSS、HTML的复合体，让我们可以复用逻辑、样式、功能。

现在我们来回忆一下《Vue学习笔记(三) 》，这篇文章里面我们讲了事件系统、监听事件、模块化、模块化的好处。以及JavaScript的模块化发展历史，JavaScript的如何使用模块, 表单的输入与绑定、组件的简单使用。但对于Vue的核心概念组件来说我们并未介绍多少，我们只是让她小小的露了个面，演示了一个组件如何声明，如何使用。但这种使用组件的方式并不讨喜，原因在于这种方式的组件的内容: HTML、CSS、JavaScript都在字符串里面。比较适应的是逻辑、样式都不复杂的场景，一旦这些内容稍微庞大一点点，庞大的字符串就会让我们头晕脑胀。

对此的解决方案是单文件组件，将文件放在一个文件中，后缀为.vue。但我们想接着开发前端应用还不大够，你会发现后面单文件应用基本上都会选择用Vue CLI来构建, 那Vue CLI 是什么？ Vue我们是认识的，那CLI呢？首先CLI是Commend-Line Interface的缩写, Commend-Line Interface 命令行界面，用户通过键盘输入指令，计算机接收指令后，予以执行。上面是维基百科的定义，下面我们来看下Vue CLI是如何介绍自身的:

> Vue CLI 是一个基于 Vue.js 进行快速开发的完整系统，提供：
>
> - 通过 `@vue/cli` 实现的交互式的项目脚手架。
> - 通过 `@vue/cli` + `@vue/cli-service-global` 实现的零配置原型开发。
> - 一个运行时依赖 (@vue/cli-service)，该依赖：
>   - 可升级；
>   - 基于 webpack 构建，并带有合理的默认配置；
>   - 可以通过项目内的配置文件进行配置；
>   - 可以通过插件进行扩展。
> - 一个丰富的官方插件集合，集成了前端生态中最好的工具。
> - 一套完全图形化的创建和管理 Vue.js 项目的用户界面。
>
> Vue CLI 致力于将 Vue 生态中的工具基础标准化。它确保了各种构建工具能够基于智能的默认配置即可平稳衔接，这样你可以专注在撰写应用上，而不必花好几天去纠结配置的问题。与此同时，它也为每个工具提供了调整配置的灵活性，无需 eject。

我们再来回忆一下《让我们来来前端工程化》讨论了什么:

> 随着浏览器页面的越来越复杂，许多问题变浮现了出来，也许许多事情都是这样，当问题的规模比较小的时候，通常比较容易解决。一旦问题的规模开始增长，问题的解决难度也随着上涨。页面越来越复杂，那就意味着代码量也在上升，早期的页面简单，JavaScript的设计者没有为这门语言准备模块化，虽然语言本身没有自带，JavaScript社区就自发的为JavaScript设计模块，比较为人所熟知的模块化系统是:
>
> - AMD —— 最古老的模块系统之一，最初由require.js库来实现。
> - CommonJS —— 为Node.js服务器创建的模块系统。
> - UMD —— 另外一个模块系统，建议作为通用的模块系统，它与AMD和CommonJS都兼容。
>
> 现在 ，它们都在慢慢成为历史的一部分，但你仍然可以在旧有的项目找到它们，语言级别的模块系统在2015年的时候出现在了标准(ES6)中，此后逐渐发展，现在已经得到了所有主流浏览器和Node.js的支持。 模块化解决了在JavaScript如何复用代码的问题，从一定程度上来说降低了代码量和代码复杂度。但这仅仅解决了一个问题，另一个值得注意的问题是，在使用了npm的情况下，最终的页面该如何交付，开发页面的时候我们还有CSS、SASS、JPG等各种资源，我们该如何组织这些文件，WebPack是这个问题的一个答案，WebPack会分析JavaScript之间的依赖关系，将我们项目的文件进行分类，放到对应的文件夹里。WebPack这是一个打包工具，它提供的能力还有模块捆绑，所谓模块捆绑指的是将页面所需的模块捆绑后才能一个大文件或几个文件以减少请求数量。
>
> 除此之外，还需要面对的问题是适配低版本浏览器的问题(IE浏览器走开)，有些JavaScript语法好用但是低版本的不支持，但能不能我们只写一套，由工具帮我们自动产生低版本浏览器的代码呢？ 这也就是Babel。

为了快速高质量的构建项目，我们引入了一个又一个工具，但是该如何快速的搭建起一个项目的模板呢？ 我们的愿景是这些的通过一些命令行命令就能将webpack、node.js、babel都引入到项目中，并且这些里面已经写了一些默认配置，有一个简单的示例。我只需要写我需要的页面就行了。这也就是Vue CLI。

再读一遍Vue自我介绍的话:

> Vue CLI 致力于将 Vue 生态中的工具基础标准化。它确保了各种构建工具能够基于智能的默认配置即可平稳衔接，这样你可以专注在撰写应用上，而不必花好几天去纠结配置的问题。与此同时，它也为每个工具提供了调整配置的灵活性，无需 eject。

## 开启第一个单文件项目吧

本项目首先需要一个node环境，要求Node的版本在14.18.1以上，虽然Vue 的最新大版本已经到了3.2了，但我们这里的Vue大版本选择还是2.5.2。后面会专门再研究3和2的区别。我们本次借助Vue CLI来构建项目，我们借助VS Code来编写代码，我的VS Code安装的插件有:

- vue-helper: Vue代码提示和组件跳转。

第一步打开VS Code, 点击终端, 在终端中输入命令: 

```javascript
# 这个命令代表的意思是全局安装vue cli
npm install --global @vue/cli
```

然后在终端里面输入:

```vue
# Vue 脚手架就会帮我们创建一个叫my-project的项目
# 在创建过程中会给你一些选择，比如输入项目描述啊之类的
vue init webpack my-project
```

## 项目结构

最终的项目结构如下图所示:

![Vue项目结构](http://tvax4.sinaimg.cn/large/006e5UvNgy1h9mzomsi6bj319d0f0q3j.jpg)

- .eslintrc.js：代码检测工具, 统一代码风格
- .babellrc:  可以在JavaScript使用新特性，并在生产环境将其转换为可以跨浏览器运行的旧代码，也可以在里面配置babel插件。
- package.json: npm的配置文件，在这里我们可以引用第三方JavaScript库。
- .postcssrc.js: 可以使用CSS中使用新特性，并在生产环境将其转换为可以跨浏览器运行的代码。
- src: 这个是Vue应用的核心目录
  - main.js：这是应用的入口文件。目前它会初始化应用并且指定将应用挂在到index.html文件中的哪个HTML元素上。通常还会做一些注册全局组件或者添加额外的Vue库的操作。
  - App.vue: 这是Vue应用的根结点组件，目前里面会有一个示例组件。
  - assets: 这个目录用来存放像CSS、图片这种静态资源。但是他们放在代码目录下面，所以可以使用webpack来操作和处理。

 ## 运行Vue项目

之前我用Vue写的页面都是在html里面，我们直接用浏览器打开即可。那现在的项目我们该如何看页面效果呢？我们在终端里输入npm run dev 即可, 像下面的图一样：

![如何运行项目](http://tvax3.sinaimg.cn/large/006e5UvNgy1h9n2mzxggsj31300o3djv.jpg)

本质上执行的还是

```javascript
webpack-dev-server --inline --progress --config build/webpack.dev.conf.js
```

然后在控制台就可以看到:

![项目运行之后的结果](http://tvax1.sinaimg.cn/large/006e5UvNgy1h9n2rm2gl7j30r604aweg.jpg)

然后我们在浏览器访问这个地址：

![浏览器效果图](http://tvax3.sinaimg.cn/large/006e5UvNgy1h9n2tdblh8j30xs0h2wf2.jpg)

## 开启第一个组件

先来复习一下什么是组件，在Vue 3的中文文档倒是没给出定义, 只是简单论述了一下: 

> 组件允许我们将 UI 划分为独立的、可重用的部分，并且可以对每个部分进行单独的思考。

好像说了什么，又好像什么都没说，既然Vue的中文文档没给出组件的定义(也可能是我没看到)，那我们这里就尝试推敲出一个定义,  在网页中我们通常并不止使用HTML，我们用html搭建基本的内容骨架，用CSS对内容骨架进行美化，用JavaScript为实现业务逻辑和页面控制。当我们想要复用的时候通常是三者都想复用，而不仅仅是复用其中一项，基于此我们尝试给出组件的定义就是:

> 一种包含内容(HTML)、样式(CSS)、逻辑(JavaScript)的组合体。

我们来直观感受一下组件, 在我们刚才建立的项目中就有一个组件，在src/components下面，它叫HelloWorld.vue, 里面的内容如下所示:

```vue
<template>
  <div class="hello">
    <h1>{{ msg }}</h1>
    <h2>Essential Links</h2>
    <ul>
      <li>
        <a
          href="https://vuejs.org"
          target="_blank"
        >
          Core Docs
        </a>
      </li>
      <li>
        <a
          href="https://forum.vuejs.org"
          target="_blank"
        >
          Forum
        </a>
      </li>
      <li>
        <a
          href="https://chat.vuejs.org"
          target="_blank"
        >
          Community Chat
        </a>
      </li>
      <li>
        <a
          href="https://twitter.com/vuejs"
          target="_blank"
        >
          Twitter
        </a>
      </li>
      <br>
      <li>
        <a
          href="http://vuejs-templates.github.io/webpack/"
          target="_blank"
        >
          Docs for This Template
        </a>
      </li>
    </ul>
    <h2>Ecosystem</h2>
    <ul>
      <li>
        <a
          href="http://router.vuejs.org/"
          target="_blank"
        >
          vue-router
        </a>
      </li>
      <li>
        <a
          href="http://vuex.vuejs.org/"
          target="_blank"
        >
          vuex
        </a>
      </li>
      <li>
        <a
          href="http://vue-loader.vuejs.org/"
          target="_blank"
        >
          vue-loader
        </a>
      </li>
      <li>
        <a
          href="https://github.com/vuejs/awesome-vue"
          target="_blank"
        >
          awesome-vue
        </a>
      </li>
    </ul>
  </div>
</template>

<script>
export default {
  name: 'HelloWorld',
  data () {
    return {
      msg: 'Welcome to Your Vue.js App'
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
h1, h2 {
  font-weight: normal;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
}
</style>

```

我们来简单的看一下这个组件的组成: 

- template(html内容)
- script(JavaScript行为): 在这个组件中, 我们只导出了该组件,名字是helloWorld。注意到之前我们写的示例data是一个对象，现在data是一个函数而且必须是一个函数，这样各个组件就能够互不影响。
- style: 样式

那我们该如何使用这个组件呢，首先我们App.vue中引入该组件, 然后再注册：

```vue
import HelloWorld from './components/HelloWorld.vue'

export default {
  name: 'App',
  components: {
    HelloWorld
  }
}
```

然后我们在App.vue中直接将这个组件名当标签一样使用即可，像下面这样:

```vue
<template>
  <div id="app">
    <img src="./assets/logo.png">
    <router-view/>
    <hello-world/>
    <hello-world/>
  </div>
</template>

<script>
import HelloWorld from './components/HelloWorld.vue'

export default {
  name: 'App',
  components: {
    HelloWorld
  }
}
</script>

<style>
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
```

下面是浏览器中的效果:

![映射关系](http://tvax2.sinaimg.cn/large/006e5UvNgy1h9n409iemdj317k0gm0w1.jpg)



就像很多其他的前端框架一样，组件是构建Vue应用中非常重要的一部分。组件可以把一个很大的应用程序拆分成独立创建和管理的不相关区块，然后彼此按需传递数据，这些小的代码块可以方便更容易的理解和测试。

Vue把模板、相关脚本和CSS一起整合放在.vue结尾的一个单文件中。这些文件最终会通过JavaScript(后面统称为JS)打包工具(如Webpack)处理，这意味着你可以使用构建工具来完成更多复杂的组件，比如Babel、TypeScript、SCSS。

## 待办表单组件

注意上面我们的一句话:

> 组件可以把一个很大的应用程序拆分成独立创建和管理的不相关区块，然后彼此按需传递数据，这些小的代码块可以更方便更容易理解和测试。

这其实在说的是, 我们将应用拆分为若干组件，那组件之间如何传递数据，我们将在下面的例子中演示，这个例子来自MDN Web docs，地址是:

> https://developer.mozilla.org/zh-CN/docs/Learn/Tools_and_testing/Client-side_JavaScript_frameworks/Vue_first_component

后面的介绍也多基于这个例子，只不过会加入一点我个人的理解。刚才那个组件来自Vue CLI，现在让我们自己创建一个组件，首先我们在src/components目录下建立一个ToDoItem.vue。然后再这个文件下建立三个标签: template、script、style。像下面一样: 

```vue
<template>
  
</template>

<script>
export default {

}
</script>

<style>

</style>
```

现在让我们来丰富这个组件的内容:

```vue
<template>
  <div>
     <input type="checkbox" id="todo-item" checked="false"/>
     <label for="todo-item"> 我的待办</label>
  </div>
</template>

<script>
export default {
  name: 'ToDoItem'
}
</script>

<style>

</style>
```

然后我们在App.vue中使用这个组件，要使用一个组件，是固定的步骤，第一步引入这个组件，第二步注册这个组件:

```vue
<template>
  <div id="app">
    <h1>To-Do List</h1>
  <ul>
    <li>
      <to-do-item></to-do-item>
    </li>
  </ul>
  </div>
</template>

<script>
import ToDoItem from './components/ToDoItem.vue'
export default {
  name: 'App',
  components: {
    ToDoItem
  }
}
</script>

<style>
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
```

效果如下:

![效果](http://tvax3.sinaimg.cn/large/006e5UvNgy1h9n54d95csj31dx05lwee.jpg)



但目前来说我们的组件还是有些呆板，他不大聪明，依照HTML规范，ID必须唯一，我们只能在页面上包含一次。但由于浏览器的宽容，你会发现包含两次也照样能渲染出来。除此之外，label标签的内容是固定的，这不是动态的。那如何让这个组件聪明起来呢，这也就是props

## props

在Vue中，注册props的方式有两种:

- 以字符串数组的方式列出props，数组中的每个实体对应一个prop名称。

```vue
Vue.component('my-component',{
	// 声明 props
	props:['message'],
	// 在模板中使用绑定语法来绑定到prop
template: '<div>{{message}}</div>'
}) 
```

- 将props定义为一个对象，每个key对应于prop名称。将props列为对象可以指定默认值，将props标记为required，执行基本的对象类型(特别是JavaScript基本类型)，并执行简单的prop校验。

> 注意：prop 验证只能在 development 模式下进行，所以你不能在生产环境中严格依赖它。此外，prop 验证函数在组件实例创建之前被调用，因此它们不能访问组件状态 (或其他 props)

```vue
Vue.component('my-component', {
  // 声明 props
  props: {
    message: {
      type: String,
      default: 'Hello'
    }
  },
  template: '<div>{{ message }}</div>'
})
```

对于针对 ToDoItem 组件，我们将使用对象注册法，我们在默认导出的对象{}对象上添加一个props对象，该props属性含有一个空对象。在这个对象里面我们添加两个key为label和done属性。label的值应该是一个带有两个属性的对象，第一个是required属性，如果它的值是true。那么这就相当于告诉Vue说，每个该组件的实例必须有一个label字段。如果没有，Vue提示会给出警告。第二个是添加一个type属性，如果它的值是String类型，这等于在告诉Vue，我们希望type属性的值是String类型。

现在转向done prop， 首先添加一个default属性，默认值是false，这意味着当没有给ToDoItem组件传递done prop，done prop的值会是false。default 不是必需的，我们只在非required里才需要default。然后添加一个type属性，值为Boolean，这将告诉Vue，我们希望这个Prop的值是JavaScript的Boolean类型。现在ToDoItem的script变成了下面这样：

```vue
<script>
export default {
  name: 'ToDoItem',
  props: {
    label: {required: true, type: String},
    done: {default: false, type: Boolean}
  }
}
</script>
```

现在组件声明了它所需要的值，我们就可以在template里面使用这些变量值，像下面这样:

```vue
<template>
  <div>
     <input type="checkbox" id="todo-item" checked="false"/>
     <label for="todo-item"> {{label}}</label>
  </div>
</template>
```

再次运行项目，你会发现浏览器的控制台会输出警告:

```vue
[Vue warn]: Missing required prop: "label"
found in
---> <ToDoItem> at src/components/ToDoItem.vue
       <App> at src/App.vue
         <Root>
```

这是因为我们在ToDoItem将label prop标记为必传，但是我们在使用这个组件的时候却没有传入。现在让我们来修复这个问题，在使用这个组件的实验传递进去:

```vue
<template>
  <div id="app">
    <h1>To-Do List</h1>
  <ul>
    <li>
      <to-do-item label="hello prop" done="true"></to-do-item>
    </li>
  </ul>
  </div>
</template>

<script>
import ToDoItem from './components/ToDoItem.vue'
export default {
  name: 'App',
  components: {
    ToDoItem
  }
}
</script>

<style>
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
```

实际效果:

![效果图](http://tvax3.sinaimg.cn/large/006e5UvNgy1h9nbs2af7aj30e002zmx0.jpg)



前面我们提到, html的标签的id不允许重复，如果你重复了，没问题的原因在于浏览器比较宽容。现在我们来尝试为label的for和input的id产生唯一值，在项目中我们用到了npm，这意味着我们可以大量使用第三方库，我们这里用来产生唯一id的库叫lodash，首先我们在终端里面尝试安装它:

```javascript
npm install --save lodash.uniqueid
```

 然后在script标签里面引入:

```vue
import uniqueId from 'lodash.uniqueid';
```

然后在data里面声明一个id，并将data的id和复选框的id、标签的for属性进行绑定，如下所示:

```vue
<template>
  <div>
     <input type="checkbox" :id="id" :checked="isDone"/>
     <label :for="id"> {{label}}</label>
  </div>
</template>

<script>
import uniqueId from 'lodash.uniqueid'
export default {
  name: 'ToDoItem',
  props: {
    label: {required: true, type: String},
    done: {default: false, type: Boolean}
  },
  data () {
    return {
      isDone: this.done,
      id: uniqueId('todo-')
    }
  }
}
</script>

<style>

</style>
```

然后打开浏览器观察源码为发现，label的for和复选框的id都产生了唯一键。

## 总结一下

在《让我们来来前端工程化》我们简单讨论了前端工程化，工程化的一个标志就是引入越来越多的工具，而Vue CLI则将这些工具一站式集成，通过一行命令我们就能轻易的集成这些工具，简单的搭建一个前端工程。组件可以把一个很大的应用程序拆分成独立创建和管理的不相关区块，然后彼此按需传递数据，这些小的代码块可以方便更容易的理解和测试。当我们将应用程序拆分，那随之伴随而来的一个问题就是组件之间该如何通信，这里我们通过一个小的待办组件，介绍了父子组件之间传递数据的方式，也就是props。

## 参考资料

- mdn web docs https://developer.mozilla.org/en-US/docs/Learn/Tools_and_testing/Client-side_JavaScript_frameworks/Vue_computed_properties
