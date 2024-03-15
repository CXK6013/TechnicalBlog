# B-树如何让你的查询更快

> 好久没翻译文章了，感觉自己的英文功底有点下降，前几篇文章都着眼于网络I/O了，这篇文章之后我们开始看点数据库、数据结构之类的内容。

> 原文链接: https://blog.allegro.tech/2023/11/how-does-btree-make-your-queries-fast.html

> 已经征得作者的同意可以进行翻译。

[TOC]



**B-tree** is a structure that helps to search through great amounts of data. It was invented over 40 years ago, yet it is still employed by the majority of modern databases. Although there are newer index structures, like LSM trees, **B-tree** is unbeaten when handling most of the database queries.

B-树是一种搜索大量数据的结构，发明于四十年前，现在仍然被用于现代的数据库。尽管已经有了一些新的索引结构，像LSM(Log-Structured-Merge-Tree)树(译者注, LSM树并不像是B树、红黑树一样是严格的数据结构，其实是一种存储结构，目前HBase,LevelDB,RocksDB这些NoSQL存储都是采用的LSM树)， 在处理大多数数据库查询的时候，B-树仍然是无与伦比的。

After reading this post, you will know how **B-tree** organises the data and how it performs search queries.

阅读本篇文章之后，你将会了解B-树如何组织数据以及如何执行搜索查询。

## 源起

In order to understand **B-tree**， let’s focus on **Binary Search Tree (BST)** first.

为了让我们理解B-tree，让我们先来将目光放在二叉搜索树

Wait, isn’t it the same?

等等，这难道不是一样的吗？

What does “B” stand for then?

那B-树的B代表什么? 

According to wikipedia.org ， Edward M. McCreight, the inventor of B-tree, once said:

> “the more you think about what the B in B-trees means, the better you understand B-trees.”

根据维基百科， B-树的发明者 Edward M. McCreight曾经说过: 

> 你越是思考B-树的B代表些什么，就能越能理解B-树。

Confusing **B-tree** with **BST** is a really common misconception. Anyway, in my opinion, BST is a great starting point for reinventing B-tree. Let’s start with a simple example of BST:

把B-树和二叉搜索树混为一谈是一个非常普遍的误解，不管如何，在我看来，二叉搜索树是重塑B-树的一个很好的起点，让我们先从一个非常简单的二叉搜索树开始:

![](https://a.a2k6.com/gerald/i/2024/03/13/4tgu.webp)



The greater number is always on the right, the lower on the left. It may become clearer when we add more numbers.

右边节点的数字总是比双亲节点要打，左边节点的数据总是比双亲节点的小。我们再添加一些数字会变得再清晰一些。

This tree contains seven numbers, but we need to visit at most three nodes to locate any number. The following example visualizes searching for 14. I used SQL to define the query in order to think about this tree as if it were an actual database index.

![](https://a.a2k6.com/gerald/i/2024/03/13/xxv4.webp)

现在这颗二叉搜索树包含七个节点，但是我们最多需要访问三个节点才能找到我们想要找到的数字，下面示例搜索14这个数字的过程，这礼物我使用SQL去定义查询，以便将这棵树视为实际数据库索引。

![](https://a.a2k6.com/gerald/i/2024/03/13/4ulv.webp)

## 硬件

In theory, using Binary Search Tree for running our queries looks fine. Its time complexity (when searching) is O(logn) , [same as B-tree](https://en.wikipedia.org/wiki/B-tree). However, in practice, this data structure needs to work on actual hardware. An index must be stored somewhere on your machine.

理论上来说，使用二叉搜索树来运行我们的查询看起来没有问题，它的搜索花费的时间复杂度是O(logn)，和B-树一样。然后，在实践中，数据结构需要工作在实际的硬件上。索引必须存储在你机器上的某个位置上。

The computer has three places where the data can be stored:

- CPU caches
- RAM (memory)
- Disk (storage)

计算机有三个位置存储数据:

- CPU缓存
- RAM(memory) 内存
- Disj(存储) 磁盘

The cache is managed fully by CPUs. Moreover, it is relatively small, usually a few megabytes. Index may contain gigabytes of data, so it won’t fit there.

缓存完全被CPU管理。此外，它相对较小，通常只有几兆字节。索引可能包含几千兆的数据，所以这里不适合。

Databases vastly use Memory (RAM). It has some great advantages:

- assures fast random access (more on that in the next paragraph)
- its size may be pretty big (e.g. AWS RDS cloud service [provides instances](https://aws.amazon.com/rds/instance-types/) with a few terabytes of memory available).

数据库大量使用内存，内存有一些非常棒的优点:

- 快速随机存取(将在下一章节介绍)
- 容量可以非常大(（例如，AWS RDS 云服务提供可用内存为几 TB 的实例)

Cons? You lose the data when the power supply goes off. Moreover, when compared to the disk, it is pretty expensive.

缺点是断点的时候会丢失数据，而且相对于磁盘，它相当昂贵。

Finally, the cons of a memory are the pros of a disk storage. It’s cheap, and data will remain there even if we lose the power. However, there are no free lunches! The catch is that we need to be careful about random and sequential access. Reading from the disk is fast, but only under certain conditions! I’ll try to explain them simply.

最后内存的缺点就是存储器的优点，它很便宜，即使断电数据也会保留在那里。然而，天下没有免费的午餐，问题在于我们需要谨慎对待顺序访问和随机访问。只有在指定的条件下，磁盘读取数据才会很快。我会试着简单解释一下。

### 随机访问和顺序访问

Memory may be visualized as a line of containers for values, where every container is numbered.

内存可以形象的理解为一排排存放述职的容器，每个容器有对应的编号。

![](https://a.a2k6.com/gerald/i/2024/03/14/3jkyi.webp)

Now let’s assume we want to read data from containers 1, 4, and 6. It requires random access:

现在让我们假设我们需要从编号1,4,6这三个容器上读取数据，这需要随机访问:

![](https://a.a2k6.com/gerald/i/2024/03/14/ow8nn.webp)

And then let’s compare it with reading containers 3, 4, and 5. It may be done sequentially:

然后和读编号为3,4,5的容器进行比较，它就可以按顺序完成。

The difference between a “random jump” and a “sequential read” can be explained based on Hard Disk Drive. It consists of the head and the disk.

随机跳转和顺序读取的不同可以用磁盘驱动器来解释，磁盘由磁头和磁盘组成。

![](https://a.a2k6.com/gerald/i/2024/03/14/igcj.webp)



“Random jump” requires moving the head to the given place on the disk. “Sequential read” is simply spinning the disk, allowing the head to read consecutive values. When reading megabytes of data, the difference between these two types of access is enormous. Using “sequential reads” lowers the time needed to fetch the data significantly.

"随机跳转"要求磁头移动到磁盘上的指定位置。“顺序读取”只需要旋转磁盘，让磁头读取连续的值， 在读取兆字节的数据时，这两种访问方式之间的差距是巨大的。使用”顺序读取“可以大大的降低获取数据所需的事件。

Differences in speed between random and sequential access were researched in the article “The Pathologies of Big Data” by Adam Jacobs, [published in Acm Queue](https://queue.acm.org/detail.cfm?id=1563874). It revealed a few mind-blowing facts:

Adam Jacobs发表在Acm Queue上发表的文章“The Pathologies of Big Data” ，研究了随机访问和顺序访问在速度上的差异。文章揭示了一些令人震惊的事实。

- Sequential access on HDD may be hundreds of thousands of times faster than random access. 🤯

​    顺序访问在机械硬盘上的访问速度比随机访问快几十万倍。

- It may be faster to read sequentially from the disk than randomly from the memory.

​	从磁盘顺序读取有可能比从内存读取更快

Who even uses HDD nowadays? What about SSD? This research shows that reading fully sequentially from HDD may be faster than SSD. However, please note that the article is from 2009 and SSD developed significantly through the last decade, thus these results are probably outdated.

但是现在谁还用机械硬盘？ 那固态硬盘呢?  这项研究显示机械硬盘上的顺序完全读取数据可能比固态硬盘更快。不过请注意，这篇文章是2009年的，而固态硬盘在过去十年得到了长租的发展。这些结果可能已经过时了。

To sum up, the key takeaway is **to prefer sequential access wherever we can**. In the next paragraph, I will explain how to apply it to our index structure.

总之，关键就在于就是尽可能选择顺序访问。下一个章节，我们将解释如何将其应用到我们的索引结构上。

##  优化对树的顺序访问

- Binary Search Tree may be represented in memory in the same way as [the heap](https://en.wikipedia.org/wiki/Binary_heap):

  二叉搜索树在内存中的表示方法和堆相同

  - parent node position is i

    父节点的位置是i

  - left node position is 2i

  ​       左节点的位置是2i

  - right node position is 2i+1

    右节点的位置是2i + 1 

That’s how these positions are calculated based on the example (the parent node starts at 1):

这是根据示例计算出来的位置(父节点从1开始)

![](https://a.a2k6.com/gerald/i/2024/03/14/j0hf.webp)

According to the calculated positions, nodes are aligned into the memory:

根据被计算出来的位置，节点被对齐到内存中

![](https://a.a2k6.com/gerald/i/2024/03/14/j1pm.webp)

Do you remember the query visualized a few chapters ago?

你还记得我们前面讨论的可视化查询吗?

![](https://a.a2k6.com/gerald/i/2024/03/14/iskr.webp)

That’s what it looks like on the memory level:

这就是在内存级别的样子:

![](https://a.a2k6.com/gerald/i/2024/03/14/iuoe.webp)



When performing the query, memory addresses 1, 3, and 6 need to be visited. Visiting three nodes is not a problem; however, as we store more data, the tree gets higher. Storing more than one million values requires a tree of height at least 20. It means that 20 values from different places in memory must be read. It causes completely random access!

执行查询的时候，内存地址1,3,6会被访问，访问三个节点不是问题，然而，如果我们存储了更多数据，这棵树就可能变得更高。存储超过100万个值需要一颗高度至少为20的树。这意味着必须从内存不同位置读取20个值，这会导致完全的随机访问。

### 页面

While a tree grows in height, random access is causing more and more delay. The solution to reduce this problem is simple: grow the tree in width rather than in height. It may be achieved by packing more than one value into a single node.

树在增高的同时，随机访问会导致越来越多的延迟。解决这一问题也很简单: 让树变宽而不是高度增长。可以通过将多个值打包到一个节点来实现。

It brings us the following benefits: 它有以下好处：

- the tree is shallower (two levels instead of three)

​    树更浅，两层而不是三层。

- it still has a lot of space for new values without the need for growing further

  仍然有大量的空间可以容纳新的值，而无需进一步增长。

The query performed on such index looks as follows: 在这种索引上执行查询如下图所示:

![](https://a.a2k6.com/gerald/i/2024/03/14/qc6dv.webp)

Please note that every time we visit a node, we need to load all its values. In this example, we need to load 4 values (or 6 if the tree is full) in order to reach the one we are looking for. Below, you will find a visualization of this tree in a memory:

请注意每次我们访问一个节点，我们都需要加载这个节点所有的值，在这个例子中，我们需要加载4个值(如果树是满的，就需要6次)才能找到我们需要的值。下面是这棵树在内存中的展示

![](https://a.a2k6.com/gerald/i/2024/03/14/3rqe3.webp)

Compared to [the previous example](https://blog.allegro.tech/2023/11/how-does-btree-make-your-queries-fast.html#optimizing-a-tree-for-sequential-access) (where the tree grows in height), this search should be faster. We need random access only twice (jump to cells 0 and 9) and then sequentially read the rest of values.

与上一个示例相比(树的高度不断增加)，搜索应当更快，我们仅需要随机访问两次(跳转到0和9单元)，然后顺序读取剩余的值。

This solution works better and better as our database grows. If you want to store one million values, then you need:

随着数据量越来越大，这种解决方案的好处会更加明显，如果你想要存储100万个值，然后你需要

- Binary Search Tree which has **20** levels

  二叉搜索树的有20层

OR

- 3-value node Tree which has **10** levels

  只有10层的3值节点树。

Values from a single node make a page. In the example above, each page consists of three values. A page is a set of values placed on a disk next to each other, so the database may reach the whole page at once with one sequential read.

单个节点的值构成一个额页面，在上面的例子中，每个页面由三个值组成。页面是磁盘上一组相邻的值，因此数据库进需要一次顺序访问读取，就能同时读取整个页面，

And how does it refer to the reality? [Postgres page size is 8kB](https://www.postgresql.org/docs/current/storage-toast.html#:~:text=PostgreSQL uses a fixed page,tuples to span multiple pages.). Let’s assume that 20% is for metadata, so it’s 6kB left. Half of the page is needed to store pointers to node’s children, so it gives us 3kB for values. BIGINT size is 8 bytes, thus we may store ~375 values in a single page.

它与现实又是如何联系的呢？ Postgres的页面大小只有8kb，让我们有20%是元数据，那么还剩下6kb。页面的一半需要存储指向子节点的指针，所以给我们存储值就只需要3kb，BIGINT的大小是8 bytes，因此我们能存储375个值再单个页面里面。

Assuming that some pretty big tables in a database have one billion rows, how many levels in the Postgres tree do we need to store them? According to the calculations above, if we create a tree that can handle 375 values in a single node, it may store **1 billion** values with a tree that has only **four** levels. Binary Search Tree would require 30 levels for such amount of data.

假设数据库有一些超级大的表有10亿条记录，那么我们在postgres树种需要多层才能存储? 根据上面的计算，单个节点可以存储375个值，它可以只有四级的树来存储10亿个值。对于如此大量的数据，二叉搜索树将需要30层来存储。

To sum up, placing multiple values in a single node of the tree helped us to reduce its height, thus using the benefits of sequential access. Moreover, a B-tree may grow not only in height, but also in width (by using larger pages).

总之，在单个节点里面存储放置多个值有助于我们可以减少树的高度。我们因此就能从顺序访问中受益。然后B-树不仅可以增加高度，也可以通过增加页面大小来增加宽度。

### 平衡

There are two types of operations in databases: writing and reading. In the previous section, we addressed the problems with reading the data from the B-tree. Nonetheless, writing is also a crucial part. When writing the data to a database, B-tree needs to be constantly updated with new values.

在数据库中有两种基本的操作: 写和读。在上一节中我们讨论了从B-树中读取数据的问题。然而写入数据也是一个非常关键的点，向数据库中写入数据的时候，B-树需要不断更新新值。

The tree shape depends on the order of values added to the tree. It’s easily visible in a binary tree. We may obtain trees with different depths if the values are added in an incorrect order.

树的形状取决于添加进入树的值的顺序，这在二叉树中很容易看到，如果数值添加顺序不正确，我们可能会得到不同深度的树。

![](https://a.a2k6.com/gerald/i/2024/03/14/ke4w.webp)

When the tree has different depths on different nodes, it is called an unbalanced tree. There are basically two ways of returning such a tree to a balanced state:

1. Rebuilding it from the very beginning just by adding the values in the correct order.
2. Keeping it balanced all the time, as the new values are added.

B-tree implements the second option. A feature that makes the tree balanced all the time is called self-balancing.

#### 自平衡算法示例

Building a B-tree can be started simply by creating a single node and adding new values until there is no free space in it.



If there is no space on the corresponding page, it needs to be split. To perform a split, a „split point” is chosen. In that case, it will be 12, because it is in the middle. The „Split point” is a value that will be moved to the upper page.



Now, it gets us to an interesting point where there is no upper page. In such a case, a new one needs to be generated (and it becomes the new root page!).

And finally, there is some free space in the three, so value 14 may be added.



Following this algorithm, we may constantly add new values to the B-tree, and it will remain balanced all the time!





*At this point, you may have a valid concern that there is a lot of free space that has no chance to be filled. For example, values 14, 15, and 16, are on different pages, so these pages will remain with only one value and two free spaces forever.*

*It was caused by the split location choice. We always split the page in the middle. But every time we do a split, we may choose any split location we want.*

*Postgres has an algorithm that is run every time a split is performed! Its implementation may be found in the [_bt_findsplitloc() function in Postgres source code](https://github.com/postgres/postgres/blob/54ccfd65868c013a8c6906bc894bc5ea3640740a/src/backend/access/nbtree/nbtsplitloc.c#L87). Its goal is to leave as little free space as possible.*

## 总结一下

## Summary

In this article, you learned how a B-tree works. All in all, it may be simply described as a Binary Search Tree with two changes:

- every node may contain more than one value
- inserting a new value is followed by a self-balancing algorithm.

Although the structures used by modern databases are usually some variants of a B-tree (like B+tree), they are still based on the original conception. In my opinion, one great strength of a B-tree is the fact that it was designed directly to handle large amounts of data on actual hardware. It may be the reason why the B-tree has remained with us for such a long time.





## 译者对B-树和B+树的理解



## MySQL 对B+树的处理





让我们从HashMap的哈希算法谈起

HashMap  哈希算法 让

## 翻译参考资料

[1] LSM树详解  https://zhuanlan.zhihu.com/p/181498475 