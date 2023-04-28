# JMM 学习笔记(一) ISA与JMM

[TOC]

## 前言

其实关于并发、多线程这方面的文章，我已经写过了一些: 

- 《当我们说起多线程与高并发时》

- 《Java多线程学习笔记(一) 初遇篇》
- 《Java多线程学习笔记(二) 相识篇》
- 《Java多线程学习笔记(三) 甚欢篇》
- 《Java多线程学习笔记(五) 长乐无极篇》
- 《Java多线程学习笔记(六) 长乐未央篇》
- 《Java多线程编程范式(一) 协作范式》

《当我们说起多线程与高并发时》是总领全篇，我们从操作系统的发展讲起，为什么要有线程这个概念出现。《Java多线程学习笔记(一) 初遇篇》讲Java平台下的线程，如何使用和创建，以及引入线程后所面临的问题，为了解决线程安全问题，Java引入的机制，这也是《Java多线程学习笔记(二) 相识篇》讨论的问题,《Java多线程学习笔记(三) 甚欢篇》是讲线程协作，即如何让线程之间协作去处理任务，《Java多线程学习笔记(五) 长乐无极篇》讲了CompletableFuture，这个强大的异步编排组件，《Java多线程学习笔记(六) 长乐未央篇》 讲ForkJoin模式，《Java多线程编程范式(一) 协作范式》 讲使用Java提供并发核心库来解决一些问题。但是在《Java多线程学习笔记(一) 初遇篇》我们的讨论相对还比较粗糙，当时我的想法是先基本搭建一个模型来快速的熟悉Java的并发编程，在实践中先用起来，我们没有直接讨论线程安全，什么是线程安全，这个问题在当时的我去看，没有找到一个很完美的定义，还有并发模型，并发是难以验证的，那我们该如何验证，我们将统一收拢，统一回答这些问题。

## 从指令集架构谈起

单看指令集架构来说，这是一个有些相对陌生的名词，让我们从生活中稍微常见的事物讲起，也就是苹果电脑Mac，很多程序员都喜欢Mac，Mac中现在比较热的一款是Mac m1、m2了，喜欢苹果的人，对m1和m2相当喜欢，这里说的m1和m2也就是CPU的代称，这两款CPU的指令集架构是ARM，那什么是指令集架构？ 在回答这个问题的时候，我们还是要清楚《程序是如何运行的(一)》这篇文章的图: 

![图片九.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bc746191f0eb40589d4d5f379cb05972~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)



从这幅图我们可以看到指令集架构是硬件系统和软件的桥梁，连接了硬件和软件。那他是什么呢?

> An Instruction Set Architecture (ISA) is part of the abstract model of a computer that defines how the CPU is controlled by the software. The ISA acts as an interface between the hardware and the software, specifying both what the processor is capable of doing as well as how it gets done.
>
> 指令集架构是计算机抽象模型的一部分定义了CPU如何被软件控制。指令集充当硬件和软件之间的接合点，规定了处理器能够做什么，以及如何完成。
>
> The ISA provides the only way through which a user is able to interact with the hardware. It can be viewed as a programmer’s manual because it’s the portion of the machine that’s visible to the assembly language programmer, the compiler writer, and the application programmer.
>
> 指令集是用户和计算机之间进行交互的唯一途径。它可以被看做是程序员的手册，因为它是程序员(汇编语言程序员，编译器开发者、应用程序程序员) 可以看到的机器部分。
>
> The ISA defines the supported data types, the registers, how the hardware manages main memory, key features (such as virtual memory), which instructions a microprocessor can execute, and the input/output model of multiple ISA implementations. The ISA can be extended by adding instructions or other capabilities, or by adding support for larger addresses and data values.    
>
> 指令集架构定义了数据类型、寄存器、硬件如何管理主存，关键特性(如虚拟内存)，微处理器可以执行哪些指令，输入输出模型。指令集架构可以通过增加指令、其他功能、增加对更大地址和数据值的支持来进行扩展。
>
> --ARM官网

高级语言的一个普遍共性就是，内置了不同的数据类型，



## 跨平台面临的问题





## 出来吧，我的JMM





## JMM测试利器-JCStress



### JCStress为什么而生?



### 如何使用



## 参考资料

