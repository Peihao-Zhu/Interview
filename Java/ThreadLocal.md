https://www.jianshu.com/p/98b68c97df9b

## ThreadLocal是什么

ThreadLocal不是用来解决共享对象的多线程访问的，其使得各个线程能够保持各自独立的对象。每个线程的变量互不干扰，在高并发场景下，可以实现无状态调用。

介绍完可能还是有点懵逼，再多讲一下：在并发编程的时候，成员变量如果不做任何处理是线程不安全的，各个线程都在操作同一个变量，显然是不行的。



## ThreadLocal的内部结构

![WeChat5ac2608c26caa9cf22cff6ebe8b64173](/Users/zhupeihao/Library/Containers/com.tencent.xinWeChat/Data/Library/Caches/com.tencent.xinWeChat/2.0b4.0.9/1ab32f76abfa38116a670b7ac59c1027/dragImgTmp/WeChat5ac2608c26caa9cf22cff6ebe8b64173.png)

- 每个Thread都有一个Map
- Map里面存储了本地对象(ThreadLocal)和线程的变量副本（value）
- 每个Thread内部的Map是由ThreadLocal来维护的

Thread类内部定义了ThreadLocalMap

```java
public class Thread implements Runnable {
    /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;
}
```

ThreadLocal类提供了几个核心方法：

```java
public T get()
public void set(T value)
public void remove()
```

#### get()方法

```java
/**
 * Returns the value in the current thread's copy of this
 * thread-local variable.  If the variable has no value for the
 * current thread, it is first initialized to the value returned
 * by an invocation of the {@link #initialValue} method.
 *
 * @return the current thread's value of this thread-local
 */
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null)
            return (T)e.value;
    }
    return setInitialValue();
}

ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}

private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}

protected T initialValue() {
    return null;
}
```

步骤：
 1.获取当前线程的ThreadLocalMap对象threadLocals
 2.从map中获取线程存储的K-V Entry节点。
 3.从Entry节点获取存储的Value副本值返回。
 4.map为空的话返回初始值null，即线程变量副本为null，在使用时需要注意判断NullPointerException。



这里可能对ThreadLocalMap的内部结构会有些疑惑，是不是和HashMap一样呢？

它并没有实现Map接口，用独立的方式实现Map功能，其内部的Entry也独立实现。

```java
static class Entry extends WeakReference<ThreadLocal> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal k, Object v) {
        super(k);
        value = v;
    }
}
```

Entry继承自WeakReference（弱引用，生命周期只能存活到下次GC前），**但只有Key是弱引用类型的，Value并非弱引用。**

## Hash冲突怎么解决

和HashMap的最大的不同在于，ThreadLocalMap结构非常简单，没有next引用，也就是说ThreadLocalMap中解决Hash冲突的方式并非链表的方式，而是采用线性探测的方式，所谓线性探测，就是根据初始key的hashcode值确定元素在table数组中的位置，如果发现这个位置上已经有其他key值的元素被占用，则利用固定的算法寻找一定步长的下个位置，依次判断，直至找到能够存放的位置。

**所以这里引出的良好建议是：每个线程只存一个变量，这样的话所有的线程存放到map中的Key都是相同的ThreadLocal，如果一个线程要保存多个变量，就需要创建多个ThreadLocal，多个ThreadLocal放入Map中时会极大的增加Hash冲突的可能。**



## ThreadLocalMap的问题

由于ThreadLocalMap的key是弱引用，而Value是强引用。这就导致了一个问题，ThreadLocal在没有外部对象强引用时，发生GC时弱引用Key会被回收，而Value不会回收，如果创建ThreadLocal的线程一直持续运行，那么这个Entry对象中的value就有可能一直得不到回收，发生内存泄露。

**如何避免泄漏**
 既然Key是弱引用，那么我们要做的事，就是在调用ThreadLocal的get()、set()方法时完成后再调用remove方法，将Entry节点和Map的引用关系移除，这样整个Entry对象在GC Roots分析后就变成不可达了，下次GC的时候就可以被回收。

如果使用ThreadLocal的set方法之后，没有显示的调用remove方法，就有可能发生内存泄露，所以养成良好的编程习惯十分重要，使用完ThreadLocal之后，记得调用remove方法。

```java
ThreadLocal<Session> threadLocal = new ThreadLocal<Session>();
try {
    threadLocal.set(new Session(1, "Misout的博客"));
    // 其它业务逻辑
} finally {
    threadLocal.remove();
}
```

## 应用场景

```java
private static final ThreadLocal<Session> threadLocal = new ThreadLocal<Session>();

//获取Session
public static Session getCurrentSession(){
    Session session =  threadLocal.get();
    //判断Session是否为空，如果为空，将创建一个session，并设置到本地线程变量中
    try {
        if(session ==null&&!session.isOpen()){
            if(sessionFactory==null){
                rbuildSessionFactory();// 创建Hibernate的SessionFactory
            }else{
                session = sessionFactory.openSession();
            }
        }
        threadLocal.set(session);
    } catch (Exception e) {
        // TODO: handle exception
    }

    return session;
}
```

为什么？每个线程访问数据库都应当是一个独立的Session会话，如果多个线程共享同一个Session会话，有可能其他线程关闭连接了，当前线程再执行提交时就会出现会话已关闭的异常，导致系统异常。此方式能避免线程争抢Session，提高并发下的安全性。



使用ThreadLocal的典型场景如数据库连接管理，线程会话管理等场景，只适用于独立变量副本的情况，如果变量为全局共享的，则不适用在高并发下使用。

## 总结

- 每个ThreadLocal只能保存一个变量副本，如果想要上线一个线程能够保存多个副本以上，就需要创建多个ThreadLocal。
- ThreadLocal内部的ThreadLocalMap键为弱引用，会有内存泄漏的风险。
- 适用于无状态，副本变量独立后不影响业务逻辑的高并发场景。如果如果业务逻辑强依赖于副本变量，则不适合用ThreadLocal解决，需要另寻解决方案。


