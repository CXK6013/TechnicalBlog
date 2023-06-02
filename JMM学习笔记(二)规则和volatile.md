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

![C语言执行](D:\工作文档\C语言执行.png)

然后我们将语句二注释，将语句一解除注释，看下执行效果:

![C语言注释语句二](D:\工作文档\C语言注释语句二.png)

我们可以看到循环次数上升，只是将I/O语句注释，速度得到了飞快的提升，就几乎不需要花费时间一样，我执行了十几次，我将循环的次数提升到了1000w，但是速度仍然很快。我们来计算下语句二不注释的情况下，平均每次循环执行的时间是0.003/10^4^,  那我们将语句二注释换成语句一，那么花费的时间，由于分子是0，那么可以认为不需要花费时间？ 可能有朋友觉得还是不够直观我们就姑且取时间为10^-8^ , 然后我们的循环次数是10^8^, 那么花费的时间就是1 / 10^16^,  我们可以看到相差的数量级，近似在。0.003/10^4^是有I/O指令花费的时间。两者的比例是1:3 × 10^9^, 可以看到有I/O指令会非常慢, 换句话说一条I/O指令所花费的时间够计算指令执行3×10^9^次， 假设我们有如下一个程序: 

![image-20230602173830269](C:\Users\chenxingke\AppData\Roaming\Typora\typora-user-images\image-20230602173830269.png) 

蓝色区域的代表有3×10^9^条计算指令，绿色部分代表有一条I/O指令，处理器在执行完计算指令之后，就发出I/O指令，等磁盘上写入或从磁盘上读取，在这段时间内CPU处于空闲状态，利用率也就是百分之之五十，但实际的程序中我们碰到的问题可能是三十条计算指令，一条I/O指令，二十条计算指令，一条I/O指令，换句话说，如果计算机只能执行一个程序，那么CPU的使用率接近于零，换言之软件并没有享受到计算能力的提升，那我们该如何提升呢，答案就是并发，交替的执行程序，在等待I/O指令的时候，CPU切换到了其他程序上进行执行。就像我们取烧热水一样，我们将水灌进热水壶之后，打开开关，我们就会取做别的事情。那交替的执行会带来什么问题，我们先在有两个程序，都加载进了计算机开始执行:





。如果我们的程序执行的计算指令和I/O指令是:1，如下图所示:  



上面这个例子改编自B站操作系统课程《【哈工大】操作系统 李治军（全32讲）》P8，



引入进程

引入线程

多个CPU共同读取一个变量

由此引出JMM

但是在解决旧问题的同时也带来了新问题:   

> In multiprocessor systems, processors generally have one or more layers of memory cache, which improves performance both by speeding access to data (because the data is closer to the processor) and reducing traffic on the shared memory bus (because many memory operations can be satisfied by local caches.).
>
> 

> Memory caches can improve performance tremendously, but they present a host of new challenges. What, for example, happens when two processors examine the same memory location at the same time? Under what conditions will they see the same value?



第二条是Java做为一个跨平台的语言，要做到跨平台要解决的问题。





## 内存模型制定了规则









## Volatile







## 参考资料

- 清华 操作系统原理  https://www.bilibili.com/video/BV1uW411f72n?p=8&vd_source=aae3e5b34f3adaad6a7f651d9b6a7799
- JSR 133 (Java Memory Model)   http://www.cs.umd.edu/~pugh/java/memoryModel/jsr-133-faq.html
- 【哈工大】操作系统 李治军（全32讲） https://www.bilibili.com/video/BV19r4y1b7Aw?p=8&vd_source=aae3e5b34f3adaad6a7f651d9b6a7799







