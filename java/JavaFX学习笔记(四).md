# JavaFX学习笔记(四) 布局组件和UI组件的引入

[TOC]

## 前言

看本篇之前建议先看前作:

- 《JavaFx教程(一) 基本概念》
- 《JavaFX学习笔记(二) 关键特性与基本概念》
- 《JavaFX学习笔记(三) 架构与图形系统》

(一) 的主要内容是:

- JavaFX是什么? 
-  如何用IDEA建立一个JavaFX项目,
- 变换、缩放、渲染。

(二)的主要内容是关键特性和基本概念 , 其实这篇基本就是将Oracle出的教程进行了翻译(三也是)，在这篇中我们可以看到JavaFX提供的能力: 

- 2D和3D的图形化能力、图片操作API、Canvas
- UI组件, JavaFX 提供了丰富的UI组件。JavaFX也内置了统计图表，如果你想做一些统计图表，可以直接使用。
- 字体支持，你可以在JavaFX中替换字体、美化字体。
- 有了UI组件之后，那如何进行摆放呢，这也就是布局组件。
- 如果你想对组件进行美化，你也可以在JavaFX中使用CSS。
- 如果你不想使用在Java代码里面处理UI控件的样式，JavaFX提供FXML，可以在FXML写样式和UI组件，
- JavaFX也支持动效、过渡动画、3D、2D变换。
- 如果你想做一个播放器的话，JavaFX中也支持集成媒体。
- 如果你是一名Web前端工作者，那双向绑定你肯定熟悉，JavaFX也实现了，集合和UI控件实现绑定，集合更新的时候会自动刷新到UI控件，在JavaFX中实现这一点，要用指定的集合。
- 对于一些特殊文字，像希伯来文，JavaFx也做好了处理。
- JavaFX也做了对高DPI的支持，如果所处的平台有GPU，那么JavaFX将会启用硬件加速来让显示变得更加流畅。
- 如果你的应用本身已经用Swing或SWT，JavaFX也可以嵌入进来。只是JavaFX的渲染逻辑与事件处理方式与Swing、AWT不一样，这是在嵌入的时候需要注意的地方。

(三)的话则介绍了JavaFX平台的架构，暴露在最上层的是给开发者使用的API和场景图(场景图并非是JavaFX首创)，场景图的逻辑结构是一个层次分明的树，场景图中的元素除了根节点之外，每个结点都有一个父结点，一个或多个子结点。每个结点可以设置视觉效果、透明度、变换、处理事件。JavaFX的图形渲染会根据不同的平台选择不同的渲染方式，在不同的平台调用的渲染引擎不同，如果所处的平台有显卡或者强大的GPU，那么JavaFX会选择硬件加速来渲染图形，如果没有，JavaFX会进行软件渲染。这被称为Prism。处理事件在JavaFX平台中由Glass Windowing Toolkit负责，GWT使用的是操作系统的事件队列，这与AWT不一样, AWT维护了一个单独的事件队列，这种设计存在很大的缺陷。

在JavaFX应用中任何时候，至少活跃两个线程，JavaFX应用线程任何场景图必须由这个线程加载，管道渲染线程，这个线程支持并发渲染。多媒体线程负责在场景图中同步最新的帧。多媒体引擎负责处理媒体，Web引擎组件基于Webkit，这是一个开源的Web浏览器引擎，支持HTML5，CSS、JavaScript，DOM，SVG。得益于该引擎，在JavaFX中就可以享受到Web世界所提供的丰富内容。

以现在的观点来看(一)有些粗糙，对一个例子没有做详细介绍，就开始引出后面的移动、缩放、旋转。但本篇不是全盘推翻(一)，本篇是在(一)的基础上进行重构。这个系列最终的一个目的是构建起一个小型的学生成绩管理系统，在构建这个系统的过程中我们引入各种各样的组件，我只会在需要用到的时候才会介绍这个组件。

本系列的文章使用IDEA作为开发JavaFX的IDE，为了方便的使用Java领域的各种第三方类库，所以项目本身会用到maven，如果你不懂maven，可以先看下:

>  《Maven学习笔记》

这个系列的文章最好是熟练掌握JavaSE, 如果对JavaSE熟练掌握，建议先去将整个JavaSE看完。

在(一)里面我们构建JavaFX的maven项目是先用maven命令产生了一个模板，然后在用这个模板建立maven项目。其实IDEA也提供了对JavaFX的支持，如果你的IDEA版本在2021.2以上包括2021.2，IDEA也提供了JavaFX项目向导，方便快捷的构建一个JavaFX项目。

![新 JavaFX 项目向导](https://www.jetbrains.com/idea/whatsnew/2021-2/img/Java_JavaFXWizard.gif)

##  Hello World

各个语言中基本以在控制台输出Hello World当做第一个入门例子，我们也遵循这个传统，下面是我们在JavaFX世界中的Hello World。

```java
public class HelloWorld extends Application {
    @Override
    public void start(Stage stage) throws IOException {
        Button button = new Button(); // 语句一
        button.setText("hello world"); // 语句二
        button.setOnAction(event ->{
            System.out.println("hello world");
        }); // 语句三
        StackPane root = new StackPane(); // 语句四
        root.getChildren().add(button); // 语句五
        Scene scene = new Scene(root,600,600); // 语句六
        stage.setTitle("hello world"); // 语句七
        stage.setScene(scene); // 语句八
        stage.show(); // 语句九
    }
    public static void main(String[] args) {
        launch();
    }
}
```

效果如下图所示:



当我们点击按钮，控制台会输出hello world。HelloWorld这个类继承了Application(javafx.application.Application)，重写的start方法是所有JavaFX应用的主要入口。在上面的程序中，在语句一种我们声明了一个按钮, 在语句二中我们为按钮设置了显示内容hello world,  在语句三中我们为按钮的点击事件绑定了行为，当点击按钮，会在控制台输出helloworld。语句四我们声明了一个布局容器StackPane，在JavaFX中由各种各样的布局容器，StackPane会将装入的控件堆叠显示，向下面这样:



### 小小总结一下



##  表格组件





## 登录





##  小小成型





## 参考资料

