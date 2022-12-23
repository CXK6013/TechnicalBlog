# JavaFX学习笔记(二) 布局与UI控件

[TOC]

## 概述

![架构](http://tvax3.sinaimg.cn/large/006e5UvNgy1h96xtuhd7uj30f904xq50.jpg)



上图是说明了JavaFX 平台各个各个架构组件，我们将简要描述每个组件的作用以及各个组件之间的联系。最上层是Java FX 公开API(给开发者使用的)Scene Graph。在Java FX 公开API下面是运行JavaFX 代码的引擎，这个引擎由JavaFX 高性能图形引擎: Prism, 一个小巧而又高巧的窗口系统: Glass。一个多媒体引擎，一个Web引擎。尽管这些组件对外部不看见，下面这些描述可以让你理解JavaFX应用是如何运行的。

##  Scene Graph (场景图)

- The JavaFX scene graph, shown as part of the top layer in [Figure 2-1](https://docs.oracle.com/javase/8/javafx/get-started-tutorial/jfx-architecture.htm#BABDFFDG), is the starting point for constructing a JavaFX application. It is a hierarchical tree of nodes that represents all of the visual elements of the application's user interface. It can handle input and can be rendered.

  JavaFX 场景图是上图的最上层，是构建JavaFX应用的起点，场景图的结构是一颗分层的结点树，所有应用程序用户界面的所有可视元素都在上面。它可以处  理输入，并且可以渲染。

- A single element in a scene graph is called a node. Each node has an ID, style class, and bounding volume. With the exception of the root node of a scene graph, each node in a scene graph has a single parent and zero or more children. It can also have the following:

  场景图的单个元素我们称之为结点，每个结点都有ID、样式、边界体积。场景图中的元素除了根结点之外，每个结点都有一个父节点，一个或个子结点。

  - Effects, such as blurs and shadows

    ​    视觉效果(如模糊和阴影)

  - Opacity

    透明度

  - Transforms

    ​    变换

  - Event handlers (such as mouse, key and input method)

    ​    事件处理(键鼠 输入)

  - An application-specific state

    应用程序的状态

- Unlike in Swing and Abstract Window Toolkit (AWT), the JavaFX scene graph also includes the graphics primitives, such as rectangles and text, in addition to having controls, layout containers, images and media.

  与Swing和AWT不同，JavaFX场景图包含基本的图形基元(如矩形、文本)。除此之外还有空间、布局容器，图像、多媒体，

- For most uses, the scene graph simplifies working with UIs, especially when rich UIs are used. Animating various graphics in the scene graph can be accomplished quickly using the javafx.animation APIs, and declarative methods, such as XML doc, also work well.

​			通常来说，场景图可以简化使用UI组件的工作，特别是当需要使用复杂UI的时候。使用Javafx.animation 的APIS和声明式方法(XML)可以快速的场景图快速			完成。		

- The `javafx.scene` API allows the creation and specification of several types of content, such as:

  javafx.scene API允许创建和指定不同类型的内容，举例如下:

  - **Nodes**: Shapes (2-D and 3-D), images, media, embedded web browser, text, UI controls, charts, groups, and containers

    结点: 形状(2D和3D)、图像、媒体、嵌入式浏览器、文本、UI图表、容器

  - **State**: Transforms (positioning and orientation of nodes), visual effects, and other visual state of the content

    变换(结点的位置、方向)，视觉效果和其他视觉状态

  - **Effects**: Simple objects that change the appearance of scene graph nodes, such as blurs, shadows, and color adjustment

    改变场景图结点的简单对象，如模糊、阴影、色彩调整。

- Java Public APIs for JavaFX Features

  The top layer of the JavaFX architecture shown in [Figure 2-1](https://docs.oracle.com/javase/8/javafx/get-started-tutorial/jfx-architecture.htm#BABDFFDG) provides a complete set of Java public APIs that support rich client application development. 架构图上最底层的部分提供了一组公开APIs支持客户端应用的开发。下面是JavaFX的特性:

  - Allow the use of powerful Java features, such as generics, annotations, multithreading, and Lamda Expressions (introduced in Java SE 8).

    ​	可以用强大的Java特性，比如泛型、注解、多线程、Lambda表达式

  - Make it easier for Web developers to use JavaFX from other JVM-based dynamic languages, such as Groovy and JavaScript.

    其他使用基于JVM平台的语言Groovy、JavaScript，掌握JavaFX会更加容易。

  - Allow Java developers to use other system languages, such as Groovy, for writing large or complex JavaFX applications.

    允许Java开发者使用其他语言来构建大型、复杂的JavaFX应用。

  - Allow the use of binding which includes support for the high performance lazy binding, binding expressions, bound sequence expressions, and partial bind reevaluation. Alternative languages (like Groovy) can use this binding library to introduce binding syntax similar to that of JavaFX Script.    支持绑定、高性能的延迟绑定，绑定序列表达式、部分绑定重新刷新。其他语言像Groovy可以使用绑定库来实现和JavaFX类似的语法。

  -   Extend the Java collections library to include observable lists and maps, which allow applications to wire user interfaces to data models, observe changes in those data models, and update the corresponding UI control accordingly.

  ​        阔扎

## Graphics System(图形系统)

The JavaFX Graphics System, shown in blue in [Figure 2-1](https://docs.oracle.com/javase/8/javafx/get-started-tutorial/jfx-architecture.htm#BABDFFDG), is an implementation detail beneath the JavaFX scene graph layer. It supports both 2-D and 3-D scene graphs. It provides software rendering when the graphics hardware on a system is insufficient to support hardware accelerated rendering.

上面架构图中的蓝色部分是JavaFX的图形系统，	

### Prism(棱镜)

Two graphics accelerated pipelines are implemented on the JavaFX platform:

JavaFX实现了两种图形加速管道

- **Prism** processes render jobs. It can run on both hardware and software renderers, including 3-D. It is responsible for rasterization and rendering of JavaFX scenes. The following multiple render paths are possible based on the device being used:

  Prism处理渲染工作，支持硬件和软件渲染，包括3D。它负责JavaFX场景图的栅格化和渲染，在不同的设备，JavaFX会唤起不同的渲染引擎

  - DirectX 9 on Windows XP and Windows Vista

     在 Windows XP and Windows Vista 使用DirectX 9 

  - DirectX 11 on Windows 7

  ​     在 Windows7 上使用 使用DirectX 911

  - OpenGL on Mac, Linux, Embedded

  ​     Max和Linux上唤起OpenGL

  - Software rendering when hardware acceleration is not possible

    当没有硬件加速的时候，会使用软件渲染。

    The fully hardware accelerated path is used when possible, but when it is not available, the software render path is used because the software render path is already distributed in all of the Java Runtime Environments (JREs). This is particularly important when handling 3-D scenes. However, performance is better when the hardware render paths are used.

    如果可以尽可能使用硬件加速，但是当硬件加速不被支持，软件渲染就会被启用，因为软件渲染已经集成在JRE。当处理3D渲染，硬件加速性能会更好

- **Quantum Toolkit** ties Prism and Glass Windowing Toolkit together and makes them available to the JavaFX layer above them in the stack. It also manages the threading rules related to rendering versus events handling.

​	 量子工具包将Prism和Glass Windowing Toolkit结合起来，并暴露接口给外部使用。并处理显示和事件处理之间的线程规则。

###  Glass Windowing Toolkit

> The Glass Windowing Toolkit, shown in beige in the middle portion of [Figure 2-1](https://docs.oracle.com/javase/8/javafx/get-started-tutorial/jfx-architecture.htm#BABDFFDG), is the lowest level in the JavaFX graphics stack. Its main responsibility is to provide native operating services, such as managing the windows, timers, and surfaces. It serves as the platform-dependent layer that connects the JavaFX platform to the native operating system.

Glass 是玻璃的意思，这里暂时没想到什么好的翻译, Glass Windowing Toolkit 使用缩写GWT代表，GWT在上面架构图中位于中间部分，在JavaFX的图形栈中属于底层。主要负责本机操作系统服务，比如管理窗口、定时器和显示。他连接了JavaFX平台和操系统。

> The Glass Windowing  Toolkit is also responsible for managing the event queue. Unlike the Abstract Window Toolkit (AWT), which manages its own event queue, the Glass toolkit uses the native operating system's event queue functionality to schedule thread usage. Also unlike AWT, the Glass toolkit runs on the same thread as the JavaFX application. In AWT, the native half of AWT runs on one thread and the Java level runs on another thread. This introduces a lot of issues, many of which are resolved in JavaFX by using the single JavaFX application thread approach.

 GWT还负责管理事件队列，但不像AWT自己做了一个事件队列一样，GWT使用的是操作系统的事件队列来使用线程进行处理。和AWT不一样，GWT和JavaFX在一个线程中执行。在AWT中则分为两个线程。这种设计存在很大的缺陷。在JavaFx中使用单个线程来解决这些问题。

- Threads

> The system runs two or more of the following threads at any given time：
>
> 任何时候在JavaFX应用中至少活跃着两个线程
>
> - **JavaFX application thread**: This is the primary thread used by JavaFX application developers. Any ”live” scene, which is a scene that is part of a window, must be accessed from this thread. A scene graph can be created and manipulated in a background thread, but when its root node is attached to any live object in the scene, that scene graph must be accessed from the JavaFX application thread. This enables developers to create complex scene graphs on a background thread while keeping animations on 'live' scenes smooth and fast. The JavaFX application thread is a different thread from the Swing and AWT Event Dispatch Thread (EDT), so care must be taken when embedding JavaFX code into Swing applications.
>
>   JavaFx应用线程:  这是开发人员使用的主要线程。任何活动的场景(窗口中的场景)必须由这个线程访问。场景图可以被后台线程创建和操作，但是一旦将场景图的根结点附加到场景中的实时对象，就必须从JavaFX应用线程去执行。这有助于开发者人员创建复杂的场景图，同时让处于显示的场景动画保持流畅和快速。JavaFX应用线程与Swing和AWT事件分发线程不同，因此将JavaFX代码嵌入到Swing应用程序中时，必须小心。
>
> - **Prism render thread**: This thread handles the rendering separately from the event dispatcher. It allows frame N to be rendered while frame N +1 is being processed. This ability to perform concurrent processing is a big advantage, especially on modern systems that have multiple processors. The Prism render thread may also have multiple  threads that help off-load work that needs to be done in rendering.
>
> ​      管道渲染线程与事件分发线程是两个线程，支持并发渲染，这种并发渲染能力是一个相当大的优势，特别是在拥有多处理器的现代计算机系统来说。
>
> ​      管道渲染线程可能会有到个渲染线程，这些线程有助于分担渲染所需的工作。
>
> - **Media thread**: This thread runs in the background and synchronizes the latest frames through the scene graph by using the JavaFX application thread.
>
>   多媒体线程，这个线程运行在后台，通过使用JavaFX使用线程在场景图中同步最新的数据(帧)。

- Pulse

> A pulse is an event that indicates to the JavaFX scene graph that it is time to synchronize the state of the elements on the scene graph with Prism. A pulse is throttled at 60 frames per second (fps) maximum and is fired whenever animations are running on the scene graph. Even when animation is not running, a pulse is scheduled when something in the scene graph is changed. For example, if a position of a button isD changed, a pulse is scheduled.

   pulse(脉冲)是一种JavaFX场景图的事件，当此事件触发就代表场景图需要Prism再次渲染。该事件的最高频率为每秒最高60帧，在场景图中运行动画时触发。即使没有运行动画，场景图中的某些内容也会触发脉冲，例如按钮的位置发生了改变，则会触发脉冲事件。

> When a pulse is fired, the state of the elements on the scene graph is synchronized down to the rendering layer. A pulse enables application developers a way to handle events asynchronously. This important feature allows the system to batch and execute events on the pulse.
>
> 脉冲被触发时，场景图上的元素状态将再次被渲染(或译为同步到渲染层)。脉冲为匮乏人员提供了一种异步处理事件的方法。我们可以在脉冲上批处理和执行事件。
>
> Layout and CSS are also tied to pulse events. Numerous changes in the scene graph could lead to multiple layout or CSS updates, which could seriously degrade performance. The system automatically performs a CSS and layout pass once per pulse to avoid performance degradation. Application developers can also manually trigger layout passes as needed to take measurements prior to a pulse.
>
> 布局和CSS也脉冲事件相关。场景图中的大量修改操作可能导致布局和CSS多次被更新，这会严重的降低性能。系统会在脉冲被触发的时候，自动的再次渲染布局和CSS。开发人员也可以手动触发重新布局。以便在脉冲触发之前观察组件的样式、位置。
>
> The Glass Windowing Toolkit is responsible for executing the pulse events. It uses the high-resolution native timers to make the execution.
>
> GWT使用高解析定时器(由操作系统)来处理脉冲事件。这就意味着JavaFX可以在较短的时间间隔触发脉冲，从而使场景图上的动画流畅且精确。

我其实觉得我的翻译是有问题，但是也许是我没接触过GUI开发，这个词确实是存在的，具体的在微软的开发者文档有所提及，但我还是不大理解。

###  Media and Images

> JavaFX media functionality is available through the `javafx.scene.media` APIs. JavaFX supports both visual and audio media. Support is provided for MP3, AIFF, and WAV audio files and FLV video files. JavaFX media functionality is provided as three separate components: the Media object represents a media file, the MediaPlayer plays a media file, and a MediaView is a node that displays the media.
>
> JavaFX的多媒体功能位于javafx.scene.media下。JavaFX支持音频和视频。支持的视频格式有MP3，AIFF，WAV音频文件。FLV视频文件。
>
> JavaFX提供了三种组件: Media类代表媒体文件，MediaPlayer负责播放多媒体文件。MediaView是负责显示多媒体。
>
> The Media Engine component, shown in green in [Figure 2-1](https://docs.oracle.com/javase/8/javafx/get-started-tutorial/jfx-architecture.htm#BABDFFDG), has been designed with performance and stability in mind and provides consistent behavior across platforms. For more information, read theIncorporating Media Assets into JavaFX Applicationsdocument.
>
> 多媒体引擎组件是上面架构图中的绿色部分，高性能、稳定性强且跨平台。如果你想获得更多相关信息，参看http://www.oracle.com/pls/topic/lookup?ctx=javase80&id=JFXMD

###  Web Component

> The Web component is a JavaFX UI control, based on Webkit, that provides a Web viewer and full browsing functionality through its API. 
>
> Web组件是一个基于Webkit的JavaFx UI控件,通过API提供Web浏览器的功能。
>
> This Web Engine component, shown in orange in [Figure 2-1](https://docs.oracle.com/javase/8/javafx/get-started-tutorial/jfx-architecture.htm#BABDFFDG), is based on WebKit, which is an open source web browser engine that supports HTML5, CSS, JavaScript, DOM, and SVG. It enables developers to implement the following features in their Java applications:
>
> Web 引擎组件是上面架构图的橘黄色部分，基于WebKit，WebKit是一个开源的Web浏览器引擎，支持HTML5，CSS、JavaScript，DOM，SVG。能够让开发者享用以下特性。
>
> - Render HTML content from local or remote URL
>
> ​    从本地或者URL中渲染HTML内容。
>
> - Support history and provide Back and Forward navigation
>
>   支持历史记录并提供后退、前进导航。
>
> - Reload the content
>
>   重新加载内容 
>
> - Apply effects to the web component
>
> ​    应用动效于Web组件。
>
> - Edit the HTML content
>
>      编辑HTML内容
>
> - Execute JavaScript commands
>
> ​      执行JavaSceipt命令
>
> - Handle events
>
>   事件处理
>
> This embedded browser component is composed of the following classes:
>
>  集成的浏览器组件在下面这些类里
>
> - `WebEngine` provides basic web page browsing capability.
>
> ​     WebEngin提供基本的web页面浏览功能
>
> - `WebView` encapsulates a WebEngine object, incorporates HTML content into an application's scene, and provides fields and methods to apply effects and transformations. It is an extension of a `Node` class.
>
> ​    WebView封装了WebEngine对象，将HTML内容并入应用的场景图。并提供应用效果和转换字段和方法，扩展了Node类。
>
> In addition, Java calls can be controlled through JavaScript and vice versa to allow developers to make the best of both environments. For more detailed overview of the JavaFX embedded browser, see the Adding HTML Content to JavaFX Applications document.
>
> 除此之外，Java可以调用JavaScript，JavaScript也可以调用Java。更多的的细节在JavaFX Embedded brower。参看Adding HTML Content to JavaFX Applications document. https://docs.oracle.com/javase/8/javafx/embedded-browser-tutorial/overview.htm#JFXWV135

- CSS

> JavaFX Cascading Style Sheets (CSS) provides the ability to apply customized styling to the user interface of a JavaFX application without changing any of that application's source code.
>
> JavaFX可以使用CSS提供的能力定制用户界面，且不用修改任何应用的源代码。
>
>  CSS can be applied to any node in the JavaFX scene graph and are applied to the nodes asynchronously.
>
> CSS可以被应用于JavaFX场景图中的任何结点，支持异步应用。
>
>  JavaFX CSS styles can also be easily assigned to the scene at runtime, allowing an application's appearance to dynamically change.
>
> CSS样式也可以在运行时应用，允许应用外观动态改变。
>
> [Figure 2-2](https://docs.oracle.com/javase/8/javafx/get-started-tutorial/jfx-architecture.htm#BABICCFD) demonstrates the application of two different CSS styles to the same set of UI controls.
>
> 下面的图在使用了两种不同的CSS样式在一组相同的UI控件中。
>
> Figure 2-2 CSS Style Sheet Sample
>
> ![Description of Figure 2-2 follows](https://docs.oracle.com/javase/8/javafx/get-started-tutorial/img/css-style-sample.gif)
> [Description of "Figure 2-2 CSS Style Sheet Sample"](https://docs.oracle.com/javase/8/javafx/get-started-tutorial/img_text/css-style-sample.htm)
>
> JavaFX CSS is based on the W3C CSS version 2.1 specifications, with some additions from current work on version 3. The JavaFX CSS support and extensions have been designed to allow JavaFX CSS style sheets to be parsed cleanly by any compliant CSS parser, even one that does not support JavaFX extensions.
>
> JavaFX 的CSS基于W3C CSS 2.1版本，并从CSS 3版本添加了一些额外的内容。JavaFX CSS提供了对标准CSS的扩展,任何符合CSS规范的解析器都可以解析这些扩展，即时这些解释器不支持JavaFX对CSS的扩展,符合标准的CSS解析器也能解析。
>
>  This enables the mixing of CSS styles for JavaFX and for other purposes (such as for HTML pages) into a single style sheet. All JavaFX property names are prefixed with a vendor extension of ”`-fx-`”, including those that might seem to be compatible with standard HTML CSS, because some JavaFX values have slightly different semantics.
> 我们可以将JavaFX的CSS样式和嵌入网页的样式放入一个CSS文件中。所有JavaFX的CSS样式属性名都会带有一个-fx-前缀,包括那些可能与标准HTML、CSS语义看起来相同的属性，因为他们在JavaFX有不同的语义。(比如字体大小,Web中用font-size,JavaFX用-fx-font-size来表示,原因在于JavaFX中的字体大小和Web中略有不同)
>
> For more detailed information about JavaFX CSS, see the  Skinning JavaFX Applications with CSS document.
> 更多JavaFX相关的细节信息,参看Skinning JavaFX Applications with CSS document.这份文档,下面是地址。
>
> https://docs.oracle.com/javase/8/javafx/user-interface-tutorial/css_tutorial.htm#JFXUI733

- UI Controls

> The JavaFX UI controls available through the JavaFX API are built by using nodes in the scene graph. They can take full advantage of the visually rich features of the JavaFX platform and are portable across different platforms. JavaFX CSS allows for theming and skinning of the UI controls.

JavaFX的控件可以用JavaFX API的在场景图中使用nodes来构建。可以充分利用JavaFx平台的丰富特性和跨平台。JavaFX CSS目前可以对UI控件的皮肤和主题进行设置。下面是示例

> [Figure 2-3](https://docs.oracle.com/javase/8/javafx/get-started-tutorial/jfx-architecture.htm#CHDCDBCA) shows some of the UI controls that are currently supported. These controls reside in the `javafx.scene.control` package
> 这些控件在javafx.scene.control这个包下面
>
> Figure 2-3 JavaFX UI Controls Sample
>
> ![Description of Figure 2-3 follows](https://docs.oracle.com/javase/8/javafx/get-started-tutorial/img/uicontrols.png)
> [Description of "Figure 2-3 JavaFX UI Controls Sample"](https://docs.oracle.com/javase/8/javafx/get-started-tutorial/img_text/uicontrols.htm)
>
> For more detailed information about all the available JavaFX UI controls, see the Using JavaFX UI Controls and the API documentation for the `javafx.scene.control` package.
> 更多关于JavaFX UI控件的信息参看Using JavaFX UI Controls and the API documentation这个章节
> 想要了解更多的JavaFX UI控件的信息，参看Using JavaFX UI Controls(http://www.oracle.com/pls/topic/lookup?ctx=javase80&id=JFXUI)和API 文档(https://docs.oracle.com/javase/8/javafx/api/)

- Layout

> Layout containers or panes can be used to allow for flexible and dynamic arrangements of the UI controls within a scene graph of a JavaFX application. The JavaFX Layout API includes the following container classes that automate common layout models:
>在JavaFX的场景图中, 布局容器和pane(玻璃板,觉得这么译不合适)可以灵活动态的调整UI控件。JavaFX的布局API包括以下常见自动化容器布局模型，下面是对应的类
>
> - The `BorderPane` class lays out its content nodes in the top, bottom, right, left, or center region.
> 边框窗格的布局分为上下左右和中心区域。
> - The `HBox` class arranges its content nodes horizontally in a single row.
> HBox 类将结点水平的放在一行。
> - The `VBox` class arranges its content nodes vertically in a single column.
VBox 将结点垂直放在一列
> - The `StackPane` class places its content nodes in a back-to-front single stack.
> StackPane class 将结点从后往前放在它的堆栈中。
> - The `GridPane` class enables the developer to create a flexible grid of rows and columns in which to lay out content nodes.
GridPane Clss允许开发者在行和列中排放结点。
> - The `FlowPane` class arranges its content nodes in either a horizontal or vertical ”flow,” wrapping at the specified width (for horizontal) or height (for vertical) boundaries.
> FlowPane class将内容结点按水平或垂直方向流式排列，在指定的宽度或高度，内容结点到达边界后自动换行。
> - The `TilePane` class places its content nodes in uniformly sized layout cells or tiles
> TilePane Class将内容结点放在大小相同的单元格中。
> - The `AnchorPane` class enables developers to create anchor nodes to the top, bottom, left side, or center of the layout.
>AnchorPane class可以让开发者在布局的上下左右创建锚结点。
> To achieve a desired layout structure, different containers can be nested within a JavaFX application.
> 为了实现所需的布局结构，可以在JavaFX应用程序中嵌套不同的容器。
> To learn more about how to work with layouts, see the [Working with Layouts in JavaFX](http://www.oracle.com/pls/topic/lookup?ctx=javase80&id=JFXLY) article. For more information about the JavaFX layout API, see the API documentation for the `javafx.scene.layout` package.
> 了解更多布局相关的信息,可以参看Working with Layouts in JavaFX。更多布局相关的API,参看javafx.scene.layout包的接口文档。

- 2-D and 3-D Transformations

> Each node in the JavaFX scene graph can be transformed in the x-y coordinate using the following `javafx.scene.tranform` classes:
> JavaFX场景图中每个结点都可以使用javafx.scene.tranform中的类进行在x-y坐标上进行变换。
> - `translate` – Move a node from one place to another along the x, y, z planes relative to its initial position.
 沿着x,y,z将一个结点从一个位置移动到另一个位置
> - `scale` – Resize a node to appear either larger or smaller in the x, y, z planes, depending on the scaling factor.
> 缩放,依赖于缩放因子，在x、y、z上改变结点的大小。
> - `shear` – Rotate one axis so that the x-axis and y-axis are no longer perpendicular. The coordinates of the node are shifted by the specified multipliers.
> 
> - `rotate` – Rotate a node about a specified pivot point of the scene.
> 在场景图对结点进行旋转。
> - `affine` – Perform a linear mapping from 2-D/3-D coordinates to other 2-D/3-D coordinates while preserving the 'straight' and 'parallel' properties of the lines. This class should be used with `Translate`, `Scale`, `Rotate`, or `Shear` transform classes instead of being used directly.
> 仿射 
> To learn more about working with transformations, see the [Applying Transformations in JavaFX](https://docs.oracle.com/javase/8/javafx/visual-effects-tutorial/transforms.htm#JFXTE139) document. For more information about the `javafx.scene.transform` API classes, see the [API documentation](https://docs.oracle.com/javase/8/javafx/api/).
理解更多变换的信息。
- Visual Effects

> The development of rich client interfaces in the JavaFX scene graph involves the use of Visual Effects or Effects to enhance the look of JavaFX applications in real time. The JavaFX Effects are primarily image pixel-based and, hence, they take the set of nodes that are in the scene graph, render it as an image, and apply the specified effects to it.
>
> Some of the visual effects available in JavaFX include the use of the following classes:
>
> - `Drop Shadow` – Renders a shadow of a given content behind the content to which the effect is applied.
> - `Reflection` – Renders a reflected version of the content below the actual content.
> - `Lighting` – Simulates a light source shining on a given content and can give a flat object a more realistic, three-dimensional appearance.
>
> For examples on how to use some of the available visual effects, see the [Creating Visual Effects](https://docs.oracle.com/javase/8/javafx/visual-effects-tutorial/visual_effects.htm#JFXTE191) document. For more information about all the available visual effects classes, see the [API documentation](https://docs.oracle.com/javase/8/javafx/api/) for the `javafx.scene.effect` package.

## 参考资料

[1] 高解析度计时器 https://learn.microsoft.com/zh-cn/windows-hardware/drivers/kernel/high-resolution-timers
