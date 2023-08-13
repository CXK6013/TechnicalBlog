# 且谈软件架构(二)  模块化与MVC

[TOC]

## 前言

我一贯不喜欢手册式的文章，就告诉你一些定律、经验，我更愿意完整的告诉我的经验，我的理论是如何得出的，读我的文章，就好像在和我进行交谈，本篇可以认为是经验之谈，所谓经验不是定理，就是这些经验部分具备普适性，部分不具备普适性，具体情况要具体分析。本身本篇的标题是从MVC走向DDD，主要还是在掘金看到了转转技术团队的《转转价格系统DDD实践》这篇文章，其中提到:

> 在使用传统的mvc模式下，我们往往使用三层架构，即controller，service，dao或者其类似的方式。这种架构会把所有的业务逻辑堆积在service之下，领域实体只做数据传输，没有行为。随着项目的迭代，可能出现service臃肿的情况，大量业务逻辑，把service搞成一个胖子，业务逻辑就会变得混乱不堪，理解和维护成本极大。

但是从MVC走向DDD，这个标题给人一种感觉是DDD一定优于MVC，事实上对于那种不会进行过多迭代的小型系统而言，没必要进行DDD，在《且谈软件架构》中我们将软件发展类比到城市发展，像一座城市，经济快速发展会吸引人，越来越多的人涌入城市，城市变得拥挤，城市开始着手扩建，那么对于软件也是，庞大的用户量涌入，软件开始追求越来越多强的性能，越来越强的可靠性，所以对于告诉发展的城市，会有乒乓乓乓的声音，那是城市的心脏跳动的声音，建筑的声音越多，这座城市就越是在发展，同样的如果没有那么多居住人口，就没有必要对城区进行扩建，扩建了也用不上这也就是浪费,。不同的城市有不同的发展历程，也就有着不同的产业结构，所谓的结构我们也在《且谈软件架构》中探讨过了，结构即架构，当我在搜索引擎搜索Architecture，第一个出来的就是建筑相关的词条:

![](https://a.a2k6.com/gerald/i/2023/08/12/5hwzf.png)

所以这个架构就是跟建筑学借的词，是软件行业对于建筑行业的学习和借鉴，用一个物理、实在的“建筑”来比喻一个抽象、虚拟的软件系统。在维基百科中Architecture的意思是:

> Architecture is the art and technique of designing and building, as distinguished from the skills associated with construction
>
> 建筑是设计与构建的技术与艺术，有别于建筑的相关的技能。

Architecture 也被译为架构，所以架构就是设计与构建的技术与艺术，这里有两个我们熟悉的单词，design 和 build，设计与构建，产品进行需求设计，开发根据需求设计进行代码设计，而自动化构建工具则将我们代码发布，变成用户可使用的服务，这里面有一个又一个的构建，也有一个又一个的设计。在设计上就可以衍生出艺术，从最初的有一个庇护之所，到追求舒适赏心悦目，我在搜索引擎搜索艺术这个关键字，顺着链接点进去看到了一幅画:

![](https://a.a2k6.com/gerald/i/2023/08/12/41hf.png)

我在看到这幅画的时候，我的心感到很平静，连带遭遇的被强迫的无意义加班带来的焦灼情绪也舒缓了很多，我的鼠标在滑动的时候，划到的叶子也在跟着动，这幅画的主题是In Rhythm With Nature, 也就是与自然和谐相处，这幅画的灵感来自于Carl Linnaeus的花钟，Linnaeus是著名的植物学家和分类学家，他建立了一套现代生物识别、命名和分类系统。他的独特花园设计利用了不同植物的自然昼夜节律，通过观察它们花朵的开放和闭合来标记一天中的时间。与我们的植物朋友一样，人类也会通过昼夜节律对环境变化做出反应。这个位于大脑中的 24 小时内部时钟帮助调节我们的睡眠-觉醒周期。从上午到下午，我们会进入一个越来越警觉的状态，然后在傍晚降低警觉，为夜间睡眠做准备。与自然共舞》将视听与您的当地时间同步，从而尊重我们的昼夜节律。 

现代人的睡眠大多没有那么节律，一般都是在手机里面刷无可刷，才会放下手机准备睡觉，想起了一句诗: 久在樊笼里，复得返自然。所以艺术是什么? 

> 指凭借技巧、意愿、想象力、经验等综合人为因素的融合与平衡，以创作隐含美学的器物、环境、影像、动作或声音的表达模式，也指和他人分享美的感觉或有深意的情感与意识的人类用以表达既有感知且将个人或群体体验沉淀与展现的过程。 来自维基百科

在技术之上派生了艺术，就像建筑，有些材料放在哪里，可以被人建造出舒适美观的建筑，可以被建造成丑陋而又让人不适的建筑，那对于一个软件呢，如果有对应的美学指导，我想应该也是有艺术的，对于用户直接接触的部分有着对应的设计美学指导，界面的显示无疑是艺术的，这方面也有商业的权衡，比如迅雷的安卓版就塞满了广告，很难用，这已经不是艺术不艺术的问题了，是很难用的问题。 那么我们将目光依然拉回到构建界面的代码呢，代码构建的页面本身是一种服务，所以速度要快，在代码层面上我更认为是实用美学，复用性、稳定性、可靠性、可维护性、可扩展性构成了代码美学的六个点，如果我们要讨论代码的艺术的话，我认为都是基于这六个点来讨论，那么为了达到这六个点，我们采用的手段一般也就是分层，分层就是分类，根据不同的功能将代码分散到各个组件功能中，分类降低了复杂度，降低了大脑的符合，这一方面分层或分类做的比较成功的是网络模型的分层，每层负责不同的功能。那软件代码中是如何分层的呢? 

## MVC

首先我是一名Java后端程序员，Java是我的主语言，我所熟悉的也就是服务端开发，这里谈的也就是我的经验，当我刚学Java EE的时候，前端代码和后端代码放在一起，我们用包对代码进行管理，不同的功能放在不同的包里，不同的功能放在不同的包里面, 那什么是包?  我想起刚学Java的时候，那是在大几的时候，我那个时候还只会C和C++，我在一个文件夹下写代码，到后面学Java也是，我没有建包，我在Eclipse默认的文件夹下面写代码，后面看别人建了包，在包下面建代码，我也有样学样，我开始建包，在包下面建class，但是刚开始我写的类都没有一个主题或者相似的地方，所以我只有一个包，到后面我开始学Servlet、JSP、JDBC，代码量越来越多了，我开始分包，包像一个房间一样容纳不同的功能: 

![](https://a.a2k6.com/gerald/i/2023/08/08/7206c.png)

上面我大致上分了三个包，包的最后一个单词就是这个包的功能，controller代表控制器，dao和service代表model，jsp所在的文件夹属于view，这也就是MVC:

![](https://bhimgs.com/i/2023/08/09/j4fv57.png)

图片来自于mdn web docs，视图层也就是用户直接接触的界面，定义数据该如何显示，控制器接收用户请求，将请求转发给模型层。 上面举了一个三层交互的例子，用户点击加入购物车, 视图层发送请求到控制器这一层，控制器这一层从视图层接收到请求，通知对应的模型去添加，模型中定义了数据结构，添加数据，然后更新视图层对应的显示。 在那个刚学Java EE的时候，我就通过包来进行功能分层，这么做的目的是避免一个包里面放太多代码，就像居住的房子，我们根据不同的需要将不同的房间承担不同的职能，如果一个房间承担了太多职能，比如杂物间承担了卧室、厨房、杂物间的功能，那么进入这个房间的人就会觉得这个房间乱糟糟的，更关键在于不可预测的反应，可能不小心就会碰到锅碗瓢盆。

> 所有堆积如山的东西，都是不可预测的。
>
> 简化系统的首选方法，就是将一个大系统，转变为多个更小的子系统组成的系统。 《系统、数学和爆炸》

这也是我们分层的目的，降低复杂度，将一个大系统，转变为多个更小的子系统组成的系统。 转变为更小的系统还有一个妙用就是方便复用，就像是一个方法主体提供了A功能，但是在实现A功能的时候调用了B功能，在方法中也就体现为十几行代码，但是只想要用到方法中的B功能就没办法剥离出来，他们捆在一起。随着功能的丰富，一个项目下会有越来越多的功能，越来越多的包，这些包也可以进行划分，对外提供功能，在后端领域我们一般使用maven来构建项目，一般我们会有一个项目总的文件夹，然后下面的文件夹层级是:

```maven
src/main/包
src/resources 
```

这些包对外提供一个主体功能，但是随着功能的迭代，功能在逐渐变多，但是如果外部想要使用的话，还是以jar的形式，但是如果我只想使用部分功能呢，我就得不得不接受所有，我们的结构像下面这样:	

![](https://s3.bhimgs.com/i/2023/08/11/sf5to1.png)

以fastJson这个json框架为例，我们对他的期待也就是对象转json，json转对象了，但是fast-json 的依赖有:

```
<dependency>
        <groupId>javax.servlet</groupId>
            <artifactId>javax.servlet-api</artifactId>
            <version>3.1.0</version>
            <scope>provided</scope>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>javax.ws.rs</groupId>
            <artifactId>javax.ws.rs-api</artifactId>
            <version>2.0.1</version>
            <scope>provided</scope>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.apache.cxf</groupId>
            <artifactId>cxf-rt-transports-http</artifactId>
            <version>3.1.2</version>
            <scope>provided</scope>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.apache.cxf</groupId>
            <artifactId>cxf-rt-frontend-jaxrs</artifactId>
            <version>3.1.2</version>
            <scope>provided</scope>
            <optional>true</optional>
        </dependency>

        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-websocket</artifactId>
            <version>4.3.7.RELEASE</version>
            <scope>provided</scope>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-webmvc</artifactId>
            <version>4.3.7.RELEASE</version>
            <scope>provided</scope>
            <optional>true</optional>
        </dependency>

        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-messaging</artifactId>
            <version>4.3.7.RELEASE</version>
            <scope>provided</scope>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.data</groupId>
            <artifactId>spring-data-redis</artifactId>
            <version>1.8.6.RELEASE</version>
            <scope>provided</scope>
            <optional>true</optional>
        </dependency>

        <dependency>
            <groupId>com.squareup.retrofit2</groupId>
            <artifactId>retrofit</artifactId>
            <version>2.5.0</version>
            <scope>provided</scope>
        </dependency>
        
        <dependency>
            <groupId>com.squareup.okhttp3</groupId>
            <artifactId>okhttp</artifactId>
            <version>3.6.0</version>
            <scope>provided</scope>
        </dependency>

        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-spring-web</artifactId>
            <version>2.6.1</version>
            <scope>provided</scope>
            <optional>true</optional>
        </dependency>

        <dependency>
            <groupId>io.javaslang</groupId>
            <artifactId>javaslang</artifactId>
            <version>2.0.6</version>
            <scope>provided</scope>
        </dependency>

        <dependency>
            <groupId>org.glassfish.jersey.core</groupId>
            <artifactId>jersey-common</artifactId>
            <version>2.23.2</version>
            <scope>provided</scope>
        </dependency>

        <dependency>
            <groupId>joda-time</groupId>
            <artifactId>joda-time</artifactId>
            <version>2.10</version>
            <scope>provided</scope>
        </dependency>

        <dependency>
            <groupId>net.sf.trove4j</groupId>
            <artifactId>core</artifactId>
            <version>3.1.0</version>
            <scope>provided</scope>
        </dependency>

        <dependency>
            <groupId>org.javamoney.moneta</groupId>
            <artifactId>moneta-core</artifactId>
            <version>1.3</version>
            <scope>provided</scope>
        </dependency>
   <dependency>
            <groupId>io.airlift</groupId>
            <artifactId>slice</artifactId>
            <version>0.36</version>
            <scope>provided</scope>
 </dependency>
```

注意javax.ws.rs-api这个依赖，这个依赖定义了一组接口，来自于JSR-311，旨在定义一个统一的规范使得Java程序员可以使用一套固定的接口来开发REST应用，具体的实现由第三方框架完成，javax.ws.rs通过spi机制来加载对应的实现，那这部分功能否单独做个依赖出来呢?  也就是将fastjson根据能力再进行拆分， 哪些是fastjson最基本的能力，哪些是其他扩展，我看了下fastjson2就是用这种思路将其进行拆分:

![](https://a.a2k6.com/gerald/i/2023/08/13/4lt6.jpg)

里面有core、扩展。我想起高中生物生命系统的结构层次:

![](https://a.a2k6.com/gerald/i/2023/08/13/wzu3.jpg)

对于代码的发展也是这样，最初是类，然后将类联合起来是包，然后包形成模块有特定的职能。模块化着眼于分离职责，增强复用性。JDK 在9也进行了模块化，JDK也是类似的思路进行拆分，JDK 模块化的目标是:

- 可靠的配置 — 模块提供了在编译期和执行期加以识别模块之间依赖项的机制，系统可以通过这些依赖项确保所有模块的子集合能满足程式的需求。
- 强封装 — 模块中的包只有在模块显式导出时可供其他模块访问，即使如此，其他模块也无法使用这些包，除非它明确指出它需要其他模块的功能。由于潜在攻击者只可以访问较少的类别，因此提高了平台的安全性。您会发现，模块化可帮助您做出更简洁、更合乎逻辑的设计。
- 可扩展的 Java 平台 — 在这之前，Java 平台是由大量的包组成的庞然大物，使其难以开发、维护和发展，还不能轻易地被子集化。现在，平台被模块化为 95 个模块（此数字可能随着 Java 的发展而变化），您可以创建定制运行时，其中仅包含应用或您定位的设备所需的模块。例如，如果设备不支持 GUI，您可以创建一个不包含 GUI 模块的运行时，显著降低运行时的大小。
- 更佳的平台完整性 — 在 Java 9 之前，您可以使用平台中许多不预期供应用使用的类别，通过强封装，这些内部 API 真正会被封装并隐藏在使用该平台的应用中。如果您的代码依赖这些内部 API，则可能会对转移旧有代码到以模块化的 Java 9 造成问题。
- 提高性能 — JVM 使用各种优化技术来提高应用性能。JSR 376 表明，当预先知道某技术仅在特定模块中被使用，这些技术会更有效。

Java平台是由大量的包组成的庞然大物，难以开发、维护和发展，不能定制运行时，仅包含我所需的模块。fastjson 1也是这个毛病，好在fastjson2就开始拆分，根据不同的职能，将不同的包分拆到不同的模块中。Java平台也是选中了一个最小的运行时，也就是最基础的base模块，jdk的其他模块都需要这个模块提供的能力，jdk各个模块之间的图如下所示:

![](https://a.a2k6.com/gerald/i/2023/08/13/vtg4.png)

## 微服务与模块化

回忆一下我们在《写给小白看的Spring Cloud入门教程》讲的微服务，我们选择微服务架构的原因是代码量在不断的变大，我们期待更强的可伸缩性，可扩展性，所以我们选择了微服务，根据业务将我们的系统拆成若干个大小适当的微服务，然后我们就开始借助于Spring Cloud 和 Spring Boot 来搭建微服务，我们用PPT化的架构图是很漂亮的: 

![](https://a.a2k6.com/gerald/i/2023/08/13/nwy.jpg)

但是这幅图也有缺陷，缺陷就在于没画出各个服务之间的关系，比如A服务需要和B服务通信，那么接口在B服务提供，如果入参和出参是一个类的话，双方就需要共享这一部分，那么由于A服务和B服务不在一个模块，那么A服务该如何引用到B服务的数据类型呢，在Java中主要是类，在B服务中复制一份，复制到A里面，那如果将来B原来提供的接口不满足于A的要求，需要添加字段，那B和A都需要跟着改动，那如果B提供的接口被七个微服务所使用，那么这七个微服务都需要改？ 这太恐怖了吧，这不还是耦合在一起了嘛，说好的解耦合呢，更聪明的方式是，每个服务内部专门建一个模块，专门给外部调用。那这样每个服务又是一个不大不小的单体服务:

![](https://a.a2k6.com/gerald/i/2023/08/13/6okzb.jpg)

api模块被其他服务所引用，而api模块又需要引用基础模块的能力，或者你也可以根据自己的需要，定义粒度更细的依赖，让给外部调用的依赖做到轻一些。将微服务拆分到适当的大小，这是一句很美妙的话，人们常说，不要太大，也不要太小，但这是空话，我该根据什么原则来将微服务拆到合适的大小呢。这个问题的一个答案叫领域模型。我们完全没有可能在一篇文章里完全将使用领域模型来拆分微服务讲清楚，这里我们先引出这个问题，后文我们会进行细致的讨论。

## 总结一下

随着代码量的不断增大，一个项目逐步在具备一个又一个功能，也在不断的变得复杂，一个项目具备越来越多的功能也不利于复用:

> 所有堆积如山的东西，都是不可预测的。简化系统的首选方法，就是将一个大系统，转变为多个更小的子系统组成的系统。 《系统、数学和爆炸》

所以我们有了许多方法来拆分，微观上不属于这个类的职责被剥离出去，宏观上，根据一定的功能将包分拆出去，比如fastjson，剔除自己的扩展，有一个最核心的依赖，这样方便复用。我曾经将这种拆分方法应用到拆分微服务中，但是与朋友交流之后发现微服务更倾向于独立、自治，各个服务之间应当是平等的，不应当存在核心包，基础包，每个微服务倾向于自治，尽可能的少依赖第三方，这也是领域建模要回答的问题，如何恰到好处的拆分微服务。

## 参考资料

[1]  架构、构架、结构、框架之间有什么区别？ https://www.zhihu.com/question/32105413

[2]  互联网协议入门(一)  https://www.ruanyifeng.com/blog/2012/05/internet_protocol_suite_part_i.html

[3] Entity层、DAO层、Service层、Controller层 https://www.jianshu.com/p/133f80c5af9b

[4] Modules, not microservices  https://news.ycombinator.com/item?id=34230641

[5] 拜托，别在 agent 中依赖 fastjson 了 https://mp.weixin.qq.com/s/ZYSiPGBQZLljZE0ESMM2tg

[6]  译 软件中的安全漏洞是什么? 献给外行人的软件漏洞指南  https://mp.weixin.qq.com/s/gLRBVLGguW196pzBLXBDSw

[7] 据报道称“浏览器内核有上千万行代码”，浏览器内核真的很复杂吗？ https://www.zhihu.com/question/290767285/answer/1200063036

[8] 《系统、数学和爆炸》*https://pjonori.blog/posts/systems-math-explosions/*
