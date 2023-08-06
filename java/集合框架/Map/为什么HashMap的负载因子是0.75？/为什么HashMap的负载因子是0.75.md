# HashMap的0.75可能只是一个经验值

[TOC]

## 前言

还是要面对HashMap的，这是个高频面试点，以前本身想着一口气讲投HashMap的，但是一口气讲投HashMap想来非常消耗肺活量，篇幅也让人生畏，所以将其分拆为几篇，每篇是独立的主题，最后又将主题合并起来。本篇就来看HashMap, 看的就是HashMap的构造函数:

```java
static final float DEFAULT_LOAD_FACTOR = 0.75f;
public HashMap() {
   this.loadFactor = DEFAULT_LOAD_FACTOR; 
}
```

这也是HashMap的一道面试题，为什么HashMap的负载因子选择了0.75，有人从代码注释上入手:

> As a general rule, the default load factor (.75) offers a good tradeoff between time a. space costs. Higher values decrease the space overhead but increase the lookup cost (reflected in most of the operations of the HashMap class, including get and put). The expected number of entries in the map and its load factor should be taken into account when setting its initial capacity, so as to minimize the number of rehash operations. If the initial capacity is greater than the maximum number of entries divided by the load factor, no rehash operations will ever occur.
>
> 一般而言，默认负载因子是0.75在时间和空间的成本上进行了很好的权衡，过大虽然会减少空间开销，但是会增加查找开销(反应在HashMap的大多数操作中，包括get和put)。设置初始容量的时候，预期的键值对数目和负载因子应当被考虑，避免过度扩容。如果初始容量大于预期的最大键值对除以负载因子，就会发生扩容操作。

这好像说了什么，又好像什么都没说，在0和1之间有很多数字，为什么偏偏就选中了0.75。在python里面也有类似于HashMap的集合，Python中是0.762，在Go中是0.65，Dart中是0.68。好像都在0.7附近。网上有其他答案是从泊松分布入手的，从泊松分布入手的大概是没有好好看HashMap的注释:

> Because TreeNodes are about twice the size of regular nodes, we  use them only when bins contain enough nodes to warrant use (see TREEIFY_THRESHOLD). 
>
> 因为TreeNode的大小约是普通结点的两倍，所以我们使用TreeNode仅当在桶上有足够的结点才会去使用(TREEIFY_THRESHOLD)，
>
> And when they become too small (due to removal or resizing) they are converted back to plain bins.  In  usages with well-distributed user hashCodes, tree bins are rarely used.  
>
> 而当它们由于移除或扩容操作，它们会被转为普通的哈希桶。哈希分布良好的情况下，几乎很少使用树结构。
>
> Ideally, under random hashCodes, the frequency of  nodes in bins follows a Poisson distribution (http://en.wikipedia.org/wiki/Poisson_distribution) with a  parameter of about 0.5 on average for the default resizing  threshold of 0.75, although with a large variance because of  resizing granularity. Ignoring variance, the expected  occurrences of list size k are (exp(-0.5) * pow(0.5, k) /
>  factorial(k)). 
>
> 理想情况下，哈希值随机，负载因子为0.75的情况下，尽管由于粒度调整会产生较大的方差，桶中的节点分布频率遵从参数为0.5的泊松分布。桶里出现一个的概率为0.6，超过8个的概率已经小于千万分之一。
>
> The first values are:
>  0:    0.60653066
>  1:    0.30326533
>  2:    0.07581633
>  3:    0.01263606
>  4:    0.00157952
>  5:    0.00015795
>  6:    0.00001316
>  7:    0.00000094
>  8:    0.00000006
>  more: less than 1 in ten million

我们可以看到这个只是解释链表在有几个节点转为红黑树的原因，已经默认是在负载因子为0.75下进行的讨论，还是没有解释负载因子为什么是0.75。在知乎上随手翻问题的答案，有人给了一个看起来比较靠谱的链接:

https://stackoverflow.com/questions/10901752/what-is-the-significance-of-load-factor-in-hashmap/10901821#10901821。

## 一种可能的答案

我们知道，在理想情况下，对于散列算法我们有一个简单的假设，散列函数应当易于计算，并且能够均匀的分布所有键，即对于任意键，0到M-1之间的每个整数都有相等的可能性。也就是说如果我们的数组容量为s，放在0-s-1这些位置上的概率是相等的也就是
$$
\frac{1}{s}
$$
那我们现在有n个key，数组大小还是s，那么我们现在就要求出现0碰撞为0的概率，也就是说要么不出现碰撞，要么至少出现一次碰撞，设每次不出现碰撞的概率为p, 则出现碰撞的概率为1-p。现在我们来求两个不相等的元素出现在一个位置的概率, 首先将用h代替哈希函数，hash碰撞意味着h(k1) = h(k2) , 我们用$F_x$ 标记事件h(k1) = h(k2) = x, 每个键映射到任意一个桶的概率是:
$$
\frac{1}{s}
$$
我们用E来标记两个不同的key出现在相同位置这个事件:
$$
E=⋃\limits_{x}F(x)
$$
x = 2 , 意味着两个不同的key出现在了相同位置上，x=3意味着三个元素被计算到一个位置上。E是所有这类事件的并集，对每个特定的位置，两个key都在这个位置的概率为: 
$$
\frac{1}{s^2}
$$
由于E是所有事件的并集所以, 所以我们是对概率求和，一共有s个位置，每个位置都是:
$$
\frac{1}{s^2}
$$
那么累加起来就是
$$
\frac{1}{s}
$$
也就是说出现hash碰撞的概率我们可以用$\frac{1}{s}$来表示，如果对于上述不理解，用扔骰子这个例子来解释就是，一个骰子，连续扔两次，两次点数为一样的概率，这个骰子当然要是均匀的，也就是说每一面出现的概率都是相同的，这个概率为$\frac{1}{s}$。 让我们的问题再超前走一步，我们向s个桶里放了n次，0次碰撞的概率，在均匀假设下，这符合二项式分布：
$$
(n,k)p^k(1-p)^(n-k)
$$

n为实验次数, k为成功次数，我们希望一次也不碰撞，上面的式子化简为:


$$
(n,0)\frac{1}{s}^0(1-\frac{1}{s})^n
$$

我们希望不发生碰撞的概率大一些也就是$\frac{1}{2}$, 我们的式子就变成了下面这样:
$$
(1-\frac{1}{s})^n >= \frac{1}{2}
$$
这里我们借助对数进行来对不等式进行处理，鉴于一些同学的对数放缩可能忘记了，我们这里再复习一下对数:
$$
c =  a^b
$$
然后我们有:
$$
b = \log_{a} c
$$
a被称为底数，x被称为真数，当 0 < a < 1, 单调递减，1 < a的时候单调递增，所以我们有下面的变换:
$$
\log_{a} c > \log_{a} d
$$
则c > d , 所以对于上面的对数我们可以用对数来进行变换:
$$
n ln(1-\frac{1}{s}) >= -ln2;
1- \frac{1}{s} < 1 ,  ln 1 = 0, ln({1-\frac{1}{s}}) < 0。
n <= \frac{-ln2}{ln\frac{s-1}{s}};
\frac{s-1}{s} 可以看成\frac{s}{s-1}^-1。
n <= \frac{ln2} {\ln\frac{s}{s-1}}
$$
然后同时除以s，我们只关注等于的情况:
$$
\frac{n}{s} = \frac{ln2}{s ln\frac{s}{s-1}}
$$
$\frac{n}{s}$刚好是扩容因子，现在让桶不断的变大，我们观察扩容因子会向哪个值靠近，这个问题就等价于求极限:
$$
\lim_{s \to \infty} \frac{ln2}{s ln\frac{s}{s-1}}
$$
然后我们将s =   m + 1,  s 趋于无穷就可以被代换为m 趋于无穷，两者是等价无穷大，然后上面的式子就变形为:
$$
\lim_{m \to \infty} \frac{ln2}{m+1 ln\frac{m+1}{m}}
$$
然后我们将$x = \frac{1}{m}$, 则m趋于无穷，x趋于0, 于是式子变为:
$$
\lim_{s \to 0} \frac{ln2} {(\frac{1}{x + 1}) ln (1 +x)}
$$
而$ln(1 + x)$ 和 $x$ 又是等价无穷小，所以式子变换为:
$$
\lim_{s \to 0} \frac{ln2} {(\frac{1}{x + 1}) x}
$$
于是当s 趋于0的时候，n/s整体向ln2靠近，而ln2则是0.69314718055995，我们直到HashMap的容量大小是2的次方，乘以ln2得到的也是一个小数，所以我们需要向上取整，所以取了0.75。 在做对数变换的时候，我们取的是以e为底的对数，但是我们上面提到，我们用大于1的底数进行替换，仍然成立。但为什么取e呢，取e上面的逻辑走的通了，这很牵强，建立的建设也是在hash函数完美理想的情况下。下面我们谈一下为什么当链表的节点为8个的时候，才转为红黑树，为什么符合泊松分布。

## 为什么是泊松分布?

什么是概率，由原因结果，什么概率，一种定义事件发生概率的方法是利用事件发生的相对频率。定义如下:  假设有一个样本空间为S的实验，它在相同的条件下可重复进行，对于样本空间S中的事件E，记n(E)为n次重复实验中事件E的发生次数，那么，该事件发生发生的概率P(E)就定义如下:
$$
P(E) =\lim_{m \to \infty}\frac{n(E)}{n}
$$
即定义频率P(E)为E发生的次数占实验次数的比例的极限，也即E发生频率的极限。上述定义虽然很直观，但是有一个致命的缺陷在于你怎么知道n(E)/n会收敛到一个固定的常数，而且如果进行另一次重复实验，它会收敛到同样的常数。有人可能回答，我实验了好多次啊， 这种回答可能基于概率建立在n(E) / n趋于某常数值这样一个公设上面，但它不够简单，更为通用的是，假定一些更简单、更为显而易见的公理，然后去证明频率在某种意义下趋于一个常数极限不是更合情合理嘛。这也就是现代概率公理化方法。那什么是数理统计，由结果看原因，这里不再展开数理统计的讨论，本篇不涉及这个。所以我们在算概率的时候，要先知道原因，也就是说，知道骰子的均匀性。上面我们在算概率的时候用到了二项式分布，如果我们令n足够大，p充分小，而使得np保持适当的大小时，参数为(n,p)的二项随机变量可近似地看做是参数为λ = np的泊松随机变量，这里不给出证明过程，我们只给出泊松分布的分布列:
$$
P(X = k) = \frac{e^{-\lambda} \lambda^k}{k!}
$$
再把上面的注释拉出来看一下:

> Ideally, under random hashCodes, the frequency of  nodes in bins follows a Poisson distribution (http://en.wikipedia.org/wiki/Poisson_distribution) with a  parameter of about 0.5 on average for the default resizing  threshold of 0.75, although with a large variance because of  resizing granularity. Ignoring variance, the expected  occurrences of list size k are (exp(-0.5) * pow(0.5, k) /
>  factorial(k)).  
>
> 理想情况下，哈希值随机，负载因子为0.75的情况下，尽管由于粒度调整会产生较大的方差，桶中的节点分布频率遵从参数为0.5的泊松分布。桶里出现一个的概率为0.6，超过8个的概率已经小于千万分之一。

那λ也就是0.5， 至于怎么得出呢，也没给出过程，λ=np,  np是二项式分布的数学期望，这是一个加权平均，E(X) = np，p为1/s,  在理想情况下，一次不冲突的概率是0.75，这里也不知道是怎么推导出来的，在冲突的概率下推导出的np = 0.5 , 然后算这个桶里面出现一个元素的概率为0.60653066，出现8个元素的概率为0.00000006。 所以我觉得HashMap的默认负载因子是一个经验值，链表由八个结点变为红黑树也是一个经验值，建立在np= 0.5的基础上。

## 写在最后

这是我毕业时我看到的问题，我看了许多推导，感觉都是差了一些，不完备，这次就系统而完善的对这个问题进行讨论，有可能我也有遗漏的地方。欢迎指出。

## 参考资料

[1] Java 8系列之重新认识HashMap  https://tech.meituan.com/2016/06/24/java-hashmap.html

[2] 【JAVA】HashMap的负载因子为什么是0.75 https://segmentfault.com/a/1190000023308658

[3] What is the significance of load factor in HashMap? https://stackoverflow.com/questions/10901752/what-is-the-significance-of-load-factor-in-hashmap/10901821#10901821

[4]散列表、全域散列与完全散列 https://zhuanlan.zhihu.com/p/300827204 

[5] SUHA https://en.wikipedia.org/wiki/SUHA_(computer_science)

[6] 算法 第四版