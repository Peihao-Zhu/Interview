

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

这样这个抓取控制器酒喝依赖对象产生了耦合度，这是不符合松耦合的原则的，如果要抓去淘宝网站，就又要写一个新的抓取器了。我们可以将对象创建的操作放到Ioc容器，然后将对象通过构造参数或者属性定义依赖关系传入到这个控制器里.

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