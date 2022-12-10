# JavaScript学习笔记(一) promise和async/wait

[TOC]

## 前言

最近我在学习前端相关的知识，没有意识到的一个问题是我用的Ajax是异步的，如下面代码所示:

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
    <h2>{{ product }} X are in stock .</h2>
    <ul v-for = "product in products" >
        <li>
            <input type="number" v-model.number="product.quantity">
                      {{product.name}} 
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
<div>
    app.products.pop();
</div>
<script src="https://cdn.jsdelivr.net/npm/vue@2/dist/vue.js"></script>
<script>
    const app = new Vue({
        el:'#app',
        data:{
            product:'Boots',
            products:[
                'Boots',
                'Jacket'
            ]
        },
        computed:{
            totalProducts(){
                return this.products.reduce((sum,product) => {
                    return sum + product.quantity;
                },0);
            }
        },
        created(){
            fetch('http://localhost:8080/hello').then(response=> response.json()).then(json => {
                console.log("0");
                this.products = json.products;
            })  
            fetch('http://localhost:8080/hello').then(response=> response.json()).then(json => {
                console.log("1");
            })  
        }
    })
</script>
</body>
</html>
```

按照我的理解是应该先输出0,后输出1。但事实上并非如此，因为fetch是异步的，这里的异步我们可以简单的理解为执行fetch的执行者并不会马上执行，所以会出现1先在控制台输出这样的情况:

![实际输出结果](http://tvax1.sinaimg.cn/large/006e5UvNgy1h8agwlrm0kj30yq0jm0tx.jpg)

但我的愿望是在第一个fetch任务执行之后再执行第二个fetch任务, 其实可以这么写:

```javascript
fetch('http://localhost:8080/hello').then(response=> response.json()).then(json => {
                console.log("0");
                this.products = json.products;
                fetch('http://localhost:8080/hello').then(response=> response.json()).then(json => {
                console.log("1");
      })  
})  
```

这看起来像是一种解决方案，那如果说我有A、B、C、D四个任务都是异步的都想按A B C D这样的顺序写，如果向上面的写法，嵌套调用，那看起来就相当的不体面。由此就引出了promise。

## promise

### 基本使用

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
    
</body>
<script>    
  new Promise(function(resolve, reject) {
              setTimeout(()=>{
                    console.log(0);
                    resolve("1");    
               },3000)
     }).then((result)=> console.log(result)); 
</script>
</html> 
```

如上面代码所示，我们声明了一个Promise对象，并向其传递了一个函数，函数的行为是3秒后再控制台输出0,并调用resolve函数，resolve函数会触发then()。所以我们上面的两个fetch用promise就可以这么写。

```javascript
fetch('http://localhost:8080/hello').then(response=> response.json()).then(json => {
   console.log("0");
   this.products = json.products;
               
}).then(()=>{
     fetch('http://localhost:8080/hello').then(response=> response.json()).then(json => {
         console.log("1");
     })  
})  
```

fetch直接返回的就是Promise对象，then之后也还是一个Promise对象。Promise对象的构造函数语法如下:

```javascript
let promise = new Promise(function(resolve, reject) {
 
});
```

对于一个任务来说，任务最终状态有两种: 一种是执行成功、一种是执行失败。参数resolve和reject是由JavaScript自身提供的回调，是两个函数, 由 JavaScript 引擎预先定义，因此我们只需要在任务的成功的时候调用resolve，失败的时候reject即可。注意任务的最终状态只能是成功或者失败，你不能先调用resolve后调用reject，这样任务就会被认定为失败状态。注意，如果任务在执行的过程中出现了什么问题，那么应该调用reject，你可以传递给reject任意类型的参数，但是建议使用Error对象或继承自Error的对象。

### Promise的状态

![Promise状态](http://tvax3.sinaimg.cn/large/006e5UvNgy1h8aj6ci75mj30ta0ecgm9.jpg)

一个Promise刚创建任务未执行完毕，状态处于pending。在任务执行完毕调用resolve时，状态转为fulfilled。任务执行出现异常调用error，状态转为reject。

### then、catch、finally

then的中文意为然后，连贯理解起来也比较自然，任务执行完毕然后执行，语法如下:

```javascript
promise.then(
  function(result) { /* 处理任务正常执行的结果 */ }, 
  function(error) { /* 处理任务异常执行的结果 */ }
);
```

第一个函数将在 promise resolved 且接收到结果后执行。第二个函数将在 promise rejected 且接收到error信息后执行。

下面是示例:

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
    
</body>
<script>
    let promiseSuccess = new Promise(function(resolve,reject){
        setTimeout(() => resolve("任务正确执行完毕"),1000);    
    })
    promiseSuccess.then(result=>{
        alert(result);
    });
    let promiseFailed = new Promise(function(resolve,reject){
        setTimeout(() => reject("任务执行出现异常"),1000);    
    })
    promiseFailed.then(result=>{
        alert(result);
    },error=>{
        alert(error);
    });
</script>
</html>
```

如果你只想关注任务执行发生异常，可以将null作为then的第一个参数, 也可以使用catch:

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
    
</body>
<script>
    new Promise(function(resolve,reject){
        reject("发生了错误");
    }).catch(error=>{
        alert(error);
    })
    new Promise(function(resolve,reject){
        reject("发生了错误");
    }).then(null,error=> alert(error));
</script>
</html>
```

像try catch 中的finally子句一样，promise也有finally，不同于then，finally没有参数，我们不知道promise是否成功，不过关系，finally的设计意图是执行“常规”的完成程序。当promise中的任务结束，也就是调用resolve或reject，finally将会被执行:

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
    
</body>
<script>
    new Promise(function(resolve,rejetc){
          resolve("hello");   
    }).then(result => alert(result),null).finally(()=>alert("执行清理工作中"));
</script>
</html>
```

then catch  finally 的执行顺序为跟在Promise后面的顺序。

### Promise链与错误处理

通过上面的讨论，我们已经对Promise有一个基本的了解了，如果我们有一系列异步任务，为了讨论方便让我们把它称之为A、B、C、D，异步任务的特点是虽然他们在代码中的顺序是A、B、C、D，但实际执行顺序可能四个字母的排列组合，但是我们希望是A、B、C、D，那我们用Promise可以这么写:

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
    
</body>
<script>
    new Promise(function(resolve,reject){
        alert("我是A任务");
        resolve("请B任务执行");
    }).then(result=>{
       alert(result+"收到请求,B任务开始执行");
       return "B 任务执行完毕,请C任务开始执行";
    },null).then(result=>{
        alert(result+"收到请求,C任务开始执行");
        return "C 任务执行完毕,请D任务开始执行";
    },null).then(result =>{
          alert(result+"D任务开始执行,按A、B、C、D序列执行完毕");
    },null)
</script>
</html>
```

运行流程为: 初始化promise，然后开始执行alert后resolve，此时结果滑动到最近的处理程序then，then处理程序被调用之后又创建了一个新的Promise，并将值传给下一个处理程序，以此类推。随着result在处理程序链中传递，我们可以看到一系列alert调用:

- 我是A任务
- 请B任务执行收到请求,B任务开始执行
- B 任务执行完毕,请C任务开始执行收到请求,C任务开始执行
- C 任务执行完毕,请D任务开始执行D任务开始执行,按A、B、C、D序列执行完毕

这样做之所以是可行的，是因为每个对.then的调用都会返回了一个新的promise，因此我们可以在其之上调用下一个.then。

当处理程序返回一个值时(then)，它将成为该promise的result，所以将使用它调用下一个.then。

上面的是按A、B、C、D执行，我们也会期待A任务执行之后，B、C、D都开始执行，这类似于跑步比赛，A是开发令枪的人，听到信号，B、C、D开始跑向终点。下面是一个示例:

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
    
</body>
<script>
let promise  = new Promise(function(resolve,reject){
    setTimeout(() => resolve(1),1000);
})

promise.then(function(result){
    alert(result); // 1
    return result * 2;
})

promise.then(function(result){
    alert(result); // 1
    return result * 2;
})

promise.then(function(result){
    alert(result); // 1
    return result * 2;
})

</script>
</html>
```

弹出结果将只会是1。

### Promise API

#### Promise.all

假设我们希望并行执行多个任务，并且等待这些任务执行完毕才进行下一步，Promise为我们提供了Promise.all:

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
    
</body>
<script>
   var  proOne  = new Promise(resolve=> setTimeout(()=> resolve(1),3000);
   var  proTwo  = new Promise(resolve=> setTimeout(()=> resolve(2),2000);
   var proThree = new Promise(resolve=> setTimeout(()=> resolve(3),1000));
   Promise.all([proOne,proTwo,proThree]).then(alert);
</script>
</html>
```

即使proOne任务花费时间最长，但它的结果仍让是数组中的第一个。如果人一个promise失败即调用了reject，那么Promise.all就被理解reject，完全忽略列表中其他的promise:

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
    
</body>
<script>
   var  proOne  = new Promise(resolve=> setTimeout(()=> resolve(1),1000));
   var  proTwo  = new Promise((resolve,reject)=> setTimeout(()=> reject(2),1000));
   var  proThree = new Promise(resolve=> setTimeout(()=> resolve(3),1000));
   Promise.all([proOne,proTwo,proThree]).catch(alert);
</script>
</html>
```

最后会弹出2。

Promise的语法如下:

```
// iterable 代表可迭代对象
let promise = Promise.all(iterable);
```

iterable 可以出现非Promise类型的值如下图所示：

```javascript
Promise.all([
  new Promise((resolve, reject) => {
    setTimeout(() => resolve(1), 1000)
  }),
  2,
  3
]).then(alert); // 1, 2, 3
```

#### Promise.allSettled

Promise.all中任意一个promise的失败，会导致最终的任务失败，但是有的时候我们希望看到所有任务的执行结果，有三个任务A、B、C执行，即使A失败了，那么B和C的结果我们也是关注的，由此就引出了Promise.allSettled:

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
    
</body>
<script>
// 这三个地址用于获取用户信息。
// 即使一个失败了我们仍然对其他的感兴趣
let urls = [
  'https://api.github.com/users/iliakan',
  'https://api.github.com/users/remy',
  'https://no-such-url'
];
Promise.allSettled(urls.map(url => fetch(url)))
  .then(results => { // (*)
    results.forEach((result, num) => {
      if (result.status == "fulfilled") {
        alert(`${urls[num]}: ${result.value.status}`);
      }
      if (result.status == "rejected") {
         alert(`${urls[num]}: ${result.reason}`);
      }
    });
  });
</script>
</html>
```

注意Promise.allSettled是新添加的特性，旧的浏览器可能不支持，那就需要我们包装一下，更为专业的说法是polyfill:

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
    
</body>
<script>
    if(!Promise.allSettled){
        const rejectHandler = reason => ({status:'rejected',reason});
        const resolveHandler = value => ({status:'fulfilled',value});

        Promise.allSettled = function(promises){
            const convertedPromises = promises.map(p => Promise.resolve(p)).then(resolveHandler,rejectHandler);
            Promise.all(convertedPromises);
        }
    }
</script>
</html>
```

promises通过map中的Promise.resolve(p)将输入值转换为Promise，然后对每个Promise都添加.then处理程序。这样我们就可以使用Promise.allSettled来获取所有给定的promise的结果，即使其中一些被reject。

#### Promise.race

有的时候我们也只关注第一名，那该怎么做呢? 由此引出Promise.race:

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
    
</body>
<script>
 Promise.race([
    new Promise((resolve,reject) => setTimeout(()=> resolve(1),1000)),
    new Promise((resolve,reject) => setTimeout(()=> resolve(2),2000)),   
    new Promise((resolve,reject) => setTimeout(()=> resolve(3),2000))
 ]).then(alert);
</script>
</html>
```

这里第一个Promise最快，所以alert(1)。

#### Promise.any

但最快有可能是犯规的对吗？ 即这个任务reject，有的时候我们只想关注没有违规的第一名，由此引出Promise.any:

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
    
</body>
<script>
 Promise.race([
    new Promise((resolve,reject) => setTimeout(()=> reject(1),1000)),
    new Promise((resolve,reject) => setTimeout(()=> resolve(2),2000)),   
    new Promise((resolve,reject) => setTimeout(()=> resolve(3),2000))
 ]).then(alert);
</script>
</html>
```

虽然第一个promise最快，但是它犯规了，所以alert的是2。那要是全犯规了呢, 那就需要catch一下:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=S, initial-scale=1.0">
    <title>Document</title>
</head>
<body>
    
</body>
<script>
     Promise.any([
    new Promise((resolve,reject) => setTimeout(()=> reject("犯规1"),1000)),
    new Promise((resolve,reject) => setTimeout(()=> reject("犯规2"),2000))
 ]).catch(error =>{
    console.log(error.constructor.name); // AggregateError
    console.log(error.errors[0]); // 控制台输出犯规1
 	console.log(error.errors[1]); // 控制台输出犯规2
  } )
</script>
</html>
```

## async/await 简介

有的时候异步方法可能比较庞大，直接用Promise写可能不大雅观，由此我们就引出async/await, 我们只用在需要用到Promise的异步函数上写上async这个关键字，这个函数的返回值将会自动的被包装进Promise:

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
    
</body>
<script>
async function f() {
  return 1;
}
f().then(alert); // 1
</script>
</html>
```

但是我们常常会碰到这样一种情况, 我们封装了一个函数，这个函数上有async，但是内部的实现有两个操作A、B，B操作需要等到A完成之后再执行，由此就引出了await:

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
</body>
<script>
async function demo(){
     let responseOne = fetch('http://localhost:8080/hello');
     console.log("1");
     let json  = await responseOne.json; // 语句一
     let response = fetch('http://localhost:8080/hello');
     let jsonTwo  = await responseOne.json; // 语句二
     console.log("2");
}
demo(); 
</script>
</html>
```

这里用async/wait改写了我们开头的例子，JavaScript引擎执行到语句一的时候会等待fetch操作完成，才执行await后的代码。这一看好像清爽了许多。注意await不能在没有async的函数中执行。相比较于Promise.then，它只是获取promise结果的一个更加优雅的写法，并且看起来更易于读写。那如果等待的promise发生了异常呢，该怎么进行错误处理呢？有两种选择，一种是try catch，另一种是用异步函数返回的promise产生的catch来处理这个错误：

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
    
</body>
<script>
async function demo01(){
    try{
         let response = await fetch('/no-user-here');
         let user = await response.json();
    }catch(error){
        alert(error);
    }
}
demo01(); 

async function demo02(){
    let response = await fetch('/no-user-here');
    let user = await response.json();
    
}
demo02().catch(alert);
</script>
</html>
```

## 总结一下

JavaScript也有异步任务，如果我们想协调异步任务，就可以选择使用Promise，当前的我们只有一个异步任务，我们希望在异步任务回调，那么可以在用Promise.then，如果你想关注错误结果，那么可以用catch，如果你想在任务完成之后做一些清理工作，那么可以用Promise的finally。现在我们将异步任务的数目提升，提升到三个，如果我们想再这三个任务完成之后触发一些操作，那么我们可以用Promise.all，但是但Promise.all的缺陷在于，一个任务失败之后，我们看不到成功任务的结果，如果任务成功与失败的结果，那么就可以用Promise.allSettled。但有的时候我们也指向关注“第一名”，那就用Promise.race，但有的时候我们也只想要没犯规的第一名，这也就是Promise.any。有的时候我们也不想用then回调的这种方式，这写起来可能有点烦，那就可以用async/await 。

## 参考资料

- 《现代JavaScript教程》 https://zh.javascript.info/async-await