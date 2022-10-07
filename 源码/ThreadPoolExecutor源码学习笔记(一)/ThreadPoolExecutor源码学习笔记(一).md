# ThreadPoolExecutor 学习笔记(一) 池化的思想

> 本篇是我们源码系列的第二篇, 看ThreadPoolExecutor，即线程池的源码，我们本篇主要探究线程池是怎么实现池化的。后面我们会探究常见的连接池:
>
> 数据库连接池、http连接池。对比他们的实现，我们来体会池化的思想。我们本篇不做笔记式的看源码，主要的思路是用一个问题连接起整个文章的脉络。
>
> 本篇我们提出的问题主要是，线程池是如何实现池化的。

[TOC]

## ThreadPoolExecutor概述

在Java中如何创建线程呢？ 可能面试题会让你这么回答:

- 继承Thread 
- 实现Runnable接口
- Callable
- ThreadPoolExecutor

但我看来在Java中创建线程的方式只有一种，就是new Thread().start()。 线程并不专属于Java，它是一个操作系统的概念，创建线程，需要调用操作系统提供的接口，才能创建线程，继承Thread只是创建一个Java世界的类而已，Runnable、Callable也是，ThreadPoolExecutor,也是用过Thread.start方法来创建线程。我们知道，线程是有自己的生命周期的，这里不再赘述，那线程池是如何不让线程死亡的呢?  本期我们就来走进ThreadPoolExecuror，探究这个问题. 

我们先大致来看ThreadPoolExecutor上面的注释:

![ThreadPoolExecutor概览](http://tvax4.sinaimg.cn/large/006e5UvNly1h3cdurfp6zj30p90i618u.jpg)

作者Doug Lea，似乎Java里面的并发库来自于这位老爷子，这位老爷子一向是注释比代码多，既然在这个变量上打了这么多注释，那么相比这个变量比较重要，我们重点来看一下这个注释。ctl是一个原子类型的变量，有两个字段与之关联，一个是workerCount, 这个是正在处于运行状态的线程数目。为了将workerCount收拢进int型中，我们限制了线程的数量，2的29次方-1个(悄悄的吐槽一下，这个Doug Lea 你不要多想，完全够用的)。如果这个在未来出了什么问题，我们也可以改为Long类型的原子类，并调整移位和掩码常数。但是在这样的需要出现之前，用int会更快更简单(此处看了一下JDK 19的协程)，

>  The workerCount is the number of workers that have been permitted to start and not permitted to stop. The value may be transiently different from the actual number of live threads, for example when a ThreadFactory fails to create a thread when asked, and when exiting threads are still performing bookkeeping before terminating



## 如何复用线程









## 写在最后





## 参考资料

