# 重学Java之泛型的基本使用

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

我当时回答的时候是将Integer和int当做不同的类型来思考的，我回答的是可以，但是我的朋友说，这是不行的。后面我想了一下是泛型擦除嘛，朋友点头说是的。其实我对泛型擦除是有所耳闻的，只是平时觉得对我用处不大，所性就一直放在那里，不去思考。最近也在一些工具类库，用到了泛型，发现自己对泛型的理解还是有所欠缺，所以今天就重新学习泛型，顺带梳理一下自己对泛型的理解，后面发现都揉在一篇文章里面，篇幅还是有些过大，这里就分拆两篇。

- 泛型的基本的使用
- 泛型擦除、实现、向前兼容、与其他语言的对比。

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
> 简而言之，泛型可以使得在定义类、接口和方法时可以将类型作为参数。就像在方法中声明形式参数一样，类型参数提供了一种方式，让你可以在不同的输入使用相同的代码。不同之处在于，形式参数输入的是指，而类型参数的输入是类型。使用泛型的代码相对于非泛型的代码有很多优点:

- > Stronger type checks at compile time. A Java compiler applies strong type checking to generic code and issues errors if the code violates type safety. Fixing compile-time errors is easier than fixing runtime errors, which can be difficult to find.

  编译时进行更强的类型检查，编译器会对使用了泛型代码进行强类型检查，如果类型不安全，就会报错。编译时的错误会比运行时的错误，容易修复和查找。

- > Elimination of casts. The following code snippet without generics requires casting:
  >
  > 消除转换，下面代码片段是没有泛型所需的转换

  ```java
   List list = new ArrayList();
   list.add("hello world");
   String s = (String) list.get(0);
  ```

- > When re-written to use generics, the code does not require casting:
  >
  > 当我们用泛型重写, 代码就不需要类型转换

  ```java
  List<String> list = new ArrayList();
  list.add("hello world");
  String s =  list.get(0);
  ```

- > Enabling programmers to implement generic algorithms. 
  >
  > 使得程序员能够通用(泛型)算法。
  >
  > By using generics, programmers can implement generic algorithms that work on collections of different types, can be customized, and are type  
  >
  > safe and easier to read.
  >
  > 用泛型，程序员能够可以在不同类型的集合上工作，可以被被定制，并且类型是安全的，更容易阅读。

简单总结一下，引入泛型的好处，将类型当做参数，可以让开发者可以在不同的输入使用相同的代码，我的理解是，提升代码的可复用性，在编译时执行更强的类型检查，消除类型转换，用泛型实现通用的算法。那该怎么使用呢? 

##  泛型如何使用

### Hello World

上面我们提到泛型是类型参数，那我们如何传递给一个类，类型呢，类似于方法，我们首先要声明形式参数，它跟在类名后面，放在<>里面，在里面我们可以声明接收几个类型参数，如下所示:

```java
class name<T1, T2, ..., Tn> {}
```

下面是一个简单的泛型使用示例:

```java
public class Car<T>{
    private T data;
    
    public T getData() {
        return data;
    }
    public void setData(T data) {
        this.data = data;
    }
    public static void main(String[] args) {
        Car<Integer> car = new Car<>();
        car.setData(1);
        Integer result = car.getData();
    }
}
```

在没有泛型之前，我们的代码如果想实现这样的效果就只能用Object，在使用的时候进行强制类型转换像下面这样:

```java
public class Car{
    private Object data;

    public Object getData() {
        return data;
    }
    public void setData(Object data) {
        this.data = data;
    }
    public static void main(String[] args) {
        Car car = new Car();
        car.setData(1);
        Integer result = (Integer) car.getData();
    }
}
```

但类型转换的错误通常在运行时才能被发现，如果能在编译时发现，不是更好嘛。类型参数可以是指定的任何非原始类型: 类类型、接口类型、数组类型、甚至是另一个类型变量。同样的规则也可以被应用于泛型接口。

### 类型命名惯例

按照惯例，类型参数明示是单个大写字母的，常见的类型参数名称如下:

- E- 元素 广泛被Java集合框架所使用
- K - key 
- N -  数字
- Y - 类型
- V - 值
- S,U,V etc - 2nd, 3rd, 4th types

### 原始类型(Raw Type)

泛型类和泛型接口没有接收类型参数的名字，拿上面的Car类举例, 为了给传递参数类型，我们在创建car对象的时候就会给一个正常的类型:

```java
Car<Integer>  car = new Car<>();
```

如果未提供类型参数，你将创建一个Car的原始类型:

```java
Car car = new Car();
```

因此，Car是泛型类Car的原始类型，然而非泛型类、接口就不是原始类型。现在我们有一个类叫Dog， 这个Dog类不接收类型参数, 如下代码参数:

```java
class Dog{
    private String name;
   // get/set 构造省略
}
```

Dog就不是一个原始类型，原因在于Dog没有接收泛型参数。这里来讲下我的理解，一般方法需要的参数，调用方没有提供，编译不通过。为什么泛型没有引入此设计呢，不传递类型参数，那不通过编译不是更好嘛。那让我们回忆一下，泛型是从JDK的哪个版本开始引入的？没错，JDK 5引入的，也就是说如果我们引入泛型，但是又强制要求泛型类的代码，比如集合框架，在使用的时候必须传递类型参数，那么意味着JDK 5之前的项目在升级JDK 之后就会跑不起来，向前兼容可是Java的特色，于是Java将原来的框架进行泛型化，为了向前兼容，创造了原始类型这个概念，那有泛型的类，不传递类型参数，里面的类型是什么类型呢？当然是Object。C#引入泛型的时候，也面临了这个问题，不同于Java的兼容从前设计，加入了一套平行于一套泛型化版本的新类型。我们完全没有可能在一篇文章里面将泛型设计讨论清楚，我们将在后续的文章讨论泛型的演进。本篇我们着重于了解Java泛型的设计与实现。

在一些老旧的项目中(这里的老旧指的是JDK 5.0之前的Java项目)，你会看见原始类型, 因为在JDK 5.0之前，Java的许多API都没有泛型化(或通用化)， 如集合框架。当使用原始类型的时候，原始类型将获得泛型之前的行为，像上面的Car对象，在调用getData()方法的时候，会返回Object类型，这么做是为了向后兼容，这里是为了确保新代码可以和旧代码相互操作，Java编译器允许在新的代码中使用旧版本的代码和类库，Java语言的设计者考虑到了向后兼容性。 这里倒是获得了一些新的概念，以前我的脑海里面就没有向后兼容这个概念，只有向前兼容，那什么是向前兼容呢？ 我也好像只有模糊的概念，我在写的时候，思考了一下向前兼容这个词，向前面兼容，这个是前是指以前，还是前方呢? 上面提到的向后兼容指的是，后面的代码可以用之前的代码，向前兼容指的是，JDK 5之前的代码可以运行在JDK 5之后的版本上，这也就是二进制兼容性，Java所强调的兼容性，是"二进制向后兼容性"。例如说，一个在Java 1.2,1.4版本上可以正常运行的Class文件，放在Java 5、6、7、8的JRE(包括JVM与标准库)上仍然要可以正常运行。"Class文件"这里就是Java程序的“二进制表现”。 需要特别抢到的是, "二进制兼容性"并不等于"源码兼容性"(source compatibility)。既然谈到了，向前兼容、向后兼容，我们不妨讨论的再仔细一点，软件是一个很大的词，某种程度上来说，操作系统也是一个软件，对于系统的兼容性来说，向后兼容可以理解为Windows 10系统能够兼容运行Windows 3.1开发的程序上，Windows 10具备向后兼容性，这个向后中的后可以理解为过去，而不是以后指未来，**backward**。我们上面讨论的向后兼容也就是这个语义。向前兼容呢，Forward Compatibility, Windows 3.1能兼容运行Windows 10开发的程序，这就可以说明Windows 3.1 具有向前兼容性，一般操作系统都向后兼容。所以JDK 引入泛型的时候，将以前没有泛型的代码视为原始类型，是一种向后兼容的设计，为了Java的承诺，二进制兼容性。所以上面的用词还是有些问题，讨论问题的时候没有确定主体。







### 泛型方法





### 有界类型参数



###  泛型，继承，子类型



### 类型推断



### 通配符



### 上界通配符



### 下界通配符



## 泛型擦除





## 泛型限制





## 总结一下



##  参考资料

- Java 不能实现真正泛型的原因是什么？ https://www.zhihu.com/question/28665443/answer/1873474818
- 软件的「向前兼容」和「向后兼容」如何区分？ https://www.zhihu.com/question/47239021
- What Binary Compatibility Is and Is Not  https://docs.oracle.com/javase/specs/jls/se8/html/jls-13.html#jls-13.2
