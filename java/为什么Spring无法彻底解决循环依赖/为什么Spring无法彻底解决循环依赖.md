# 记一次排查循环依赖的经历



## 前言

Spring是如何解决循环依赖的，想来类似解读的文章已经如汗牛充栋，我在之前背面试题的时候也是将其认为是高频面试题去准备，但是背的时候也是记了个大概，但是记得一年前的一个项目，在windows上运行的好好的，在Linux上就报循环依赖，当时用了@Lazy注解解决这个问题，其实当时心里还是有疑问的，想不明白为什么一个jar在windows上运行就没问题，在Linux上就报循环依赖这让我颇为不解，但是注意力不在这个上面，再加上用@Lazy解决了，于是这个问题就被抛到脑后。今年再次碰到了这个问题，项目在windows上运行没问题，但是在Linux上运行就报循环依赖，后面用@Lazy也没解决这个问题，于是开始探索背后的原因，因为之前背过面试题, Spring是如何解决循环依赖的，也看了一些文章，但是实际上循环依赖该出现还是出现了，这肯定是认知出现了问题。

## Spring是如何解决循环依赖的

那么要提到是如何解决循环依赖的，那么首先就要考察这么一个概念，即什么是循环依赖，使用了Spring帮助我们管理对象之后，我们可以在一个类里面这样写:

```java
@Service
public class A
  @Autowired  
  private B b;
}
@Service
public class B{
  @Autowired   
  private A a;   
}
```

上面的代码非常简单，我们在启动Spring Boot web项目的时候会首先加载容器，容器扫描到类A上面，发现A上面有Service注解，就会创建A类的实例，然后再创建的时候发现成员变量需要注入，于是就去创建B，扫描到B的时候发现B需要A来注入，但是注意此时我们就是因为扫描到A才来创建B，但是再创建B的时候又需要A，这就像是鸡生蛋的问题。这也就是循环依赖，存在两个类A、B，A里面的成员变量有B，B里面的成员有A。但是上面的代码在Spring Boot web启动的时候，还是能启动的。这让我想到是不是我的Spring Boot 版本比较低，默认是允许循环依赖，我想起Spring 2.6.x版本开始禁用循环依赖，改为由一个开关控制:

```yaml
spring:
  main:
    allow-circular-references: true
```

然后我们升级一下Spring Boot版本到2.6，试试看，然后果然项目启动报错。Spring 官方提到了循环依赖，并如此说道:

> You can generally trust Spring to do the right thing. It detects configuration problems, such as references to non-existent beans and circular dependencies, at container load-time. Spring sets properties and resolves dependencies as late as possible, when the bean is actually created.
>
> 通常情况你可以相信Spring总会做正确的事情，它会在启动容器加载的时候，检测配置问题，比如引用了一个不存在的bean，循环依赖。Spring 在实际创建bean的时候会尽可能晚的设置属性、解决依赖。

那么如何解决的也就呼之欲出了，简单的说就是尽可能晚一点做依赖注入，也就是说在初始化A的时候，发现A需要B，但是此时B还没初始化完成，选择初始化其他需要依赖注入的变量进行注入，如果需要注入的变量依然没有在容器里面初始化完成，也就是说，先将能A的实例中能依赖注入的注入，然后返回出去，B拿到了A，然后B的实例bean创建完成，然后为A进行依赖注入。

Spring的官方文档也是这么说的:

> If you use predominantly constructor injection, it is possible to create an unresolvable circular dependency scenario.
>
> 如果你主要使用构造器注入，那就可能会出现无法被解决的循环依赖。

> For example: Class A requires an instance of class B through constructor injection, and class B requires an instance of class A through constructor injection. If you configure beans for classes A and B to be injected into each other, the Spring IoC container detects this circular reference at runtime, and throws a `BeanCurrentlyInCreationException`.
>
> 例如：类A通过构造函数注入需要类B的一个实例，而类B通过构造函数注入需要类A的一个实例。如果你配置了类A和类B的bean以相互注入，Spring IoC容器会在运行时检测到这种循环引用，并抛出一个`BeanCurrentlyInCreationException`异常。

> One possible solution is to edit the source code of some classes to be configured by setters rather than constructors. Alternatively, avoid constructor injection and use setter injection only. In other words, although it is not recommended, you can configure circular dependencies with setter injection. 
>
> 一种解决方案是改代码使用set方法注入。另一种选择是避免使用构造器注入，而仅使用set注入。尽管不推荐这样做，但你可以使用set注入来解决循环依赖。

这就意味着要求类里面有一个无参构造函数才能尽可能晚的设置属性、依赖注入。如果里面没有无参构造函数，Spring 无法延迟依赖注入呢，那么即使在Spring里面打开了允许循环依赖的开关，也是无济于事的:

```java
@Component
public class AService {
    private final BService bService;
 
    public AService(BService bService) {
        this.bService = bService;
    }
}
@Component
public class BService {
   
    private final AService aService;
    
    public BService(AService aService) {
        this.aService = aService;
    }
}
```

![](https://a.a2k6.com/gerald/i/2023/10/06/irqw.jpg)

对应也有解决方案，我们可以使用@Lazy注解:

```java
@Component
public class AService {
    private final BService bService;

    public AService(@Lazy BService bService) {
        this.bService = bService;
    }
}
```

打上了@Lazy注解之后，容器在通过构造函数来创建AService这个实例的时候就会BService属性的加载，在加载BService的时候发现这个属性被@Lazy修饰，那么就不会直接加载BService，而是产生了一个代理对象来做依赖注入，这样AService就能够在容器中初始化完成了。

在翻Spring文档的时候有意外收获，看到了这个: 

> The Spring team generally advocates constructor injection as it enables one to implement application components as *immutable objects* and to ensure that required dependencies are not `null`. 
>
> Spring团队提倡使用构造器进行依赖注入，因为它使得可以将应用程序组件时限为不可变对象，并确保所需要的依赖项不为空。
>
> Furthermore constructor-injected components are always returned to client (calling) code in a fully initialized state. As a side note, a large number of constructor arguments is a *bad code smell*, implying that the class likely has too many responsibilities and should be refactored to better address proper separation of concerns.  
>
> 此外，构造函数注入的组件在返回给调用方的时候总是完全初始化的状态。顺便提一下，构造函数有太多参数是一种代码的坏味道，暗示着该类具有过多的职责，应该进行重构以更好的实现关注点分离。
>
> 《spring-framework-docs》见参考文档1

这里面暗含的意思是项目在迭代的过程中会慢慢出现违背职责分离原则的状况，这就需要重构，代码不是一成不变的。看到这里可能就有同学会说了，你难道就想告诉我通过构造器做依赖注入的，Spring的默认加载机制无法解决循环依赖？ 你不是也说了嘛，通过@Lazy方式就可以解决了。所以你就想告诉我这些嘛，好无趣的文章。对此我想说，当然不是，请让我细细道来。

## 溯源

### 先上代码

别的先不说，让我们先上代码:

```java
// 该类位于org.springframework.boot.issues.gh6045.server
@Service
public class MyServer {
    Logger logger = LoggerFactory.getLogger(MyServer.class);

    @Autowired
    ExampleService exampleService;

    public MyServer() {
        logger.info("MyServer initialized.");
    }

    public void execute() {
    }
}
// 该类位于 org.springframework.boot.issues.gh6045.service下面
@Service
public class ExampleService {

    Logger logger = LoggerFactory.getLogger(ExampleService.class);

    MyServer myServer;

    @Autowired
    public ExampleService(MyServer myServer) {
        this.myServer = myServer;
        logger.info("ExampleService initialized.");

    }
    public void execute() {
    }
}
@Configuration
// WORKING
/*@ComponentScan(basePackages = {
    "org.springframework.boot.issues.gh6045.server",
    "org.springframework.boot.issues.gh6045.service",
    "org.springframework.boot.issues.gh6045"})*/

// throws BeanCurrentlyInCreationException
/*@ComponentScan(basePackages = {"org.springframework.boot.issues.gh6045",
    "org.springframework.boot.issues.gh6045.server",
   "org.springframework.boot.issues.gh6045.service"})*/
// throws BeanCurrentlyInCreationException
@ComponentScan(basePackages = "org.springframework.boot.issues.gh6045")
@EnableAutoConfiguration
public class Application {

    public static void main(String[] args) {
        SpringApplication springApp = new SpringApplication(Application.class);
        springApp.run(args);
    }
}
```

上面的代码来自于https://github.com/patoi/spring-boot-issues/tree/gh-6045/gh-6045，这个问题也是这个issue报出来的，本篇的文章就是依托于这个issue而展开的，运行这段代码，你会发现在windows上面都是正常工作的，那照正常流程是不是可以说没有复现呢，当然不能，因为在其他操作系统上会有不同的表现，那问题该如何复现呢? 我们可以再切换成一个等价的例子: 

```java
@Component
public class ATest0 {

    private ATest1 a;
    
    @Autowired
    public void setA(ATest1 a) {
        this.a = a;
    }
}
@Component
public class ATest1 {
    private final ATest0 a;

    public ATest1(ATest0 a) {
        this.a = a;
    }
}
```

尽管产生了循环依赖还是可以正常工作的，在包里面的文件顺序是ATest0在ATest1前面，ATest0是set注入，ATest1是构造器注入，如果让构造器注入的文件顺序在set注入之前呢，也就是说我们将ATest1修改名称为ATest0，而将ATest0改为ATest1:

```java
@Component
public class ATest0 {
    
   private final ATest1 a;
    
   public ATest0(ATest1 a) {
        this.a = a;
    }
}
@Component
public class ATest1 {

    private ATest0 a;

    @Autowired
    public void setA(ATest0 a) {
        this.a = a;
    }
}
```

就会报循环依赖的错误:

![](https://a.a2k6.com/gerald/i/2023/10/15/2oz8.jpg)



这里其实可以下基本结论就是初始化Bean的顺序不同会导致循环依赖，尽管事实上你不应该写出循环依赖，但我们还是要分析一下问题出在哪里。 到这里说个省流的答案吧，部分循环依赖依赖于创建bean的顺序，而bean的创建顺序又依赖于从jar中提取class文件的顺序，而从jar中提取class文件的顺序又依赖于maven等打包工具添加class的顺序，而maven打包扫描的顺序在不同平台上无法保证一致，因此导致了windows上面没有出现循环依赖，部分Linux版本出现了循环依赖。

除此之外，Spring是如何判断项目里面有循环依赖的呢，在创建Bean之前会将这个bean的名称放到一个set里面，如果这个bean需要其他bean的注入，我们姑且就用A来代称，然后就转而去寻找其他bean，这个被需要的bean还没有被创建，被需要的bean就用B来代称，就去创建B，然后向这个集合里面添加B，set里面就有A和B，发现B需要A，然后接着创建A，发现A在集合里面已经有了，因此判定出现了循环依赖。因为Bean的创建是一个自动化的过程，在根据Bean定义去创建的过程是发现A创建的过程中需要B，转而就去寻找B，没有就创建，检查也就是在创建Bean的时候进行检查。

下面是详细的探索过程， 让我们首先从@ComponentScan是如何起作用开始说起。

### @ComponentScan 如何起作用

我们还是用IDEA来调试，来猜一下这个注解是如何起作用的，首先进入ComponentScan的源码，然后摁着ctrl看这个类在哪里被使用:

![](https://a.a2k6.com/gerald/i/2023/10/06/wv7r.jpg)

我们点进这个类来看一下：

![](https://a.a2k6.com/gerald/i/2023/10/06/4rdk.jpg)

我们看下这个parse方法的逻辑:

![](https://a.a2k6.com/gerald/i/2023/10/06/3i6.jpg)

![](https://a.a2k6.com/gerald/i/2023/10/06/xis7.jpg)

这个scanner是ClassPathBeanDefinitionScanner，我们看下它的doScan方法:

```java
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
		Assert.notEmpty(basePackages, "At least one base package must be specified");
		Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
		for (String basePackage : basePackages) {
            // 这里去包里面提取class文件并构造成bean
			Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
			for (BeanDefinition candidate : candidates) {
				ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
				candidate.setScope(scopeMetadata.getScopeName());
				String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
				if (candidate instanceof AbstractBeanDefinition) {
					postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
				}
				if (candidate instanceof AnnotatedBeanDefinition) {
					AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
				}
				if (checkCandidate(beanName, candidate)) {
					BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
					definitionHolder =
							AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
					beanDefinitions.add(definitionHolder);
					registerBeanDefinition(definitionHolder, this.registry); // 语句二
				}
			}
		}
		return beanDefinitions;
}
```

语句二的作用事实上就是加载对应的bean了，这个registry是BeanDefinitionRegistry，记得我们在《如何向Spring IOC 容器 动态注册bean》一文中就是用BeanDefinitionRegistry的registerBeanDefinition方法完成动态注册bean的，但BeanDefinitionRegistry是一个接口，那在ClassPathBeanDefinitionScanner用的是BeanDefinitionRegistry的哪个实现类呢？ 这个registry是通过ClassPathBeanDefinitionScanner的构造函数进行初始化的，ClassPathBeanDefinitionScanner的构造函数有4个，最终都是调用到了参数最多的构造函数:

```java
public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters,
			Environment environment, @Nullable ResourceLoader resourceLoader) 
```

在这个构造参数打断点我们来观察是BeanDefinitionRegistry的哪个实现类:

![](https://a.a2k6.com/gerald/i/2023/10/06/6oamk.jpg)

 我们接着看AnnotationConfigServletWebServerApplicationContext，看看registerBeanDefinition的实现:

![](https://a.a2k6.com/gerald/i/2023/10/06/y9s6.jpg)

我们转到GenericApplicationContext这个来看下:

```java
// 省略无关代码
public class GenericApplicationContext extends AbstractApplicationContext implements BeanDefinitionRegistry {
	
    private final DefaultListableBeanFactory beanFactory;
    
    @Override
	public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException {
		this.beanFactory.registerBeanDefinition(beanName, beanDefinition);
	}
}
```

registerBeanDefinition的逻辑如下:

![](https://a.a2k6.com/gerald/i/2023/10/06/11642.jpg)

#### 总结一下这个阶段

![](https://a.a2k6.com/gerald/i/2023/10/15/3r7ts.jpg)

在registerBeanDefinition这个方法也就还是将beanName和beanDefinition放到一个map里，将beanDefinitionName放到一个List里面，然后就结束了。

那什么时候开始正式创建Bean呢，还得从SpringApplication的run方法开始看起。

### 那什么时候才正式开始创建bean

Spring Boot启动的流程是由main线程调用SpringApplication的run方法启动:

![](https://a.a2k6.com/gerald/i/2023/10/06/7erlj.jpg)

refreshContext这个逻辑如下:

```java
private void refreshContext(ConfigurableApplicationContext context) {
	if (this.registerShutdownHook) {
			shutdownHook.registerApplicationContext(context);
    }
	refresh(context);
}
// ConfigurableApplicationContext是接口，实现类是AnnotationConfigServletWebServerApplicationContext，
protected void refresh(ConfigurableApplicationContext applicationContext) {
    applicationContext.refresh();
}
// AnnotationConfigServletWebServerApplicationContext的refresh()方法如下
public final void refresh() throws BeansException, IllegalStateException {
		try {
			super.refresh();
		}
		catch (RuntimeException ex) {
			WebServer webServer = this.webServer;
			if (webServer != null) {
				webServer.stop();
			}
			throw ex;
		}
}
```

最终走到了AbstractApplicationContext的refresh方法上:

![](https://a.a2k6.com/gerald/i/2023/10/06/12ccb.jpg)

​		![](https://a.a2k6.com/gerald/i/2023/10/06/7hc85.jpg)

![](https://a.a2k6.com/gerald/i/2023/10/06/s60.jpg)

这个getBean往下看事实上是调用了doGetBean:

![](https://a.a2k6.com/gerald/i/2023/10/15/2w3v.jpg)

![](https://a.a2k6.com/gerald/i/2023/10/15/rmfhd.jpg)

​	然后我们看下getSingleton方法的执行逻辑:

​	![](https://a.a2k6.com/gerald/i/2023/10/15/2wn1.jpg)



那它是怎么判定的呢? 

```java

private final Set<String> inCreationCheckExclusions = Collections.newSetFromMap(new ConcurrentHashMap<>(16));

private final Set<String> singletonsCurrentlyInCreation = Collections.newSetFromMap(new ConcurrentHashMap<>(16));

protected void beforeSingletonCreation(String beanName) {
  if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.add(beanName)) {
			throw new BeanCurrentlyInCreationException(beanName);
  }
}
```

![](https://a.a2k6.com/gerald/i/2023/10/15/40n7t.jpg)

 那想来逻辑是先创建ATest0，然后向这个Set里面添加，创建ATest0这个Bean需要ATest1,然后再次走了这段逻辑，但是ATest1在创建的时候又需要ATest0，然后接着走了这段逻辑，原来是这么判断循环依赖的呀。

我们还是回到创建Bean的那段逻辑，也就是DefaultListableBeanFactory的preInstantiateSingletons，来跟一下ATest0是如何被实例化的, 其实还是上面的逻辑:

getBean => doGetBean => 执行到getSingleton方法，传入了一个lambda函数去创建Bean。

![](https://a.a2k6.com/gerald/i/2023/10/15/4bf0k.jpg)

![](https://a.a2k6.com/gerald/i/2023/10/15/4btfa.jpg)

![](https://a.a2k6.com/gerald/i/2023/10/15/2bb.jpg)

![](https://a.a2k6.com/gerald/i/2023/10/15/lzuc.jpg)

然后在这个方法调试，就不知道怎么回事跳到createBean了:

![](https://a.a2k6.com/gerald/i/2023/10/15/n9ql.jpg)

我调试了好几次，都没有捕捉到是怎么调到这里的，于是想到了获取当前方法的调用栈来观察:

![](https://a.a2k6.com/gerald/i/2023/10/15/ndtf.jpg)

那到这里说明我们的想法是正确的，就是创建ATest1需要ATest0，然后去创建ATest0，然后发现创建ATest0做依赖注入需要ATest1，接着创建ATest1，于是触发了循环依赖检查的方法beforeSingletonCreation抛出异常。到这里我们已经发现了，如果set注入在前面就没问题。那么为什么在windows上就没问题呢，放到服务器上就出现了循环依赖呢？ 让我们从classpath说起。

### 被忽视的classpath

对于一个后端程序员一天的工作大概是打开IDEA，然后启动项目，所谓:

![](https://a.a2k6.com/gerald/i/2023/10/06/4hzsk.png)

一般现在都是Spring Boot web项目，然后我们会找到main函数，然后正常启动或者debug模式启动，当我们启动的时候隐含的一个过程就是启动的时候加载项目所需的依赖和项目自身的class文件，那么在启动的时候项目会去哪里搜寻呢?  答案是class path，class path是class search path的缩写，这个路径是运行时会扫描的路径。那么默认的class path是什么呢？ 我们在一个Spring Boot web项目中加上下面的代码:

```java
System.out.println(System.getProperty("java.class.path"));
```

就会发现输出的有当前项目target目录的全路径，我的项目名是simple-spring，所以输出的路径是D:\IdeaProjects\simple-spring\target\classes，注意到项目启动依赖的class放在不同的文件夹下，然后就会输出多个路径，在windows上分隔符是；在 Solaris 系统中以冒号（:）分隔类路径。在IDEA启动的时候找的是target下面的class。但是我们启动的时候用java -jar命令启动的，这里就有些不一样，那么究竟是哪里不一样呢？

### 哪里不一样呢？

回到扫描包的代码中去，也就是ClassPathBeanDefinitionScanner的doScan方法：

![](https://a.a2k6.com/gerald/i/2023/10/15/3dn4.jpg)

![](https://a.a2k6.com/gerald/i/2023/10/15/nr0a.jpg)

  我们看下它是如何读取的:

![](https://a.a2k6.com/gerald/i/2023/10/15/nx5e.jpg)

那我们以java -jar方式调试的时候走的是哪个方法，我们在IDEA以jar的方式运行然后调试，那么要在IDEA里面调试jar应当怎么做呢:

![](https://a.a2k6.com/gerald/i/2023/10/15/hiw.jpg)

![](https://a.a2k6.com/gerald/i/2023/10/15/4o345.jpg)

然后会发现在jar模式下:

![](https://a.a2k6.com/gerald/i/2023/10/15/o307.jpg)

然后一直往下走会走到doFindPathMatchingJarResources这个方法里面，也就是从jar里面提取文件:

![](https://a.a2k6.com/gerald/i/2023/10/15/of6z.jpg)

​	拿到jarFile之后遍历获取jar文件:

![](https://a.a2k6.com/gerald/i/2023/10/15/3jbf.jpg)

其实到这一步基本上已经拿到了jar中class文件，那么是在哪一步拿的呢，在roorDirURL.openConnection 进入下一步发现进不去，原因在于我们的pom里面缺少依赖:

```xml
<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-loader-tools</artifactId>
            <version>2.7.14</version>
 </dependency>
 <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-loader</artifactId>
            <version>2.7.14</version>
</dependency>
```

然后进入下一步: 

![](https://a.a2k6.com/gerald/i/2023/10/21/3rgdm.jpg)

### 读取顺序是如何确定的呢？

理一理目前的思绪，现在我们已知的是在Spring下面循环依赖与Bean的创建顺序有关，在上面的例子中我们已经看到，构造函数注入的bean先于Set注入，两个bean有循环依赖，Spring也没给我们解决，那么Bean的创建顺序又依赖于从jar里面读取class的顺序，那这个读取顺序是怎么确定的呢？ 在openConnection里面执行了语句一:

```java
@Override
protected URLConnection openConnection(URL url) throws IOException {
   if (this.jarFile != null && isUrlInJarFile(url, this.jarFile)) {
      return JarURLConnection.get(url, this.jarFile); // 语句一
   }
   try {
      return JarURLConnection.get(url, getRootJarFileFromUrl(url));
   }
   catch (Exception ex) {
      return openFallbackConnection(url, ex);
   }
}
```

然后我们再看doFindPathMatchingJarResources方法，拿到URLConnection，走的逻辑是:

```java
if (con instanceof JarURLConnection) {
   // Should usually be the case for traditional JAR files.
   JarURLConnection jarCon = (JarURLConnection) con;
   ResourceUtils.useCachesIfNecessary(jarCon);
   jarFile = jarCon.getJarFile();
   jarFileUrl = jarCon.getJarFileURL().toExternalForm();
   JarEntry jarEntry = jarCon.getJarEntry();
   rootEntryPath = (jarEntry != null ? jarEntry.getName() : "");
}
```

```java
Set<Resource> result = new LinkedHashSet<Resource>(8);
// 也就是获取jarFile的entries迭代
for (Enumeration<JarEntry> entries = jarFile.entries(); entries.hasMoreElements();) {
   JarEntry entry = entries.nextElement(); 
   String entryPath = entry.getName();
   if (entryPath.startsWith(rootEntryPath)) {
      String relativePath = entryPath.substring(rootEntryPath.length());
      // 满足条件 
      if (getPathMatcher().match(subPattern, relativePath)) {
         // 将其加入到result里面,
         result.add(rootDirResource.createRelative(relativePath));
      }
   }
}
```

result是Set集合，Set是无序的？ 但带上了Linked，所以内部有一个链表实现？ 我们看下他的注释:

> Hash table and linked list implementation of the Set interface, with predictable iteration order. This implementation differs from HashSet in that it maintains a doubly-linked list running through all of its entries
>
> Set 接口的哈希表和链表实现，具有可预测的迭代顺序。这种实现方式与 HashSet 不同，它维护一个贯穿所有条目的双链表。该链表定义了迭代顺序，即元素插入集合的顺序（插入顺序）。

所以遍历顺序即为插入顺序，而插入顺序即为创建bean的顺序，那迭代顺序是如何确定的?  我们在openConnection可以看到在执行这个方法的时候，里面的成员变量jarFile已经完成了初始化，那么这个初始化是在什么时候执行的呢，我们找下这个Handler的构造函数，看看这个构造函数在什么时候被调用, 注意这个Handler是位于org.springframework.boot.loader.jar下面，里面刚好有一个这样的构造函数:

```java
public Handler(JarFile jarFile) {
   this.jarFile = jarFile;
}
```

我们还是找Handler这个类在哪里被调用:

![](https://a.a2k6.com/gerald/i/2023/10/21/jch.jpg)

那JarFile又是被谁调用的呢？

![](https://a.a2k6.com/gerald/i/2023/10/21/569fp.jpg)

看源码吧，就是这么试探，试试这个，找这个JarFile的使用地方太多了，不好看，就找到这个构造函数去看看再哪里被使用，如果试错了，就接着试探别的，科学的精神就是如此，大胆猜想。然后我们转到JarFileArchive这个类去打断点，看看这个类的调用栈:

![](https://a.a2k6.com/gerald/i/2023/10/21/57cl0.jpg)

JarFile的entries方法就是个迭代器，那么里面的逻辑是什么样呢? 

```java
@Override
public Enumeration<java.util.jar.JarEntry> entries() {
    return new JarEntryEnumeration(this.entries.iterator());
}
```

那entries是在什么时候赋值的呢？ 

![](https://a.a2k6.com/gerald/i/2023/10/21/5atm5.jpg)

然后这个entries顺序还是没找到在哪里？难道是方向错了？ 再看一眼读取jarFile的逻辑：

```
Set<Resource> result = new LinkedHashSet<>(8);
    for (Enumeration<JarEntry> entries = jarFile.entries(); entries.hasMoreElements();) {
		JarEntry entry = entries.nextElement();
		String entryPath = entry.getName();
		if (entryPath.startsWith(rootEntryPath)) {
			String relativePath = entryPath.substring(rootEntryPath.length());
			if (getPathMatcher().match(subPattern, relativePath)) {
						result.add(rootDirResource.createRelative(relativePath));
			}
		}
}
public java.util.jar.JarEntry nextElement() {
	return this.iterator.next();
}
public JarEntry next() {
	this.validator.run();
    if (!hasNext()) {
		 throw new NoSuchElementException();
	 }
	 int entryIndex = JarFileEntries.this.positions[this.index];
	 this.index++;
	 return getEntry(entryIndex, JarEntry.class, false, null);
}
```

next方法是通过移动下标来实现访问Jar中文件信息的，这是不是意味着jar里面的文件顺序依赖于创建时候的创建顺序呢？如果没人干扰的话我想是的，于是问题就转变为了，打包时候的顺序是怎么确定的。那现在debug一下maven? 犯不上，犯不上，我们手里有GPT，可以请教一下GPT，然后GPT回答我AbstractJarMojo的createArchive方法，然后我就去看这个方法的源码: 

![](https://a.a2k6.com/gerald/i/2023/10/21/10jfb.jpg)



![](https://a.a2k6.com/gerald/i/2023/10/21/10vzm.jpg)

这个调用的addDirectory的方法来自于AbstractArchiver，然后在这个方法里面传入的最终交给了PlexusIoFileResourceCollection:

```java
  public void addFileSet( @Nonnull final FileSet fileSet )
        throws ArchiverException
    {
        final File directory = fileSet.getDirectory();
        if ( directory == null )
        {
            throw new ArchiverException( "The file sets base directory is null." );
        }

        if ( !directory.isDirectory() )
        {
            throw new ArchiverException( directory.getAbsolutePath() + " isn't a directory." );
        }

        // The PlexusIoFileResourceCollection contains platform-specific File.separatorChar which
        // is an interesting cause of grief, see PLXCOMP-192
      	// 我们来看这个类
        final PlexusIoFileResourceCollection collection = new PlexusIoFileResourceCollection();
        collection.setFollowingSymLinks( false );

        collection.setIncludes( fileSet.getIncludes() );
        collection.setExcludes( fileSet.getExcludes() );
        collection.setBaseDir( directory );
        collection.setFileSelectors( fileSet.getFileSelectors() );
        collection.setIncludingEmptyDirectories( fileSet.isIncludingEmptyDirectories() );
        collection.setPrefix( fileSet.getPrefix() );
        collection.setCaseSensitive( fileSet.isCaseSensitive() );
        collection.setUsingDefaultExcludes( fileSet.isUsingDefaultExcludes() );
        collection.setStreamTransformer( fileSet.getStreamTransformer() );
        collection.setFileMappers( fileSet.getFileMappers() );
        collection.setFilenameComparator( getFilenameComparator() );

        if ( getOverrideDirectoryMode() > -1 || getOverrideFileMode() > -1 || getOverrideUid() > -1
            || getOverrideGid() > -1 || getOverrideUserName() != null || getOverrideGroupName() != null )
        {
            collection.setOverrideAttributes( getOverrideUid(), getOverrideUserName(), getOverrideGid(),
                                              getOverrideGroupName(), getOverrideFileMode(),
                                              getOverrideDirectoryMode() );
        }

        if ( getDefaultDirectoryMode() > -1 || getDefaultFileMode() > -1 )
        {
            collection.setDefaultAttributes( -1, null, -1, null, getDefaultFileMode(), getDefaultDirectoryMode() );
        }

        addResources( collection );
 }
```

然后接着看PlexusIoFileResourceCollection这个类的逻辑，看看添加进入的资源是怎么被处理的:

![](https://a.a2k6.com/gerald/i/2023/10/21/5fvu.jpg)

```java
public Iterator<PlexusIoResource> getResources()
        throws IOException
    {
        final DirectoryScanner ds = new DirectoryScanner();
        final File dir = getBaseDir();
        ds.setBasedir( dir );
        final String[] inc = getIncludes();
        if ( inc != null && inc.length > 0 )
        {
            ds.setIncludes( inc );
        }
        final String[] exc = getExcludes();
        if ( exc != null && exc.length > 0 )
        {
            ds.setExcludes( exc );
        }
        if ( isUsingDefaultExcludes() )
        {
            ds.addDefaultExcludes();
        }
        ds.setCaseSensitive( isCaseSensitive() );
        ds.setFollowSymlinks( isFollowingSymLinks() );
        ds.setFilenameComparator( filenameComparator );
        ds.scan(); // 这里执行扫描动作

        final List<PlexusIoResource> result = new ArrayList<>();
        if ( isIncludingEmptyDirectories() )
        {
            String[] dirs = ds.getIncludedDirectories();
            addResources( result, dirs );
        }

        String[] files = ds.getIncludedFiles();
        addResources( result, files );
        return result.iterator();
    }
```

DirectoryScanner的scan方法事实上调用了scandir方法,scandir 开头有一句这个调用:

```java
// dir 是File类型
String[] newfiles = dir.list();
```

```
public String[] list() {
    return normalizedList();
}
```

```java
private final String[] normalizedList() {
    @SuppressWarnings("removal")
    SecurityManager security = System.getSecurityManager();
    if (security != null) {
        security.checkRead(path);
    }
    if (isInvalid()) {
        return null;
    }
    String[] s = fs.list(this);
    if (s != null && getClass() != File.class) {
        String[] normalized = new String[s.length];
        for (int i = 0; i < s.length; i++) {
            normalized[i] = fs.normalize(s[i]);
        }
        s = normalized;
    }
    return s;
}
```

File的list实现在windows下面的实现是WinNTFileSystem，在Linux上的实现是UnixFileSystem ，list方法的实现如下:

```java
@Override
public String[] list(File f) {
        long comp = Blocker.begin();
        try {
            return list0(f);
        } finally {
            Blocker.end(comp);
        }
}
private native String[] list0(File f);
```

接着看native的实现:

![](https://a.a2k6.com/gerald/i/2023/10/21/130zi.jpg)

所以最后还是系统函数在不同文件系统上的返回不一样，所以导致在不同bean的创建顺序不一样，导致在windows上正常运行的项目

## 总结一下

如果你还能看到这里，在分析这个问题的时候想到Java的一次编译，处处运行，又想起对Java的调侃，一次编译，处处调试。分析这个问题的时候我一开始是知道Java获取目录下的文件，在不同的文件系统下面的表现是不同的，但一开始我认为是在扫描jar中的class文件的时候间接的调用了File的list或listFile这个方法，于是一直找，一直找，但是找着找着，发现自己的思路出现了问题，于是想到应该是这个顺序是在打包的时候就被确定了的，这个还是跟GPT对聊的时候，GPT给我了一定的提示，我才转变方向，这个提示我们要善于使用工具，因为你在问GPT的时候，相当于又整理了一遍上下文，整理的过程就会有新的线索。再有就是看源码也掌握了一些技巧，一方面调试的时候看每一步调用，有的时候一个方法下一步跳到你不知道的地方，这个时候可以接着IDEA的调用栈来观察，

![](https://a.a2k6.com/gerald/i/2023/10/22/1crr.jpg)

一方面也复习了一些语法:

```java
int entryIndex = JarFileEntries.this.positions[this.index];
```

JarFileEntries是内部类，所以访问外部成员变量就得通过这个引用去访问，刚看到这个还懵了一下，想来是因为工作中用到内部类不多，才导致对这个语法不熟悉。很久的时候，我以为看源码都有标准路线，事实上每个人看源码的过程形成的路线就是自己的标准路线，我以前一直以为自己的看源码的方式不对，所以看的磕磕绊绊的，但是事实上就是这样，无论你看了多少看源码的技巧，你总得自己一行行的debug，找出问题的答案，我看的时候总是感觉跟破案一样，在收集线索找到找到问题的答案，不断的回溯，这个过程也在不断的查资料。如果你仔细阅读了这篇文章，本篇其实在调试的过程有几个小技巧，就是看方法调用栈，然后调试的时候进行试探猜测，不要对猜测进行排斥，猜测也是一种探索的方式，然后猜他就是在这里调用的，然后验证，如果猜错了，然后再猜其他的，本身就是在揣测代码的运行，每一步可以不那么确定，不断梳理思绪，问题就会变得慢慢清晰起来。

## 参考资料

[1]  Spring 官方文档 https://docs.spring.io/spring-framework/docs/4.3.10.RELEASE/spring-framework-reference/htmlsingle/#beans-dependency-resolution

[2] JVM 中不正确的类加载顺序导致应用运行异常问题分析 https://ost.51cto.com/posts/15217 

[3] Spring 如何解决循环依赖   https://www.cnblogs.com/dw3306/p/15995834.html

[4] 循环依赖中的 一个set注入，一个构造器注入 https://zhuanlan.zhihu.com/p/484570587?

[5] getBean简单介绍 https://www.cnblogs.com/zfcq/p/15925542.html

[6] Spring-Boot原理及应用布署(中信银行) https://www.cnblogs.com/aspirant/p/16143539.html

