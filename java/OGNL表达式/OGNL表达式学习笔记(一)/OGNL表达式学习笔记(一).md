# OGNL表达式学习笔记(一) 基本特性与基本概念

[TOC]

## 前言

```java
<select id="findActiveBlogWithTitleLike"
     resultType="Blog">
  SELECT * FROM BLOG
  WHERE state = ‘ACTIVE’
  <if test="title != null">
    AND title like #{title}
  </if>
</select>
```

上面是一个MyBatis里面一个非常场景的动态SQL，如果传入的变量title != null 满足条件，最终执行的SQL就是:

```
# 如果title变量的值是%title%
SELECT * FROM BLOG WHERE state = ‘ACTIVE’ and  AND title like '%title%'
```

在我刚学MyBatis的时候认为title != null这个表达式是由Java编译器解析返回结果的，事实上后面也的确如此，后面我又觉得对MyBatis学习的不系统，这也就是：

- 《假装是小白之重学MyBatis(一)》
- 《假装是小白之重学MyBatis(二)》

(如果是在公众号没搜到这篇文章，可以去掘金、思否看)这篇的由来，在这篇文章我重新梳理JDBC、MyBatis之间的关系，重新学习了MyBatis的相关概念，在MyBatis官网看到下面这句话:

> 使用动态 SQL 并非一件易事，但借助可用于任何 SQL 映射语句中的强大的动态 SQL 语言，MyBatis 显著地提升了这一特性的易用性。如果你之前用过 JSTL 或任何基于类 XML 语言的文本处理器，你对动态 SQL 元素可能会感觉似曾相识。在 MyBatis 之前的版本中，需要花时间了解大量的元素。借助功能强大的基于 OGNL 的表达式，MyBatis 3 替换了之前的大部分元素，大大精简了元素种类，现在要学习的元素种类比原来的一半还要少。

再加上Arthas中也用到了OGNL表达式，但是我在用到Arthas的时候都是一边用一边查，效率着实不高，今天就来系统的梳理一下OGNL表达式的语法。有朋友不知道什么是Arthas，这里简单介绍一下:

> Arthas 是一款线上监控诊断产品，通过全局视角实时查看应用 load、内存、gc、线程的状态信息，并能在不修改应用代码的情况下，对业务问题进行诊断，包括查看方法调用的出入参、异常，监测方法执行耗时，类加载信息等，大大提升线上问题排查效率。

举一个例子，在线上某个方法执行出现异常，我们想查看某个方法执行的入参和出参，甚至直接让不重启执行我们的命令，我们就可以在Arthas中使用OGNL表达式来进行表达式求值。本篇我们不对Arthas做过多介绍，集中在OGNL表达式本身。

## 怎么用

让我们回到上面的例子中:

```java
<select id="findActiveBlogWithTitleLike"  resultType="Blog">
  SELECT * FROM BLOG
  WHERE state = ‘ACTIVE’
  <if test="title != null">
    AND title like #{title}
  </if>
</select>
```

在Mapper中我们的入参是Student对象，里面有一个title字段，首先要取值，我们首先就要引入OGNL的依赖:

```xml
<dependency>
    <groupId>ognl</groupId>
    <artifactId>ognl</artifactId>
    <version>3.4.1</version>
</dependency>
```

下面我们将演示如何从方法中取值:

```java
@Test
public  void OgnlDemo() throws OgnlException {
    Student student = new Student();
    student.setTitle("hello ognl");
    System.out.println(Ognl.getValue("title.length", student));
    System.out.println(Ognl.getValue("title.toCharArray()[0]", student));
}
```

上面这个例子表示我们要从student中取出title字段，然后再调用toCharArray方法将title转成字符数组，然后再从这个字符数组中取出第0个元素。输出结果为h，OGNL总是从一个对象开始，毕竟OGNL也是 Object-Graph  Navigation Language 对象导航图语言，对象导航，那在Java里面一切都是对象，那么就是去对象寻访属性，我们可以对集合进行投影、选择运算，也可以对对象使用Lambda表达式。那么什么是投影(projection)、选择(selection)，投影和选择是一个通用的概念，如下面的SQL:

```sql
select a, b, c from foobar where x=3;
```

a , b , c 是投影部分，表示要从表里面选取哪些列，where x = 3 是选择部分，表示要从表里选择哪些行。

在OGNL表达式里面最基本的单位是导航链, 通常称之为“链” ，最基本的链由以下部分组成:

| 表达元素部分 | 示例                                            |
| ------------ | ----------------------------------------------- |
| 属性名       | 像title和title.length                           |
| 方法调用     | title.toCharArray(), 返回的是一个字符数组       |
| 数组索引     | title.toCharArray()[0] 返回字符数组的第一个对象 |

我们可以将链理解为从对象里面获取的东西，student.name就代表我希望从student获取一个名叫name的属性，但对象里面要有get方法。不然就会报:

```java
ognl.NoSuchPropertyException: com.example.socket.controller.Student.title
```

### 简单总结一下

OGNL表达式用于用于访问对象的字段、方法。所以我们在使用OGNL表达式的时候需要一个对象，这在OGNL表达式中被称为root对象，以此来声明由哪个对象来执行这个表达式，返回结果。所有的OGNL表达式都在一个特定的数据环境中运行。OGNL的上下文环境是一个Map结构，称之为OgnlContext。Root对象也会被添加到上下文环境当中。

### 操纵OgnlContext

我们还是去Ognl这个类里面去找, 看看有没有对象的方法:

![](https://a.a2k6.com/gerald/i/2023/07/30/vp3ab.jpg)



然后发现直接创建OgnlContext需要四个参数，然后翻来翻去，在Ognl找到一个createDefaultContext方法，会返回上下文对象。我们借助这个方法来操纵上下文对象。示例如下:

```java
@Test
public  void OgnlContextDemo() throws OgnlException {
        Student student = new Student();
        student.setTitle("hello ognl");
        Map<Object,Object> map = new HashMap<>();
        // 创建上下文对象
        OgnlContext defaultContext = Ognl.createDefaultContext(student);
        map.put("init","init");
        map.put("student",student);
        // 将map放入上下文对象中
        // withValues 是将我们传入的Map遍历,
        // 放入OgnlContext自己的Map中
        // 所以我们需要在调用withValues方法之前,向Map里面放值
        defaultContext.withValues(map);
        // 从上下文里面取值需要加上#,不加#默认从root对象里面取值
        System.out.println(Ognl.getValue("#init", defaultContext,student));
        System.out.println(Ognl.getValue("#student.title", defaultContext,student));
        System.out.println(Ognl.getValue("title", defaultContext,student));
}
```

### 设置值

前面我们都是从root对象中获取值，有获取我们自然就会想到设置值，获取值是getValue，那么设置值就是setValue, 我们做出这样的推断， 在Ognl表达式中刚巧也有这个方法:

```java
@Test
public void ognlSetValue() throws OgnlException {
    Student student = new Student();
    Ognl.setValue("title",student,"hello world");
    System.out.println(Ognl.getValue("title", student));
}
```

### 调用静态方法和静态变量

静态方法和静态变量依附于类，所以我们调用的时候要加上包名，我们在Student里面随手放置一个静态变量和静态方法：

```java
private static String TEST_STATIC_VALUE = "test_static_value";
public static String  getStudent(){
  return "hello world";
}
```

```java
@Test
public void callStaticValueAndStaticMethod() throws OgnlException {
  System.out.println(Ognl.getValue("@com.example.socket.controller.Student@getStudent()", null));
  System.out.println(Ognl.getValue("@com.example.socket.controller.Student@TEST_STATIC_VALUE", null));
}
```

普通方法通过.来访问，静态方法和静态变量通过@来访问。

### 构建集合和对象

我们也可以通过表达式来创建集合和对象:

```java
@Test
public void  createCollection() throws OgnlException {
    Student student = new Student();
    OgnlContext context = Ognl.createDefaultContext(student);
    // context 也可以为null,里面会默认创建 #代表请求OGNL创建Map
    Object map = Ognl.getValue("#{'foo' :'foo value','bar' : 'bar value' }",context,student);
    // 这会被解析为List
    Object list = Ognl.getValue("{'hello','world'}", context,student);
    System.out.println(list);
    System.out.println(map);
    // new com.example.socket.controller.Student('qqqq') // 寻找Student的有参构造
 	// new com.example.socket.controller.Student()	// 寻找无参构造
    Object result = Ognl.getValue("new com.example.socket.controller.Student('qqqq')", context,student);
    System.out.println(result);
}
```

我们有了集合就可以开始进行投影和选择操作了。

### 投影

```java
@Test
public void collectionProjection() throws OgnlException {
    // 要进行投影操作,首先我们要有一个集合
    List<Student> studentList = new ArrayList<>();
    Student student = new Student();
    student.setName("zs");
    student.setTitle("a1");
    studentList.add(student);

    student = new Student();
    student.setTitle("b1");
    student.setName("lisi");
    studentList.add(student);
    OgnlContext defaultContext = Ognl.createDefaultContext("");
    defaultContext.put("list",studentList);
    // 从上下文里面出list元素中的title属性
    Object list = Ognl.getValue("#list.{title}",defaultContext,new Student());
    System.out.println(list);
    // 将title 和 name 进行拼接
    Object contactList = Ognl.getValue("#list.{title + name}",defaultContext,new Student());
    System.out.println(contactList);
}
```

### 选择

投影的操作符有三个:

- ?  所有满足条件的
- ^  从满足条件的选出第一个
- $  从满足条件的选出最后一个

示例:

```java
@Test
public void collectionSelection() throws OgnlException {
        // 要进行投影操作,首先我们要有一个集合
        List<Student> studentList = new ArrayList<>();
        Student student = new Student();
        student.setName("zs");
        student.setTitle("a1");
        studentList.add(student);

        student = new Student();
        student.setTitle("a2");
        student.setName("lisi");
        studentList.add(student);

        student = new Student();
        student.setTitle("b3");
        student.setName("lisi");
        studentList.add(student);


        OgnlContext defaultContext = Ognl.createDefaultContext("");
        defaultContext.put("list",studentList);
        // 从上下文里面出list元素中的title属性
        Object allList = Ognl.getValue("#list.{? #this.title.toCharArray[0] == 'a' }",defaultContext,new Student());
        System.out.println(allList);
        Object firstElementList = Ognl.getValue("#list.{^ #this.title.toCharArray[0] == 'a' }",defaultContext,new Student());
        System.out.println(firstElementList);
        Object lastElementList = Ognl.getValue("#list.{$ #this.title.toCharArray[0] == 'a' }",defaultContext,new Student());
        System.out.println(lastElementList);
}
```

### 常用操作符

OGNL 借鉴了 Java 的大部分运算符，并添加了一些新的运算符。在大多数情况下，OGNL 对给定运算符的处理与 Java 的相同，但有一个重要的注意事项，即 OGNL 本质上是一种无类型语言。这意味着 OGNL 中的每个值都是一个 Java 对象，而 OGNL 会尝试从每个对象中强制获得适合其使用情况的含义。

#### 且或运算符

```java
@Test
public void operatorDemo() throws OgnlException {
        List<Student> studentList = new ArrayList<>();
        OgnlContext defaultContext = Ognl.createDefaultContext("");
        Student student = new Student();
        student.setName("zs");
        student.setTitle("a1");
        studentList.add(student);
        defaultContext.put("list",studentList);
        // 且操作符
        Object logicAndOne = Ognl.getValue("#list !=  null && #list.size() > 0 ",defaultContext,student);
        System.out.println(logicAndOne);
        Object logicAndTwo = Ognl.getValue("#list != null and  #list.size() > 0 ",defaultContext,student);
        System.out.println(logicAndTwo);

        // 或运算符
        Object logicOrOne = Ognl.getValue("#list !=  null ||  #list.size() > 0 ",defaultContext,student);
        System.out.println(logicOrOne);
        Object logicOrTwo = Ognl.getValue("#list != null or  #list.size() > 0 ",defaultContext,student);
        System.out.println(logicOrTwo);
}
```

#### 顺序运算符

#### getValue

```
e1,e2
```

getValue方法对上述进行求值就是返回e2, setValue对上述表达式会先取得e1的值，然后设置给e2。老实说，我对顺序运算符并不理解，仅从取值的角度有些多余，像下面这样:

```java
@Test
public void  sequenceDemo01() throws OgnlException {
   Student student = new Student();
   student.setName("zs");
   student.setTitle("a1");
   Object valueResult = Ognl.getValue("name,title", student);
   System.out.println(valueResult);
   Object sameValueResult = Ognl.getValue("title", student);
   System.out.println(sameValueResult);
}
```

前后输出的结果是一致的，所以我认为顺序运算符是有些鸡肋的，于是我在搜索引擎上搜索，在Spring 项目上看到了一个issue:

```java
String mathEl = "a=3,b=3,(a+1)*(b-1)";
Object value2 = Ognl.getValue(mathEl, new HashMap<String, Object>());
System.out.println(value2);
SpelExpressionParser parser = new SpelExpressionParser();
Object value1 = parser.parseExpression(mathEl).getValue();
System.out.println(value1);
```

提issue的老兄叫timnick-snow, 说在mathEl这个表达式在OGNL中运行良好,在Spring EL表达式解析不成功。mathEl在OGNL的结果是8，也就是(3+1) * (2-1)的结果，顺序运算符可以用来声明变量执行数学计算。

#### setValue

```java
@Test
public void  sequenceDemo03() throws OgnlException {
    Student student = new Student();
    student.setName("zs");
    student.setTitle("a1");
    OgnlContext ognlContext = Ognl.createDefaultContext(student);
    Ognl.setValue("name,title",student,"hello world");
    // 输出结果为hello world
    System.out.println(Ognl.getValue("title", ognlContext, student));
}    
```

#### 加减运算符

非数字执行拼接，数字执行数学的加法。减法运算符只能工作于数字。如下示例:

```java
@Test
public void addOperator() throws OgnlException {
        Student student = new Student();
        student.setName("zs");
        student.setTitle("a1");
        Object valueResult = Ognl.getValue("name+title", student);
        // 输出结果为zsa1
        System.out.println(valueResult);

        Object minus  = Ognl.getValue("3-1", student);
        // 输出结果为2
        System.out.println(minus);
}
```

#### 比较运算符

```java
 @Test
 public void comparisonDemo() throws OgnlException {
        // 小于等于 非数字会要求实现Comparable接口, 如果没有实现会将参与运算的当作数字
        // <  和 lt 等价
        Student student = new Student();
        student.setName("a1");
        OgnlContext context = Ognl.createDefaultContext(student);

        context.put("list", Stream.of("a1","a2").collect(Collectors.toList()));
        System.out.println(Ognl.getValue("1 < 2 and 1 lt 2", new HashMap<>()));
        // <=  和  lte  等价
        System.out.println(Ognl.getValue("1 <= 2 and 2 lte 2", new HashMap<>()));
        // > 和 gt  等价
        System.out.println(Ognl.getValue("1 > 2 or  2 gt 1", new HashMap<>()));
        // >= gte  等价
        System.out.println(Ognl.getValue("1 >= 2 and   2 gte 2", new HashMap<>()));
        // in 语法
        System.out.println(Ognl.getValue("name in #list",context, student));
        // not in 语法
        System.out.println(Ognl.getValue(" 'sss' not in #list",context, student));
 }
```

## 写在最后

本篇最初就是将OGNL的Language Guide和Developer Guide翻译一下，写了一半发现，如果我是读者大致不喜欢这样的文章，因为我不关心OGNL的发展历史，我只想让你在基本概念这篇告诉该怎么用，常用，怎么验证。所以翻译了一半，又推翻了重写。其实翻译OGNL的Guide也不算是偷懒，原因在于，我希望告诉读者我是怎么得出这个结论的，我是如何得出这个判断的，我求知的过程，我不想只讲结论，我更愿意讲推导过程，我认为这比固定知识点更重要，授人以鱼不如授人以渔乎。但是在翻译的时候，感觉完全翻译，有效信息也不多，我想讲推导过程也许只用在文末讲一句看了参考资料[1]和参考资料[2] ，[1]是基本来源, [2]是参考了这篇文章的讲解方式，但是[2]是基于OGNL的3.1.19来讲解的，到OGNL3.4.1解析的方式发生了一点改变。

## 参考资料

[1] OGNL https://commons.apache.org/proper/commons-ognl/

[2] Ognl 表达式的基本使用方法  https://jueee.github.io/2020/08/2020-08-15-Ognl%E8%A1%A8%E8%BE%BE%E5%BC%8F%E7%9A%84%E5%9F%BA%E6%9C%AC%E4%BD%BF%E7%94%A8%E6%96%B9%E6%B3%95/

[3] Support OGNL's comma operator in SpEL https://github.com/spring-projects/spring-framework/issues/27766