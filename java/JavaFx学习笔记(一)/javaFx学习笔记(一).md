# JavaFx教程(一) 基本概念

> 今年打算踏出领域之外，做一点桌面开发，在几个桌面开发之间选择了一下，最终还是选择了JavaFx，原因这个也是用Java做的，不用再学习一门语言。JavaFx还有一个游戏引擎，打算后面写一点游戏玩玩。那么JDK版本的选择呢，选择了JDK 11，很大的原因在于JDK 带有 Jlink可以对JDK进行裁剪，让最终生成的平台包尽可能的小。
>
> javaFx 有个十分不错的文档: https://fxdocs.github.io/docs/html5/#_overview 
>
> 本篇教程基本上依据这个文档。阅读本系列教程的前提是有JavaSE的基础，maven熟练掌握。

[TOC]

##  前言

在我迈入大学的时候，我没想到第一门编程语言出来的是一个黑窗口，这让我十分失落，这让我一点动力都没有，甚至让我一度十分失落。后面我买了本C语言游戏开发，觉得蛮有意思的，但是投入精力不多，我当时的主要精力还是用在数学上。后面开了Java课程，我其实也没那么有兴致，翻到教科书后面发现有桌面开发，于是兴奋起来，但是发现Java Swing比较丑:

![Swing巨丑](http://tva1.sinaimg.cn/large/006e5UvNgy1h6pntcjxj4j30jc0djq3e.jpg)

于是放弃了学习的想法。但是偶然间发现了Java出了JavaFx，看了看界面发现长的不错，是个“标致的小姑娘”，再加上有个游戏引擎，我打算开发写点游戏玩玩，就打算简单学习一下。

## javaFx 是啥？

> JavaFX 是一个开源的下一代客户端应用平台，适用于基于Java构建的桌面、移动端和嵌入式系统。 --- javaFx官网

所以JavaFx是一个应用平台，我们可以借助JavaFx来构建图形化应用。

## 从Hello World开始

JDK 11 的安装这里不再赘述，除此之外还需要一个maven archetype，第一个例子我们先跟着做。下面是安装JavaFx archetype 的脚本:

```shell
git clone https://github.com/openjfx/javafx-maven-archetypes.git
cd javafx-maven-archetypes
mvn clean install
```

![archetype](http://tvax3.sinaimg.cn/large/006e5UvNgy1h6hrphuz4sj30m80iit9h.jpg)

```shell
# 坐标如下
GroupId = org.openjfx
ArtifactId = javafx-archetype-simple
```

![将这个移除掉](http://tvax2.sinaimg.cn/large/006e5UvNgy1h6hrrmut22j30u60fbq4g.jpg)

稍微解释一下，JDK 11 进行了模块化，rt这个包引入不到。

![运行这个运用](http://tva2.sinaimg.cn/large/006e5UvNgy1h6hrtou0hcj312g0ghmzl.jpg)

就会出现一个窗口:

![出现了这个窗口](http://tvax1.sinaimg.cn/large/006e5UvNgy1h6hrzffk1tj318c0p73zo.jpg)

这个maven 模板产生的java Fx 的依赖版本是 13 ，我将其改为了11。下面来解释一下这个代码：

```java
public class App extends Application {

    @Override
    public void start(Stage stage) {
        var javaVersion = SystemInfo.javaVersion();
        var javafxVersion = SystemInfo.javafxVersion();

        var label = new Label("Hello, JavaFX " + javafxVersion + ", running on Java " + javaVersion + ".");
        var scene = new Scene(new StackPane(label), 640, 480);
        stage.setScene(scene);
        stage.show();
    }

    public static void main(String[] args) {
        launch();
    }

public class SystemInfo {
	
    // 获取当前Java的版本
    public static String javaVersion() {
        return System.getProperty("java.version");
    }
	
    // 获取当前javaFx的版本
    public static String javafxVersion() {
        return System.getProperty("javafx.version");
    }
}
```

出来的图形的数据结构如下所示: 

![结构展示](http://tvax2.sinaimg.cn/large/006e5UvNgy1h6pjvqk7awj30p90jbdfq.jpg)

Stage 有舞台的意思, 可以理解为JavaFx对当前操作系统窗口的包装，Scene则有场景的意思，所以讨巧一点的理解是，javaFx的设计者将桌面开发理解为戏剧的演出，戏剧需要有舞台，而戏剧则有不同的场景，不同的场景有不同的布置，在上面的“hello world”这个小戏剧中，我们上面只摆了StackPane，StackPane上放了Label。上面是一种比较简单的数据结构图，更为通用的如下：

![scene graph](http://tva1.sinaimg.cn/large/006e5UvNgy1h6pkvt8e1tj311j0lvmyc.jpg)

直接或者间接依附于Scene的元素，统称为Scene Graph，这里将这个词译为场景图，其实看到Graph，我想到的是数据结构里面的图，觉得这个翻译不合适，但是又找不出太好的译名，就姑且用这个译名吧。JavaFx中任意一个窗口都可以表示为上面的数据结构，在任意给定时间，一个Stage都可以有一个Scene附加到它。

JavaFx中场景图中的所有元素都是Node对象。Node有三种类型: 跟、分支、叶。根结点时唯一没有父结点的结点，看到这里有同学可能会问，Scene不是Node的父结点吗？ 从数据结构上看是，但是Scene本身并不是Node对象，我们可以将Scene理解为大地。分支结点和叶子结点的区别在于叶子结点没有子结点。在场景图中，父结点的许多属性由子结点共享，

##  变换-Transformations

在JavaFx中变换分为三种: 平移、缩放、旋转。我们将以下面的例子来演示这三种变换:

```java
public class TransformApp extends Application {

    private Parent createContent(){
        // 矩形设置长宽高
        Rectangle box = new Rectangle(100,50, Color.BLUE);
        transform(box);
        return new Pane(box);
    }
	
    
    private void transform(Rectangle box) {
    	// 在这里写变换的代码,下面只展示这个方法
        
    }


    @Override
    public void start(Stage primaryStage) throws Exception {
        primaryStage.setScene(new Scene(createContent(),300,300,Color.GRAY));
        primaryStage.show();
    }

    public static void main(String[] args) {
        launch(args);
    }
}
```

在windows 上运行此程序出来的效果图如下：

![transform转换](http://tva4.sinaimg.cn/large/006e5UvNgy1h6plr9cpwqj308d096t8i.jpg)

### Translate  移动

在JavaFx中，结点可以有三维变换(X、Y或Z)和二维变换(X、Y)，但是我们的示例应用程序是2D的，所以我们将只考虑X和Y轴。 在JavaFx和计算机图形学中，translate表示移动。下面的程序将上面的矩形在X轴上平移100像素，在Y轴上平移200像素。

![平移](http://tva1.sinaimg.cn/large/006e5UvNgy1h6plw8ofjqj308f094web.jpg)

代码如下:

```java
private void transform(Rectangle box) {
	// 在这里写变换的代码,下面只展示这个方法
    box.setTranslateX(100);
    box.setTranslateY(200);
}
```



### Scale 缩放

我们可以使用缩放来使得结点变大或者变小，默认情况下，结点在每个轴上的缩放值为1。我们将上面的矩形缩放1.5倍:

```java
private void transform(Rectangle box) {
	// 在这里写变换的代码,下面只展示这个方法
    box.setScaleX(1.5);
    box.setScaleY(1.5);
}
```

效果图:

![1.5倍放大](http://tva1.sinaimg.cn/large/006e5UvNgy1h6pm2f81ezj30kj0cxdgc.jpg)

### Rotate 旋转

代码:

```java
private void transform(Rectangle box) {
   // 先平移再旋转,比较明显 
   box.setTranslateX(100);
   box.setTranslateY(200);
    // 旋转30度
   box.setRotate(30);
}
```

效果:

![旋转效果图](http://tvax4.sinaimg.cn/large/006e5UvNgy1h6pm7nbjg5j308d0980sl.jpg)

## 事件处理 - Event Handling

事件通知，从字面意思理解就行，就是结点上出现了某事件，来通知监听的人。举一个例子，我们登录系统输入完用户名、密码，点击登录就有对应的监听将你输入的信息发送到服务端。事件通常是事件系统的原语。一般来说，一个事件系统有以下三个职责: 

- 触发事件
- 通知监听该事件的监听方
- 处理事件

事件通知有javaFx平台自动完成，我们就只考虑触发事件、监听事件以及如何处理他们就好了。

```java
public class EventApp extends Application {
    @Override
    public void start(Stage primaryStage) throws Exception {
        // 创造一个按钮
        Button button = new Button("监听点击事件");
        // 监听按钮点击事件,点击按钮输出helloworld
        button.setOnAction(o->{
            System.out.println("hello world");
        });
        // 
        Pane pane = new Pane(button);
        primaryStage.setScene(new Scene(pane,300,200));
        primaryStage.show();
    }
}
```

![效果图](http://tvax1.sinaimg.cn/large/006e5UvNgy1h6pmo4oz2oj308b06b743.jpg)

键鼠事件是最常见的，每个Node都有封装好的方法来监听处理这些事件。

##  写在最后

目前来说JavaFx的参考资料相对有点少，我在学习中也是一波三折，官网的开发者文档着实有点不友好，所幸在搜索引擎上以JavaFx 瞎搜，搜到了不错的参考资料:https://fxdocs.github.io/docs/html5/#_overview。但是这篇文档感觉还是不够面向初学者，好在是串了一条主线，本文是基于这篇文章，再加上了自己的理解的。



