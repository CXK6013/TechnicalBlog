# Stream学习笔记(二)map与reduce

> 《Stream学习笔记(一)》里我们介绍了Stream里面常用的基本操作，今天我们再来介绍一些。

## 前言

我们今天主要介绍的是Stream中的map与reduce方法，为什么介绍这个呢？ 原因在于我之前看过一个大数据领域的框架叫MapReduce，在这个大数据框架中核心关键词就是Map和Reduce，同时这两个关键词也是MapReduce框架中的两个关键函数，Map函数的作用是从获取输入并将其做为key-value对，当作函数的入参，经过Map函数的处理，返回key-value对。Reduce对结果进行处理也就是合并，下面的图演示了MapReduce过程:

![MapReduce](http://tva1.sinaimg.cn/large/006e5UvNgy1h7u8u4h4b4j30zq0lbmzc.jpg)



input输入，split分割，map映射转换，combine组合，reduce是一个动词，减少数量、价格等。这让我很迷惑，从示意图来看reduce执行的是最后的合并过程，我们姑且将其理解为聚合吧。在上面的图例中，combine想key相同的进行聚合，最后分成四组，而reduce阶段则是对每个小块本身进行聚合。基本同Stream的reduce函数是类似的。

## reduce

Java的Stream里面也有map、reduce。我们这里先讲reduce，reduce这个函数更难理解一点，在Stream中reduce函数一共有三个重载:

```java
1. Optional<T> reduce(BinaryOperator<T> accumulator);
2. T reduce(T identity, BinaryOperator<T> accumulator);
3. <U> U reduce(U identity,BiFunction<U, ? super T, U> accumulator, BinaryOperator<U> combiner);
理解难度从上往下依次递增。
```

我们一个一个来看，第一个有下面这样的注释:

```java
	boolean foundAny = false;
     T result = null;
     for (T element : this stream) {
         if (!foundAny) {
             foundAny = true;
             result = element;
         }
         else
             result = accumulator.apply(result, element);
     }
     return foundAny ? Optional.of(result) : Optional.empty();
```

第一个函数等价于上面的代码，上面的代码首先从集合中找到第一个元素，然后和集合中的其他元素做apply运算，得到结果之后，这个结果再和其他元素做运算，直到所有元素参与运算完毕。这样说可能有点抽象，一个简单的例子是累加求和，如下图所示

![reduce过程](http://tvax1.sinaimg.cn/large/006e5UvNgy1h82c121l7ej30zf0k7q3q.jpg)

所以reduce函数表达的意思是: **`reduce()`** 方法对流中的每个元素按序执行一个由您提供的 **reducer** 函数，每一次运行 **reducer** 会将先前元素的计算结果作为参数传入，最后将其结果汇总为单个返回值。第一次执行reduce函数时，没有先前元素的计算结果，可以从外部指定(reduce函数的第二个重载), 如果外部没指定默认选取第一个元素中作为上一次reduce的计算结果(对应reduce函数的第一个重载)。

![reduce过程-2](http://tvax1.sinaimg.cn/large/006e5UvNgy1h82fzubfcaj31330k7jta.jpg)

那第三个重载是做什么的，或者说第三个函数的第三个参数是用来做什么的？首先让我们来看下上面的注释: 

```java
Performs a reduction on the elements of this stream, using the provided identity, accumulation and combining functions. 
使用提供的初始值、accumulation、combining对流执行reduction操作。等价于下面这个函数
This is equivalent to:
 	 U result = identity;
     for (T element : this stream)
         result = accumulator.apply(result, element)
     return result;
but is not constrained to execute sequentially.
但是不限于顺序流
The identity value must be an identity for the combiner function. This means that for all u, combiner(identity, u) is equal to u. Additionally, the combiner function must be compatible(兼容的) with the accumulator function; for all u and t, the following must hold:
 combiner.apply(u, accumulator.apply(identity, t)) == accumulator.apply(u, t)
特征值一定要作为combine函数的特征, 这也就是说对于任意元素u，combine(identity,u) 与 u相等。除此之外,combiner一定要与accumulator函数兼容也就是说，对于任意的u、t, 要求满足 combiner.apply(u, accumulator.apply(identity, t)) == accumulator.apply(u, t)
```

看这个注释是不是有点云里雾里，没关系，我们看例子，在例子中分析combiner和accumulator在流里面是如何运作的。

```java
 @Test
 public void reduceDemo03(){
        String[] arr = {"lorem", "ipsum", "sit", "amet"};
        List<String> strs = Arrays.asList(arr);
        int ijk = strs.stream().reduce(1,
                (identity, element) -> {
                    System.out.println("execute Thread name"+Thread.currentThread().getName());
                    System.out.println("Accumulator, identity = " + identity + ", element = " + element);
                    return identity + element.length();
                },
                (leftResult, rightResult) -> {
                    System.out.println("execute Thread name"+Thread.currentThread().getName());
                    System.out.println("combine, identity = " + leftResult + ", element = " + rightResult);
                    return leftResult * rightResult;
                });
        System.out.println(ijk);
    }

```

上面的例子，我们传入的accumulator函数是将传入的元素进行相加，combiner函数将传入的元素进行相乘。执行结果如下：

![reduce-3](http://tva2.sinaimg.cn/large/006e5UvNgy1h83elvwxk8j30j006jq3l.jpg)



好像我们传入的combine函数压根就没用，我们换成并行流试试看：

```java
 @Test 
 public void reduceDemo05(){
        String[] arr = {"lorem", "ipsum", "sit", "amet"};
        List<String> strs = Arrays.asList(arr);
        int reduceResult  = strs.parallelStream().reduce(1,
                (identity, element) -> {
                    System.out.println("Accumulator execute Thread name: "+Thread.currentThread().getName());
                    System.out.println("Accumulator, identity = " + identity + ", element = " + element);
                    return identity + element.length();
                },
                (leftResult, rightResult) -> {
                    System.out.println("combine execute Thread name"+Thread.currentThread().getName());
                    System.out.println("combine, identity = " + leftResult + ", element = " + rightResult);
                    return leftResult * leftResult;
                });
        System.out.println("reduceResult: "+reduceResult);
    }
```

输出结果如下:

![并行reduce结果](http://tvax2.sinaimg.cn/large/006e5UvNgy1h83fo73bcdj30f908wdh4.jpg)

我们换成并行流之后combine函数就执行了，执行过程为先让我们传入的identity和流中的元素做accumulator运算, 流里面又有四个元素，经过accumulator运算之后，得到了四个结果，最后由combine函数将这四个结果进行合并。

![并行combine过程](http://tva4.sinaimg.cn/large/006e5UvNgy1h83fre7276j312a0irmym.jpg)

理解了这个过程之后我们再来看上面的注释: combiner.apply(u, accumulator.apply(identity, t)) == accumulator.apply(u, t), 我们取上面过程的值代入看看式子是不是成立,  u是执行完accumulator操纵之后的值，所以我们选择u=6，t是流中的元素，我们选取ipsum

-  combiner.apply(6, accumulator.apply(1, 5)) = 36
- accumulator.apply(6, 5) = 30

combiner.apply(u, accumulator.apply(identity, t)) == accumulator.apply(u, t)不成立，如果成立会由什么样的效果？ 我们尝试改写上面的reduceDemo05，让combiner.apply(u, accumulator.apply(identity, t)) == accumulator.apply(u, t)，看看会是什么结果:

```java
 @Test
 public void reduceDemo05(){
        String[] arr = {"lorem", "ipsum", "sit", "amet"};
        List<String> strs = Arrays.asList(arr);
        int parallelStreamReduceResult  = strs.parallelStream().reduce(0,
                (identity, element) -> {
                    System.out.println("Accumulator execute Thread name: "+Thread.currentThread().getName());
                    System.out.println("Accumulator, identity = " + identity + ", element = " + element);
                    return identity + element.length();
                },
                (leftResult, rightResult) -> {
                    System.out.println("combine execute Thread name"+Thread.currentThread().getName());
                    System.out.println("combine, identity = " + leftResult + ", element = " + rightResult);
                    return leftResult + rightResult;
                });
        int  streamResult = strs.stream().reduce(0,
                (identity, element) -> {
                    System.out.println("Accumulator execute Thread name: " + Thread.currentThread().getName());
                    System.out.println("Accumulator, identity = " + identity + ", element = " + element);
                    return identity + element.length();
                },
                (leftResult, rightResult) -> {
                    System.out.println("combine execute Thread name" + Thread.currentThread().getName());
                    System.out.println("combine, identity = " + leftResult + ", element = " + rightResult);
                    return leftResult + rightResult;
                });

        System.out.println("parallelStreamReduceResult: "+parallelStreamReduceResult);
        System.out.println("streamResult: "+streamResult);
    }
```

![并行与串行](http://tvax1.sinaimg.cn/large/006e5UvNgy1h83gbcq10pj30e50aegmt.jpg)

结论是对combiner.apply(u, accumulator.apply(identity, t)) == accumulator.apply(u, t)成立, 则reduce的第三个重载函数在并行流和串行流下结果相等。

## 并行流中如何指定自定义线程池

记得之前有人再微信群里问过一个问题，并行流中如何指定自定义的ForkJoinPool，如果不指定所有的并行流都会使用一个线程池，我们有的时候想再流中指定线程池，我记得我当时的回答是不能，想来是有些误认子弟了,  事实上可以通过下面的方式用自定义的线程池:

```java
 @Test
 public void forkJoinPool(){
        ForkJoinPool forkJoinPool = new ForkJoinPool(4);
        forkJoinPool.submit(()->{
            Stream.of("lorem", "ipsum", "sit", "amet").parallel().forEach(e->{
                System.out.println(Thread.currentThread().getName());
            });
        });
  }
```

输出结果:

![输出结果](http://tvax2.sinaimg.cn/large/006e5UvNgy1h83h0199ocj30cj040gll.jpg)

## 参考资料

- What is MapReduce?  https://www.talend.com/resources/what-is-mapreduce/
- Array.prototype.reduce()  https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Array/Reduce
- 关于java源码中Stream#reduce的一个解读  https://www.bigbrotherlee.com/index.php/archives/274/