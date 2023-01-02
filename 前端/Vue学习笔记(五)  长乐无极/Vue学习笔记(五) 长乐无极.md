# Vue学习笔记(五) 长乐无极

> 下一篇是长乐未央，其实是我记错了，将长乐未央、长毋相忘，记成了长乐无极，长乐未央。在汉代，长乐未央和长毋相忘，是两句常用的祝福语，意思是长久快乐，永远不要忘记对方。后面的主线就是通过一些例子来介绍Vue的一些常用概念。当前这个例子依然来自mdn web docs。加上了我自己的理解。

[TOC]

## 进化: 组件列表

书接上回(有种章回体的感觉了，哈哈)，上回我们构建了一个待办组件，并借助props来实现父子组件之间进行通信。父子组件简单的说就是假设一个组件，我们姑且称之为A，使用了另一个组件，我们姑且称之为B。那么我们称A为父组件，B为子组件，也就是父子组件。但是我们的待办通常不会只有一个，假设我们有10个待办，我们不会想将组件标签写10次，我们会想到循环，没错这就是我们前文中提到的v-for指令，首先我们在App.vue中声明一个data函数，然后在函数中我们返回的对象中包含一个数组。

```vue
data() {
    return {
      ToDoItems: [
        { label: 'JavaFX教程(一)', done: false },
        { label: '使用JavaFX打包一个平台包', done: true },
        { label: '写C语言教程', done: true },
        { label: 'SDL库', done: false }
      ]
};
```

目前好像没问题，但是我们并不想每次修改待办，都会将所有的待办都重新创建一遍，为了帮助Vue优化列表中元素的呈现，Vue需要在v-for创建出来的元素有唯一的键，这些键最好是字符串或数值，这里我们就可以选择使用第三方库来产生唯一的键， 也就是lodash包的uniqueid()方法来产生唯一键。首先让我们停止服务，然后再终端中输入以下命令：

```javascript
npm install --save lodash.uniqueid
# 如果报 找不到uniqueid
npm i --save-dev @types/lodash.uniqueid
```

然后在app.vue中引入这个这个方法, 

```javascript
import uniqueId from 'lodash.uniqueid';
```

然后我们的app.vue就变成了下面这样:

```vue
<template>
  <div id="app">
    <h1>To-Do List</h1>
  <ul>
    <li v-for="item in ToDoItems" :key="item.id">
      <to-do-item :label="item.label"  :done="item.done" ></to-do-item>
    </li>
  </ul>
  </div>
</template>

<script>
import ToDoItem from './components/ToDoItem.vue'
import uniqueId from 'lodash.uniqueid'
export default {
  name: 'App',
  components: {
    ToDoItem
  },
  data () {
    return {
      ToDoItems: [
        { id: uniqueId('todo-'), label: 'JavaFX教程(一)', done: false },
        { id: uniqueId('todo-'), label: '使用JavaFX打包一个平台包', done: true },
        { id: uniqueId('todo-'), label: '写C语言教程', done: true },
        { id: uniqueId('todo-'), label: 'SDL库', done: false }
      ]
    }
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

目前app的css不会发生变化，后面的代码示例就暂时不展示。我们现在已经为每一个待办产生了一个id，那待办组件的id就显的有点多余了，我们将待办组件的id声明为prop，同时移除掉lodash相关的代码, 现在待办组件的script就变成了下面这样:

```vue
<script>
export default {
  name: 'ToDoItem',
  props: {
    label: {required: true, type: String},
    done: {default: false, type: Boolean},
    id: {required: true, type: String}
  },
  data () {
    return {
      isDone: this.done
    }
  }
}
</script>
```

app.vue 中将id传递给待办组件:

```vue
<template>
  <div id="app">
    <h1>To-Do List</h1>
  <ul>
    <li v-for="item in ToDoItems" :key="item.id" >
      <to-do-item :label="item.label"  :done="item.done" :id="item.id" ></to-do-item>
    </li>
  </ul>
  </div>
</template>
```

## 进化: 待办表单

现在我们的程序仍然可交互性较差，待办的数据都被固定在程序里面，不能添加。为了让我们的程序交互性更强，让我们制作一个待办表单组件。固定的步骤，我们首先在src/components下面添加一个ToDoForm.vue，然后我们在template里面添加一个待办表单，表单里面包含label，input、button。如下所示:

```vue
<template>
  <form>
      <label for = "new-todo-input">
        还有哪些没做
      </label>
      <input type="text" id="new-todo-input" name = "new-todo" autocomplete="off"/>
      <button type="submit"> 添加待办</button>
  </form>
</template>

<script>
export default {
    name: 'ToDoForm'
}
</script>
```

然后我们在App组件里面，引入并注册这个组件。

```javascript
import ToDoForm from './components/ToDoForm.vue'
```

```vue
components: {
    ToDoItem,
    ToDoForm
},
```

现在template变成了下面这样:

```vue
<template>
  <div id="app">
    <h1>To-Do List</h1>
    <to-do-form></to-do-form>
  <ul>
    <li v-for="item in ToDoItems" :key="item.id" >
      <to-do-item :label="item.label"  :done="item.done" :id="item.id" ></to-do-item>
    </li>
  </ul>
  </div>
</template>
```

页面效果如下:

![添加待办表单](http://tvax3.sinaimg.cn/large/006e5UvNgy1h9o22womdgj30ga064glq.jpg)

对于表单的按钮来说点击submit的默认行为是将表单里面的数据发送回服务器，现在我们还没有服务器。我们只能来模拟，我们在submit事件上绑定一个方法，该方法将输入的待办加入到待办列表里。这里我们回忆一下vue实例的几个属性:

- data 属性 用于建立双向绑定，单文件组件中必须声明为一个函数
- compute 计算属性，用于计算data属性。
- methods  方法属性，在这里面我们声明事件发生时触发的方法。

现在我们需要监听事件，那就是需要methods属性, 首先我们在表单里面声明我们关注的事件和事件发生后触发的方法

```vue
 <form @submit="onSubmit"></form>
<script>
export default {
  name: 'ToDoForm',
  methods: {
    onSubmit () {

    }
  }
}
</script>
```

但是上面我们提到表单里面的submit事件会默认将表单的数据提交到服务器，这目前不是我们所需要的，所以我们需要阻止事件的默认行为，在原生JavaScript中我们使用的是event.preventDefault来阻止，在Vue中也对此做了包装，在Vue中这种语法我们称之为 **event modifiers** (事件修饰符)，修饰符被添加到事件的末尾，和事件之间通过点来进行连接。像下面这样:

```vue
<template>
  <form @submit.prevent="onSubmit">
      <label for = "new-todo-input">
        还有哪些没做
      </label>
      <input type="text" id="new-todo-input" name = "new-todo" autocomplete="off"/>
      <button type="submit"> 添加待办</button>
  </form>
</template>

<script>
export default {
  name: 'ToDoForm',
  methods: {
    onSubmit () {

    }
  }
}
</script>

```

下面是修饰符列表：

- `.stop`：停止传播事件。等效于常规 JavaScript 事件中的 Event.stopPropagation()。
- `.prevent`：阻止事件的默认行为。等效于 Event.preventDefault()
- `.self`：仅当事件是从该确切元素分派时触发处理程序。
- `{.key}`：仅通过指定键触发事件处理程序。 多词键只需转换为 kebab 大小写（例如 `page-down`）。
- `.native`：监听组件根（最外层的包装）元素上的原生事件。
- `.once`：监听事件，直到它被触发一次，然后不再触发。
- `.left`：仅通过鼠标左键事件触发处理程序。
- `.right`：仅通过鼠标右键事件触发处理程序。
- `.middle`：仅通过鼠标中键事件触发处理程序。
- `.passive`：等效于在 vanilla JavaScript 中使用addEventListene 创建事件监听器时传入 `{ passive: true }` 参数。4

有了监听事件我们就可以进行下一步，接收表单的值并将其同步到待办列表上，我们可以用双向绑定来自然的完成这个完成，也就是v-model。现在我们的表单标签如下所示:

```html
<form @submit.prevent="onSubmit">
```

script标签中的值如下所示:

```vue
export default {
  name: 'ToDoForm',
  data () {
    return {
      label: ''
    }
  },
  methods: {
    onSubmit () {
      console.log(this.label)
    }
  }
}
```

现在你在待办表单输入值，点击提交，控制台就会输出你输入的值。但我们的监听事件目前仍然存在一些缺陷，对于空格这样的输入我们一惯是不关心的，在Vue中我们可以通过trim来去除前后的空格。v-model当前是通过input事件来更新data变量中的值，这意味着我们每次点击按键都会同步数据，但我们希望的是在点击提交按钮或不再输入之后再同步值，所以我们需要修改v-model所关心的事件，也就是change事件。我们通过lazy修饰符可以完成这个改变，现在我们的输入框变成了下面这样:

```html
  <input type="text" id="new-todo-input" name = "new-todo" autocomplete="off" v-model.trim.lazy="label"/>
```

下一个问题是当添加待办这个按钮被点击，也就是说一个待办被提交，我们该怎么通知待办列表组件。在Vue的世界里，这个答案叫自定义事件。在我们的程序中待办列表的组件是App组件来传递给它的，所以我们应该是在App组件中监听这个消息。要完成这一件事首先我们要改变上面onSubmit事件中的内容，在Vue中发出自定义事件的方法是emit，这是Vue内置的方法，我们在调用的时候需要在方法名上加上$ , 标准的语法如下所示:

```vue
#eventName 事件名称 args是附加参数 都会传递给监听此事件的方法
vm.$emit( eventName, […args] )
```

```vue
this.$emit('todo-added', this.label);
```

值得注意的是事件处理程序区分大小写并且不能包含空格，Vue在进行转换的时候会将事件名转为小写，所以在Vue里面无法监听以大写字母命名的事件。子组件发出此事件之后，我们在父组件使用的地方监听此事件：

```vue
<template>
  <div id="app">
    <h1>To-Do List</h1>
    <to-do-form @todo-added="addToDo"></to-do-form>
  <ul>
    <li v-for="item in ToDoItems" :key="item.id" >
      <to-do-item :label="item.label"  :done="item.done" :id="item.id" ></to-do-item>
    </li>
  </ul>
  </div>
</template>
```

然后再addToDo里面将待办表单组件传递过来的数据更新到页面,像下面这样:

```vue
<script>
import ToDoItem from './components/ToDoItem.vue'
import uniqueId from 'lodash.uniqueid'
import ToDoForm from './components/ToDoForm.vue'
export default {
  name: 'App',
  components: {
    ToDoItem,
    ToDoForm
  },
  data () {
    return {
      ToDoItems: [
        { id: uniqueId('todo-'), label: 'JavaFX教程(一)', done: false },
        { id: uniqueId('todo-'), label: '使用JavaFX打包一个平台包', done: true },
        { id: uniqueId('todo-'), label: '写C语言教程', done: true },
        { id: uniqueId('todo-'), label: 'SDL库', done: false }
      ]
    }
  },
  methods: {
    addToDo (toDoLabel) {
      this.ToDoItems.push({id: uniqueId('todo-'), label: toDoLabel, done: false})
    }
  }
}
</script>
```

现在我们的待办就有了交互性，我们在输入框中输入，点击提交之后，页面就会出现新的待办。但目前我们的程序仍然不太完善，我们在输入完成，点击提交，输入框的内容没被清空。除此之外，对于空格我们虽然做了去除，但是我们输入空格最后到达监听方法的是一个长度为0的字符串，我们不应该处理这类数据。所以我们待办表单的输入应当在点击提交之后清空输入框的内容，对于空输入，我们应当直接返回，像下面这样:

```vue
onSubmit () {
      if (this.label === '') {
        return
      }
      this.$emit('todo-added', this.label)
      this.label = ''
}
```

现在我们构建的待办开始感觉有交互性，但是她不好看，我们需要美化它。

## 进化: CSS 样式化组件

在Vue中使用CSS样式的方法有三种:

- 外部CSS文件
- 单个文件组件(.vue文件)中的全局样式。
- 单个文件组件中组件范围的样式。

我们将分别演示这三种使用方法。

### 外部CSS文件的样式

首先，我们在src/assets目录下建一个名为reset.css的文件。Webpack将处理此文件夹的文件。这意味着我们可以使用CSS预处理器(如SCSS)或后处理器(如PostCSS)。然后将下列内容添加进去:

```css
/*reset.css*/
/* RESETS */
*,
*::before,
*::after {
  box-sizing: border-box;
}
*:focus {
  outline: 3px dashed #228bec;
}
html {
  font: 62.5% / 1.15 sans-serif;
}
h1,
h2 {
  margin-bottom: 0;
}
ul {
  list-style: none;
  padding: 0;
}
button {
  border: none;
  margin: 0;
  padding: 0;
  width: auto;
  overflow: visible;
  background: transparent;
  color: inherit;
  font: inherit;
  line-height: normal;
  -webkit-font-smoothing: inherit;
  -moz-osx-font-smoothing: inherit;
  -webkit-appearance: none;
}
button::-moz-focus-inner {
  border: 0;
}
button,
input,
optgroup,
select,
textarea {
  font-family: inherit;
  font-size: 100%;
  line-height: 1.15;
  margin: 0;
}
button,
input {
  /* 1 */
  overflow: visible;
}
input[type="text"] {
  border-radius: 0;
}
body {
  width: 100%;
  max-width: 68rem;
  margin: 0 auto;
  font: 1.6rem/1.25 "Helvetica Neue", Helvetica, Arial, sans-serif;
  background-color: #f5f5f5;
  color: #4d4d4d;
  -moz-osx-font-smoothing: grayscale;
  -webkit-font-smoothing: antialiased;
}
@media screen and (min-width: 620px) {
  body {
    font-size: 1.9rem;
    line-height: 1.31579;
  }
}
/*END RESETS*/

```

然后在src/main.js中引入:

```javascript
import './assets/reset.css';
```

现在页面变成了下面这样:

![使用了CSS之后的效果图](http://tvax3.sinaimg.cn/large/006e5UvNgy1h9o4uzo4r2j30q30740sz.jpg)

### 单个文件组件(.vue文件)中的全局样式

上面我们用reset.css统一了样式，我们也希望在我们的应用程序里面所有的组件部分样式统一，虽然将这些样式添加到reset.css是可以的，但是将它添加到style标签也是可以的，注意要在App.vue中演示。现在我们更新App.vue中的style标签，更新内容如下：

```css
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
/* Global styles */
.btn {
  padding: 0.8rem 1rem 0.7rem;
  border: 0.2rem solid #4d4d4d;
  cursor: pointer;
  text-transform: capitalize;
}
.btn__danger {
  color: #fff;
  background-color: #ca3c3c;
  border-color: #bd2130;
}
.btn__filter {
  border-color: lightgrey;
}
.btn__danger:focus {
  outline-color: #c82333;
}
.btn__primary {
  color: #fff;
  background-color: #000;
}
.btn-group {
  display: flex;
  justify-content: space-between;
}
.btn-group > * {
  flex: 1 1 auto;
}
.btn-group > * + * {
  margin-left: 0.8rem;
}
.label-wrapper {
  margin: 0;
  flex: 0 0 100%;
  text-align: center;
}
[class*="__lg"] {
  display: inline-block;
  width: 100%;
  font-size: 1.9rem;
}
[class*="__lg"]:not(:last-child) {
  margin-bottom: 1rem;
}
@media screen and (min-width: 620px) {
  [class*="__lg"] {
    font-size: 2.4rem;
  }
}
.visually-hidden {
  position: absolute;
  height: 1px;
  width: 1px;
  overflow: hidden;
  clip: rect(1px 1px 1px 1px);
  clip: rect(1px, 1px, 1px, 1px);
  clip-path: rect(1px, 1px, 1px, 1px);
  white-space: nowrap;
}
[class*="stack"] > * {
  margin-top: 0;
  margin-bottom: 0;
}
.stack-small > * + * {
  margin-top: 1.25rem;
}
.stack-large > * + * {
  margin-top: 2.5rem;
}
@media screen and (min-width: 550px) {
  .stack-small > * + * {
    margin-top: 1.4rem;
  }
  .stack-large > * + * {
    margin-top: 2.8rem;
  }
}
/* End global styles */
#app {
  background: #fff;
  margin: 2rem 0 4rem 0;
  padding: 1rem;
  padding-top: 0;
  position: relative;
  box-shadow: 0 2px 4px 0 rgba(0, 0, 0, 0.2), 0 2.5rem 5rem 0 rgba(0, 0, 0, 0.1);
}
@media screen and (min-width: 550px) {
  #app {
    padding: 4rem;
  }
}
#app > * {
  max-width: 50rem;
  margin-left: auto;
  margin-right: auto;
}
#app > form {
  max-width: 100%;
}
#app h1 {
  display: block;
  min-width: 100%;
  width: 100%;
  text-align: center;
  margin: 0;
  margin-bottom: 1rem;
}
```

现在你会发现我们的待办列表变成了一个卡片。

![样式化之后的待办](http://tvax3.sinaimg.cn/large/006e5UvNgy1h9o5ejjnrkj30kt099gm0.jpg)

为了让他更好看一些，我们将app.vue中声明的按钮样式应用ToDoForm的按钮，在.vue中使用css和在标准html中添加样式的做法相同，通过class属性选中样式。所以我们的待办表单按钮变成了下面这样:

```html
<button type="submit"  class="btn btn__primary btn__lg">
           点击添加
</button>
```

但目前label的距离还待办之间的距离还太过紧凑，我们对待办表单的改造如下:

```vue
<template>
    <form @submit.prevent="onSubmit">
        <h2 class="label-wrapper">
        <label for="new-todo-input" class="label__lg">
          还有什么没做?
        </label>
        </h2>
      <input
        type="text"
        id="new-todo-input"
        name="new-todo"
        autocomplete="off"
        v-model.lazy.trim="label"
        class="input__lg"
      />
      <button type="submit"  class="btn btn__primary btn__lg">
           点击添加
      </button>
    </form>
</template>
```

现在我们的待办列表变成了下面这个样子:

![待办列表美化](http://tvax4.sinaimg.cn/large/006e5UvNgy1h9o5njeh58j30jt0e3gm2.jpg)

### 单个文件组件中组件范围的样式

  现在还剩待办列表组件我们没有美化，为了让这个样式仅应用待办列表组件，所以我们需要在待办列表的style标签里面加上scope，然后加上下面的样式:

```css
<style scoped>
.custom-checkbox > .checkbox-label {
  font-family: Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  font-weight: 400;
  font-size: 16px;
  font-size: 1rem;
  line-height: 1.25;
  color: #0b0c0c;
  display: block;
  margin-bottom: 5px;
}
.custom-checkbox > .checkbox {
  font-family: Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  font-weight: 400;
  font-size: 16px;
  font-size: 1rem;
  line-height: 1.25;
  box-sizing: border-box;
  width: 100%;
  height: 40px;
  height: 2.5rem;
  margin-top: 0;
  padding: 5px;
  border: 2px solid #0b0c0c;
  border-radius: 0;
  appearance: none;
}
.custom-checkbox > input:focus {
  outline: 3px dashed #fd0;
  outline-offset: 0;
  box-shadow: inset 0 0 0 2px;
}
.custom-checkbox {
  font-family: Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  font-weight: 400;
  font-size: 1.6rem;
  line-height: 1.25;
  display: block;
  position: relative;
  min-height: 40px;
  margin-bottom: 10px;
  padding-left: 40px;
  clear: left;
}
.custom-checkbox > input[type="checkbox"] {
  -webkit-font-smoothing: antialiased;
  cursor: pointer;
  position: absolute;
  z-index: 1;
  top: -2px;
  left: -2px;
  width: 44px;
  height: 44px;
  margin: 0;
  opacity: 0;
}
.custom-checkbox > .checkbox-label {
  font-size: inherit;
  font-family: inherit;
  line-height: inherit;
  display: inline-block;
  margin-bottom: 0;
  padding: 8px 15px 5px;
  cursor: pointer;
  touch-action: manipulation;
}
.custom-checkbox > label::before {
  content: "";
  box-sizing: border-box;
  position: absolute;
  top: 0;
  left: 0;
  width: 40px;
  height: 40px;
  border: 2px solid currentcolor;
  background: transparent;
}
.custom-checkbox > input[type="checkbox"]:focus + label::before {
  border-width: 4px;
  outline: 3px dashed #228bec;
}
.custom-checkbox > label::after {
  box-sizing: content-box;
  content: "";
  position: absolute;
  top: 11px;
  left: 9px;
  width: 18px;
  height: 7px;
  transform: rotate(-45deg);
  border: solid;
  border-width: 0 0 5px 5px;
  border-top-color: transparent;
  opacity: 0;
  background: transparent;
}
.custom-checkbox > input[type="checkbox"]:checked + label::after {
  opacity: 1;
}
@media only screen and (min-width: 40rem) {
  label,
  input,
  .custom-checkbox {
    font-size: 19px;
    font-size: 1.9rem;
    line-height: 1.31579;
  }
}
</style>
```

现在它变的更加漂亮：

![更加漂亮的待办](http://tvax1.sinaimg.cn/large/006e5UvNgy1h9o5sfv395j30kn0ft74u.jpg)

## 进化: 计算属性

让我们来接着完善这个待办，它看起来还是有些不完美的地方，没有显示出已经已办的和未半的。一种直接的思路是在插值表达式中进行运算像下面这样:

```html
<h2>{{ToDoItems.filter(item => item.done).length}} out of {{ToDoItems.length}} items completed</h2>
```

但我们不应该在插值表达式中写入太多计算逻辑，一方面这不利于维护，另一方面页面每次渲染都要重新再进行计算，对于更为复杂的页面或更为复杂的表达式这会严重影响性能。更好的解决方案是计算属性。我们首先在App组件中声明一个计算属性，像下面这样:

```vue
computed: {
    listSummary () {
      const completeNum = this.ToDoItems.filter(item => item.done).length
      return `${completeNum} out of ${this.ToDoItems.length} items completed`
    }
}
```

然后在template取出计算属性:

```html
  <div id="app">
    <h1>To-Do List</h1>
    <to-do-form @todo-added="addToDo"></to-do-form>
    <h2 id="list-summary">{{listSummary}}</h2>
   <ul aria-labelledby="list-summary" class="stack-large">
    <li v-for="item in ToDoItems" :key="item.id" >
      <to-do-item :label="item.label"  :done="item.done" :id="item.id" ></to-do-item>
    </li>
  </ul>
  </div>
```

现在我们的应用变成了下面这样:

![计算属性](http://tvax4.sinaimg.cn/large/006e5UvNgy1h9o65dbecbj30mv0jct9j.jpg)

但当我们添加待办会发现总的待办数量会发生变化，当我们点击完成待办，已完成的待办数量没有发生变化。原因在于，点击完成的时候没有更新待办数据的状态。现在我们对待办组件进行改造，当我们完成了待办，待办组件需要告知父组件App，让父组件及时的更新data中的数据。依旧是emit。

```vue
<template>
  <div class="custom-checkbox">
     <input type="checkbox" :id="id" :checked="isDone" class="checkbox" @change="$emit('todo-complete')"/>
     <label :for="id" class="checkbox-label"> {{label}}</label>
  </div>
</template>
```

我们在App.vue中处理该事件:

```html
<div id="app">
    <h1>To-Do List</h1>
    <to-do-form @todo-added="addToDo"></to-do-form>
    <h2 id="list-summary">{{listSummary}}</h2>
   <ul aria-labelledby="list-summary" class="stack-large">
    <li v-for="item in ToDoItems" :key="item.id" >
      <to-do-item :label="item.label"  :done="item.done" :id="item.id" @todo-complete="updateToDoStatus(item.id)"></to-do-item>
    </li>
  </ul>
</div>
```

在methods声明处理该事件的方法:

```javascript
updateToDoStatus (toDoId) {
      const toDoUpdate = this.ToDoItems.find(item => item.id === toDoId)
      toDoUpdate.done = !toDoUpdate.done
}
```

##  总结一下

到现在我们的待办应用大致已经成型，但还是不够完美，我们还要继续完善下去。但都塞在一篇，那这篇的篇幅就太长了。我们来回忆一下本篇我们讲了什么，上一篇我们讲了父组件通过prop来向子组件传递数据，这一篇我们讲了子组件向父组件传递数据的方式和计算属性，以及在Vue中如何使用CSS样式。打个预告，这些样式在JavaFX中也会用到，我们也会用JavaFX构建一个待办，如果你对JavaFX不敢兴趣，那么你可以忽略这句话。

## 参考资料

- mdn web docs https://developer.mozilla.org/en-US/docs/Learn/Tools_and_testing/Client-side_JavaScript_frameworks/Vue_computed_properties