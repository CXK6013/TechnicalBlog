# 如何向Spring IOC 容器 动态注册bean

> 本文的大纲如下

![文章大纲](http://tvax3.sinaimg.cn/large/006e5UvNly1h40gp5dxv7j30nn07xmx9.jpg)

## 从一个需求谈起

这周遇到了这样一个需求，从第三方的数据库中获取值，只是一个简单的分页查询，处理这种问题，我一般都是在配置文件中配置数据库的地址等相关信息，然后在Spring Configuration 注册数据量连接池的bean，然后再将数据库连接池给JdbcTemplate, 但是这种的缺陷是，假设填错了数据库地址和密码，或者换了数据库的地址和密码，在配置文件里面重启之后，都需要重启应用。

我想能不能动态的向Spring IOC容器中注册和加载bean呢，项目在界面上填写数据库的地址、用户名、密码，存储之后，将JdbcTemplate和另一个数据库连接池加载到IOC容器中。答案是可以的，我经过一番搜索写出了如下代码:

```java
@Component
public class BeanDynamicRegister {
    private final ConfigurableApplicationContext configurableApplicationContext;

    public BeanDynamicRegister(ConfigurableApplicationContext configurableApplicationContext) {
        this.configurableApplicationContext = configurableApplicationContext;
    }

    /**
     * 此方法提供出去,供其他bean动态的向IOC容器中注册bean。
     * 代表使用构造器给bean赋值
     *
     * @param beanName bean名
     * @param clazz    bean类
     * @param args     用于向bean的构造函数中添加值 如果loadType是set,则要求传递map.map的key为属性名,value为属性值
     * @param <T>      返回一个泛型
     * @param loadType
     * @return
     */
    public <T> T registerBeanByLoadType(String beanName, Class<T> clazz, LoadType loadType, Object... args) {
        BeanDefinitionBuilder beanDefinitionBuilder = BeanDefinitionBuilder.genericBeanDefinition(clazz);
        if (args.length > 0) {
            // 将参数加入到构造函数中
            switch (loadType) {
                case CONSTRUCTOR:
                    for (Object arg : args) {
                        beanDefinitionBuilder.addConstructorArgValue(arg);
                    }
                    break;
                case SETTER:
                    Map<String, Object> propertyMap = (Map<String, Object>) args[0];
                    for (Map.Entry<String, Object> stringObjectEntry : propertyMap.entrySet()) {
                        beanDefinitionBuilder.addPropertyValue(stringObjectEntry.getKey(), stringObjectEntry.getValue());
                    }
                    break;
                default:
                    break;
            }

        }
        BeanDefinition beanDefinition = beanDefinitionBuilder.getRawBeanDefinition();
        BeanDefinitionRegistry beanDefinitionRegistry = (BeanDefinitionRegistry) configurableApplicationContext.getBeanFactory();
        beanDefinitionRegistry.registerBeanDefinition(beanName, beanDefinition);
        return configurableApplicationContext.getBean(beanName, clazz);
    }

    public <T> T getBeanByName(String beanName,Class<T> requiredType){
        return  configurableApplicationContext.getBean(beanName,requiredType);
    }


    /**
     * 如果用户换了地址和密码,向IOC容器中移除bean。 重新注册
     *
     * @param beanName
     */
    public void removeBean(String beanName) {
        BeanDefinitionRegistry beanDefinitionRegistry = (BeanDefinitionRegistry) configurableApplicationContext.getBeanFactory();
        beanDefinitionRegistry.removeBeanDefinition(beanName);
    }
}
@SpringBootTest
class SsmApplicationTests {

    @Autowired
    private LoadBeanService loadBeanService;


    private NamedParameterJdbcTemplate jdbcTemplate;

    @Autowired
    private BeanDynamicRegister beanDynamicRegister;

    @Test
    public void test() {
        loadBeanService.loadDataSourceTest("root", "root");
        jdbcTemplate = beanDynamicRegister.getBeanByName("jdbcTemplateOne", NamedParameterJdbcTemplate.class);
        System.out.println("--------" + jdbcTemplate);
    }
}

```

结果： ![成功赋值](http://tva2.sinaimg.cn/large/006e5UvNly1h40g8u8s9xj30tz07rgme.jpg)

我们就到这里了吗？ 我们观察一下上面将一个bean加载到Spring IOC容器里经过了几步:

- BeanDefineBuilder 构造BeanDefinition
- 然后BeanDefinitionRegistry将其注册到IOC容器中。(这一步事实上只完成了注册，还未完成Bean的实例化，属性填充)

联系我们前面的文章《Spring Bean 的生命周期》，我们将Spring 的生命周期理解为“Spring 给我们提供的一些扩展接口，如果bean实现了这些这些接口，应用在启动的过程中会回调这些接口的方法。” ,  这个理解并不完善，缺少了解析BeanDefinition这个阶段。

## Spring Bean的生命周期再完善

### BeanDefinition

那BeanDefinition是什么？ BeanDefinition是一个接口，我们进Spring 官网(https://docs.spring.io/spring-framework/docs/5.2.x/spring-framework-reference/core.html#beans-child-bean-definitions)大致看一下:

> A bean definition can contain a lot of configuration information, including constructor arguments, property values, and container-specific information, such as the initialization method, a static factory method name, and so on. A child bean definition inherits configuration data from a parent definition. The child definition can override some values or add others as needed. Using parent and child bean definitions can save a lot of typing. Effectively, this is a form of templating.
> bean 的定义信息可以包含许多配置信息，包括构造函数参数，属性值和特定于容器的信息，例如初始化方法，静态工厂方法名称等。子 bean 定义可以从父 bean 定义继承配置数据。子 bean 的定义信息可以覆盖某些值，或者可以根据需要添加其他值。使用父 bean 和子 bean 的定义可以节省很多输入（实际上，这是一种模板的设计形式）。

这段说的可能有点抽象, 你点BeanDefinition进去，你就会发现有很多熟悉的面孔：

![BeanDefinition的作用域](http://tvax2.sinaimg.cn/large/006e5UvNly1h40gv9zam7j30na0de3zz.jpg)

Bean的作用域: 单例，还是多例。

![lazy-init](http://tvax2.sinaimg.cn/large/006e5UvNly1h40gw3t2wzj30m30dtwfs.jpg)

lazyInit是否是懒加载。

这些都是描述Spring Bean的信息，我们可以类比到Java中的类，每个类都会有class属性，我们在配置类或者xml中的配置Bean的元信息，也被映射到这里。供IOC容器将Bean加入时使用。所以我们可以为对Spring Bean的生命周期的理解打一个补丁:

- 从xml或配置类中解析BeanDefintion
- BeanDefinition 注册，此时还未完成Bean的实例化。

我们可以打断点来验证一下:

![BeanDefaultRegistry的实现类](http://tvax3.sinaimg.cn/large/006e5UvNly1h40i0cmkw3j31fo0p87cm.jpg)

![BeanDefinitionMap中](http://tvax2.sinaimg.cn/large/006e5UvNly1h40i2ka4v5j30yz0odjvz.jpg)

- Bean 实例化 
- Bean的属性赋值+依赖注入
- Bean的初始化阶段的方法回调
- Bean的销毁。

![Bean的生命周期](http://tvax2.sinaimg.cn/large/006e5UvNly1h40ixx5t8uj30va0jb0va.jpg)

## Bean 加入IOC容器的几种方式

我们这里再来总结一下一个Bean注入Spring IOC容器的几种形式:

- 启动时加入

  - 配置类: @Configuration+@Bean
  - 配置文件: xml
  - 注解形式
    - @Component
    - @Service
    - @Controller
    - @Repository
    - @import
    - @Qualifier 
    - @Resource
    - @Inject

- 运行时加入

  - ImportBeanDefinitionRegistrar 

  - 手动构造BeanDefinition注入(我们上面就是自己手动构造BeanDefinition注入)
  - 借助BeanDefinitionRegistryPostProcessor注入

  这三种最终都是通过BeanDefinitionRegistry来注入的，ImportBeanDefinitionRegistrar是一个接口，留给我们实现的方法如下:

  ```java
  default void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
  }
  ```

​	BeanDefinitionRegistryPostProcessor也是一个接口，留给我们实现的方法如下：

```java
void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;
```

## 总结一下

有种越学越不会的感觉。

## 参考资料

- 180804-Spring之动态注册bean  https://blog.hhui.top/hexblog/2018/08/04/180804-Spring%E4%B9%8B%E5%8A%A8%E6%80%81%E6%B3%A8%E5%86%8Cbean/
- 从spring容器中动态添加或移除bean  https://blog.csdn.net/qq_20161461/article/details/125034315
- 《从 0 开始深入学习 Spring》  https://juejin.cn/book/6857911863016390663