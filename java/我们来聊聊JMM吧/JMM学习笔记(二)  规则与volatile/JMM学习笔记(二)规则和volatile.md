# JMM学习笔记(二) 规则和volatile

> 学生: 老师，我想请问为什么在月球上物体也会有坠落呢，和在地球上一样诶？
>
> 老师: 因为他们都遵循相同的规律, 我们根据规律就可以预测行为。但你观察到没有在月球上的坠落速度会慢一点。

## 到底什么是内存模型? 

这里我们来回忆一下《JMM 学习笔记(一) 跨平台的JMM》讲述的东西，在这篇文章里面有两条线, 第一条是硬件性能提升带来的问题，在单核时代，提升CPU的方向是优化架构性能和提升主频速度，但是遗憾的是主频并不能无限制的提升，主频提高过一个拐点之后，功耗会爆炸提升。但我们还需要更强、更快的CPU，多核是一剂良药，引入了多核以后，如何提升CPU运算性能的问题得到了解决，我们就可以通过多核来不断的提升CPU的性能了，某种程度上来说，我们也可以理解为提供给软件的计算资源在不断的增加，那摆在开发者面前的一个问题是如何更好的使用这些计算资源，如何提升计算资源的使用率。我们将结合操作系统的发展历史来回答这个问题。让我们从纸带计算机开始讲起，如下图所示

![图片一.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bd1466d5c8be4d0f9621ba999bd68043~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

一边读一边执行想来是那时的内存比较小，但是随着技术的发展，内存在慢慢的变大，高级语言开始出现，我们来观察一下一台计算机同时只能执行一个程序会遇到哪些问题, 我们先来看下下面一个很简单C语言小程序:

```c
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
// argv 用于接收以命令行方式启动传递的参数
int main(int argc , char* argv[])
{
    int to ,sum = 0;
    // 字符串转int
    to = atoi(argv[1]);
    // 打印传入的外部参数
    printf("sum: %d\n",to);
    // 获取output.txt的指针
    FILE* fp = fopen("output.txt", "w");
    // 获取当前时间
    clock_t start_time = clock();
    for(int i = 0 ; i <= to ; i++){
        sum = sum + i;
        // sum = sum + 1; 语句一
        // fprintf 将sum写入到fp指向的文件
        fprintf(fp,"%d",sum); // 语句二
    }
    fclose(fp); 
    clock_t end_time  = clock(); // 获取当前系统时间
    double total_time = (double)(end_time - start_time) / CLOCKS_PER_SEC;  // 计算总运行时间（秒
    printf("Total runtime: %.12f seconds\n", total_time);
}
```

然后我们以命令行方式启动这个程序:

![C语言执行](https://github.com/CXK6013/TechnicalBlog/assets/45529222/0ee0ac4a-d899-435e-819f-0a24076af254)


然后我们将语句二注释，将语句一解除注释，看下执行效果:

![C语言注释语句二](https://github.com/CXK6013/TechnicalBlog/assets/45529222/56bd8458-debf-4609-a705-823c4c8e20c4)


我们可以看到循环次数上升，只是将I/O语句注释，速度得到了飞快的提升，就几乎不需要花费时间一样，我执行了十几次，我将循环的次数提升到了1000w，但是速度仍然很快。我们来计算下语句二不注释的情况下，平均每次循环执行的时间是0.003/10^4^,  那我们将语句二注释换成语句一，那么花费的时间，由于分子是0，那么可以认为不需要花费时间？ 可能有朋友觉得还是不够直观我们就姑且取时间为10^-8^ , 然后我们的循环次数是10^8^, 那么花费的时间就是1 / 10^16^,  我们可以看到相差的数量级。0.003/10^4^是有I/O指令花费的时间。两者的比例近似在1:3 × 10^9^, 可以看到有I/O指令会非常慢, 换句话说一条I/O指令所花费的时间够计算指令执行3×10^9^次， 假设我们有如下一个程序: 

![image-20230602173830269](https://github.com/CXK6013/TechnicalBlog/assets/45529222/b8a6a90d-50c4-4c7d-80a4-34e98fc59e81)

蓝色区域的代表有3×10^9^条计算指令，绿色部分代表有一条I/O指令，处理器在执行完计算指令之后，就发出I/O指令，等磁盘上写入或从磁盘上读取，在这段时间内CPU处于空闲状态，利用率也就是百分之之五十，但实际的程序中我们碰到的问题可能是三十条计算指令，一条I/O指令，二十条计算指令，一条I/O指令，换句话说，如果计算机只能执行一个程序，那么CPU的使用率接近于零，换言之软件并没有享受到计算能力的提升，那我们该如何提升呢，答案就是并发，交替的执行程序，在等待I/O指令的时候，CPU切换到了其他程序上进行执行。就像我们取烧热水一样，我们将水灌进热水壶之后，打开开关，我们就会取做别的事情。那交替的执行会带来什么问题，我们先在有两个程序，都加载进了计算机开始执行: 

![V0MJPE.jpeg](https://i.imgloc.com/2023/06/03/V0MJPE.jpeg)

程序1执行到第53条指令，需要将CPU切换到程序2开始执行，程序2也有I/O指令，我们同样也不让CPU闲着，将CPU切回到程序1执行，那么这个时候CPU需要知道切换到程序2执行之前，这个程序执行到了哪里和指定到那个位置的数据(也就是程序执行到53条指令的时候，寄存器里面存储的数据)，也就是我们需要一块内存来记录运行中程序的相关数据，我们看到再引入并发提高CPU的利用率的时候，刻化运行中的程序就需要再引入一些新的概念，这是相对于静态的程序不一样的地方，这也就是进程。引入进程之后，CPU的使用率得到了提高，但遇到了新的问题 ，比如一个音乐播放软件，先从音频文件中读取数据，然后对数据进行解压缩，然后执行播放，我们这里用伪代码表示就是:

```java
main{
    while(true){
     	read(); // I/O  加载数据     
        decomress(); // CPU 解压
        play(); // 播放
    }
}
```

我们的思路是读一些放一些，但是I/O很慢，这就会导致我们听到的歌曲，唱一句等一些时间再唱一句，我们也可以想到将这三个功能拆成三个进程，先让读数据的进程先跑一段时间，再执行解压和播放进程，但是多进程之间共享资源可是个问题，除此之外，进程也需要消耗，我们能否提出一种实体，每一个实体共享地址空间，这些实体也交付给CPU并发的执行，这也就是线程，创建线程的代价要远小于进程切换代价，所需要的空间也远小于进程，但在软件领域就是这样我们引入一个机制去解决旧的问题，就会带来新的问题:

> In multiprocessor systems, processors generally have one or more layers of memory cache, which improves performance both by speeding access to data (because the data is closer to the processor) and reducing traffic on the shared memory bus (because many memory operations can be satisfied by local caches.).
>
> 在多处理器系统内，处理器通常有一级或多级内存，用来加速读写数据改善性能，因为缓存里CPU更近，又可以减少内存总线上的数据传输(因为一些内存操作可以由本地缓存来满足)

> Memory caches can improve performance tremendously, but they present a host of new challenges. What, for example, happens when two processors examine the same memory location at the same time? Under what conditions will they see the same value?
>
> 缓存可以极大的提升性能，但是也带来了一系列新挑战，比如当两个处理同时检查同一内存位置会发生什么? 在什么条件下它们会看到相同的值。《JSR 133 (Java Memory Model) FAQ》

我们可以将这个同一内存位置理解为多核CPU操纵共享变量，为了解决这个问题，就需要制定协议，这也就是缓存一致性协议，这类的协议有MESI、MSI、MOSI等，使用较为广泛的就是MESI协议，MESI解决了缓存一致性问题，但其实也存在一个性能弱点， 处理器执行写内存操作时，必须等待其他所有处理器将其告诉缓存中相应副本数据删除并接收到这些处理器回复的消息之后，才能将数据写入告诉缓存。为了规避和减少这种等待造成的写操作延迟，硬件设计者引入了写缓冲器和无效化队列。第二条是Java做为一个跨平台的语言, 该如何处理不同平台之间的差异。那这里的平台是什么意思？ 我们在学习Docker的时候也看到过平台这个词，我们来回忆一下Docker是如何介绍自己的:

> Docker is a platform for developers and sysadmins to build, run, and share applications with containers.
>
> Dokcer 是一个开发者和系统管理员构建、运行、共享容器应用的平台。

那Java跨的平台呢？ 在Oracle 出的教程《**The Java™ Tutorials**》是如是说的：

> A *platform* is the hardware or software environment in which a program runs.
>
> 平台是程序运行的软件或硬件环境。
>
>  We've already mentioned some of the most popular platforms like Microsoft Windows, Linux, Solaris OS, and Mac OS. Most platforms can be described as a combination of the operating system and underlying hardware.
>
> 我们已经提及了一些广泛使用的平台，像Windows、Linux、Solaris OS、MacOS。大多数平台可以被描述为操作系统和底层硬件的组合。
>
>  The Java platform differs from most other platforms in that it's a software-only platform that runs on top of other hardware-based platforms.
>
> Java平台和其他大多数平台的不同之处在于，它是一个纯软件平台，在其他基于硬件的平台之上运行。

 当我开始学习Java，Java向我承诺:

> Your applications are *portable* across multiple platforms. Write your applications once, and you never need to port them--they will run without modification on multiple operating systems and hardware architectures.
>
> 你的应用很容易可以跨平台，编写一次，不需要移植，无需修改，就能运行在多个不同的操作系统和硬件架构上。

当Java喊出那句“一次编写，到处运行”的口号时，Java面临的是一个差异化的世界，不同指令集架构的CPU，不同的操作系统，不同的处理器架构有着不同的内存模型，我们这里再来复习一下:

> 从处理器的角度来说，读内存操作的实质是从指定的RAM地址加载数据，因此这种内存操作也被称为Load，写内存操作的实质是将数据存储到指定地址表示的RAM存储单元中，因此内存操作通常被称为Store。所以，内存重排序实际上只有以下四种可能:

![iLa68v.jpeg](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4efbfecdf9a44493bb3dcfc12ce338cd~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

X86系列的处理器只存在最后一种问题，所以X86系列的处理器被称为具有强内存模型的处理器，其他平台的处理器则四种重排序都有可能出现，但是我们也不能完全禁止内存重排序，除非我们能够接受性能降低。这也就是并发问题的来源之一，有序性，有序性指的是在什么情况下一个处理器运行的一个线程所执行的内存访问操作在另外一个处理器上运行的其他线程看来是乱序的。所谓乱序，是指内存访问操作的顺序看起来像是发生了变化。我们再回忆一下JSR 133中的话:

> Memory caches can improve performance tremendously, but they present a host of new challenges. What, for example, happens when two processors examine the same memory location at the same time? Under what conditions will they see the same value?
>
> 缓存可以极大的提升性能，但是也带来了一系列新挑战，比如当两个处理同时检查同一内存位置会发生什么? 在什么条件下它们会看到相同的值。

这说的也就是可见性问题，现代计算机的执行单位是线程，上面的话换一句话说也就是一个线程对共享变量的更新在什么情况下对另一个线程可见。可见性是有序性的基础，有序性描述的是一个处理器上运行的线程对共享变量做所的更新，在其他处理上运行的其他线程是以什么样的顺序观察到这些更新。在有序性的基础上我们又希望如果我们需要，访问某个共享变量的操作从其执行线程以外的任何线程来看，该操作要么已经执行结束，要么尚未发生，即其他线程不会“看到” 该操作的中间效果，这也就是原子性。做最终要回答的问题还是，一个线程对共享变量的更新在什么情况下会另一个线程可见。

Java的想法是在各个不同的处理器内存模型之间建立一个属于Java的内存模型: 

> The Java Memory Model was an ambitious undertaking; it was the first time that a programming language specification attempted to incorporate a memory model which could provide consistent semantics for concurrency across a variety of architectures.
>
> Java 内存模型是一项雄心勃勃的工作，这是第一次有高级语言试图引入一个内存模型为各种架构的并发性提供一致的语义。
>
>  Unfortunately, defining a memory model which is both consistent and intuitive proved far more difficult than expected. JSR 133 defines a new memory model for the Java language which fixes the flaws of the earlier memory model. In order to do this, the semantics of final and volatile needed to change.  
>
> 不幸的是，定义一个一致又直观的内存模型比预想的要困难的多，JSR 133 为Java语言定义了一个新的内存模型，修复了早期内存模型的缺陷。为了做到这一点，final和volatile的语义需要改变。
>
> 《JSR 133》

Java内存模型的总体目标是:

> the goal of JSR 133 was to create a set of formal semantics that provides an intuitive framework for how volatile, synchronized, and final work.
>
> JSR 133的目标是创建一套形式化语义，为volatile、synchronized和final关键字的工作方式提供直观的框架。

更详细一点的目标是:

1. Preserving existing safety guarantees, like type-safety, and strengthening others. For example, variable values may not be created "out of thin air": each value for a variable observed by some thread must be a value that can reasonably be placed there by some thread.

> 保留现有的安全保障，如类型安全，并加强其他。例如变量的值不能“凭空产生”： 某个线程观察到的每个变量值都必须是某个线程可以合理设置的值。

2. The semantics of correctly synchronized programs should be as simple and intuitive as possible.

> 正确的同步语义应当是尽可能的简单而又直观的。

3. The semantics of incompletely or incorrectly synchronized programs should be defined so that potential security hazards are minimized.

> 对不完全或不正确的同步程序的愿意进行定义，以便将潜在的安全隐患降到最低。

4. Programmers should be able to reason confidently about how multithreaded programs interact with memory.

> 程序员应该能够自信地推理出多线程与内存的互动方式

5. It should be possible to design correct, high performance JVM implementations across a wide range of popular hardware architectures.

> 应该可以在广泛流行的架构上设计出正确、高性能的JVM实现。

6. A new guarantee of *initialization safety* should be provided. If an object is properly constructed (which means that references to it do not escape during construction), then all threads which see a reference to that object will also see the values for its final fields that were set in the constructor, without the need for synchronization.

> 应该提供一个新的初始化安全保证，如果一个对象是被正确创建(这意味着对它的引用在构造过程中没有被逃逸)，那么所有看到该对象的引用的线程也将看到在构造函数中设置字段的最终值，而不需要同步。

7. There should be minimal impact on existing code.

> 最小化对已有代码的影响

简单总结一下就是: 修复旧模型的bug的同时保证高性能，设计一套直观的框架让程序员可以直观的推导出来多线程与内存的交互方式。

## 内存模型定义了什么？

> The Java Memory Model describes what behaviors are legal in multithreaded code, and how threads may interact through memory. 

> 内存模型描述了多线程代码中的哪些行为是合法的，线程如何内存进行交互。

> Java includes several language constructs, including volatile, final, and synchronized, which are intended to help the programmer describe a program's concurrency requirements to the compiler. 
>
> Java 语言中提供了多个特性、机制，包括volatile、final和synchronized，帮助程序员向编译器描述并发需求。
>
> The Java Memory Model defines the behavior of volatile and synchronized, and, more importantly, ensures that a correctly synchronized Java program runs correctly on all processor architectures.
>
> Java内存模型定义了volatile 和synchronized的行为，更重要的是，确保在所有处理器架构上的程序能够被正确同步。

到这里我们可以看出Java内存模型为我们做的一切，当我们编写Java程序的时候，Java Memory Model屏蔽掉了各种硬件和操作系统的内存访问差异，Java内存模型给出了一组规则或规范， 定义了程序中各个变量(包括示例字段，静态字段和构成数组对象的元素)的访问方式，规范了Java虚拟机与计算机内存是如何协同工作的，JVM运行程序的实体是线程，而每个线程创建时JVM都会为其创建一个工作内存(有些文献也称为栈空间)，用于存储线程私有的数据，而Java内存模型中规定所有变量都必须存储在内存，主内存是共享内存区域，所有线程可以访问，但线程对遍历的操作(读取赋值等)必须在工作内存中进行，首先要将变量从主内存拷贝的自己的工作内存空间，然后对遍历进行操作，操作完成后再将变量写会主内存，不能直接操作主内存中的变量，工作内存中存储着主内存中的变量副本拷贝。工作内存是每个线程的私有数据区域，因此不同的线程无法访问对方的工作内存，线程间的通信(传值) 必读通过主内存来完成。

![VCwi1A.jpeg](https://i.imgloc.com/2023/06/04/VCwi1A.jpeg)

那一个线程对共享变量的更新在什么情况下对另一个线程可见？ JMM使用happens-before这个术语来回答。

## happens-before

happens发生，before之前，所以happens-before是之前发生，这里我们先换个角度来理解这个概念. 





## 进一步详解JMM





## 其他语言需要内存模型嘛？





## 写在最后







## 参考资料

- 清华 操作系统原理  https://www.bilibili.com/video/BV1uW411f72n?p=8&vd_source=aae3e5b34f3adaad6a7f651d9b6a7799
- JSR 133 (Java Memory Model)   http://www.cs.umd.edu/~pugh/java/memoryModel/jsr-133-faq.html
- 【哈工大】操作系统 李治军（全32讲） https://www.bilibili.com/video/BV19r4y1b7Aw?p=8&vd_source=aae3e5b34f3adaad6a7f651d9b6a7799






