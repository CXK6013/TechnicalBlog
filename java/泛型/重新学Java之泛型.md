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

我当时回答的时候是将Integer和int当做不同的类型来思考的，我回答的是可以，但是我的朋友说，这是不行的。后面我想了一下是泛型擦除嘛，朋友点头说是的。其实我对泛型擦除是有所耳闻的，只是平时觉得对我用处不大，所性就一直放在那里，不去思考，后面也在想是不是跟自动装箱拆箱有关系。最近也在一些工具类库，用到了泛型，发现自己对泛型的理解还是有所欠缺，所以今天就重新学习泛型，顺带梳理一下自己对泛型的理解，后面发现都揉在一篇文章里面，篇幅还是有些过大，这里就分拆两篇。

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

我们在来看下软件兼容，以安卓软件为例，每年都在发大版本，但是安卓手机现在的版本就是什么样的都有，2023年最新的安卓版本是13，但我手机的安卓版本是安卓11，那我去应用市场下载软件的时候，丝毫不考虑下载的软件是否能正常运行，原因就在于基本上软件也保留了一定的向前兼容。举一个例子来说，Android11的存储权限变更导致APP无法访问根目录文件，但是为了让为安卓11开发的软件能够跑在低版本的安卓上，这就要求开发者向前兼容。

### 泛型方法 Generic Method

> *Generic methods* are methods that introduce their own type parameters. This is similar to declaring a generic type, but the type parameter's scope is limited to the method where it is declared. Static and non-static generic methods are allowed, as well as generic class constructors.
>
> 所谓泛型方法指的就是方法上引入参数类型的方法，这与声明泛型类似。但是类型参数的范围仅于声明的范围。允许静态和非静态方法，也允许泛型构造函数。

下面是一个泛型静态方法: 

```java
// 例子来自于: The Java™ Tutorials
public class Pair<K,V> {
    private K key;
    private V value;
    // 泛型构造函数
    public Pair(K key, V value) {
        this.key = key;
        this.value = value;
    }
	// getters, setters etc.
}
public class CompareUtil {
    // 静态泛型方法
    public static <K,V> boolean compare(Pair<K,V> p1,Pair<K,V> p2){
        return p1.getKey().equals(p2.getKey()) && p1.getValue().equals(p2.getValue());
    }
    // 泛型方法
  	// 返回值左侧声明接收几个类型参数
    public <T1,T2> void compare(T1 t1,T2 t2){

    }
}
```

使用示例如下:

```java
 Pair<Integer, String> p1 = new Pair<>(1, "apple");
 Pair<Integer, String> p2 = new Pair<>(1, "apple");
 //  但是往往没人这么写,compare左侧的实际类型参数可以通过p1,p2推断出来。
 boolean isSame = CompareUtil.<Integer, String>compare(p1, p2);
```

我们更习惯的写法如下:

```java
boolean isSame = CompareUtil.compare(p1,p2);
```

 上面的特性, 我们称之为类型推断(type,inference) ,允许开发者将一个泛型方法作为普通方法来调用，而不需要在角括号中指定一个类型。更详细的讨论见下方的类型推断。

### 有边界的类型参数(Bounded type Parmeters)

有的时候我们希望对泛型进行限制，比如我写了一个比较方法，但是这个比较方法想限制传递进来的实际类型参数，只能为数字类型，这就需要对传入的类型参数加以限制，像下面这样:

```java
public <U extends Number> boolean compare(U u){
        return false;
}
```

U extends Number，compare接收的参数只能是Number或Number子类的实例，extends后面跟上界。我们传入String类型，编译就会不通过:

```java
// IDE 中会直接报错
CompareUtil.compare("2");
```

说到这里想起了《数学分析》上的确界原理: **任一有上界的非空实数集必有上确界（最小上界）；同样任一有下界的非空实数集必有下确界（最大下界）**。

当我们限制了泛型的上界，那我们就可以在泛型方法里面调用上界类的方法, 像下面这样:

```java
public static  <U extends Number> boolean compare(U u){
   u.intValue();
   return false;
}
```

 但有的时候一个上界可能还不够，我们希望有多个上界：

```java
<T extends B1 & B2 & B3>
```

 Java中虽然不支持多继承，但是可以实现多个接口，但是如果多个上界中某个上界是类，那么这个类一定要出现在第一个位置，如下所示:

```java
class A {}
interface B {}
interface C {}
class D <T extends A & B & C>
```

如果A不在第一个位置，就会编译报错。

#### 有界类型参数和泛型方法

有界类型参数是实现通用算法的关键，思考下面一个方法，该方法计算数组中大于指定元素elem的元素数量, 我们可能这么写:

```java
public static <T> int countGreaterThen(T[] anArray,T elem){
     int count = 0;
     for (T t : anArray) {
         if (t > elem ){
            count++;
         }
     }
    return count;
}
```

但由于你没有限制泛型参数的范围，上面的方法报错原因也很简单，原因在于操作符号(>)只能用于基本数据类型，比如short，int，double，long，float，byte，char。对象之间不能使用(>)，但这些数据类型都有包装类，包装类都实现了Comparable接口，我们就可以这么写:

```java
public static <T extends Comparable> int countGreaterThen(T[] anArray,T elem){
  int count = 0;
  for (T t : anArray) {
      if (t.compareTo(elem) > 0){
           count++;
       }
   }
   return count;
}
```

###  泛型，继承，子类型

我想你也知道，如果类型兼容，你可以将一个类型的对象引用指向另一个类型的对象，例如你可以将Object的引用指向Integer对象，原因在于Integer是Object类的子类:

```java
Object someObject = new Object();
Integer someInteger = new Integer(10);
someObject = someInteger;
```

在面向对象的术语中，这种被称为“is a” 关系，因为Integer是一种Object，所以允许赋值，但是Integer 也是Number的一种，所以下面的代码也是有效的:

```java
public void someMethod(Number n){};
someMethod(new Integer(10));
someMethod(new Double(10.1)); // ok
```

 现在我们来看下面这个方法:

```java
public void boxTest(Car<Number> n);
```

 这个方法接收哪些类型参数？ 单纯看方法签名,  我们可以看到，它接收的是Box\<Number>类型的参数，那它能接收Box\<Integer>、Box\<Double>之类的参数嘛，当然是不允许的：

```java
// 编译不会通过
CompareUtil.boxTest(new Car<Integer>());
```

 当我们使用泛型编程的时候，这是一个常见的误解，但这是一个需要学习的重要概念:

![diagram showing that Box<Integer> is not a subtype of Box<Number>](https://docs.oracle.com/javase/tutorial/figures/java/generics-subtypeRelationship.gif)

给两个具体类型A和B，比如Number和Integer，MyClass\<A>和MyClass\<B>之间是没关系的，但不管A和B是否有关系，MyClass\<A>和MyClass\<B>都有一个共同父类叫Object。

#### 泛型类和子类型

我们可以实现或继承一个泛型类和接口，两个泛型类、接口之间的关系由继承和实现的语句决定。用集合框架的例子来讲就是ArrayList\<E>` implements `List\<E>`, and `List\<E> extends Collection\<E>。所以ArrayList\<String>是List\<String>的一个子类型，而List\<String>是Collection\<String>的一个子类型。如果我们想定义自己的List接口，它将一个泛型P的可选值和List的每个元素都关联起来。它的声明可能像下面这样:

```java
interface PayloadList<E,P> extends List<E> {
  void setPayload(int index, P val);
}
```

下面参数类型是`List<String>`的子类型:

- `PayloadList<String,String>`
- `PayloadList<String,Integer>`
- `PayloadList<String,Exception>`

### 通配符

> In generic code, the question mark (`?`), called the *wildcard*, represents an unknown type. The wildcard can be used in a variety of situations: as the type of a parameter, field, or local variable; sometimes as a return type (though it is better programming practice to be more specific). The wildcard is never used as a type argument for a generic method invocation, a generic class instance creation, or a supertype.
>
> 在泛型代码中 ，？被称为通配符，代表未知类型。通配符可以在各种情况下使用:  作为参数、字段或局部变量的类型；有时作为返回类型(尽管更好的编程实际是更具体的)。通配符从不用作泛型方法的调用，泛型类示例创建或父类型的类型参数。 《java  Tutorial》

其实看到这块的时候，我对这个通配符是有点不了解的，我将这个符号理解为和T、V一样的泛型参数名，但是我用？去取代T的时候，发现IDEA里面出现了错误提示。那代表？号是特殊的一类泛型符号，有专门的含义。 假如我们想制作一个处理List\<Number>的方法，我们希望限制集合中的元素只能是Number的子类，我们看了上面的有界类型参数就可能会很自然的写出下面的代码:











#### 上界通配符



#### 类型推断



#### 下界通配符





## 泛型擦除





## 泛型限制





## 总结一下



##  参考资料

- Java 不能实现真正泛型的原因是什么？ https://www.zhihu.com/question/28665443/answer/1873474818
- 软件的「向前兼容」和「向后兼容」如何区分？ https://www.zhihu.com/question/47239021
- What Binary Compatibility Is and Is Not  https://docs.oracle.com/javase/specs/jls/se8/html/jls-13.html#jls-13.2
- 《**The Java™ Tutorials**》 https://docs.oracle.com/javase/tutorial/java/generics/methods.html
