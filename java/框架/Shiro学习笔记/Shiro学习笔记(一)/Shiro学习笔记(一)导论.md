# Shiro学习笔记(一)  基本概念与使用

[TOC]

## Shiro能帮助我们干什么？

> **Apache Shiro™** is a powerful and easy-to-use Java security framework that performs authentication, authorization, cryptography, and session management. With Shiro’s easy-to-understand API, you can quickly and easily secure any application – from the smallest mobile applications to the largest web and enterprise applications. -《Apache Shiro 官网》
>
> Apache Shiro 是Java领域内的一款简单易用而又强大的一款安全框架，主要用于登录验证、授权、加密、会话管理。Shiro拥有简单应用的API，您可以快送的使用它来保护您的应用，不管是小到手机应用还是到大的Web企业级应用。

从上面的一段话我们可以提取到以下信息:

- Shiro简单易用而又强大
- 主要应用于登录验证、授权、加密、会话管理。

第一点需要在中使用中慢慢体会，第二点是我们主要关注的，这里我们一点一点的讲。

### 登录验证 authentication 与 会话管理

这里我们回忆一下HTTP协议和Servlet, 早期的HTTP协议是无状态的, 这个无状态我们可以这么理解，你一分钟前访问和现在访问一个网站，服务端并不认识你是谁，但是对于Web服务端开发者来说, 这便无法实现访问控制，某些信息只能登录用户才能看，各自看各自的。这着实的有点限制住了Web的发展，为了让HTTP协议有状态，RFC-6235提案被获得批准，这个提案引入了Cookie, 是服务器发送到用户浏览器并保存在本地的一小块数据，它会再浏览器下次向同一服务器再发起请求时被携带并发送到服务器上，服务端就可以实现“识别”用户了。

在原生的Servlet场景中如下:

![登录流程图](http://tvax3.sinaimg.cn/large/006e5UvNly1h344akrlxqj310g0fhaek.jpg)

代码示例:

```java
/**
 * 拦截所有的请求
 */
@WebFilter(urlPatterns = "/*")
public class LoginCheckFilter implements Filter {

    /**
     * 不重写init方法,过滤器无法起作用
     * @param filterConfig
     * @throws ServletException
     */
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        System.out.println("----login check filter init-----");
    }
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        HttpServletRequest servletRequest = (HttpServletRequest) request;
        HttpSession session = servletRequest.getSession();
        // 这里采取了暂时写死,目前只放行登录请求的URL
        String requestUri = servletRequest.getRequestURI();
        if ("/login".equals(requestUri)){
            // 放行,此请求进入下一个过滤器
            chain.doFilter(request,response);
        }else {
            Object attribute = session.getAttribute("currentUser");
            if (Objects.nonNull(attribute)){
                chain.doFilter(request,response);
            }else {
                request.getRequestDispatcher("/login.jsp").forward(request,response);
            }
        }
    }
}
@WebFilter(urlPatterns = "/login")
public class LoginFilter implements Filter {

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        System.out.println("------login filter init-----");
    }

    /**
     * 由登录过滤器来执行登录验证操作
     * 要求用户名和密码不为空的情况下才进行下一步操纵
     * 此处省略判空操作
     * 只做模拟登录
     */
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        HttpServletRequest servletRequest = (HttpServletRequest) request;
        String userName = (String) servletRequest.getAttribute("userName");
        String password = (String)servletRequest.getAttribute("password");
        // 这里假装去数据库去查账号和密码
        HttpSession session = servletRequest.getSession();
        // 生成session,
        session.setAttribute("currentUser",userName + password);
        // 放到下一个过滤器,如果这是最后一个过滤器,那么这个请求会被放行到对应的Servlet中
        chain.doFilter(request,response);
    }
}
@WebServlet(value = "/hello")
public class HttpServletDemo extends HttpServlet {
   
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        System.out.println("hello world");
        super.doGet(req, resp);
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        super.doPost(req, resp);
    }
}
```

前面提到, 为了实现让HTTP协议有“状态”, 浏览器引入了Cookie,  存放服务端发送给客户端的数据, 为了使服务端区分不同的用户,  服务端程序引入了Session这个概念，浏览器首次请求服务端，服务端会在HTTP协议中告诉客户端，需要再Cooke里面记录一个SessionID，以后每次请求把这个SessionId发送到服务器，这样服务器就能区分不同的客户端了。

![JSessionId](http://tvax3.sinaimg.cn/large/006e5UvNgy1h3451bkqnij30s10anq8p.jpg)

有了这样的对应关系，我们就可以在Session中保存当前用户的信息。其实我们这里只做了登录，还没有做登出，登出应该将对应的Session中清除掉。这套逻辑Shiro帮我们做好了，那自然是极好的，但仅仅是如此的话，还不足以让我们使用Shiro, 我们接着翻官方文档。

### 一个小插曲

因为很久没写Servlet了，这次去下载Tomcat，下了一个10.0的, 但是发现10.0版本跟JDK 8不太兼容，折腾了许久, 请求都没办法到达Servlet，要不就是报Servlet初始化失败，最高只好退回8.5版本，但是还是发现请求到达不了我写的Servlet中, 我是基于注解的形式配置的映射，但是web.xml的有个属性我忘记了，那就是metadata-complete, 此属性为true, 不会扫描基于注解的Servlet，此注解为false, 才会启用此Servlet。

![meta-complete = false](http://tva3.sinaimg.cn/large/006e5UvNly1h346ju71xfj30l508gtbv.jpg)

### Shiro的特性

![ShiroFeatures](http://tvax4.sinaimg.cn/large/006e5UvNly1h34664sf77j30dn071ta8.jpg)

Authentication和Session Management: Shiro帮我们做好了, 那自然是极好的。

Cryptography: 加密, 其实Java标准库也提供了实现，那Shiro能提供一套更简单易用的API更好。

Authorization：授权, 这个值得我们注意一下，写到这里我想起我第一个构建的Web系统，当时只考虑了普通的用户，没有考虑资源控制的问题，当时已经快到预定的时间了，说来惭愧, 只好做了一个非常粗糙的权限控制，为用户添加了一个权限标识字段，1代表什么角色，2代表什么角色能看哪些页面，也就是说资源是固定的，相当的僵硬。后面才了解到在这方面有一套比较成熟而自然的RBAC(Role-Based Access Controle 基于角色的访问控制)模型，即角色-用户-权限(资源)，也就是说一个用户拥有若干角色，每一个角色拥有若干权限和资源，这样我们就实现了权限和用户的解耦合。

基本概念我们大致论述完之后，我们来进一步深入的看这些特性:

- Authentication 登录

   >- **Subject Based** – Almost everything you do in Shiro is based on the currently executing user, called a Subject.
   >  And you can easily retrieve the Subject anywhere in your code. This makes it easier for you to understand and work with Shiro in your applications.
   >
   >  基于主体，在Shiro中做的任何事情都基于当前正在活动的用户，在Shiro中称之为主体，你可以在代码的任何地方取到当前的主体。
   >
   >  这将让你在使用和理解shiro变的轻松起来。
   >
   >- **Single Method call** – The authentication process is a single method call.
   >  Needing only one method call keeps the API simple and your application code clean, saving you time and effort.
   >
   >    简单的方法调用，非常简单，节省时间和精力。
   >
   >- **Rich Exception Hierarchy** – Shiro offers a rich exception hierarchy to offered detailed explanations for why a login failed.
   >  The hierarchy can help you more easily diagnose code bugs or customer services issues related to authentication. In addition, the richness can help you create more complex authentication functionality if needed.
   >
   >  丰富的异常体系，Shiro提供了完备的异常体系来解释登录为什么失败。这个异常体系可以帮助诊断定制服务的相关bug和issue。除此之外，还可以帮助创建功能更加丰富的应用。
   >
   >- **‘Remember Me’ built in** – Standard in the Shiro API is the ability to remember your users if they return to your application.
   >  You can offer a better user experience to them with minimal development effort.
   >
   >  记住我，标准的Shiro API提供了记录密码的功能，只需少量的配置，就能给用户提供更佳的体验。
   >
   >- **Pluggable data sources** – Shiro uses pluggable data access objects (DAOs), called Realms, to connect to security data sources like LDAP and Active Directory.
   >  To help you avoid building and maintaining integrations yourself, Shiro provides out-of-the-box realms for popular data sources like LDAP, Active Directory, and JDBC. If needed, you can also create your own realms to support specific functionality not included in the basic realms.
   >
   >   可插拔的数据源，Shiro提供了一个可插拔的数据权限对象，在shiro中我们称之为Realms，我们用这个去安全的连接像LDAP、Active Directory的数据源。
   >
   >为了避免开发者做重复的工作，Shiro 提供了开箱即用的连接指定数据源的Realm是，像LDAP、Active Directory 、JDBC。 如果你需要你也可以创建自定义的Realms。
   >
   >- **Login with one or more realms** – Using Shiro, you can easily authenticate a user against one or more realms and return one unified view of their identity.
   >  In addition, you can customize the authentication process with Shiro’s notion of an authentication strategy. The strategies can be setup in configuration files so changes don’t require source code modifications – reducing complexity and maintenance effort.
   >
   >  支持一个和多个Realm登录，使用Shiro 你可以轻松的完成用户使用多个Realm登录，并且方式统一。除此之外，你也可以可以使用Shiro的身份验证策略自定义登录过程，验证策略支持写在配置文件中，所以当验证策略改变的时候，不需要更改代码。

看来上面的特性概述，你会发现原来登录需要考虑这么多，原先我们的视角可能只在数据库的数据源，事实上对于WEB系统来说还可以引入其他数据源，但是你不用担心，这么多需要考虑的东西，原先你自己来写登录，可能还会考虑遗漏的地方，但是Shiro都帮你写好。我们还有什么理由不学它呢。这里还是需要重点讲一下Shiro的Realms概念，我们回忆一下JDBC，可以让我们Java程序员写一套API就能做到跨数据库，那多个数据源呢，我们能否也抽象出一个接口，做到登录的时候跨数据源呢？ 其实Realms就是这个思路，也抽象出了一个接口，对接不同的数据源来实现登录认证:

![Realm继承图](http://tvax1.sinaimg.cn/large/006e5UvNgy1h348nc9bj8j30zi0ajdj5.jpg)

本质上是一个特定的安全DAO(Data Access Object), 封装与数据源连接的细节，得到Shiro所需要的相关数据。在配置Shiro的时候，我们必须指定至少Realm来实现认证(authentication) 和授权(authorization).



我们只重点介绍一个特性来体会Shiro的强大，其他特性我们只简单介绍转眼的点:

- Cryptography：Shiro指出Java的密码体系比较难用(The Java Cryptography Extension (JCE) can be complicated and difficult to use unless you’re a cryptography expert, Java的扩展非常难用，当然你要是个密码专家就当我没说)，Shiro设计的密码学的API更简单易用。

  PS: 其实还指出了Java密码学扩展的一些问题，算是对Java密码学相关库的吐槽了。有兴致可以去看看，我们这里不做过多介绍。

- SessionManagement

  > 可以被用于SSO，获取用户登录和退出都相当方便。SSO是单点登录，方便的和各种应用系统做集成。

- Authorization

  > 下面是授权的几个经典问题:
  >
  > 这个用户是否可以编辑这个账号
  >
  > 这个用户是否有权限看这页面
  >
  > 这个用户是否有权限使用这个按钮

  Shiro回答了以上问题，并且非常灵活、简单、容易使用。

 Shiro帮我们做了这么多，而且简单，我们可以省了很多工作，这就是学习Shiro的理由。

#### Shiro核心概念概述

在Shiro中的架构中有三个主要概念: 

- subject(当前主体, 可以理解为当前登录用户,)

  在Shiro中可以使用下面代码来获取当前登录用户:

```java
  Subject currentUser = SecurityUtils.getSubject();
```

- SecurityManager

> 为所有用户提供安全保护，内嵌了很多安全组件，那么如何设置它呢？ 也取决于不同的环境, Web 程序中通常是在Web.xml中指定Shiro的Servlet过滤器，这就完成了一个SecurityManager 实例，其他类型的应用程序我们也有其他选项。

- Realms

> Realm 事实上是Shiro和你应用安全数据的桥梁或连接器。

![Realms](http://tva4.sinaimg.cn/large/006e5UvNgy1h34agslrz2j312t09fdgp.jpg)

三者之间的关系:

![三个验证状态](http://tvax2.sinaimg.cn/large/006e5UvNgy1h34b69sunnj311t0dc76o.jpg)

## 用起来

首先引入maven依赖：

```xml
  <dependency>
      <groupId>org.apache.shiro</groupId>
      <artifactId>shiro-core</artifactId>
      <version>1.9.0</version>
    </dependency>
    <dependency>
      <groupId>org.slf4j</groupId>
      <artifactId>slf4j-simple</artifactId>
      <version>1.7.21</version>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.slf4j</groupId>
      <artifactId>jcl-over-slf4j</artifactId>
      <version>1.7.21</version>
      <scope>test</scope>
    </dependency>
```

#### 当然是从Hello World开始

```java

    public static void testAuthentication(){
        // 设置SecurityManager 
        // SecurityManager 负责将用户提交过来的username、password和realm中的进行对比
        // 判断是否可以登录
        DefaultSecurityManager defaultSecurityManager = new DefaultSecurityManager();
    
        // 这是一个简单的Realm,直接在代码里面存储账号和密码
        SimpleAccountRealm simpleAccountRealm = new SimpleAccountRealm();
        simpleAccountRealm.addAccount("hello world","hello world");
        // 将realm 纳入到 DefaultManager的管辖之下
        defaultSecurityManager.setRealm(simpleAccountRealm);
        // 通过此方法设置SecurityManager
        SecurityUtils.setSecurityManager(defaultSecurityManager);
        
        // 用户登录和密码凭证
        UsernamePasswordToken token = new UsernamePasswordToken("hello world", "hello world");
        
        // 获取subject
        Subject subject = SecurityUtils.getSubject();
        
        // 由subject将token提交给SecurityManager
        subject.login(token);
        // 登录成功会返回true
        System.out.println("login status:"+subject.isAuthenticated());
        // 退出
        subject.logout();
        // 退出之后是false
        System.out.println("login status:"+subject.isAuthenticated());
    }
```

#### 加密示例

>  Cryptography is the process of hiding or obfuscating data so prying eyes can’t understand it. Shiro’s goal in cryptography is to simplify and make usable the JDK’s cryptography support.
>
> 密码学是隐藏或混淆数据的过程，防止数据被窃取。Shiro在密码学方面的目标是简化JDK标准密码学库的使用

接下来让我们下用Shiro的密码学相关的API有多简单易用.

- MD5 JDK标准库的实现

```java
 private static void testMD5JDK() {
        try {
            String code = "hello world";
            // MD5 是 MessageDigest的第五个版本
            MessageDigest md = MessageDigest.getInstance("MD5");
            byte[] targetBytes = md.digest(code.getBytes());
            //输出的是MD5的十六进制形式
            System.out.println(targetBytes);
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        }
    }
```

			- Shiro的实现:

```java
private static void testMD5Shiro() {
    String hex = new Md5Hash("hello world").toHex();
    System.out.println(hex.getBytes());
}
```

这么一看Shiro的实现确实简单一些，更直观。

#### 授权示例

在Shiro中将授权分成以下两种:

- Permission Defined

> Permissions are the most atomic level of a security policy and they are statements of functionality. Permissions represent what can be done in your application. A well formed permission describes a resource types and what actions are possible when you interact with those resources. Can you *open* a *door*? Can you *read* a *file*? Can you *delete* a *customer record*? Can you *push* a *button*?
>
> Common actions for data-related resources are create, read, update, and delete, commonly referred to as CRUD.
>
> It is important to understand that permissions do not have knowledge of *who* can perform the actions– they are just statements of *what* actions can be performed.
>
> 准许是安全策略的原子级别，表现为功能的声明。准许的意思是你可以在这个系统中做什么。形式良好的准许描述了资源类型以及可以对这些资源进行的操作。比如: 你是否可以打开这扇门？ 你是否可以读一个文件? 你是否可以删除一个客户记录？ 你是否能点击按钮？
>
> 一般来说对资源的操作有新增、读取、更新、删除，这些操作通常被称作为CRUD。
>
> 一定要理解，准许是不知道是谁可以操作这些资源的，它们只是说明这些资源可以执行哪些操作。

​	权限的粒度:

1. Resource Level - This is the broadest and easiest to build. A user can edit customer records or open doors. The resource is specified but not a specific instance of that resource.

> 资源级别: 这是最广泛的而且是最容易构建的。用户可以编辑客户记录或者打开一扇门。资源指定了，但是没有指定到具体的人祸角色上。

2. Instance Level - The permission specifies the instance of a resource. A user can edit the customer record for IBM or open the kitchen door.

> 实例级别:  某个准许指定了具体的人或者角色,  某个用户可以编辑IBM的用户记录或打开厨房的门。

3. Attribute Level - The permission now specifies an attribute of an instance or resource. A user can edit the address on the IBM customer record.

> 属性级别: 某人被允许编辑资源的某个属性，某个用户可以编辑IVM用户记录的地址。

- Role Defined

> In the context of Authorization, Roles are effectively a collection of permissions used to simplify the management of permissions and users. So users can be assigned roles instead of being assigned permissions directly, which can get complicated with larger user bases and more complex applications. So, for example, a bank application might have an *administrator* role or a *bank teller* role.
>
> 在授权的上下门，角色实际上是权限的集合，简化权限和用户的管理，因此用户可以被分配角色，而不是直接被分配权限，因为直接分配权限这对于更大的用户群和更复杂的应用程序来说会比较复杂。

Shiro支持如下两种角色:

- Implicit Roles 隐式角色

> 大多数人的角色在我们眼中属于是隐式角色，隐含了一组权限，通常的说，如果你具有管理员的角色，你可以查看患者数据。如你具有“银行出纳员”的角色，那么你可以创建账号。

- Explicit Roles 显式角色

> 显式角色就是系统明显分配的权限，如果你可以查看患者数据，那是因为你被分配到了“管理员角色”的“查看患者数据”权限。

##### 小小总结一下

上面是我翻译的Shiro对角色和权限的论述，权限的话可以理解为对某个资源的CRUD, 粒度级别有整个资源，比如一行记录的操纵权限，这行记录只有某个人才能操纵，某个人只能操纵这行记录的一部分。而角色则是权限的集合, 在我看来是实现权限和用户的解耦，比如我想对一批用户授权，我可以选一个角色，进行批量授权。那么问题又来了，我该怎么做权限控制呢，或者在Shiro中判断某个是否有这个角色呢？

- 我们可以从Subject这个类的方法中判断当前用户是否具备某个角色或者具备某个权限

  ```java
  subject.hasRole("admin") // 当前用户是否有admin这个角色
  subject.isPermitted("user:create") // 判断当前用户是否允许添加用户
  ```

- JDK 的注解, 在方法中加注解

```java
// 判断当前角色是否有admin 这个角色
@RequiresRoles("admin")
private static void requireSpecialRole() {

}
 // 判断当前用户是否允许添加用户
@RequiresPermissions("user:create")
private static void requireSpecialRole() {
 }
```

- JSP 标签(前后端分离时代了, 这个不做介绍)

Shiro 定义了一组权限描述语法来描述权限，格式为: 资源:(create || update || delete ||query)。

示例：

```java
user:create,update // 表示具备创建和更新用户的权限
user:create,update:test110 // ID为test110有 创建和更新用户的权限   
```

### 自定义Realm

我们前面一直再说Realm是Shiro和用户数据之间的桥梁, 我们大致看下这个登录过程，重点来体会一下桥梁:

1. subject.login(token). 目前Subject只有一个实现类，那就是DelegatingSubject。

![DelegatingSubject](http://tva2.sinaimg.cn/large/006e5UvNly1h34f1q1n11j30ol0er47l.jpg)

   我们上面用的是DefaultSecurityManager, 所以我们需要进入该类的方法来看是怎么执行登录的,  DefaultSecurityManager.login调用authenticate(token)来获取用户信息，authenticate()方法来自DefaultSecurityManager的父类AuthenticatingSecurityManager, 该类拥有一个Authenticator成员变量，由该成员变量调用authenticate，获取用户信息

![AuthenticatingSecurityManager](http://tvax1.sinaimg.cn/large/006e5UvNly1h34f9rh7k5j30vk0knthp.jpg)

这在某种成都上很像是模板方法模式, 父类定义骨架或通用方法，子类做调用。Authenticator是一个接口有许多实现类，但是调用的authenticate就只有两个实现：

![AbstractAuthenticator](http://tvax4.sinaimg.cn/large/006e5UvNly1h34fe41mq2j314y09y464.jpg)

![doAuthenticate](http://tva4.sinaimg.cn/large/006e5UvNly1h34ffruyz7j30wt0fvqcw.jpg)

![获取初始化进来Realms](http://tvax4.sinaimg.cn/large/006e5UvNly1h34fizmpoxj30tn068adw.jpg)

![最终调到了对应的Realms](http://tvax4.sinaimg.cn/large/006e5UvNly1h34fkq6tzej30pa093jyb.jpg)

 Realm这个接口太顶层了，我们要做自己的Realm的话还是找一个抽象类，我们上面的SimpleAccountRealm就是继承自AuthorizingRealm，重写了doGetAuthenticationInfo用于登录验证，doGetAuthorizationInfo用于做权限验证。我们自己写的如下：

```java
public class MyRealm extends AuthorizingRealm {

    /**
     * 设置Realm的名称
     */
    public MyRealm() {
        super.setName("myRealm");
    }

    /**
     * 我们这个自定义reamls就能实现权限控制了
     * @param principals
     * @return
     */
    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principals) {
        // 获取登录凭证
        String userName = (String) principals.getPrimaryPrincipal();
        // 假装这roles 是从数据库中查的
        Set<String> roles  = new HashSet<>();
        // 假装是从数据库查的
        Set<String> permissions  = new HashSet<>();
        SimpleAuthorizationInfo simpleAuthorizationInfo = new SimpleAuthorizationInfo();
        simpleAuthorizationInfo.setStringPermissions(permissions);
        simpleAuthorizationInfo.setRoles(roles);
        return simpleAuthorizationInfo;
    }

    /**
     * 随便写个空实现
     * @param userName
     * @return
     */
    private String getPassWordByUserName(String userName) {
        return "";
    }

    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token) throws AuthenticationException {
        // 获取账号
        String userName = (String) token.getPrincipal();
        String password = null;
        if (userName != null && userName != ""){
            password = getPassWordByUserName(userName);
        }else {
            return null;
        }
        if (Objects.isNull(password)){
            return null;
        }
        // 假装去查了数据库
        SimpleAuthenticationInfo simpleAuthenticationInfo = new SimpleAuthenticationInfo(userName,password,"myRealm");
        return simpleAuthenticationInfo;
    }
}
```

## 写在最后

我记得刚开始学Shiro的时候是去B站搜对应的视频, 但是视频大多都是从Shiro的基本组件开始讲，我其实只是想知道Shiro能帮我干啥，去看教程也是先从Shiro是一个安全框架讲起，然后讲架构，看了半天视频我只收获了一堆名词，我想看的东西都没看到，实在消耗我的耐心。所以本篇文章是以问题为导向，即Shiro最终帮我们做了什么，同时也调整了一些文章介绍风格，省略掉一些比较大的词，这篇文章献给当初在网上找Shiro，找了半天也没找到合心意的教程的自己。

## 参考资料

- Shiro 官方文档 https://shiro.apache.org/
- RBAC用户、角色、权限、组设计方案  https://zhuanlan.zhihu.com/p/63769951
- Shiro安全框架【快速入门】就这一篇！  https://zhuanlan.zhihu.com/p/54176956
- 理解Apache Shiro中的权限--官网  https://www.jianshu.com/p/1e6e453b9332
