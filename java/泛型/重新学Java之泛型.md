# 重新学Java之泛型

[TOC]

## 前言

本身是打算接着写JMM、JCStress，然后这两个是在公司闲暇的时候随手写的，没有推到Github上，但写点什么可以让我获得宁静的感觉，所性就从待办中拎了一篇文章，也就是这篇泛型。这篇文章来自于我朋友提出的一个问题，比如我在一个类里面声明了两个方法，两个方法只有返回类型是int，一个是Integer，像下面这样,能否通过编译: 

```java
public class DataTypeTest {    
   public int sayHello(){
        return 0;
    }
    public Integer sayHello(){
        return 1;
    }
}
```

我当时回答的时候是将Integer和int当做不同的类型来思考的，我回答的是可以，但是我的朋友说，这是不行的。后面我想了一下是泛型擦除嘛，朋友点头说是的。其实我对泛型擦除是有所耳闻的，只是平时觉得对我用处不大，所性就一直放在那里，不去思考。最近也在一些工具类库，用到了泛型，发现自己对泛型的理解还是有所欠缺，所以今天就重新学习泛型，顺带梳理一下自己对泛型的理解。

## 泛型的意义

我在学习Java的时候，看的是Oracle出的《Java Tutorials》，地址如下:

- https://docs.oracle.com/javase/tutorial/java/generics/index.html

在开篇教程如是说:

> In any nontrivial software project, bugs are simply a fact of life. Careful planning, programming, and testing can help reduce their pervasiveness, but somehow, somewhere, they'll always find a way to creep into your code. This becomes especially apparent as new features are introduced and your code base grows in size and complexity.
>
> 在任何不平凡的软件工程，bug都是不可避免的事实。仔细的规划、变成、测试可以帮助减少它们的普遍性，但不知何时，不知何地，它们总会找到一种方式渗入你的代码。随着新功能的引入和代码量的增长，这一点变得尤为明显。

> Fortunately, some bugs are easier to detect than others. Compile-time bugs, for example, can be detected early on; you can use the compiler's error messages to figure out what the problem is and fix it, right then and there. Runtime bugs, however, can be much more problematic; they don't always surface immediately, and when they do, it may be at a point in the program that is far removed from the actual cause of the problem.
>
> 幸运的是，一些bug更容易发现相对其他类型的bug，例如，编译时的bug可以在早期发现; 你可以使用编译器给出的错误信息来找出问题所在，然后在当时就解决它。然而运行时的bug就要麻烦的多，它们并不总是立即复现出来，而且当它们复现出来的时候，可能是在程序的某个点上，与问题的实际原因相去甚远。
>
> Generics add stability to your code by making more of your bugs detectable at compile time. 
>
> 泛型可以增加你的代码的稳定性，让更多错误可以在编译时被发现。

总结一下，泛型可以增强我们代码的稳定性，让更多错误可以在编译时就被发现。我一开始用的是JDK 8，在使用这个版本的时候，泛型已经进入Java十年了，泛型对于我来说是很理所当然的，就像鱼习惯了水一样。那Java为什么要引入泛型呢? 

> In a nutshell, generics enable *types* (classes and interfaces) to be parameters when defining classes, interfaces and methods. Much like the more familiar *formal parameters* used in method declarations, type parameters provide a way for you to re-use the same code with different inputs. The difference is that the inputs to formal parameters are values, while the inputs to type parameters are types. Code that uses generics has many  benefits over non-generic code:
>
> 简而言之，泛型可以使得在定义类、接口和方法时可以将类型作为参数。就像在方法中声明形式参数一样，类型参数提供了一种方式，让你可以在不同的输入重负使用相同的代码。不同之处在于，形式参数输入的是指，而类型参数的输入是类型。使用泛型的代码相对于非泛型的代码有很多优点:

- > Stronger type checks at compile time. A Java compiler applies strong type checking to generic code and issues errors if the code violates type safety. Fixing compile-time errors is easier than fixing runtime errors, which can be difficult to find.

  编译时进行更强的类型检查

- > Elimination of casts.
  > The following code snippet without generics requires casting:

- > When re-written to use generics, the code does not require casting:

- > Enabling programmers to implement generic algorithms. By using generics, programmers can implement generic algorithms that work on collections of different types, can be customized, and are type safe and easier to read.

##  泛型如何使用



## 泛型擦除

泛型应当是我们日常使用比较高的了，一般我们会如下使用: 

```java
List<String> stringList = new ArrayList<>();
```

上面的语句代表我们希望List集合中里面容纳的是String类型的元素，但是





## 总结一下

