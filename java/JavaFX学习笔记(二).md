# JavaFX学习笔记(二) JavaFx的架构



## 前言

按照我JavaFX系列的计划应该是: 

- (一) 介绍基本概念 
- (二) 布局与UI控件示例 
- (三) 一个小的桌面系统 
- (四) JavaFX 打包
- (四) FXGL(JavaFX的游戏引擎)入门与基本概念
- (五) 常用示例
- (六) 小型游戏系统的构建

以前觉得JavaFX的资料少，但是后面在搜索引擎瞎搜，就搜到了oracle写到用户指导:

![image-20221214142704397](https://user-images.githubusercontent.com/45529222/208052880-ea8bbe46-b021-4c68-8b5f-4b1fed94c8fd.png)


https://docs.oracle.com/javase/8/javase-clienttechnologies.htm

在这个用户指导里面你可以看到JavaFX的架构、基本概念、示例都有，甚至我还找到了Java 8的官方示例(JavaFX 和 Java8的):

![image-20221214143158559](https://user-images.githubusercontent.com/45529222/208052975-3e30e8e2-8e88-4f60-9c59-6e6d609eecbe.png)


https://www.oracle.com/java/technologies/downloads/

仔细一看其实官方出的指导还是很清晰的，我大致对其做了翻译:

- Getting Started with JavaFX

  - What Is JavaFX

  > 什么是JavaFX?

  - Get Started with JavaFX

  > 开启学习JavaFX之旅

  - Get Acquainted with JavaFX Architecture 

  > 熟悉javaFX的架构

  - Deployment Guide

- Graphics (图形化)

  - Getting Started with JavaFX 3D Graphics (3D图形化)
  - Use the Image Ops API (图片操作API)
  - Work with Canvas (画布)

- User Interface Components (UI组件)

  - Work with UI Controls (使用UI组件)

  > 下拉框、输入框、菜单，JavaFX都封装好了。

  - Create Charts (创建图表)

  > 统计图表

  - Add Text (字体)

  > 炫酷字体

  - Add HTML Content (支持HTML)

  > JavaFX 支持内嵌了一个浏览器解析引擎，所以我们可以在JavaFX使用Web世界的东西

  - Work with Layouts

  > 布局, 组件如何摆放

  - Skin Applications with CSS

  > 使用CSS皮肤

  - Build UI with FXML

  > 使用FXML构建UI控件

  - Handle Events

  > 事件处理

- Effects, Animation, and Media（动效、动画、音频）

  - Create Visual Effects (制作直观动效)
  - Add 2D & 3D Transformations (2D、3D变换)
  - Add Transitions & Animation (过渡动画)
  - Incorporate Media(集成媒体)

- Application Logic(应用逻辑)

  - Work with the Scene Graph(使用Scene graph)
  - Use Properties and Binding(使用属性和绑定)
  - Work with Collections(使用集合框架)

- Interoperability (交互性)

  - Use Concurrency and Threads (并发与线程)
  - Integrate JavaFX and Swing (JavaFx和Swing集成)
  - Integrate JavaFX and SWT(JavaFX和SWT集成)

- Reference (引用)

  - JavaFX API Documentation (JavaFX 接口文档)
  - CSS Reference Guide (CSS 引用指南)
  - Introduction to FXML (FXML简介）

我的评价是这份用户指南构建的非常良好，有demo有示例。我这里是对其内容理解的方式进行重构，在介绍概念方面一般是翻译JavaFX的文档，这些概念是必要的，也很简单直观。

## JavaFX 是什么？



### 概述

>  JavaFX is a set of graphics and media packages that enables developers to design, create, test, debug, and deploy rich client applications that operate consistently across diverse platforms.
>
> 我的翻译: JavaFX是一组图形化和多媒体工具包，可让开发者设计、创建、测试、调试和部署跨平台的客户端应用程序。
>
> chatGPT的翻译是: JavaFX是一套图形和媒体包，可让开发人员设计、创建、测试、调试和部署在各种平台上一致运行的丰富客户端应用程序。
>
> JavaFX是一个用于开发跨平台图形化应用程序的工具包。



### JavaFX Applications

> Since the JavaFX library is written as a Java API, JavaFX application code can reference APIs from any Java library. For example, JavaFX applications can use Java API libraries to access native system capabilities and connect to server-based middleware applications.
>
> JavaFX库使用Java来编写，所以JavaFX中可以使用任何Java库的代码。例如JavaFX应用可以使用Java库和操作系统进行交互、连接服务端应用程序。
>
>  The look and feel of JavaFX applications can be customized. Cascading Style Sheets (CSS) separate appearance and style from implementation so that developers can concentrate on coding. Graphic designers can easily customize the appearance and style of the application through the CSS. 
>
> JavaFX的界面外观支持自定义，可以使用CSS将外观与样式分离的实现可以让开发人员专心编码。设计师可以轻松的使用CSS定制应用程序的外观和样式。
>
> If you have a web design background, or if you would like to separate the user interface (UI) and the back-end logic, then you can develop the presentation aspects of the UI in the FXML scripting language and use Java code for the application logic.
>
> 如果你有网页开发北京，或者你想将UI处理逻辑与用户界面显示进行分离，那么你可以选择用FXML脚本语言来构建页面，使用Java来构建UI控制逻辑。
>
> If you prefer to design UIs without writing code, then use JavaFX Scene Builder. As you design the UI, Scene Builder creates FXML markup that can be ported to an Integrated Development Environment (IDE) so that developers can add the business logic.
>
> 如果你并不想使用写代码的方式来构建用户界面，那么这里可以使用JavaFX Scene Builder，通过拖拽的形式设计好用户界面之后可以直接导出FXML给开发人员使用，开发人员只需要在UI界面后面添加处理逻辑即可完成客户端应用程序的构建。

在翻资料的过程中，在B站看到有up主吐槽这个Scene Builder 难用，这里我们到时候试一下，看看新版本的是否那么难用。

### Availability 可用的

> The JavaFX APIs are available as a fully integrated feature of the Java SE Runtime Environment (JRE) and the Java Development Kit (JDK ). 
>
> JavaFX API是作为Java SE运行时环境和Java开发工具包的一项完全集成功能提供。
>
> Because the JDK is available for all major desktop platforms (Windows, Mac OS X, and Linux), JavaFX applications compiled to JDK 7 and later also run on all the major desktop platforms.
>
> 因为JDK能够运行在所有主流的桌面平台(Windows, Mac OS X, Linux)，因此编译到JDK7版本以上包括JDK 7的JavaFX应用程序也可以在主流版本上运行。
>
>  Support for ARM platforms has also been made available with JavaFX 8. JDK for ARM includes the base, graphics and controls components of JavaFX.
>
> JavaFX 8还提供了对ARM平台的支持，JDK FOR ARM包括JavaFX基础、 图形和控件。
>
> The cross-platform compatibility enables a consistent runtime experience for JavaFX applications developers and users.
>
> 跨平台兼容性让JavaFX开发人员和用户在运行时可以享受一致的体验。

### Key Features

这里只介绍有用的:

- **FXML and Scene Builder**. FXML is an XML-based declarative markup language for constructing a JavaFX application user interface. A designer can code in FXML or use JavaFX Scene Builder to interactively design the graphical user interface (GUI). Scene Builder generates FXML markup that can be ported to an IDE where a developer can add the business logic.

> 

- **WebView**. A web component that uses WebKitHTML technology to make it possible to embed web pages within a JavaFX application. JavaScript running in WebView can call Java APIs, and Java APIs can call JavaScript running in WebView. Support for additional HTML5 features, including Web Sockets, Web Workers, and Web Fonts, and printing capabilities have been added in JavaFX 8. See [Adding HTML Content to JavaFX Applications](https://docs.oracle.com/javase/8/javafx/embedded-browser-tutorial/overview.htm#JFXWV135).
- **Swing interoperability**. Existing Swing applications can be updated with JavaFX features, such as rich graphics media playback and embedded Web content. The `SwingNode` class, which enables you to embed Swing content into JavaFX applications, has been added in JavaFX 8. See the [SwingNode API javadoc](https://docs.oracle.com/javase/8/javafx/api/) and [Embedding Swing Content in JavaFX Applications](https://docs.oracle.com/javase/8/javafx/interoperability-tutorial/embed-swing.htm#JFXIP566) for more information.
- **Built-in UI controls** **and CSS**. JavaFX provides all the major UI controls that are required to develop a full-featured application. Components can be skinned with standard Web technologies such as CSS. The DatePicker and TreeTableView UI controls are now available with the JavaFX 8 release. See [Using JavaFX UI Controls](https://docs.oracle.com/javase/8/javafx/user-interface-tutorial/ui_controls.htm#JFXUI336) for more information. Also, the CSS Styleable* classes have become public API, allowing objects to be styled by CSS.
- **Modena theme****.** The Modena theme replaces the Caspian theme as the default for JavaFX 8 applications. The Caspian theme is still available for your use by adding the `setUserAgentStylesheet(STYLESHEET_CASPIAN)` line in your Application start() method. For more information, see the [Modena blog](http://fxexperience.com/2013/01/modena-new-theme-for-javafx-8/) at fxexperience.com
- **3D Graphics Features**. The new API classes for `Shape3D` (`Box, Cylinder, MeshView, and Sphere` subclasses), `SubScene, Material, PickResult, LightBase (AmbientLight` and `PointLight` subclasses), and `SceneAntialiasing` have been added to the 3D Graphics library in JavaFX 8. The `Camera` API class has also been updated in this release. For more information, see the [Getting Started with JavaFX 3D Graphics](https://docs.oracle.com/javase/8/javafx/graphics-tutorial/javafx-3d-graphics.htm#JFXGR256) document and the corresponding [API javadoc](https://docs.oracle.com/javase/8/javafx/api/) for `javafx.scene.shape.Shape3D`, `javafx.scene.SubScene, javafx.scene.paint.Material, javafx.scene.input.PickResult`, and `javafx.scene.SceneAntialiasing`.
- **Canvas API**. The Canvas API enables drawing directly within an area of the JavaFX scene that consists of one graphical element (node).
- **Printing API.** The `javafx.print` package has been added in Java SE 8 release and provides the public classes for the [JavaFX Printing API](https://docs.oracle.com/javase/8/javafx/api/).
- **Rich Text Support**. JavaFX 8 brings enhanced text support to JavaFX, including bi-directional text and complex text scripts, such as Thai and Hindu in controls, and multi-line, multi-style text in text nodes.
- **Multitouch Support**. JavaFX provides support for multitouch operations, based on the capabilities of the underlying platform.
- **Hi-DPI support.** JavaFX 8 now supports Hi-DPI displays.
- **Hardware-accelerated graphics pipeline**. JavaFX graphics are based on the graphics rendering pipeline (Prism). JavaFX offers smooth graphics that render quickly through Prism when it is used with a supported graphics card or graphics processing unit (GPU). If a system does not feature one of the recommended GPUs supported by JavaFX, then Prism defaults to the software rendering stack.
- **High-performance media engine**. The media pipeline supports the playback of web multimedia content. It provides a stable, low-latency media framework that is based on the GStreamer multimedia framework.
- **Self-contained application deployment** **model.** Self-contained application packages have all of the application resources and a private copy of the Java and JavaFX runtimes. They are distributed as native installable packages and provide the same installation and launch experience as native applications for that operating system.



## JavaFX的架构





## UI 控件介绍





## 总结一下



## 参考资料

- Java Platform, Standard Edition (Java SE) 8 Client Technologies https://docs.oracle.com/javase/8/javase-clienttechnologies.htm
