# Spring Bean的生命周期

> 突然觉得问答体很适合解释一些问题，本篇文章我们接着采用问答故事体的方式来讲上面这个问题，这是我之前面试的时候，被问道的。这次来彻底的探讨一下这个问题。

## Bean 生命周期概念的引入

某一天小陈觉得在这个公司有点厌倦了, 准备悄悄的跑路，于是一边上班，一边准备面试题，一边上班，这天他看到了这样一个面试题Spring Bean的生命周期，有点理解不动，他想到了他的领导，那个技术栈特别宽的领导，于是就找到了领导问: 领导我对一个Spring Bean生命周期这个概念有点不太理解，您有时间的话可以给我讲讲吗？

领导说: 等下，我正在处理一个问题，一会儿，我去找你去。过了十五分钟后，领导就来到了小陈的工位，说道：现在我们来过Spring Bean的生命周期这个概念吧！准备好了吗？小陈点了点头。

领导说道: 首先我们要弄清楚生命周期的含义，生命周期的英文是Life Cycle，简单的说就是Spring 给我们提供的一些扩展接口，如果bean实现了这些这些接口，应用在启动的过程中会回调这些接口的方法。在Spring中一个bean的一生通常就是先创建其对象，然后填充其属性，如果这个Bean实现了Spring 提供的扩展接口，那么在IOC容器加载的时候会依次回调这些方法。这些扩展接口的对应方法的回调顺序如下:

![Bean的生命周期](http://tvax4.sinaimg.cn/large/006e5UvNgy1h2w9sjuzl9j311s0diq9w.jpg)



画完图后领导问道: 这里面哪个单词你熟一些啊, 拿出来讲一下。

小陈心里想领导的这个PPT画的怎么这么熟练，还是接着回答道：我好像只见过ApplicationContext,我刚学Spring 框架的时候，用这个ApplicatonContext来获取加载进IOC容器的对象，像下面这样:

```java
public static void main(String[] args) {
        ApplicationContext applicationContext = new AnnotationConfigApplicationContext(SpringBeanConfig.class);
        applicationContext.getBean(StudentService.class);
}
```

领导点了点头，接着说道:  大多数入门都是从这样的例子开始的，说道这里看着上面的代码你想到了设计模式的哪个模式？

小陈回答道: 这让我想起了简单工厂模式，我给一个标识，由ApplicationContext给我加载对应的对象。

领导点了点头:  ApplicationContext 继承自BeanFactory, 上面的确是使用了简单工厂模式。上面接口中带有Aware的, 都继承自Aware接口。子类将对于父类来说提供的功能更为丰富。setBeanName会从换入实现BeanNameAware接口的Bean Id和Bean名称，也就是说容器在初始化Bean和填充完Bean的属性之后，会回依次回调BeanNameAware、ApplicationContextAware、BeanPostProcessor的postProcessBeforeInitialization、InitializingBean的afterPropertiesSet方法，Bean的init-method方法、BeanPostProcessor的postProcessAfterInitialization的方法。这里要注意BeanNameAware、ApplicationContextAware只会回调依次，每个Bean完成初始化之后都会回调BeanPostProcessor的两个方法。到这里其实就结束了。下面我们写个代码来演示一下上面的顺序:

```java
public class BeanLifeCycleDemo implements BeanNameAware, BeanFactoryAware, ApplicationContextAware, InitializingBean,BeanPostProcessor{

    private MuttonSoupService muttonSoupService;

    public BeanLifeCycleDemo( ) {
        System.out.println("1.构造函数初始化bean");
    }

    @Autowired
    public void setMuttonSoupService(MuttonSoupService muttonSoupService) {
        System.out.println("2.先填充属性-");
        this.muttonSoupService = muttonSoupService;
    }

    public void sayHelloWorld(){
        System.out.println("hello world");
    }

    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        System.out.println("----4-- 接着是BeanFactoryAware");
    }

    @Override
    public void setBeanName(String name) {
        System.out.println("3---- name+" + name);
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("----7 afterPropertiesSet");
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        System.out.println("------5 applicationContextAware");
    }


    public void initMethod(){
        System.out.println("6 init-method");
    }

    /**
     * @param bean
     * @param beanName
     * @return
     * @throws BeansException
     */
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("---- postProcessBeforeInitialization"+beanName);
        return bean;
    }

    /**
     * @param bean
     * @param beanName
     * @return
     * @throws BeansException
     */
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("---- postProcessAfterInitialization");
        return bean;
    }

}
@Configuration
public class SpringBeanConfig {
    @Bean(initMethod = "initMethod")
    public BeanLifeCycleDemo beanLifeCycleDemo(){
        return new BeanLifeCycleDemo();
    }
}
@SpringBootTest
class SsmApplicationTests {

    @Autowired
    private BeanLifeCycleDemo beanLifeStyleDemo;
	
    // 直接运行测试即可
    @Test
    public void test(){

    }
}    
```

输出结果:

![输出顺序](http://tvax4.sinaimg.cn/large/006e5UvNgy1h2wd0wj5swj30mz08x43s.jpg)

小陈看到了这个结果, 心里有点开心，领导翻车了，于是说道：领导你这个顺序跟上面图的顺序不一样欸。我还有一个问题，这个Bean是永生的吗？

领导淡淡的说道：我是故意演示给你看的, 注意这个BeanPostProcessor，这个接口对应的Bean需要预先加载进入才能实现上面的效果。Spring Bean的加载顺序如下图(下面这张图是网上找的)所示:

![5922080414274eeba353c0b5d4bede5f](http://tvax1.sinaimg.cn/large/006e5UvNly1h2wdagx2ywj30eq08wad8.jpg)

所以我们在写一个BeanPostProcessor的bean就能实现上面的顺序了。至于Bean生命的结束, 其实我是故意考验你的。其实也就是一个接口和一个属性而已. 接口是DisposableBean，属性在@Bean中有个destroyMethod。 这两个属性也有替代注解，init-method=@PostConstruct, @PreDestroy=destroyMethod。结合上面的图我们可以大致了解了Spring IOC容器加载Bean的流程，所谓的生命周期在某种程度上可以看成在加载Bean的过程的回调。

小陈说道: 谢谢领导，我感觉我对这个概念理解的更通透了，相对于生命周期，我更喜欢你回调的说法呢。

## 参考资料

- Spring – Bean Life Cycle https://howtodoinjava.com/spring-core/spring-bean-life-cycle/
- Spring用到哪些设计模式  https://199604.com/2188
- BeanPostProcessor —— 连接Spring IOC和AOP的桥梁  https://zhuanlan.zhihu.com/p/38208324
