



## 为什么要使用Spring Ioc

https://cloud.tencent.com/developer/article/1033720

Ioc即控制反转，在spring中其实就是依赖注入。一个对象不可能单打独斗，它总要和其他对象进行交互合作，它通过构造参数，工厂方法参数或者对象属性定义其依赖关系，然后通过第三方容器（如spring ioc）在创建该对象时注入这些依赖，这就是控制反转，该对象即被称为bean，与之相反的是对象自身直接通过构造方法控制其依赖对象的实例化，或者通过ServiceLocator定位依赖对象，我们可以通过示例比较这两种方法。

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

那么具体Ioc容器内部是怎么实现的本章并没有讲述。



## Spring Bean的生命周期

https://www.jianshu.com/p/3944792a5fff

http://c.biancheng.net/view/4261.html

Spring容器可以管理singleton作用域Bean的生命周期，Spring能精确的知道Bean何时被创建，何时初始化完成以及何时被销毁

对于prototype作用域的Bean，Spring只负责创建，实例化后就会将Bean的实例交给客户端代码管理，不再跟踪其生命周期。

Bean Factory 和ApplicationContext 是Spring两个很重要的容器，前者提供最基本的依赖注入的支持，后者在继承前者基础上进行了功能的拓展，会介绍两种容器中Bean的生命周期。

**ApplicationContext  Bean的生命周期**

![Bean的生命周期](http://c.biancheng.net/uploads/allimg/190701/5-1ZF1100325116.png)



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

