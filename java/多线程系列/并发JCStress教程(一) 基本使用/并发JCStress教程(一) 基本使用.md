# 多线程测试利器JCStress教程(一) 基本使用

## 前言

这段时间在写JMM、Volatile相关的文章，重新梳理一下自己对Java中并发模型的理解，我以前的想法是将自己看到的精炼出来一条主线，然后顺势叙述这样。大致就是从缓存一致性协议到写缓冲区与无效化队列，最后过渡到JMM，但是发现这样写来文章有点诘屈聱牙，所以我就想调整思路，能否从一个小的可复现的例子引出问题再过度到缓存一致性协议，再到JMM。但是我很想找到一种验证的工具，验证多线程问题的工具，比如指令重排序等等，因为我的想法是理论的预测，最终也要经过实践的检验方才能够让人信服。OpenJDK下面有一个JcStress的项目就可以满足我们的需求，一般碰到一个新的框架，我现在的习惯是直接看文档，然然后再尝试写示例，先学会用再说。

但是在网上搜罗良久，发现JCStress并没有对应的开发者文档，但是我在GitHub上看这个项目有示例的模块，我觉得有示例也行，于是将这个项目下载下来，但是让我惊讶的是大量的示例都缺少一个类：

![II_Result](http://tvax2.sinaimg.cn/large/006e5UvNly1h62rxfup10j30qn0gv0ta.jpg)

去各大技术群尝试询问，也是没找到答案，遂放弃直接从原始材料中直接提取资料的想法。本周在研究这个框架的过程中，发现直接引用JCStress的maven依赖就可以看到这个类，有点满头问号的感觉。但是我们的主线是找到一个验证多线程下程序的工具，这个问题我们就放到后面研究。

## 从Hello World开始

```
mvn install:install-file -DgroupId=org.openjdk.jcstress -DartifactId=jcstress-core -Dversion=1.0-SNAPSHOT -Dpackaging=jar -DgeneratePom=true -Dfile=D:\work\jcstress-core-1.0-SNAPSHOT.jar
```



