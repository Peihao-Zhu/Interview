[TOC]

## Spring Boot 相关面试题

#### 1.什么是Spring Boot

重要的是要理解 Spring Boot 并不是一个框架，它是一种创建独立应用程序的更简单方法，只需要很少或没有配置（相比于 Spring 来说）。Spring Boot最好的特性之一是它利用现有的 Spring 项目和第三方项目来开发适合生产的应用程序。

#### 2.什么是 Spring Boot Starters?

Spring Boot Starters 是一系列依赖关系的集合，因为它的存在，项目的依赖之间的关系对我们来说变的更加简单了。不需要单独添加Spring MVC,  Tomcat这样的类库。

#### 3.如何在Spring Boot应用程序中使用Jetty而不是Tomcat?

Spring Boot Web starter使用Tomcat作为默认的嵌入式servlet容器, 如果你想使用 Jetty 的话只需要修改pom.xml(Maven)或者build.gradle(Gradle)就可以了。

```java
<!--从Web启动器依赖中排除Tomcat-->
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
	<exclusions>
		<exclusion>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-tomcat</artifactId>
		</exclusion>
	</exclusions>
</dependency>
<!--添加Jetty依赖-->
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
```

## SpringBoot的自动装配原理

https://www.jianshu.com/p/88eafeb3f351

```java
package org.springframework.boot.autoconfigure;
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {
		@Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
   ......
}
```

```java
package org.springframework.boot;
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration
public @interface SpringBootConfiguration {

}
```

可以看出大概可以把 `@SpringBootApplication `看作是 `@Configuration`、`@EnableAutoConfiguration`、`@ComponentScan `注解的集合。根据 SpringBoot官网，这三个注解的作用分别是：

- `@EnableAutoConfiguration`：启用 SpringBoot 的自动配置机制
- `@ComponentScan`： 扫描被`@Component` (`@Service`,`@Controller`)注解的bean，**注解默认会扫描该类所在的包下所有的类。**
- `@Configuration`：允许在上下文中注册额外的bean或导入其他配置类

这里重点看`@EnableAutoConfiguration` 因为是这个注解实现了SpringBoot的自动装配

```java
@Import({AutoConfigurationImportSelector.class})
public @interface EnableAutoConfiguration {
    String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

    Class<?>[] exclude() default {};

    String[] excludeName() default {};
}
```

利用`@Import`注解，将所有符合自动装配条件的bean注入到IOC容器中，关于`@Import`注解原理这里就不再阐述，感兴趣的同学可以参考此篇文章：[Spring @Import注解源码解析](https://mp.weixin.qq.com/s?__biz=MzU5MDgzOTYzMw==&mid=2247484613&idx=1&sn=374f1c3cc601bd86caaff29a8bb76d7d&scene=21#wechat_redirect)

进入类`AutoConfigurationImportSelector`，观察其`selectImports`方法，这个方法执行完毕后，Spring会把这个方法返回的类的全限定名数组里的所有的类都注入到IOC容器中

```java
private static final String[] NO_IMPORTS = new String[0];

//从这里可以看出该类实现很多的xxxAware和DeferredImportSelector，所有的aware都优先于selectImports
//方法执行，也就是说selectImports方法最后执行，那么在它执行的时候所有需要的资源都已经获取到了
public class AutoConfigurationImportSelector implements DeferredImportSelector, BeanClassLoaderAware, ResourceLoaderAware, BeanFactoryAware, EnvironmentAware, Ordered {
...
public String[] selectImports(AnnotationMetadata annotationMetadata) {
        if (!this.isEnabled(annotationMetadata)) {
            return NO_IMPORTS;
        } else {
//1加载META-INF/spring-autoconfigure-metadata.properties文件
            AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader.loadMetadata(this.beanClassLoader);
//2获取注解的属性及其值（PS：注解指的是@EnableAutoConfiguration注解）
            AnnotationAttributes attributes = this.getAttributes(annotationMetadata);
//3.在classpath下所有的META-INF/spring.factories文件中查找org.springframework.boot.autoconfigure.EnableAutoConfiguration的值，并将其封装到一个List中返回
            List<String> configurations = this.getCandidateConfigurations(annotationMetadata, attributes);
//4.对上一步返回的List中的元素去重、排序
            configurations = this.removeDuplicates(configurations);
//5.依据第2步中获取的属性值排除一些特定的类
            Set<String> exclusions = this.getExclusions(annotationMetadata, attributes);
//6对上一步中所得到的List进行过滤，过滤的依据是条件匹配。这里用到的过滤器是
//org.springframework.boot.autoconfigure.condition.OnClassCondition最终返回的是一个ConditionOutcome[]
//数组。（PS：很多类都是依赖于其它的类的，当有某个类时才会装配，所以这次过滤的就是根据是否有某个
//class进而决定是否装配的。这些类所依赖的类都写在META-INF/spring-autoconfigure-metadata.properties文件里）
            this.checkExcludedClasses(configurations, exclusions);
            configurations.removeAll(exclusions);
            configurations = this.filter(configurations, autoConfigurationMetadata);
            this.fireAutoConfigurationImportEvents(configurations, exclusions);
            return StringUtils.toStringArray(configurations);
        }
    }
  protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
        List<String> configurations = SpringFactoriesLoader.loadFactoryNames(this.getSpringFactoriesLoaderFactoryClass(), this.getBeanClassLoader());
        Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you are using a custom packaging, make sure that file is correct.");
        return configurations;
    }

...
}
```

**总结**：@EnableAutoConfiguration作用就是从classpath中搜寻所有的META-INF/spring.factories配置文件，并将其中org.springframework.boot.autoconfigure.EnableutoConfiguration对应的配置项通过反射（Java Refletion）实例化为对应的标注了@Configuration的JavaConfig形式的IoC容器配置类，然后汇总为一个并加载到IoC容器。这些功能配置类要生效的话，会去classpath中找是否有该类的依赖类（也就是pom.xml必须有对应功能的jar包才行）并且配置类里面注入了默认属性值类，功能类可以引用并赋默认值。生成功能类的原则是自定义优先，没有自定义时才会使用自动装配类。

- 所以功能类能生效需要的条件：（1）spring.factories里面有这个类的配置类（一个配置类可以创建多个围绕该功能的依赖类）（2）pom.xml里面需要有对应的jar包



## @RestController vs @Controller

**`@Controller` 返回一个页面**

单独使用 `@Controller` 不加 `@ResponseBody`的话一般使用在要返回一个视图的情况，这种情况属于比较传统的Spring MVC 的应用，对应于前后端不分离的情况。

**`@RestController` 返回JSON 或 XML 形式数据**

但`@RestController`只返回对象，对象数据直接以 JSON 或 XML 形式写入 HTTP 响应(Response)中，这种情况属于 RESTful Web服务，这也是目前日常开发所接触的最常用的情况（前后端分离）。

**`@Controller +@ResponseBody` 返回JSON 或 XML 形式数据**

如果你需要在Spring4之前开发 RESTful Web服务的话，你需要使用`@Controller` 并结合`@ResponseBody`注解，也就是说`@Controller` +`@ResponseBody`= `@RestController`（Spring 4 之后新加的注解）。

> `@ResponseBody` 注解的作用是将 `Controller` 的方法返回的对象通过适当的转换器转换为指定的格式之后，写入到HTTP 响应(Response)对象的 body 中，通常用来返回 JSON 或者 XML 数据，返回 JSON 数据的情况比较多。



## 为什么要使用Spring Ioc

https://cloud.tencent.com/developer/article/1033720

Ioc即控制反转，在spring中其实就是依赖注入。将原本程序中手动创建对象的控制权，交由Spring框架管理，Ioc容器是Spring实现Ioc的载体，Ioc容器实际上是一个Map(key,value)。

一个对象不可能单打独斗，它总要和其他对象进行交互合作，它通过构造参数，工厂方法参数或者对象属性定义其依赖关系，然后通过第三方容器（如spring ioc）在创建该对象时注入这些依赖，这就是控制反转，该对象即被称为bean，与之相反的是对象自身直接通过构造方法控制其依赖对象的实例化，或者通过ServiceLocator定位依赖对象，我们可以通过示例比较这两种方法。

```java
//抓取接口
public interface Crawl {
 public void crawlPage();
}
//抓取京东网站内容的实现类
public class JingdongCrawler implements Crawl{
 @Override
 public void crawlPage() {
  System.out.println("crawl Jingdong");
 }
}
//抓取控制器
public class CrawlControl {
 private Crawl crawler;
  //构造函数
 public CrawlControl(){
   //直接控制依赖对象的实例化 JingdongCrawler
 crawler = new JingdongCrawler();
   }
 public void execute(){
 crawler.crawlPage();
   }
}

```

这样这个抓取控制器就和依赖对象产生了耦合度，这是不符合松耦合的原则的，如果要抓去淘宝网站，就又要写一个新的抓取器了。我们可以将对象创建的操作放到Ioc容器，然后将对象通过构造参数或者属性定义依赖关系传入到这个控制器里.

```java
//抓取淘宝网站内容的实现类
public class TaobaoCrawler implements Crawl{
 @Override
 public void crawlPage() {
  System.out.print("crawl taobao");
 }
}
//CrawlControl 在ioc容器中的写法
public class CrawlControl {
 private Crawl crawler;'
  //这里crawler具体的实例化操作由Ioc容器完成
 public CrawlControl(Crawl crawler){
 this.crawler = crawler;
 }
 public void execute(){
 crawler.crawlPage();
 }
}
```

Spring Ioc 中 依赖倒置原则和依赖注入的了解

https://www.zhihu.com/question/23277575/answer/169698662

![img](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20201103223328.jpg)

依赖倒置：是把高层依赖底层转成底层依赖高层。高层需要什么，底层去实现这样的需求，这样底层进行细小的改动，不会对高层有什么影响。

依赖注入：是将底层类作为参数传入上层类，实现上层类对下层类的控制。可以用构造方法传递，也可以setter等形式完成注入。

那么这个Ioc Container，就是将高层类的初始化的时候，不需要依次创建底层类，而是交给Ioc Container，我们只需要维护一个Configuration(xml或者是代码)。

Ioc的源码阅读

https://javadoop.com/post/spring-ioc



## Spring AOP

这部分在设计模式中的 `代理模式` 有详细的介绍，这里只给出简解

![SpringAOPProcess](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20201103224431.jpeg)



**Spring AOP与AspectJ有什么区别**

前者属于运行时增强，而AspectJ是编译时增强。前者基于代理（Proxying），而后者基于字节码操作。

## Spring Bean

####  Spring 中的 bean 的作用域有哪些?

- singleton : 唯一 bean 实例，Spring 中的 bean 默认都是单例的。
- prototype : 每次请求都会创建一个新的 bean 实例。
- request : 每一次HTTP请求都会产生一个新的bean，该bean仅在当前HTTP request内有效。
- session : 每一次HTTP请求都会产生一个新的 bean，该bean仅在当前 HTTP session 内有效。

#### Spring Bean的线程安全问题

cnblogs.com/myseries/p/11729800.html

线程安全问题要从单例与原型Bean分别说明

**原型(prototype)Bean**

每次创建一个线程就会创建一个新对象，所以线程之间不存在Bean共享，没有线程安全的问题

**单例Bean**

如果是无状态Bean(不会对Bean的成员执行查询以外的操作),那么就是线程安全的，比如Spring mvc的Controller、Service、Dao等。

但是对于有状态Bean，框架并没有对bean进行多线程的封装处理，需要开发人员来做线程安全的保证，最简单的方法就是改变bean的作用域，变成prototype类型

**@Controller @Service 是不是线程安全的？**

默认情况不是，因为没有加@Scope，就表示是singleton，单例的，系统只会初始化一次Controller容器，所以每次请求的都是同一个Controller容器。

那么如果加了@Scope就一定是线程安全的吗？

```java
@RestController
@Scope(value = "prototype") // 加上@Scope注解，他有2个取值：单例-singleton 多实例-prototype
public class TestController {
    private int var = 0;
    private static int staticVar = 0;

    @GetMapping(value = "/test_var")
    public String test() {
        System.out.println("普通变量var:" + (++var)+ "---静态变量staticVar:" + (++staticVar));
        return "普通变量var:" + var + "静态变量staticVar:" + staticVar;
    }
}

返回结果
-----------------------
普通变量var:1---静态变量staticVar:1
普通变量var:1---静态变量staticVar:2
普通变量var:1---静态变量staticVar:3
```

也不一定！！！因为对于static变量，所有对象都能访问到，还是会有危险。

**总结**

1、在@Controller/@Service等容器中，默认情况下，scope值是单例-singleton的，也是线程不安全的。

2、尽量不要在@Controller/@Service等容器中定义静态变量，不论是单例(singleton)还是多实例(prototype)他都是线程不安全的。

3、默认注入的Bean对象，在不设置scope的时候他也是线程不安全的。

4、一定要定义变量的话，用ThreadLocal来封装，这个是线程安全的

#### @Component和@Bean的区别？

1. 作用对象不同: `@Component` 注解作用于类，而`@Bean`注解作用于方法。
2. `@Component`通常是通过类路径扫描来自动侦测以及自动装配到Spring容器中（我们可以使用 `@ComponentScan` 注解定义要扫描的路径从中找出标识了需要装配的类自动装配到 Spring 的 bean 容器中）。`@Bean` 注解通常是我们在标有该注解的方法中定义产生这个 bean,`@Bean`告诉了Spring这是某个类的示例，当我需要用它的时候还给我。
3. `@Bean` 注解比 `Component` 注解的自定义性更强，而且很多地方我们只能通过 `@Bean` 注解来注册bean。比如当我们引用第三方库中的类需要装配到 `Spring`容器时，则只能通过 `@Bean`来实现。

#### 将一个类声明为Spring的 bean 的注解有哪些?

我们一般使用 `@Autowired` 注解自动装配 bean，要想把类标识成可用于 `@Autowired` 注解自动装配的 bean 的类,采用以下注解可实现：

- `@Component` ：通用的注解，可标注任意类为 `Spring` 组件。如果一个Bean不知道属于哪个层，可以使用`@Component` 注解标注。
- `@Repository` : 对应持久层即 Dao 层，主要用于数据库相关操作。
- `@Service` : 对应服务层，主要涉及一些复杂的逻辑，需要用到 Dao层。
- `@Controller` : 对应 Spring MVC 控制层，主要用于接受用户请求并调用 Service 层返回数据给前端页面。



#### Spring Bean的生命周期

https://www.jianshu.com/p/3944792a5fff

http://c.biancheng.net/view/4261.html

Spring容器可以管理singleton作用域Bean的生命周期，Spring能精确的知道Bean何时被创建，何时初始化完成以及何时被销毁

对于prototype作用域的Bean，Spring只负责创建，实例化后就会将Bean的实例交给客户端代码管理，不再跟踪其生命周期。

Bean Factory 和ApplicationContext 是Spring两个很重要的容器，前者提供最基本的依赖注入的支持，后者在继承前者基础上进行了功能的拓展，会介绍两种容器中Bean的生命周期。

**ApplicationContext  Bean的生命周期**

![Bean的生命周期](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20201031151936.png)



Bean的生命周期可以如下描述：

1）根据配置情况调用Bean构造方法或工厂方法实例化Bean

2）利用依赖注入完成Bean所有属性值的配置注入

3）如果 Bean 实现了 BeanNameAware 接口，则 Spring 调用 Bean 的 setBeanName() 方法传入当前 Bean 的 id 值。

4）如果 Bean 实现了 BeanFactoryAware 接口，则 Spring 调用 setBeanFactory() 方法传入当前工厂实例的引用。

5）如果 Bean 实现了 ApplicationContextAware 接口，则 Spring 调用 setApplicationContext() 方法传入当前 ApplicationContext 实例的引用。

6）如果 BeanPostProcessor 和 Bean 关联，则 Spring 将调用该接口的预初始化方法 postProcessBeforeInitialzation() 对 Bean 进行加工操作，此处非常重要，Spring 的 AOP 就是利用它实现的。

7）如果 Bean 实现了 InitializingBean 接口，则 Spring 将调用 afterPropertiesSet() 方法。

8）如果在配置文件中通过 init-method 属性指定了初始化方法，则调用该初始化方法。

9）如果 BeanPostProcessor 和 Bean 关联，则 Spring 将调用该接口的初始化方法 postProcessAfterInitialization()。此时，Bean 已经可以被应用系统使用了。

10）如果在 <bean> 中指定了该 Bean 的作用范围为 scope="singleton"，则将该 Bean 放入 Spring IoC 的缓存池中，将触发 Spring 对该 Bean 的生命周期管理；如果在 <bean> 中指定了该 Bean 的作用范围为 scope="prototype"，则将该 Bean 交给调用者，调用者管理该 Bean 的生命周期，Spring 不再管理该 Bean。

11）如果 Bean 实现了 DisposableBean 接口，则 Spring 会调用 destory() 方法将 Spring 中的 Bean 销毁；如果在配置文件中通过 destory-method 属性指定了 Bean 的销毁方法，则 Spring 将调用该方法对 Bean 进行销毁。

在applicationContext.xml文件中配置Person方法

```java
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:p="http://www.springframework.org/schema/p"
    xmlns:aop="http://www.springframework.org/schema/aop" xmlns:tx="http://www.springframework.org/schema/tx"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="
            http://www.springframework.org/schema/beans 
            http://www.springframework.org/schema/beans/spring-beans-3.2.xsd
            http://www.springframework.org/schema/context
            http://www.springframework.org/schema/context/spring-context-3.2.xsd">
                    
     <bean id="person1" destroy-method="myDestroy" 
            init-method="myInit" class="com.test.spring.life.Person">
        <property name="name">
            <value>jack</value>
        </property>
    </bean>
    
    <!-- 配置自定义的后置处理器 -->
     <bean id="postProcessor" class="com.pingan.spring.life.MyBeanPostProcessor" />
</beans>
```

```java
public class Person implements BeanNameAware, BeanFactoryAware,
        ApplicationContextAware, InitializingBean, DisposableBean {

    private String name;
    
    public Person() {
        System.out.println("PersonService类构造方法");
    }
    
    
    public String getName() {
        return name;
    }
    public void setName(String name) {
        this.name = name;
        System.out.println("set方法被调用");
    }

    //自定义的初始化函数
    public void myInit() {
        System.out.println("myInit被调用");
    }
    
    //自定义的销毁方法
    public void myDestroy() {
        System.out.println("myDestroy被调用");
    }

    public void destroy() throws Exception {
        // TODO Auto-generated method stub
     System.out.println("destory被调用");
    }

    public void afterPropertiesSet() throws Exception {
        // TODO Auto-generated method stub
        System.out.println("afterPropertiesSet被调用");
    }

    public void setApplicationContext(ApplicationContext applicationContext)
            throws BeansException {
        // TODO Auto-generated method stub
       System.out.println("setApplicationContext被调用");
    }

    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        // TODO Auto-generated method stub
         System.out.println("setBeanFactory被调用,beanFactory");
    }

    public void setBeanName(String beanName) {
        // TODO Auto-generated method stub
        System.out.println("setBeanName被调用,beanName:" + beanName);
    }
    
    public String toString() {
        return "name is :" + name;
    }
```

```java
public class MyBeanPostProcessor implements BeanPostProcessor {

    public Object postProcessBeforeInitialization(Object bean,
            String beanName) throws BeansException {
        // TODO Auto-generated method stub
        
        System.out.println("postProcessBeforeInitialization被调用");
        return bean;
    }

    public Object postProcessAfterInitialization(Object bean,
            String beanName) throws BeansException {
        // TODO Auto-generated method stub
        System.out.println("postProcessAfterInitialization被调用");
        return bean;
    }

}
```

```java
public class AcPersonServiceTest {

    public static void main(String[] args) {
        // TODO Auto-generated method stub

        System.out.println("开始初始化容器");
        ApplicationContext ac = new ClassPathXmlApplicationContext("com/test/spring/life/applicationContext.xml");
        
        System.out.println("xml加载完毕");
        Person person1 = (Person) ac.getBean("person1");
        System.out.println(person1);        
        System.out.println("关闭容器");
        ((ClassPathXmlApplicationContext)ac).close();
        
    }

}
```

启动容器以后运行结果如下：

```java
开始初始化容器
九月 25, 2016 10:44:50 下午 org.springframework.context.support.ClassPathXmlApplicationContext prepareRefresh信息: Refreshing org.springframework.context.support.ClassPathXmlApplicationContext@b4aa453: startup date [Sun Sep 25 22:44:50 CST 2016]; root of context hierarchy
九月 25, 2016 10:44:50 下午 org.springframework.beans.factory.xml.XmlBeanDefinitionReader loadBeanDefinitions
信息: Loading XML bean definitions from class path resource [com/test/spring/life/applicationContext.xml]
Person类构造方法
set方法被调用
setBeanName被调用,beanName:person1
setBeanFactory被调用,beanFactory
setApplicationContext被调用
postProcessBeforeInitialization被调用
afterPropertiesSet被调用
myInit被调用
postProcessAfterInitialization被调用
xml加载完毕
name is :jack
关闭容器
九月 25, 2016 10:44:51 下午 org.springframework.context.support.ClassPathXmlApplicationContext doClose
信息: Closing org.springframework.context.support.ClassPathXmlApplicationContext@b4aa453: startup date [Sun Sep 25 22:44:50 CST 2016]; root of context hierarchy
destory被调用
myDestroy被调用
```



**Bean Factory  Bean的生命周期**

![img](https://upload-images.jianshu.io/upload_images/3131012-249748bc2b49e857.png?imageMogr2/auto-orient/strip|imageView2/2/w/746/format/webp)

可以发现和ApplicationContext相比有以下不同：

- BeanFactory容器中，不会调用ApplicationContextAware接口的setApplicationContext()方法，
- BeanPostProcessor接口的postProcessBeforeInitialzation()方法和postProcessAfterInitialization()方法不会自动调用，必须自己通过代码手动注册
- Bean Factory容器启动的时候，不会实例化所有Bean，包括singleton，而是在调用的时候去实例化

```java
public class BfPersonServiceTest {

    public static void main(String[] args) {
        // TODO Auto-generated method stub
        System.out.println("开始初始化容器");  
        ConfigurableBeanFactory bf = new XmlBeanFactory(new ClassPathResource("com/pingan/spring/life/applicationContext.xml"));
        System.out.println("xml加载完毕");      
        //beanFactory需要手动注册beanPostProcessor类的方法
        bf.addBeanPostProcessor(new MyBeanPostProcessor());
        Person person1 = (Person) bf.getBean("person1");
            System.out.println(person1);
        System.out.println("关闭容器");
        bf.destroySingletons();
    }

}
```

```java
开始初始化容器
九月 26, 2016 12:27:05 上午 org.springframework.beans.factory.xml.XmlBeanDefinitionReader loadBeanDefinitions
信息: Loading XML bean definitions from class path resource [com/pingan/spring/life/applicationContext.xml]
xml加载完毕
PersonService类构造方法
set方法被调用
setBeanName被调用,beanName:person1
setBeanFactory被调用,beanFactory
postProcessBeforeInitialization被调用
afterPropertiesSet被调用
myInit被调用
postProcessAfterInitialization被调用
name is :jack
关闭容器
destory被调用
myDestroy被调用
```





## Java日志框架：slf4j作用及其实现原理

https://www.cnblogs.com/xrq730/p/8619156.html



## Spring MVC

**原理如下图所示：** [![SpringMVC运行原理](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20201104103924.png)](https://camo.githubusercontent.com/6889f839138de730fce5f6a0d64e33258a2cf9b5/687474703a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f31382d31302d31312f34393739303238382e6a7067)

上图的一个笔误的小问题：Spring MVC 的入口函数也就是前端控制器 `DispatcherServlet` 的作用是接收请求，响应结果。

**流程说明（重要）：**

1. 客户端（浏览器）发送请求，直接请求到 `DispatcherServlet`。
2. `DispatcherServlet` 根据请求信息调用 `HandlerMapping`，解析请求对应的 `Handler`。
3. 解析到对应的 `Handler`（也就是我们平常说的 `Controller` 控制器）后，开始由 `HandlerAdapter` 适配器处理。
4. `HandlerAdapter` 会根据 `Handler `来调用真正的处理器来处理请求，并处理相应的业务逻辑。
5. 处理器处理完业务后，会返回一个 `ModelAndView` 对象，`Model` 是返回的数据对象，`View` 是个逻辑上的 `View`。
6. `ViewResolver` 会根据逻辑 `View` 查找实际的 `View`。
7. `DispaterServlet` 把返回的 `Model` 传给 `View`（视图渲染）。
8. 把 `View` 返回给请求者（浏览器）

## Spring 中用到了哪些设计模式

https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247485303&idx=1&sn=9e4626a1e3f001f9b0d84a6fa0cff04a&chksm=cea248bcf9d5c1aaf48b67cc52bac74eb29d6037848d6cf213b0e5466f2d1fda970db700ba41&token=255050878&lang=zh_CN#rd

#### 控制反转(IoC)和依赖注入(DI)

#### 工厂模式

`BeanFactory`或`ApplicationContext` 创建bean对象

**两者对比：**

- `BeanFactory` ：延迟注入(使用到某个 bean 的时候才会注入),相比于`BeanFactory`来说会占用更少的内存，程序启动速度更快。
- `ApplicationContext` ：容器启动的时候，不管你用没用到，一次性创建所有 bean 。`BeanFactory` 仅提供了最基本的依赖注入支持，`ApplicationContext` 扩展了 `BeanFactory` ,除了有`BeanFactory`的功能还有额外更多功能，所以一般开发人员使用`ApplicationContext`会更多。

#### 单例模式

在我们的系统中，有一些对象其实我们只需要一个，比如说：线程池、缓存、对话框、注册表、日志对象、充当打印机、显卡等设备驱动程序的对象。事实上，这一类对象只能有一个实例，如果制造出多个实例就可能会导致一些问题的产生，比如：程序的行为异常、资源使用过量、或者不一致性的结果。

**使用单例模式的好处:**

- 对于频繁使用的对象，可以省略创建对象所花费的时间，这对于那些重量级对象而言，是非常可观的一笔系统开销；
- 由于 new 操作的次数减少，因而对系统内存的使用频率也会降低，这将减轻 GC 压力，缩短 GC 停顿时间。

**Spring 实现单例的方式：**

- xml:<bean id="userService" class="top.snailclimb.UserService" scope="singleton"/>``
- 注解：`@Scope(value = "singleton")`

Spring 通过 `ConcurrentHashMap` 实现单例注册表的特殊方式实现单例模式。Spring 实现单例的核心代码如下：

```java
// 通过 ConcurrentHashMap（线程安全） 实现单例注册表
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(64);

public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
        Assert.notNull(beanName, "'beanName' must not be null");
        synchronized (this.singletonObjects) {
            // 检查缓存中是否存在实例  
            Object singletonObject = this.singletonObjects.get(beanName);
            if (singletonObject == null) {
                //...省略了很多代码
                try {
                    singletonObject = singletonFactory.getObject();
                }
                //...省略了很多代码
                // 如果实例对象在不存在，我们注册到单例注册表中。
                addSingleton(beanName, singletonObject);
            }
            return (singletonObject != NULL_OBJECT ? singletonObject : null);
        }
    }
    //将对象添加到单例注册表
    protected void addSingleton(String beanName, Object singletonObject) {
            synchronized (this.singletonObjects) {
                this.singletonObjects.put(beanName, (singletonObject != null ? singletonObject : NULL_OBJECT));

            }
        }
}
```

## Spring 事务

#### 1.Spring管理事务的方式

- 编程式，在代码中硬编码(不推荐)
- 声明式，在配置文件中配置(推荐) 分为XML以及注解的方式

#### 2. Spring事务的隔离级别

**TransactionDefinition 接口中定义了五个表示隔离级别的常量：**

- **TransactionDefinition.ISOLATION_DEFAULT:** 使用后端数据库默认的隔离级别，Mysql 默认采用的 REPEATABLE_READ隔离级别 Oracle 默认采用的 READ_COMMITTED隔离级别.
- **TransactionDefinition.ISOLATION_READ_UNCOMMITTED:** 最低的隔离级别，允许读取尚未提交的数据变更，**可能会导致脏读、幻读或不可重复读**
- **TransactionDefinition.ISOLATION_READ_COMMITTED:** 允许读取并发事务已经提交的数据，**可以阻止脏读，但是幻读或不可重复读仍有可能发生**
- **TransactionDefinition.ISOLATION_REPEATABLE_READ:** 对同一字段的多次读取结果都是一致的，除非数据是被本身事务自己所修改，**可以阻止脏读和不可重复读，但幻读仍有可能发生。**
- **TransactionDefinition.ISOLATION_SERIALIZABLE:** 最高的隔离级别，完全服从ACID的隔离级别。所有的事务依次逐个执行，这样事务之间就完全不可能产生干扰，也就是说，**该级别可以防止脏读、不可重复读以及幻读**。但是这将严重影响程序的性能。通常情况下也不会用到该级别。

#### 3. @Transactional(rollbackFor = Exception.class)注解了解吗？

我们知道：Exception分为运行时异常RuntimeException和非运行时异常。事务管理对于企业应用来说是至关重要的，即使出现异常情况，它也可以保证数据的一致性。

当`@Transactional`注解作用于类上时，该类的所有 public 方法将都具有该类型的事务属性，同时，我们也可以在方法级别使用该标注来覆盖类级别的定义。如果类或者方法加了这个注解，那么这个类里面的方法抛出异常，就会回滚，数据库里面的数据也会回滚。

在`@Transactional`注解中如果不配置`rollbackFor`属性,那么事务只会在遇到`RuntimeException`的时候才会回滚,加上`rollbackFor=Exception.class`,可以让事务在遇到非运行时异常时也回滚。

关于 `@Transactional `注解推荐阅读的文章：

- [透彻的掌握 Spring 中@transactional 的使用](https://www.ibm.com/developerworks/cn/java/j-master-spring-transactional-use/index.html)