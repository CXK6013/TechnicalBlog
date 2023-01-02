# Vue学习笔记(六) 长乐未央

> 例子仍然来自Mdn Web Docs，加上了我自己的理解。长乐未央，长毋相忘。某种意义上算是MDN web docs 中Vue教程的翻译，但又加上了自己的理解。
>
> 地址是: https://developer.mozilla.org/en-US/docs/Learn/Tools_and_testing/Client-side_JavaScript_frameworks/Vue_refs_focus_management

[TOC]

## 进化: 编辑组件

现在我们的组件仍然不太完美，因为它不能编辑，输错了，就输错了。对此我们的解决方案是引入一个编辑组件。还是固定的步骤，首先在src/components下建立一个文件，我们将其命名为ToDoItemEditForm.vue。然后将下面你的代码复制到这个文件中:

```vue
<template>
  <form class="stack-small" @submit.prevent="onSubmit">
    <div>
      <label class="edit-label">Edit Name for &quot;{{label}}&quot;</label>
      <input :id="id" type="text" autocomplete="off" v-model.lazy.trim="newLabel" />
    </div>
    <div class="btn-group">
      <button type="button" class="btn" @click="onCancel">
        取消
        <span class="visually-hidden">正在编辑 {{label}}</span>
      </button>
      <button type="submit" class="btn btn__primary">
        保存
        <span class="visually-hidden">对{{label}}进行修改</span>
      </button>
    </div>
  </form>
</template>
<script>
  export default {
    props: {
      label: {
        type: String,
        required: true,
      },
      id: {
        type: String,
        required: true,
      },
    },
    data() {
      return {
        newLabel: this.label,
      };
    },
    methods: {
      onSubmit() {
        if (this.newLabel && this.newLabel !== this.label) {
          this.$emit("item-edited", this.newLabel);
        }
      },
      onCancel() {
        this.$emit("edit-cancelled");
      },
    },
  };
</script>
<style scoped>
  .edit-label {
    font-family: Arial, sans-serif;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    color: #0b0c0c;
    display: block;
    margin-bottom: 5px;
  }
  input {
    display: inline-block;
    margin-top: 0.4rem;
    width: 100%;
    min-height: 4.4rem;
    padding: 0.4rem 0.8rem;
    border: 2px solid #565656;
  }
  form {
    display: flex;
    flex-direction: row;
    flex-wrap: wrap;
  }
  form > * {
    flex: 0 0 100%;
  }
</style>
```

我们大致的解读一下这个文件，在这个文件里面我们创建了一个表单，表单中的input用于编辑待办事项的名称，用v-model和data中的newLabel建立了双向绑定。同时我们声明了这个组件接收的消息(变量)prop。

在这个表单中有一个保存和取消按钮。

- 点击保存按钮时，组件会通过emit发出item-edited事件。
- 点击取消按钮时，组件会通过emit发出edit-cancelled事件

现在我们来改造一下待办组件，为待办组件添加上编辑和删除功能。我们的设计目标是在待办下面出现一个编辑和删除按钮, 当点击编辑按钮的时候，待办组件隐藏，编辑组件出现。这是一种互斥的关系，我们需要用一个变量来表示这样的状态，我们在data里面进行声明:

```vue
data () {
    return {
      isDone: this.done,
      isEditing: false
    }
}
```

那怎么通关isEditing来控制待办组件的显示和不显示呢，我们可以使用Vue指令if else指令来完成。我们为待办组件的template最外层再添加一个div，同时也是为了控制样式，如果!isEditing = true就代表当前不处于编辑状态。同时在待办列表下面添加一个编辑和删除按钮，为编辑按钮绑定处理事件，将isEditing = true。像下面这样:

```vue
<template>
   <div class="stack-small" v-if="!isEditing">
        <div class="custom-checkbox">
            <input type="checkbox" :id="id" :checked="isDone" class="checkbox" @change="$emit('todo-complete')"/>
            <label :for="id" class="checkbox-label"> {{label}}</label>
        </div>
        <div class="btn-group">
       <button type="button" class="btn"  @click="toggleToItemEditForm">
        编辑 <span class="visually-hidden">{{label}}</span>
      </button>
      <button type="button" class="btn btn__danger" @click="deleteToDo">
        删除 <span class="visually-hidden">{{label}}</span>
      </button>
    </div>
</div>
<to-do-item-edit-form v-else :id="id" :label="label"></to-do-item-edit-form>
</template>

<script>
import ToDoItemEditForm from './ToDoItemEditForm.vue'
export default {
  name: 'ToDoItem',
  components: {
    ToDoItemEditForm
  },
  props: {
    label: {required: true, type: String},
    done: {default: false, type: Boolean},
    id: {required: true, type: String}
  },
  data () {
    return {
      isDone: this.done,
      isEditing: false
    }
  },
  methods: {
    deleteToDo () {
      this.$emit('item-deleted')
    },
    toggleToItemEditForm () {
      this.isEditing = true
    }
  }
}
</script>

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

现在当我们的页面如下所示:

![编辑](http://tvax1.sinaimg.cn/large/006e5UvNgy1h9p48gms0rj30i70mxwfo.jpg)

当点击了编辑之后:

![点击编辑之后](http://tvax3.sinaimg.cn/large/006e5UvNgy1h9p4chazv8j30ip0o0wfu.jpg)



但你会发现点击取消不能返回，点击保存没反应。原因在于我们的编辑组件发出的事件没有被待办组件所理会，当编辑待办组件发出点击取消按钮事件，待办组件应当将编辑组件隐藏，也就是将isEditing置为false。当编辑待办组件按钮发出点击保存按钮事件，我们同样应当将编辑组件隐藏，所以我们待办组件中的template的编辑待办模板标签变成了下面这样:

```vue
<to-do-item-edit-form v-else :id="id" :label="label" @item-edited="itemEdited" @edit-cancelled="editCancelled"></to-do-item-edit-form>
```

待办组件的methods多了两个方法itemEdited和editCancelled:

```vue
itemEdited (newLabel) {
  this.$emit('item-edited', newLabel)
  this.isEditing = false
},
editCancelled () {
  this.isEditing = false
}
```

现在我们点击取消就能回到待办组件，但是点击保存仍然没有反应，原因在于我们在待办组件里面发出的事件没有被App组件所处理，待办列表的数据在App里面。所以现在我们就需要转到App组件里面，处理这个事件。首先在待办组件上声明处理此事件的方法:

```vue
<to-do-item :label="item.label"  :done="item.done" :id="item.id" @todo-complete="updateToDoStatus(item.id)" @item-edited="editToDo(item.id,$event)" @item-deleted="deleteToDo(item.id)"></to-do-item>
```

 $event是一个特殊的Vue变量，用于携带子组件传递过来的数据。现在我们需要在methods添加editToDo和deleteToDo方法:

```vue
editToDo (toDoId, newLabel) {
    const toDoEdit = this.ToDoItems.find((item) => item.id === toDoId)
    toDoEdit.label = newLabel
 },
deleteToDo (toDoId) {
      const deleteToDoIndex = this.ToDoItems.findIndex(item => item.id === toDoId)
      this.ToDoItems.splice(deleteToDoIndex, 1)
 }
```

## 一个小问题

到目前为止一切看起来都很好，但是如果用一下会发现还是会有一点小问题:

- 尝试选中待办事项的复选框

  ![](http://tvax3.sinaimg.cn/large/006e5UvNgy1h9p5sykxa2j30ix0my0u0.jpg)

- 然后点击该待办事项的的编辑按钮

​     ![点击编辑按钮](http://tvax3.sinaimg.cn/large/006e5UvNgy1h9p5viqnw0j30ih0o075j.jpg)

- 然后点击取消

​      ![点击取消](http://tvax2.sinaimg.cn/large/006e5UvNgy1h9p5ws4km1j30is0n2gmr.jpg)

选中状态被丢失了，待办事项的统计也出了问题，当你选中会发现统计数据跟你预期的相反，它变成了0:

![变成了0](http://tvax3.sinaimg.cn/large/006e5UvNgy1h9p5yucthpj30j00mu75i.jpg)

原因在于加载组件时，复选框的状态取决于isDone，而isDone又取决于done，这个done又由外部传入，所以当我们选中一个初始状态为未选中的复选框然后点击编辑，又点击取消，就相当于待办组件又重新被加载，丢失了选中的状态。但是幸运的是解决这个问题也比较简单，我们可以将isDone转为一个计算属性，计算属性会保留改变。在Vue的官方文档是这么介绍计算属性的：

> **计算属性是基于它们的响应式依赖进行缓存的**。只在相关响应式依赖发生改变时它们才会重新求值

这也就是我们将其转换为计算属性的时，点击选中按钮，计算属性会被计算一次，我们点击取消返回时，由于计算属性的依赖并没有发生更新，所以我们的选中状态得以保留。你看在实践中我们对Vue的一些理解更加深入了。最初我对计算属性的理解是相比于在插值表达式中写逻辑判断，可维护性更高，就让template里面只负责显示，另一个方面是计算属性只有在相关响应式依赖发生改变才会重新求值，这对于一些大型页面来说，如果我们将复杂的逻辑判断写在插值表达式里面，每次渲染都要再执行一遍运算，这会相当损耗性能。现在我们可以借助计算属性来保留状态。

首先我们将待办组件的data中取消下面这一行:

```vue
isDone: this.done,
```

然后在待办组件的计算属性如下声明:

```vue
computed: {
    isDone () {
      return this.done
    }
},
```

现在你再保存并重新加载，就会发现问题已经解决。是不是很有成就感了呢！

## 自定义事件与原生事件

在这个例子中让我们感到有点不清晰的，恐怕也就是自定义事件和原生事件了吧。下面这个流程图会让你对事件流转更加清晰:

![事件流转图](http://tvax1.sinaimg.cn/large/006e5UvNgy1h9p7x7be7ej30tz0fsaav.jpg)

## ref与焦点管理

我们几乎完成了一个小型的应用，但目前仔细审视的话，它的体验仍然有些美中不足。比如我们只用键盘来完成上面的操作。我们可以借助tabs这个键盘上的按钮，来完成编辑、保存待办。让我们重新加载页面，然后按tab键，你会发现待办的输入框上会有一个蓝色框框，代表我们现在处于输入框，向下面这样:

![foucs管理](http://tvax3.sinaimg.cn/large/006e5UvNgy1h9p8aqtsn4j30j40n2myl.jpg)

这个蓝色的框框我们姑且称之为焦点(focus) ， 再次按下tab键，这个焦点会被移动到点击添加按钮上。再次按tab键，焦点会出现在第一个待办的复选框上。接着按，它会停留在第一个待办的编辑按钮上。然后按下Enter键，然后编辑按钮消失，出现的是我们的编辑待办组件，我们的焦点也消失了。这样的交互可能让用户的体验没有那么良好。当你再次按下tab键，焦点出现在哪里，这取决于你使用的浏览器。同样的，如果你按table让焦点再次出现，按下保存或取消编辑，焦点会再次消失。

为了给用户更好的体验，我们将添加代码来控制焦点，以便在编辑表单出现的时候，让焦点出现在编辑表单的输入框上。当用户在编辑表单中取消编辑或保存编辑，让焦点重新回到编辑按钮。 为了做到这一点，我们都需要对Vue如何工作有更加深入一点的理解。

### Virtual DOM(虚拟DOM) and refs

Vue和主流的前端框架一样，选择使用虚拟DOM来管理结点，这意味着Vue在内存中保留了应用程序所有的结点代表(原文为representation)，这意味着任何更新都会先到达内存中的结点。然后再对页面实际结点所需的更新进行批量同步。

直接读写真实DOM的结点相对于虚拟结点来说是有些昂贵的，虚拟结点会有更好的性能。然后这也意味着在框架里面你不可以通过浏览器的原生APIs来在操纵HTML元素(向Document.getElementById)，这会导致虚拟DOM和真实DOM同步出现问题(失去同步 原文为going out of sync)。

但是如果你确实是需要操纵真实DOM的结点(像设置焦点)，你可以选择使用Vue ref。对于自定义的组件，你可以选择使用refs直接访问子组件的内部结构，但是注意，要小心使用，这会让你的代码看起来难以理解。

如果你想在组件中使用ref，你需要在想要访问的元素上添加ref属性，并未该属性的值提供字符串标识符。注意在一个组件中ref必须是唯一的。

### 在待办组件里面添加ref

首先我们对ToDoItem.vue进行改造，在编辑按钮上为它添加ref:

```html
<button type="button" class="btn"  @click="toggleToItemEditForm" ref="editButton">
        编辑 <span class="visually-hidden">{{label}}</span>
</button>
```

然后我们就可以拿到这个结点了，让我们在toggleToItemEditForm里尝试获取一下ref:

```javascript
toggleToItemEditForm () {
      console.log(this.$refs.editButton)
      this.isEditing = true
}
```

点击编辑按钮你就会在控制台发现输出了ref所在按钮的结点。

### nextTick方法

当用户保存或取消他们的编辑时，我们希望焦点回到编辑按钮上。所以我们要对ToDoItem组件中的itemEdited和editCancelled中的方法进行改造。我们创建一个不带参数的方法，在这个方法中我们获取上面的ref，ref目前处于编辑按钮上，我们拿到这个按钮然后设置焦点即可。

```
focusOnEditButton () {
      const editButtonRef = this.$refs.editButton
      editButtonRef.focus() 
}
```

然后在itemEdited和editCancelled调用即可: 

```javascript
itemEdited (newLabel) {
      this.$emit('item-edited', newLabel)
      this.isEditing = false
      this.focusOnEditButton();
},
editCancelled () {
      this.isEditing = false
      this.focusOnEditButton();
},
```

但是即使做了改造，你尝试保存/取消待办也会发现，焦点没有按我们预想的回到编辑按钮上，同时在控制台也会看到下面的报错:

```vue
[Vue warn]: Error in v-on handler: "TypeError: editButtonRef is undefined"

found in

---> <ToDoItemEditForm> at src/components/ToDoItemEditForm.vue
       <ToDoItem> at src/components/ToDoItem.vue
         <App> at src/App.vue
           <Root> vue.esm.js:5105
TypeError: editButtonRef is undefined
    focusOnEditButton ToDoItem.vue:60
    editCancelled ToDoItem.vue:56
    VueJS 4
    onCancel ToDoItemEditForm.vue:43
    VueJS 38
    toggleToItemEditForm ToDoItem.vue:47
    VueJS 21
```

当我们点击编辑按钮的时候，我们还能拿到这个ref，为什么点击取消和保存就拿不到了呢？ 原因在于当我们点击编辑按钮，此时将isEditing设置true，我们将不再渲染组件的编辑按钮，这也就意味着ref引用不到按钮，因此我们无法获得编辑按钮。

但是这也似乎有些说不通，在我们访问ref之前，我们不是将isEditing 设置为false了吗？那按钮不就应该显示了吗？ 或者说渲染了吗？那为什么我们取不到呢？这也就是虚拟DOM发挥作用的地方，Vue试图优化批量更改，所以在DOM上的更新可能不会立马更新，它会放在一个队列里面，因此当我们调用focusOnEditButton时，编辑按钮尚未渲染。我们需要等到下一个DOM的更新周期之后，ref才能获取到按钮。

那有没有什么办法能让在DOM更新之后再调用focusOnEditButton方法呢，Vue提供了一个名为$nextTick的方法，该方法接收一个回调函数，回调函数会在DOM更新之后被调用，所以我们的focusOnEditButton就变成了下面这样:

```javascript
focusOnEditButton () {
      this.$nextTick(()=>{
        const editButtonRef = this.$refs.editButton
        editButtonRef.focus() 
      })
}
```

现在我们我们按编辑进入编辑表单，在编辑表单点击取消或返回会发现编辑按钮上就出现了焦点。

## Vue 的生命周期

Vue的生命周期是一个相当重要的概念，我们在前面的文章都回避了他，原因在于这个概念配合具体了例子，会让理解更加清晰。现在我们还没有完成的事情是，在我们点击编辑按钮的时候，将焦点移动到表单的输入框，但是编辑按钮位于待办组件，编辑待办的输入框位于编辑组件。所以我们不能在编辑按钮的点击事件中设置焦点。我们可以借助点击编辑按钮时都会将ToDoItemEditForm组件重新挂载来解决这个问题。

那究竟该如何入手呢？ Vue的组件会经历一系列阶段，我们称之为生命周期。这个生命周期从元素被创建添加到VDOM开始，一直到它们被虚拟DOM中被移除结束。

在每个阶段Vue都会有触发的方法，这对于数据获取之类的事情会很有用，比如你需要在组件渲染之前或属性之后获取数据。每个阶段调用的方法如下，按触发顺序进行排列：

1. beforeCreate: 在实例刚在被创建，且未初始化完成时调用。
2. created: 在实例创建完成后调用。此时实例已完成初始化，但是还没有挂载。
3. beforeMount: 在挂载开始之前被调用：相关的 render 函数首次被调用。
4. mounted: 在组件挂载之后调用。此时可以访问实例上的属性和方法。
5. beforeUpdate: 在数据更新之前调用。
6. updated: 在数据更新之后调用。此时可以访问更新后的 DOM。
7. beforeDestroy: 在实例销毁之前调用。
8. destroyed: 在实例销毁之后调用

现在让我们为ToDoItemEditForm的输入框添加一个ref，如下所示:

```html
<input :id="id" type="text" autocomplete="off" v-model.lazy.trim="newLabel"  ref="labelInput" />
```

接下来让我们再导出的script对象中添加一个属性mounted，注意这个属性和计算属性平级，我们在mounted中获取输入框的ref。

```javascript
mounted () {
    const labelInputRef = this.$refs.labelInput
    labelInputRef.focus()
}
```

注意我们这里并没有使用nextTick，原因在于在Vue的声明周期里面mounted被调用的时候，组件已经被挂载，我们可以访问属性和方法了。

## 删除的焦点移动

目前还没有考虑的一个问题是，删除的时候焦点应该移动到哪里，我想此时用户应当关注的是统计信息，也就是还有多少待办。让我们把视线转移到App.vue中的统计信息。我们还是为统计信息添加上ref。

```html
<h2 id="list-summary" ref="listSummary" tabindex="-1">{{listSummary}}</h2>
```

现在我们已经获得了统计信息的ref，我们就可以在删除待办的时候，将焦点移动到这上面:

```javascript
deleteToDo (toDoId) {
  const deleteToDoIndex = this.ToDoItems.findIndex(item => item.id === toDoId)
  this.ToDoItems.splice(deleteToDoIndex, 1)
  this.$refs.listSummary.focus()
}
```

现在当你删除一个待办，焦点将会被移动调待办的统计信息上。

## 总结一下

经过这个例子我们将(一) (二) (三)的概念串联了起来，在实践中体会理论最为深刻。按照规划来说，本身打算三篇介绍完Vue的相关理念，但是后面一边学一边发现，一些理念还是在实践中体会比较深。感谢MDN Web Docs 提供的代码示例，非常详细。



