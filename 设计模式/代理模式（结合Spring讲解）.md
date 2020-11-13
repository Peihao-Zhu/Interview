## ★★★ 理解代理模式，结合 Spring 中的 AOP 回答。

https://www.jianshu.com/p/471c80a7e831

https://blog.csdn.net/Luckydog1991/article/details/51711474

代理模式根据目的和实现方式不同可以分为以下三种：‘

​	 1) 远程代理(Remote Proxy)：为一个位于不同的地址空间的对象提供一个本地的代理对象，这个不同的地址空间可以是在同一台主机中，也可是在另一台主机中。

​	 2) 虚拟代理(Virtual Proxy)：如果需要创建一个资源消耗较大的对象，先创建一个消耗相对较小的对象来表示，真实对象只在需要时才会被真正创建。

​     3) 保护代理(Protect Proxy)：控制对一个对象的访问，可以给不同的用户提供不同级别的使用权限。

  

Java中的代理一般分为三种：**JDK静态代理**、**JDK动态代理**、**CGLIB代理**。 在Spring AOP的实现中主要应用了JDK动态代理以及CGLIB代理，



#### JDK静态代理

![img](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20201103163419.png)

```java
//接口，也可以是抽象类，具体类
public interface HelloInterface{
	void sayHello();
}
//被代理类
public class hello inplements helloInterface{
	@Override
	public void sayHello(){
		System.out.println("Hello man");
	}
}
//代理类
public class helloProxy inplements helloInterface{
	private Hello hello=new Hello();
	
	@Override
	public void sayHello(){
		System.out.println("Before");
		hello.sayHello();
		System.out.println("After");
	}
}


```

实质就是代理类为被代理类预处理消息、过滤消息并在此之后转发给被代理类。代理类本身不提供服务，而是调用被代理类中的方法来提供。

静态代理存在一个问题！！！！代理类只能为一个类提供服务，如果需要代理其他的类，需要编写很多代码。



#### 动态代理

```java
//接口
public interface HelloInterface{
	void sayHello();
}
//被代理类（目标对象）
public class hello inplements helloInterface{
	@Override
	public void sayHello(){
		System.out.println("Hello man");
	}
}
//InvocationHandler 类
public class helloProxy {
	private Object subject;
	
  public helloProxy(Object subject){
    	this.subject=subject;
  }
  //InvocationHandler 方法会自动调用invoke，并根据传入的代理对象、方法名称、参数决定调用被代理对象的哪个方法
	@Override
	public Object invoke(Object proxy, Method method, Object[] args)throws Throwable{
		System.out.println("Before"+method.getName());
		method.invoke(subject,args);
		System.out.println("After"+,method.getName());
    return null;
	}
}

//测试类

public class client{
  public static void main(String[] args){
    Hello hello=new Hello();
    InvocationHandler handler=new HelloProxy(hello);
    
    HelloInterface helloInterface=(HelloInterface)Proxy.newProxyInstance(handler.getClass().getClassLoader(),hello.getClass().getInterface(),handler);
    helloInterface.sayHello();
  }
}
```

可以发现在动态代理也要创建代理类，以及被代理类都要实现接口。但是和静态代理的区别也比较明显----在静态代理中需要对哪个接口和那个被代理类创建代理类，在编译前就需要代理类实现与被代理类相同的接口，并且在实现的方法中调用被代理类相应的方法；**但是动态代理不同，我们不知道要针对哪个接口、哪个被代理类创建代理类，这个操作是在运行时被创建的。**

更详细的说JDK静态代理是通过直接编码创建的，而JDK动态代理是利用**反射机制**在运行时创建代理类。其中在动态代理中，核心是InvocationHandler。每一个代理的实例都会有一个关联的调用处理程序（InvocationHandler）。对代理实例进行调用时（hello），将其方法的调用进行编码并指派到他的调用处理器（InvocationHandler）的invoke方法中完成，而invoke方法会根据传入的代理对象，方法名称，参数决定调用代理的哪个方法。

**JDK动态代理使用步骤**

1. 定义一个接口及其实现类
2. 自定义InvocationHandler 并重写invoke方法，在invoke方法中我们会调用原生方法（被代理类的方法）并自定义一些处理逻辑
3. 通过Proxy.newProxyInstance(ClassLoader loader, Class<?>[]interfaces,InvocationHandler h)

**获取代理对象的工厂类**

```java
public class JdkProxyFactory {
    public static Object getProxy(Object target) {
        return Proxy.newProxyInstance(
                target.getClass().getClassLoader(), // 目标类的类加载
                target.getClass().getInterfaces(),  // 代理需要实现的接口，可指定多个
                new DebugInvocationHandler(target)   // 代理对象对应的自定义 InvocationHandler
        );
    }
}
```

`getProxy()` ：主要通过`Proxy.newProxyInstance（）`方法获取某个类的代理对象

**这里总结一下JDK动态代理：代理对象不需要实现接口，但是目标对象一定要实现接口，否则不能动态代理，因为newProxyInstance的第二个参数需要传入目标对象的接口**



#### CGLIB

**上面的静态代理和动态代理都是要求目标对象实现一个接口**，但是有时候目标对象没有实现任何接口，这时可以使用CGLIB代理，也叫做子类代理，它是内存中构建一个子类对象从而实现对目标对象功能的扩展。

[CGLIB](https://github.com/cglib/cglib)(*Code Generation Library*)是一个基于[ASM](http://www.baeldung.com/java-asm)的字节码生成库，它允许我们在运行时对字节码进行修改和动态生成。CGLIB 通过继承方式实现代理。很多知名的开源框架都使用到了[CGLIB](https://github.com/cglib/cglib)， 例如 Spring 中的 AOP 模块中：如果目标对象实现了接口，则默认采用 JDK 动态代理，否则采用 CGLIB 动态代理。

**在 CGLIB 动态代理机制中 `MethodInterceptor` 接口和 `Enhancer` 类是核心。**

你需要自定义 `MethodInterceptor` 并重写 `intercept` 方法，`intercept` 用于拦截增强被代理类的方法。

```
public interface MethodInterceptor
extends Callback{
    // 拦截被代理类中的方法
    public Object intercept(Object obj, java.lang.reflect.Method method, Object[] args,
                               MethodProxy proxy) throws Throwable;
}
```

1. **obj** :被代理的对象（需要增强的对象）
2. **method** :被拦截的方法（需要增强的方法）
3. **args** :方法入参
4. **methodProxy** :用于调用原始方法



你可以通过 `Enhancer`类来动态获取被代理类，当代理类调用方法的时候，实际调用的是 `MethodInterceptor` 中的 `intercept` 方法。

**CGLIB动态代理类使用步骤**

1. 定义一个类；
2. 自定义 `MethodInterceptor` 并重写 `intercept` 方法，`intercept` 用于拦截增强被代理类的方法，和 JDK 动态代理中的 `invoke` 方法类似；
3. 通过 `Enhancer` 类的 `create()`创建代理类；



CGLIB子类代理实现方法：

1. 需要引入cglib的jar文件，但是Spring的核心包中包括了cglib功能，所以直接引入spring-core-3.2.5.jar即可。
2. 引入功能包后，就可以在内存中动态构建子类
3. 代理的类不能使用final，否则报错
4. 目标对象的方法如果为final/static，那么就不会被拦截，既不会执行对象额外的业务开发

不同于 JDK 动态代理不需要额外的依赖。[CGLIB](https://github.com/cglib/cglib)(*Code Generation Library*) 实际是属于一个开源项目，如果你要使用它的话，需要手动添加相关依赖。

```
<dependency>
  <groupId>cglib</groupId>
  <artifactId>cglib</artifactId>
  <version>3.3.0</version>
</dependency>
```

**1.实现一个使用阿里云发送短信的类**

```java
package github.javaguide.dynamicProxy.cglibDynamicProxy;

public class AliSmsService {
    public String send(String message) {
        System.out.println("send message:" + message);
        return message;
    }
}
```

**2.自定义 `MethodInterceptor`（方法拦截器）**

```java
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;

import java.lang.reflect.Method;

/**
 * 自定义MethodInterceptor
 */
public class DebugMethodInterceptor implements MethodInterceptor {


    /**
     * @param o           被代理的对象（需要增强的对象）
     * @param method      被拦截的方法（需要增强的方法）
     * @param args        方法入参
     * @param methodProxy 用于调用原始方法
     */
    @Override
    public Object intercept(Object o, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
        //调用方法之前，我们可以添加自己的操作
        System.out.println("before method " + method.getName());
        Object object = methodProxy.invokeSuper(o, args);
        //调用方法之后，我们同样可以添加自己的操作
        System.out.println("after method " + method.getName());
        return object;
    }

}
```

**3.获取代理类**

```java
import net.sf.cglib.proxy.Enhancer;

public class CglibProxyFactory {

    public static Object getProxy(Class<?> clazz) {
        // 创建动态代理增强类
        Enhancer enhancer = new Enhancer();
        // 设置类加载器
        enhancer.setClassLoader(clazz.getClassLoader());
        // 设置被代理类
        enhancer.setSuperclass(clazz);
        // 设置方法拦截器
        enhancer.setCallback(new DebugMethodInterceptor());
        // 创建代理类
        return enhancer.create();
    }
}
```

**4.实际使用**

```java
AliSmsService aliSmsService = (AliSmsService) CglibProxyFactory.getProxy(AliSmsService.class);
aliSmsService.send("java");
```

运行上述代码之后，控制台打印出：

```java
before method send
send message:java
after method send
```

**JDK动态代理和CGLIB字节码生成的区别？**

（1）JDK动态代理只能对实现了接口的类生成代理

（2）CGLIB是针对类实现代理，主要是针对指定的类生成的子类，覆盖其中的方法。



**Spring在选择用JDK还是CGLIB的依据？**

当Bean实现接口时，Spring会用JDK动态代理；反之使用CGLIB；也可以强制使用CGLIB（在spring配置中加入<aop:aspectj-autoproxy proxy-target-class="true"/>）



**CGLIB比JDK快？**

（1）使用CGLIB实现动态代理，底层采用ASM字节码生成框架，使用字节码技术生成代理类，比使用Java反射效率高。但是CGLIB不能对申明为final的方法进行代理，因为其是动态生成被代理类的子类（而final类不能被继承）