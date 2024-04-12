# 我们来聊聊HashMap吧

[TOC]

## 回忆起HashMap

###  概述HashMap

说到HashMap我脑中复现出下面这一个图:

![](https://a.a2k6.com/gerald/i/2024/04/10/2zl5.png)



也就是hash算法、数组、链表、红黑树，我放入的key-value，根据hash算法会计算出来应该放置到数组的哪个位置上，如果出现了hash冲突，也就是hash算法映射出来的下标是一个，但是使用equals方法判断不相等，那么也就是出现了hash冲突，就会数组对应的位置形成链表，链表大于8个之后，转为红黑树。为什么是8才就转为红黑树呢？ 这依赖于负载因子0.75，那什么是负载因子呢? 我们知道HashMap是一个动态容器，也就是说容器里面放不下之后，就会自动扩大容器的大小，那HashMap在什么时候扩容呢?  

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
```

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
    	// 为空就初始化一次
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null) // 语句一
        tab[i] = newNode(hash, key, value, null); // 语句二
    else {
        Node<K,V> e; K k;
        if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k)))) // 语句三
            e = p;
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value); // 语句四
        else {
            // 语句块六
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        // 语句块七
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    // 语句块八
    ++modCount;
    // 语句块九
    if (++size > threshold)
        resize();
    
    // 语句块十
    afterNodeInsertion(evict);
    return null;
}
```

这一段的逻辑为，如果数组为空，执行扩容，扩容完成，将hash的位置和n-1做与运算，得到插入的数组索引i。 这让我想起了，这让我想起了求余算法， 对正整数取M取余，余数的范围是0到M-1。

那用取余算法当做hash算法？ 那肯定有很多hash冲突, 我们可以随手举一个例子，假设数组长度是10, 我们放入的是key是17、27、37、47。这些都会放在数组索引7的这个位置上，hash冲突比较严重，下面是HashMap的hash方法源码:

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

HashMap中我们一般常用String当做key的类型，我们看下String的hash算法:

```java
public int hashCode() {
    int h = hash;
    if (h == 0 && value.length > 0) {
        char val[] = value;
        for (int i = 0; i < value.length; i++) {
            h = 31 * h + val[i];
        }
        hash = h;
    }
    return h;
}
```

也就是是用这个公式做计算: s[0]*31^(n-1) + s[1]*31^(n-2) + ... + s[n-1], s[i] 是第字符串中第i个字符。那我们自然会提出一个问题，这个算法的好处在哪里呢？ 为什么是31呢？ 带着这个问题我找到了StackOverFlow，排名第一的答案是这么回答的：

> According to Joshua Bloch's Effective Java, Second Edition (a book that can't be recommended enough, and which I bought thanks to continual mentions on Stack Overflow):
>
> 根据Joshua Bloch的《Effective Java》第二版( 这本书怎么推荐都不为过，多亏了StackOverFlow的推荐，我才买了这本书。)
>
> The value 31 was chosen because it is an odd prime. If it were even and the multiplication overflowed, information would be lost, as multiplication by 2 is equivalent to shifting.
>
> 31被选择是因为这是一个奇素数，如果是偶数乘法会溢出，信息会被丢失，乘以2和移位相同。
>
> The advantage of using a prime is less clear, but it is traditional. A nice property of 31 is that the multiplication can be replaced by a shift and a subtraction for better performance: 31 * i == (i << 5) - i. 
>
> 使用质数的优势不太明显，但是这是传统的做法。 31有一个很好的特性是乘法可以用移位和减法代替: 31 * i  == (i << 5 ) - i。

我的看法是所谓传统的做法就是一直以来就是这么做的，所以可能就是惯例，有那么一丝原因。

> Modern VMs do this sort of optimization automatically.(from Chapter 3, Item 9: Always override hashCode when you override equals, page 48)
>
> 现代虚拟机会自动做这类优化( 第三章，第9页: 当你重写equals方法时候总要重写equals方法)。

我的想法奇素数这个概念很怪，奇数的定义是不能被2整除的数是奇数，素数的定位是指在大于1的自然数中，除了1和该数自身外，无法被其他自然数整除的数。所以除了2，所有的素数都是奇数。

### 来自google开源算法

在google的一个开源库用质数1000003来计算哈希值，对应的哈希算法如下:

```java
public int hashCode() {
  int h = 1;
  h *= 1000003;
  h ^= this.firstName.hashCode();
  h *= 1000003;
  h ^= this.lastName.hashCode();
  h *= 1000003;
  h ^= this.age;
  return h;
}
```

^ 是**按位异或 (XOR)** 运算符，如果相对应位值相同，则结果为0，否则为1。

![](https://a.a2k6.com/gerald/i/2024/04/11/5zifd.png)



然后在这个仓库下面我找到了一个这样的评论:

![](https://a.a2k6.com/gerald/i/2024/04/11/3pk6.png)

我翻译了一下:

> 也许kevinb9n可以提供更多细节，我在内部找到了一个提交。Kevin在其中写道: "使用另一个人发现的hash算法，其性能远远优于*31+"(这个人已经不在google工作了，我也没在Github上看到他)。因为如果贡献给哈希代码的数字很小，31只会将它们向左移动5位，而1000003则会将它们向左移动20位。

然后kevinb9n也出来了，评论如下:

> Although I don't remember the details.... it is kind of exactly like me to shift things over a golden-ratio fraction of the way. 
>
> 我已经记不起太多细节了，但我倾向于将东西移动到黄金分割率的几分之一的位置上。
>
> Though that might argue for a multiplier of 898,459, so I guess I also thought it should be a nice simple number to the human eye.
>
> 虽然这可能会让人认为乘数是898459，但我想我也认为这对人眼来说应该是一个漂亮而简单的数字。

黄金比例是无理数，近似值为1.6180339，一个牵强的解释是, 1000003 除以1.618 = 618048。 这里是个人倾向，倒是没有太多严格证明在里面。我们接着看:

> But yeah, the idea of making it bigger was just to eat up all those initial zeros more quickly and get the bits to come back around and interfere with each other.
>
> 不过，是的，把它变大的想法只是为了更快地吃掉所有的初始0，并让这些比特位回到周围并相互干扰。

我们举个例子来解释一下，乘以一个较大的素数能更快地吃掉所有初始0:

- 如果我们选择的乘数是3，这是一个质数，然后我们看两个数高位的还是四个0，

  - 0000 1001 * 3 = 0000 0011

  - 0000 1010 * 3 = 0000 0030

- 现在我们来选择乘数是1000003，来看下乘了之后的结果:

  - 0000 1001 *  1000003 = 1001 0011 0011

  - 0000 1010 * 1000003 =  1010 0030 0030

现在高位的0就被吃掉了，两个数不同的地方就更多了。言下之意也就是说，31这个性能不佳，可能只是一个魔法数，这可能是一个惯例。在这个issue下面我还看到了一些温情的评论:

![](https://a.a2k6.com/gerald/i/2024/04/11/3tm8.png)

mjiustin回复感谢你细致而快速的回答。然后kevinb9n的回答是: 当然，如果我的工作全部都是在为别人解答问题，那么这将是世上最有趣的工作。

### 到这里小小总结一下

工程上一般用的比较多的也就是多项式Hash算法, 良好的hash算法映射出来的值，各个位置之上的0和1应当尽量不同。31可能就是一个传统的值。

####  接着聊HashMap的hash算法

上面的注释是:

> Computes key.hashCode() and spreads (XORs) higher bits of hash to lower. 
>
> 计算key 的hashCode，并通过异或操作将hash值的高位扩散到低位。

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

int是4字节，也就是32位二进制数，\>>> 是右移位运算法，这个移位该怎么理解呢？ 运算规则其实很简单，丢弃右边指定位数，左边补0。

![](https://a.a2k6.com/gerald/i/2024/04/11/6ehsm.png)

读懂这段代码之后，我们就可以来思考这段代码中异或的价值所在, int的范围是**-2147483648**到**2147483648**，前后加起来有10亿的映射空间，如果hash函数映射得比较均匀松散，一般碰撞的概率并不大。

但我们并不能一开始就new 这么大数组，内存也放不下，所以我们就要对数组进行取模操作:









> Because the table uses power-of-two masking, sets of hashes that vary only in bits above the current mask will always collide. (Among known examples are sets of Float keys holding consecutive whole numbers in small tables.)
>
> 由于哈希表使用了2的幂次作为掩码，那些仅在当前掩码以上的位有变化的哈希集合将总是发生碰撞。（已知的例子包括在小型哈希表中用作键的一组连续整数浮点数。）

这个是什么意思，让我们

>  So we apply a transform that spreads the impact of higher bits downward. There is a tradeoff between speed, utility, and quality of bit-spreading. Because many common sets of hashes are already reasonably distributed (so don't benefit from spreading), and because we use trees to handle large sets of collisions in bins, we just XOR some shifted bits in the cheapest possible way to reduce systematic lossage, as well as to incorporate impact of the highest bits that would otherwise never be used in index calculations because of table bounds
>
>  所以我们应用了一个变换,将高位的影响向下传播。在速度、实用性和位扩散质量之间有一个权衡。因为许多常见的哈希值集合已经有了合理的分布(所以不会从扩散中获益),而且因为我们使用树来处理桶中的大量冲突,所以我们只是以最便宜的方式异或一些移位的位,以减少系统性损失,并结合最高位的影响,否则这些位就永远不会在索引计算中使用,因为有表的边界



### 计算数组位置

那这里为什么是与运算呢？ 得到位置之后，然后如果对应位置没有值，就执行赋值操作。如果key存在，且调用equals方法是一个key则执行覆盖操作。

### put 方法概述

如果hash相同，但是调用equals方法不同，说明出现了hash冲突，然后看对应位置的节点是否是树节点，也就是我们上面提到的红黑树，如果树将其添加到树里面。否则准备开始遍历链表，将其添加到最后，然后判断链表的长度是否大于8，如果大于8，然后转换红黑树，插入键值对。这是我之前知道的，但是我在看源码有点不理解:

```java
static final int TREEIFY_THRESHOLD = 8;

for (int binCount = 0; ; ++binCount) {
    if ((e = p.next) == null) {
        p.next = newNode(hash, key, value, null);
        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
            treeifyBin(tab, hash);
        break;
    }
    if (e.hash == hash &&
        ((k = e.key) == key || (key != null && key.equals(k))))
        break;
    p = e;
}
```

TREEIFY_THRESHOLD - 1 = 7，不应该是等于8嘛，但是我们是从0开始的，所以0到7刚好等于8，没毛病。如果判断出来链表的某个节点和放入的key 是一个，也就是:

```java
e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k)))
```

然后我们接着看语句块七:

```java
// 语句块七
if (e != null) { // existing mapping for key
    V oldValue = e.value;
    if (!onlyIfAbsent || oldValue == null)
      e.value = value;
      afterNodeAccess(e);
      return oldValue;
}
```

putVal中onlyIfAbsent是false，所以覆盖value。在HashMap中afterNodeAccess是空方法，交给LinkedHashMap来实现。这个onlyIfAbsent为true, 应该是给putIfAbsent用的。

```java
public V putIfAbsent(K key, V value) {
    return putVal(hash(key), key, value, true, true);
}
```

也就是如果key不存在才放进去，如果存在了就返回给结果:

```java
Map<String,String> map = new HashMap<>();
map.put("email","xxx");
System.out.println(map.putIfAbsent("email", "aaa"));
// 相当于putIfAbsent 
if (map.get("email") == null){
    map.put("email","aaa");
}
```

想起来JDK 1.8 ConcurrentHashMap中putIfAbsent的性能问题:

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            tab = initTable(); // 初始化表
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) { // 如果为空通过CAS赋值
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null))) 
                break;                   // no lock when adding to empty bin
        }
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            // 性能问题就在这,都已经存在了还是加锁。
            synchronized (f) {
             // 省略无关代码
             if (tabAt(tab, i) == f) {}                
        }
    }
    addCount(1L, binCount);
    return null;
}
```

在JDK 17这个问题得到了修复:

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh; K fk; V fv;
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value)))
                break;                   // no lock when adding to empty bin
        }
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        // 既然已经存在了不需要加锁
        else if (onlyIfAbsent // check first node without acquiring lock 
                 && fh == hash
                 && ((fk = f.key) == key || (fk != null && key.equals(fk)))
                 && (fv = f.val) != null)
            return fv;
        else {
            V oldVal = null;
            synchronized (f) {
                if (tabAt(tab, i) == f) {}
            }
    }
    addCount(1L, binCount);
    return null;
}
```

回到语句八、九、十:

```java
// 语句块八
++modCount;
// 语句块九
if (++size > threshold)
  resize(); 
// 语句块十
afterNodeInsertion(evict);
```

threshold: size超过这个触发扩容，也就是容量乘以负载因子，默认负载因子是0.75，这个容量我们可以通过HashMap的构造函数传入:

```java
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);
}
```

```
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

modCount用于跟踪修改次数，这个的用处是用于禁在用迭代器遍历的时候一边添加一边移除，添加的时候增加modCount，用迭代器遍历的时候remove会进行判断:

```java
abstract class HashIterator {
        Node<K,V> next;        // next entry to return
        Node<K,V> current;     // current entry
        int expectedModCount;  // for fast-fail
        int index;             // current slot
        HashIterator() {
            expectedModCount = modCount;
            Node<K,V>[] t = table;
            current = next = null;
            index = 0;
            if (t != null && size > 0) { // advance to first entry
                do {} while (index < t.length && (next = t[index++]) == null);
            }
        }     
        public final void remove() {
            Node<K,V> p = current;
            if (p == null)
                throw new IllegalStateException();
            // 比较现有的modCount和刚初始化迭代器的expectedModCount
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            current = null;
            K key = p.key;
            removeNode(hash(key), key, null, false, false);
            expectedModCount = modCount;
        }
 }
```

所以总结一下put方法的逻辑如下: 



#### 小小的总结一下





### 由此想到其他Hash算法

sha-1 、数据库的hash索引、md5。



## 总结一下







## 参考资料

[1] Java 8系列之重新认识HashMap https://tech.meituan.com/2016/06/24/java-hashmap.html

[2]  Why does Java's hashCode() in String use 31 as a multiplier?  https://stackoverflow.com/questions/299304/why-does-javas-hashcode-in-string-use-31-as-a-multiplier

[3] JDK 源码中 HashMap 的 hash 方法原理是什么？ https://www.zhihu.com/question/20733617