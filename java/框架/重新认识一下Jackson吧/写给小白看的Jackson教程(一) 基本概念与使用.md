# 写给小白看的Jackson教程(一) 基本概念与使用

> 在前后端分离时代，服务端跟前端交互的基本数据格式就是json，在Java里面是对象的世界，但是在Java里面是对象的世界，在Spring  MVC下，我们的“return” 出去的对象是如何变成json的呢？ 再补充一下，使用Spring MVC，我们可以直接用对象去接前端传递的参数，那这个转换过程是由谁来完成的呢？这也就是Jackson。本篇我们依然采用问答体来介绍Jackson。

[TOC]



##  前言

依旧请出访谈系列的嘉宾小陈、老陈。小陈目前还是实习生，老陈是小陈的领导。某天小陈闲来无事，觉得就有必要看看面试题，突然收到了领导的微信: 

![对话](http://tva3.sinaimg.cn/large/006e5UvNgy1h6prtqblwjj30tu0mj3z8.jpg)

![对话(二)](http://tvax1.sinaimg.cn/large/006e5UvNgy1h6przrg5wsj30u30lr40v.jpg)

![对话三](http://tva4.sinaimg.cn/large/006e5UvNgy1h6ps0ryqakj30tt0m776n.jpg)

## what is Jackson？

 电脑面前的小陈挠了挠脑袋，还是学海无涯啊。打开了Jackson在Github上的仓库，看到:

> Jackson has been known as "the Java JSON library" or "the best JSON parser for Java". Or simply as "JSON for Java".
>
> Jackson身上最为人所熟知的恐怕是 Java 的JSON库，或者Java中最佳JSON解析者，或者简称为Java的JSON
>
> More than that, Jackson is a suite of data-processing tools for Java (and the JVM platform), including the flagship streaming JSON parser / generator library, matching data-binding library (POJOs to and from JSON) and additional data format modules to process data encoded in Avro, BSON, CSV, Smile, (Java) Properties, Protobuf ,TOML, XML or YAML and even the large set of data format modules to support data types of widely used data types such as Guava, Joda, PCollections and many, many more (see below).
>
> 但是Jackson不止于此，Jackson还是一套JVM平台的数据处理工具集: JSON解析与产生，数据绑定(JSON 转 对象，对象JSON)，还有处理Avro, BSON, CSV, Smile, (Java) Properties, Protobuf ,TOML, XML or YAML等数据格式等模块，还有大量的数据模块支持更加通用的数据类型处理比如：Guava、Joda Collections 等等。

总结一下就是Jackson不仅仅是一个JSON库，Jackson将自己定位为JVM平台上的数据处理工具，不仅仅是解析JSON，基本上支持了主流的数据格式解析与产生。那我现在最直接的诉求就是JSON到对象，对象到JSON那该怎么实现呢？答案就是ObjectMapper。

## 对象到JSON

Jackson做数据处理的核心类是ObjectMapper，那ObjectMapper提供了哪些方法来将对象转成JSON呢？ 我们进去找write方法开头的就行。

```java
public class ObjectMapperDemo {
    public static void objToJson() throws Exception{
        Map<String,Object> stringObjectMap = new HashMap<>();
        stringObjectMap.put("name","zs");
        stringObjectMap.put("date",new Date());
        ObjectMapper objectMapper = new ObjectMapper();
        System.out.println(objectMapper.writeValueAsString(stringObjectMap));
    }
    public static void main(String[] args) throws Exception {
        objToJson();
    }
}
```

输出结果:  ![Jackson输出](http://tvax3.sinaimg.cn/large/006e5UvNgy1h6psvehwhdj30m803fwej.jpg)



小陈看了看输出结果，为什么Date字段没变成了时间戳格式呢？ 这是默认的时间格式吗？ 如果是默认的话，有没有办法改呢？ 小陈开始搜ObjectMapper的方法， 果然ObjectMapper中就有setDateFormat方法，重新指定一下格式就好了:

```java
public class ObjectMapperDemo {

    public static void objToJson() throws Exception{
        Map<String,Object> stringObjectMap = new HashMap<>();
        stringObjectMap.put("name","zs");
        stringObjectMap.put("date",new Date());
        ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.setDateFormat(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));
        System.out.println(objectMapper.writeValueAsString(stringObjectMap));
    }
    public static void main(String[] args) throws Exception {
        objToJson();
    }
}
```

输出结果: ![注册DateFormat](http://tvax3.sinaimg.cn/large/006e5UvNgy1h6pt1oxkxaj30if025mx7.jpg)



到这里，小陈突然想起公司返回给前端的Date类型的字段都变成了yyyy-MM-dd HH:mm:ss，难道是Spring Boot 中的web自己注册的？ 于是小陈快速的搭了一个Spring Boot web项目，随意写了一个接口，想验证一下自己的猜想:

```java
@RestController
public class HelloWorldController {
    @GetMapping("hello")
    public Map<String,Object> test(String name) {
        Map<String,Object> map = new HashMap<>();
        map.put("aa",new Date());
        return map;
    }
}
```

![默认结果](http://tvax3.sinaimg.cn/large/006e5UvNgy1h6pt9dbkyfj30la02oweb.jpg)

看着浏览器的结果，小陈想其实公司还是对默认的ObjectMapper进行了改造，应该在项目的某个地方，只是自己没看到而已。

## JSON到对象

既然对象到JSON就是write开头的方法，那么我有理由相信，json到对象就是read方法开头。

![readValue的重载方法](http://tva4.sinaimg.cn/large/006e5UvNgy1h6ptn0nxnqj30v60h60vf.jpg)

 根据首个参数类型将其分为9组:

1. 首个参数是byte数组

2. 首个参数是DataInput

3. 首个参数是File对象

4. 首个参数是InputStream

5. 首个参数是JsonParser

6. 首个参数是Reader

7. 首个参数是String

8. 首个参数是URL

9. 首个参数是JsonParser

我们以首个参数为byte的为例进行研究，首个参数的类型我们可以理解为如何将JSON给ObjectMapper。

```java
public class ObjectMapperDemo {

    private  static ObjectMapper objectMapper = new ObjectMapper();

    private  static  void  initObjectMapper(){
        objectMapper.setDateFormat(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));
    }

    public static String  objToJson() throws Exception{
        Map<String,Object> stringObjectMap = new HashMap<>();
        stringObjectMap.put("name","zs");
        stringObjectMap.put("date",new Date());
        initObjectMapper();
        return  objectMapper.writeValueAsString(stringObjectMap);
    }
    public static void main(String[] args) throws Exception {
        String json = objToJson();
        jsonToObj(json);
    }
    
    private static void jsonToObj(String json) throws Exception {
        Map<String,Object> map = objectMapper.readValue(json, Map.class);
        System.out.println(map);
    }
}
```

那JavaType是用来干什么呢？我们来看下JavaType：

![JavaType](http://tva2.sinaimg.cn/large/006e5UvNgy1h6pu6n7pp5j30mo0fwt9e.jpg)

这是一个抽象类，那看它的子类: 

![javaType继承体系](http://tva4.sinaimg.cn/large/006e5UvNgy1h6pud6xh6bj310106eq5u.jpg)

这些类有什么存在意义呢？，这里有没有想到Java的泛型擦除呢？官方称之为类型擦除。JDK 5才引入了泛型，在此之前都是没有泛型的，当时引入泛型的主要目的是参数化类型，但是Java的一大优良特性就是向前兼容，所以Java在实现泛型的时候选择了类型擦除，某种程度上你可以理解为Java的泛型没有带到字节码中。那如果我想下面这样做该怎么办: 

```java
private static void jsonToObj(String json) throws Exception {
    // 这里会直接报错
    Map<String,Object> map = objectMapper.readValue(json, Map<String,Date>.class);
    System.out.println(map);
}
```

为了实现上面的效果，Jackson决定曲线救国, 推出了JavaType。那该怎么用呢，我们用CollectionType为例来介绍:

```java
// 省略get/set
public class Student {
    private int id;
    private String name;
    private String number;
    private Boolean status;
    private Map<String,Object> map;
}
private static <E> List<E> jsonToObj(String json, Class<? extends List> collectionType ,Class<E> elementType) throws Exception{
        CollectionType javaType = TypeFactory.defaultInstance().constructCollectionType(collectionType, elementType);
        return objectMapper.readValue(json,javaType);
}
```

那CollectionLikeType和MapLikeType是用来做什么的？  这个Like的意思是像，我们可以在这两个类的注释中一窥端倪:

- CollectionLikeType:

> Type that represents things that act similar to Collection; but may or may not be instances of that interface. This specifically allows framework to check for configuration and annotation settings used for Map types, and pass these to custom handlers that may be more familiar with actual type.

> 这个类型用于处理哪种行为上跟集合框架相似，但不是集合框架的实例。特别允许框架去检查用于Map类型的配置和注解，将其传递给更适合的处理器。
> 暂时还未找到符合这个要求的集合    

- MapLikeType:

> Type that represents Map-like types; things that consist of key/value pairs but that do not necessarily implement Map, but that do not have enough introspection functionality to allow for some level of generic handling. This specifically allows framework to check for configuration and annotation settings used for Map types, and pass these to custom handlers that may be more familiar with actual type.
> 处理像Map的类型，存储key-value对，但是没有实现Map接口。

  到现在来说，JDK内置的集合框架都是满足我们要求的，Apache Collections提供了对JDK集合框架的扩展：

> MultiValuedMap  是一个顶层的接口，允许存储key相同的key-value对

 现在我们来聊聊 TypeReference，这是另一种“避开泛型擦除”的一种方式，注释如是说:

> This generic abstract class is used for obtaining full generics type information by sub-classing; it must be converted to ResolvedType implementation (implemented by JavaType from "databind" bundle) to be used. Class is based on ideas from http://gafter.blogspot.com/2006/12/super-type-tokens.html , Additional idea (from a suggestion made in comments of the article) is to require bogus implementation of Comparable (any such generic interface would do, as long as it forces a method with generic type to be implemented). to ensure that a Type argument is indeed given.
> Usage is by sub-classing: here is one way to instantiate reference to generic type List<Integer>:
>     TypeReference ref = new TypeReference<List<Integer>>() { };which can be passed to methods that accept TypeReference, or resolved using TypeFactory to obtain
>
> 通过TypeReference的子类来获得完整的泛型信息，这个类的从链接中的文章得到设计灵感。通过这个类我们就可以直接将json转换为List<Student>.

使用示例: 

```java
 studentList  = objectMapper.readValue("", new TypeReference<List<Student>>() {});
```

我们也来赏析一下这篇文章，这篇文章的作者是大名鼎鼎的NEAL GAFTER(参与了JDK 1.4 到 5、Lambda表达式的设计)，这篇文章简短而又精炼，简单的讨论了如何在泛型擦除下，解决集合泛型中的泛型，在泛型擦除这种机制下，我们无法写出

```
List<Student>.class
```

 , 这也就是TypeReference的由来，TypeReference是一个抽象类，实例化该类，我们就可以通过反射获取到填入的泛型信息: 

```java
TypeReference<List<String>> x = new TypeReference<List<String>>() {};
```

 上面的参数中我们不认识的还有JsonParser，从类名上可以推断这个类负责Json解析，类上的注释如是说:

> Base class that defines public API for reading JSON content. Instances are created using factory methods of a JsonFactory instance.
>
> 定义一组读JSON内容的API，通过JsonFactory来进行实例化。

相对来说里面的方法更为原始，上面的读json，ObjectMapper就是借助JsonParser来做，里面定义了一些读json的底层方法. 本篇文章不做详细介绍，只做了解。

## JSON 特性配置

到现在为止，我们已经对读写JSON有了一个大致的了解，那我如果我想配置一些特性呢，比如说为了节省带宽，字段为null的就不返回给前端该怎么配置呢？

```java
// 在ObjectMapper中配置
objectMapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);
```

更多的特性可以在ObjectMapper的configure方法中进行配置, 下面是方法签名:

```java
// JSON 转对象
// DeserializationFeature 是一个枚举类
// 常见特性为: ACCEPT_EMPTY_ARRAY_AS_NULL_OBJECT  这个特性为将空字符串序列化为对象当作null值
public ObjectMapper configure(DeserializationFeature f, boolean state)
// 对象转JSON 
public ObjectMapper configure(SerializationFeature f, boolean state) 
```

## 总结一下

在Jackson中我们使用read来将json转对象，write来将对象转json，如果我们需要配置一些特性通过configure进行配置。

## 参考资料

- Java泛型类型擦除以及类型擦除带来的问题  https://www.cnblogs.com/wuqinglong/p/9456193.html
- Gson-如何处理泛型擦除  https://www.modb.pro/db/140112
- 程序员的福音 - Apache Commons Collections  https://zhuanlan.zhihu.com/p/396813478
