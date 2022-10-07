# Node.js 教程(一) 基本概念与基本使用

> 无题，说不上学习的原因，只是因为在学习计划里面而已。

[TOC]

## 是什么? 

我在学JavaScript的时候是在浏览器面看输出和结果，所以浏览器就是JavaScript的运行环境之一。Node.js是JavaScript的另一个运行环境，基于Chrome V8引擎，开源跨平台，可以让JavaScript在浏览器之外执行，使用Node.js可以用来开发服务端应用程序。

我们现在装下Node.js，Node.js的下载地址: https://nodejs.org/en/。

![Node.js](http://tva3.sinaimg.cn/large/006e5UvNgy1h6uia5kkcaj31990kz0t9.jpg)



下载过后一路next，next就行。然后打开命令行输入node -v， 出现下图代表安装成功: 

![node.js安装成功](http://tvax1.sinaimg.cn/large/006e5UvNgy1h6uida8s5uj30qq0dw0t0.jpg)

代表安装成功。

## Hello World

那怎么在浏览器之外运行JavaScript?  首先你需要一个js文件, 然后在终端里面执行node js文件名即可，如下图所示:

![helloWorld](http://tva1.sinaimg.cn/large/006e5UvNgy1h6uiidqm1zj30rs0ltglj.jpg)

## 还能帮我们干什么？

Node.js 可以帮助我们干啥，让我们再去翻翻官方文档:

> As an asynchronous event-driven JavaScript runtime, Node.js is designed to build scalable network applications. In the following "hello world" example, many connections can be handled concurrently. Upon each connection, the callback is fired, but if there is no work to be done, Node.js will sleep.
>
> Node.js 是一个异步事件驱动JavaScript 运行时，为构建高扩展的网络应用而生，下面的“hello world”例子，可以并发处理连接，每个连接准备完毕，回调将会被激活。但是如果没有网络连接，node.js 将会进入“睡眠”。

我下面建立了一个app.js, 里面的代码如下:

```javascript
// 引入http模块 
const http = require('http'); // 语句一

const hostname = '127.0.0.1';
const port = 3000;
// 建立一个web服务器,监听本机的3000端口
// 连接建立的时候会返回hello world
const server = http.createServer((req, res) => {
  res.statusCode = 200;
  res.setHeader('Content-Type', 'text/plain');
  res.end('Hello World');
});

server.listen(port, hostname, () => {
  // eslint-disable-next-line no-console
  console.log(`Server running at http://${hostname}:${port}/`);
});
```

效果如下:  ![Hello World](http://tva2.sinaimg.cn/large/006e5UvNgy1h6ujd0vv7qj30qh031q2y.jpg)

  所以node.js让javaScript有了开发服务端应用的能力。我们现在来仔细审视一下app.js的语句一这一行，语句一代表引入node.js提供的http模块。原生的JavaScript并没有内置网络相关的库，所以node.js提供了诸多扩展，让JavaScript具备了和主流后端一样的能力，node.js提供的扩展能力如下:

![Node.js的扩展能力](http://tva1.sinaimg.cn/large/006e5UvNgy1h6vf15fmsoj310q0j374w.jpg)

- Debugger  debug 能力
- modules    内置模块和模块化
- Console   控制台
- Cluster 集群
- Add-ons 和C/C++进行通信
- Buffer   二进制数据类型  JavaScript 语言自身只有字符串数据类型，没有二进制数据类型。但在处理像TCP流或文件流时，必须使用到二进制数据。因此在 Node.js中，定义了一个 Buffer 类，该类用来创建一个专门存放二进制数据的缓存区。
- Callbacks  回调是函数异步的产物。在给定任务完成时回调函数。node.js中大量使用回调。Node的所有API都以支持回调的方式编写。
- Crypto  加解密库
- Error Handing  错误处理
- DNS  解析域名
- net 网络库
- global 全局成员变量
- Streaming  流式数据 文件I/O相关

​    node.js 常用内置模块: 

- path  文件路径模块  不同操作系统平台的路径分隔符不同，path用来屏蔽
- fs  File System  文件系统模块 对文件进行操作
- 事件模块  node.js的核心API都是基于事件驱动的 ，在这个体系中，某些对象(发射器) 发出某一事件，我们可以监听这一事件，传入回调函数，监听事件满足，就会触发这个函数的调用。
- http模块  像项目的app.js，我们就是通过require引入了http模块，做了一个小型的http服务器。

其实在后端程序员里，模块化是理所当然的事情，因为服务端开发需要用到大量的第三方类库，所以复用代码也是理所当然的事情。但是对于JavaScript来说确并不是这样，起初的JavaScript只用来写一些简单的网页脚本，但是随着互联网的发展，现在已经有了运行大量JavaScript脚本的复杂程序，所以将JavaScript程序拆分为可按序导入的单独模块的机制就提上议程来。 但是即使有这个迫切的需求，但这个愿景却没有来得那么快，所以JavaScript社区就自发推出了模块化库: 

- CommonJS
- 基于AMD的其他模块系统
- Webpack
- Babel

Node.js 也用CommonJS提供了模块化的能力，所以在Node.js中我们就能复用别人的代码，这也就是npm

## npm

npm  是Node Package Manager 的缩写，也就是node.js的包管理器，类似于Java中的maven，安装node.js的时候会自动附带npm。那我们要引入第三方库就要做一些规定，比如包名，版本，描述，开源协议。在终端中执行: npm init。就会当前项目生成package.json。下面是我执行npm init 生成的package.json： 

```json
{
  "name": "helloworld", // 项目名
  "version": "1.0.0", // 版本
  "description": "这是一个node学习项目",// 描述
  "main": "app.js",//入口文件生命了导出哪些函数给外部使用,哪些对外部开放。
  "scripts": {
    "dev": "node helloWorld.js"
  },
  "author": "", // 作者
  "license": "ISC",// 开源协议
  "dependencies": {
    "express": "^4.18.1" // 依赖于哪些包
  }
}
```

scripts: npm 允许在 package.json 文件中，使用 scripts 字段定义脚本。我们就可以用node run dev 代替 node helloWorld.js。利用scripts编写一些复杂的脚本.上面我们提到了第三方库，那第三方库都放在哪里呢？ 下面这个连接就是第三方库地址:  https://www.npmjs.com/，npm默认去这个仓库寻找，这个库是放在国外的，所以加载起来不会很快，国内有对应的镜像: https://registry.npm.taobao.org，那如何替换默认仓库地址呢: 

```java
# 找express 去 https://registry.npm.taobao.org 找
npm --registry https://registry.npm.taobao.org install express
# 永久替换
npm config set registry https://registry.npm.taobao.org
# 通过下面命令来判断 是否替换完成。 
npm config get registry
```

在Node.js中我们可以用npm来引入第三方库。 在终端中执行如下命令: npm install express, 就相当于为我们的项目引入了express框架。安装成功项目下面会多出一个node_modules文件夹，我们引入的第三方库就在这个文件夹里。写到这里，可能有同学会问，那我有很多个项目，有些第三方库是共同引入的，能否全局引入的？ 当然可以。

```node.js
# 全局安装express. -g = -gloabl,一般默认的安装位置是:在C:\Users\你的用户\AppData\Roaming\npm\node_modules
npm install express  -g
# 查看全局安装位置
npm root -g
# 修改默认安装位置
npm config set prefix "D:\Program Files\nodejs\node_modules\npm"
```

在VsCode中安装全局包，如果不是以管理员身份来运行，就会报没权限，以管理员权限运行就不会有这个问题。

## 总结一下

node.js = javaScript Runtime + Java Script Libary。node.js 为JavaScript提供了模块化和一些库，所以我们可以使用node.js开发服务端程序。

## 参考资料

- Node入门  https://www.nodebeginner.org/index-zh-cn.html#finding-a-place-for-our-server-module
- Node addons简介  https://zhuanlan.zhihu.com/p/351997504
- 淘宝npm镜像源设置  https://segmentfault.com/a/1190000022524476
