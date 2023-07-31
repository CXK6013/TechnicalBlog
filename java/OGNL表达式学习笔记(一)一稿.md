# OGNL表达式(一)  基本介绍与入门

[TOC]



## 前言

一般来说用过MyBatis的都会对动态SQL有印象，像下面这样:

```java
<select id="findActiveBlogWithTitleLike"  resultType="Blog">
  SELECT * FROM BLOG
  WHERE state = ‘ACTIVE’
  <if test="title != null">
    AND title like #{title}
  </if>
</select>
```

在很早之前，我认为判断title != null 是将其映射为if(Blog.getTitle != null) 交给Java编译器执行，由编译器返回执行结果，后面又重新学些了MyBatis，这也就《假装是小白之重学MyBatis(一)》的由来，在重新学习MyBatis的过程中，看到了MyBatis的官方文档有下面这么一句话:

> 借助功能强大的基于 OGNL 的表达式，MyBatis 3 替换了之前的大部分元素，大大精简了元素种类，现在要学习的元素种类比原来的一半还要少。

所以if test里面解析的参数就是OGNL表达式，再加上Arthas 中诊断问题的时候也用到了OGNL表达式，但是掌握的不系统，这也就是本篇的目的，系统的整理OGNL表达式。其实OGNL表达式官网写的已经很清晰了，某种程度上来说我这里是汉化版。

## 概述

OGNL stands for Object-Graph Navigation Language; it is an expression language for getting and setting properties of Java objects, plus other extras such as list projection and selection and lambda expressions. You use the same expression for both getting and setting the value of a property.

OGNL代表对象导航图语言，它是一门为了从Java对象中获取设置属性、或从集合里面投影，选择，以及lambda 表达式的语言，你也可以使用相同的表达式来获取和设置属性值。

这里的解释投影(projection)和selection(选择)这两个词，各位可能感到有些陌生，这两个概念在SQL中也有所使用:

- **Projection** means choosing **which columns** (or expressions) the query shall return. 

> 投影指的是在查询中应当选择哪些列返回或表达式。

- **Selection** means **which rows** are to be returned.

> 选择指的是哪些列应该被返回。

```mysql
select a, b, c from foobar where x=3;
```

a ，b，c是投影部分，where x = 3是选择部分。对应集合就是要操纵集合中的哪些元素上面的哪些属性。Projection这个词其实翻译的有些怪，投影，我马上想到的就是投影仪, 但是看了看词典发现还真有这个意思，也许是引申了吧。

The `Ognl` class contains convenience methods for evaluating OGNL expressions. You can do this in two stages, parsing an expression into an internal form and then using that internal form to either set or get the value of a property; or you can do it in a single stage, and get or set a property using the String form of the expression directly.

Ognl这个类包含一些方便的方法可以用来评估OGNL表达式。你可以分两步来做，第一步将表达式解析为内部形式，然后使用内部格式来对属性来获取设置属性的值。 或者你也可以选择一步完成，直接使用表达式的字符串形式来获取设置属性。

OGNL started out as a way to set up associations between UI components and controllers using property names.

OGNL最初是作为一种使用属性名称在控制器和UI组件建立联系而设计的。

As the desire for more complicated associations grew, Drew Davidson created what he called KVCL, for Key-Value Coding Language, egged on by Luke Blanshard. Luke then reimplemented the language using ANTLR, came up with the new name, and, egged on by Drew, filled it out to its current state. 

随着更复杂关联的需求增长， 在Luke Blanshard  的推动下Drew Davidso创造了被他称为KVCL(key-value Coding Language)的语言。之后Luck使用ANTLR重新实现了这一语言，然后起了新的名字，并在Drew的推动下完善了它，使其发展到了当今这个状态。

Later on Luke again reimplemented the language using JavaCC. Further maintenance on all the code is done by Drew (with spiritual guidance from Luke).

后来Luke使用Java编译再次实现了这个语言，所有的代码维护由Drew负责(在Luke的精神指导下)。

## 介绍

Many people have asked exactly what `OGNL` is good for. Several of the uses to which `OGNL` has been applied are:

许多人会问OGNL有什么用，下面是OGNL的典型使用场景:

- A binding language between GUI elements (textfield, combobox, etc.) to model objects. Transformations are made easier by `OGNL`'s TypeConverter mechanism to convert values from one type to another (String to numeric types, for example);

   	GUI组件和对象之间的绑定，通过`OGNL`的TypeConverter机制，将值从一种类型转换为另一种类型变得更加容易（例如，从字符串转换为数值类型）。

- A data source language to map between table columns and a Swing TableModel;

  在表格列和Swing的TableModel之间进行映射的数据源语言。

- A binding language between web components and the underlying model objects;

  web组件和底层对象之间的绑定

Most of what you can do in Java is possible in `OGNL`, plus other extras such as list *projection*, *selection* and *lambda expressions*.

大部分在Java中能做的，在OGNL中都能做，比如列表投影、选择、lambda表达式。

## 语言指导 Language Guide

### 语法

Basic OGNL expressions are very simple. The language has become quite rich with features, but you don't generally need to worry about the more complicated parts of the language: the simple cases have remained that way. For example, to get at the name property of an object, the OGNL expression is simply `name`. To get at the `text` property of the object returned by the headline property, the OGNL expression is `headline.text`.

OGNL表达式非常简单，虽然这门语言的功能在变得越来越丰富，但通常你无需担心其中的复杂部分: 简单的部分仍然保持简单。例如现在要从对象获取name属性，OGNL表达式只需是name。要获取headline属性返回对象的text属性，OGNL表达式为headline.text。

What is a property? Roughly, an OGNL property is the same as a bean property, which means that a pair of get/set methods, or alternatively a field, defines a property (the full story is a bit more complicated, since properties differ for different kinds of objects; see below for a full explanation).

那么什么事属性？ 通常来说，OGNL的属性就是bean的属性，这意味着需要有一个字段，有一堆get/set方法，上下文要复杂一些。在下文有完整的解释。

The fundamental unit of an OGNL expression is the navigation chain, usually just called "chain." The simplest chains consist of the following parts:

OGNL表达式的基本单位是导航链，我们通常称之为链。最简单的链有以下三个部分组成:

| Expression Element Part | Example                                                      |
| :---------------------- | :----------------------------------------------------------- |
| Property names 属性名   | like the `name` and `headline.text` examples above           |
| Method Calls 方法调用   | `hashCode()` to return the current object's hash code        |
| Array Indices 数组索引  | `listeners[0]` to return the first of the current object's list of listeners |

All OGNL expressions are evaluated in the context of a current object, and a chain simply uses the result of the previous link in the chain as the current object for the next one. You can extend a chain as long as you like. For example, this chain:

所有 OGNL 表达式都是在当前对象的上下文中进行评估的，而链只是将链中上一个链接的结果作为下一个链接的当前对象。您可以随意扩展一个链，例如。

```
name.toCharArray()[0].numericValue.toString()
```

This expression follows these steps to evaluate:

- extracts the `name` property of the initial, or root, object (which the user provides to OGNL through the OGNL context);

  从初始对象提取name属性。

- calls the `toCharArray()` method on the resulting `String`;

  在生成的字符串上调用toCharArray方法

- extracts the first character (the one at index `0`) from the resulting array

​      从array中提取第一个字符

- gets the `numericValue` property from that character (the character is represented as a `Character` object, and the `Character` class has a method called `getNumericValue()`);

  从第一个字符上获取numericValue属性，这个字符代表Character对象，Character类名有一个名为getNumericValue的方法。

- calls `toString()` on the resulting `Integer` object. The final result of this expression is the `String` returned by the last `toString()` call.

从Integer上调用toString方法，表达式的最后结果是被最后的toString方法所返回。

Note that this example can only be used to get a value from an object, not to set a value. Passing the above expression to the `Ognl.setValue()` method would cause an `InappropriateExpressionException` to be thrown, because the last link in the chain is neither a property name nor an array index.This is enough syntax to do the vast majority of what you ever need to do.

注意这个例子只被用于展示从对象中读取值，将上述表达式传递给Ognl.setvalue方法会抛`InappropriateExpressionException` 异常，因为链中的最后一环既不是属性名也不是数组索引。

###  引用属性



### 索引



### 



## 参考资料

- OGNL官网 https://commons.apache.org/proper/commons-ognl/language-guide.html
- What are projection and selection?  https://stackoverflow.com/questions/1031076/what-are-projection-and-selection