[TOC]



## 线程池原理

https://www.cnblogs.com/dolphin0520/p/3932921.html#!comments

当并发的线程多的时候，使用线程池可以避免频繁的创建和销毁线程，实现线程的复用。

### ThreadPoolExecutor类

首先了解一下ThreadPoolExecutor的四个构造函数

```java
public class ThreadPoolExecutor extends AbstractExecutorService {
    .....
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,
            BlockingQueue<Runnable> workQueue);
 
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,
            BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory);
 
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,
            BlockingQueue<Runnable> workQueue,RejectedExecutionHandler handler);
 
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,
        BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory,RejectedExecutionHandler handler);
    ...
}
```

ThreadPoolExecutor类继承了AbstractExecutorService，并且四个构造方法的区别就是多加了一下参数，那么接下来解释一下各个参数的含义：

- **corePoolSize**：核心池的大小，这个参数跟后面讲述的线程池的实现原理有非常大的关系。在创建了线程池后，默认情况下，线程池中并没有任何线程，而是等待有任务到来才创建线程去执行任务，除非调用了`prestartAllCoreThreads()` 或者 `prestartCoreThread()`方法，从这 2 个方法的名字就可以看出，是预创建线程的意思，即在没有任务到来之前就创建 corePoolSize 个线程或者一个线程。默认情况下，在创建了线程池后，线程池中的线程数为0，当有任务来之后，就会创建一个线程去执行任务，当线程池中的线程数目达到 corePoolSize 后，就会把到达的任务放到缓存队列当中；

- **maximumPoolSize**：线程池最大线程数，这个参数也是一个非常重要的参数，它表示在线程池中最多能创建多少个线程；如果缓存队列满了，又进来了新的任务就会临时增加新的线程，一旦总的线程数超过了maximumPoolSize就会停止接受新的任务。

- **keepAliveTime**：表示线程没有任务执行时最多保持多久时间会终止。默认情况下，只有当线程池中的线程数大于 corePoolSize 时，keepAliveTime 才会起作用，直到线程池中的线程数不大于 corePoolSize，即当线程池中的线程数大于 corePoolSize 时，如果一个线程空闲的时间达到 keepAliveTime，则会终止，直到线程池中的线程数不超过 corePoolSize。但是如果调用了 `allowCoreThreadTimeOut(boolean)` 方法，在线程池中的线程数不大于 corePoolSize 时，keepAliveTime 参数也会起作用，直到线程池中的线程数为0；

- **unit**：参数 keepAliveTime 的时间单位，有 7 种取值，在 `TimeUnit` 类中有 7 种静态属性：

  ```java
  TimeUnit.DAYS;               //天
  TimeUnit.HOURS;             //小时
  TimeUnit.MINUTES;           //分钟
  TimeUnit.SECONDS;           //秒
  TimeUnit.MILLISECONDS;      //毫秒
  TimeUnit.MICROSECONDS;      //微妙
  TimeUnit.NANOSECONDS;       //纳秒
  ```

- **workQueue**：一个阻塞队列，用来存储等待执行的任务，这个参数的选择也很重要，会对线程池的运行过程产生重大影响，一般来说，这里的阻塞队列有以下几种选择：

  ```java
  ArrayBlockingQueue;
  LinkedBlockingQueue;
  SynchronousQueue;
  ```

- **threadFactory**：线程工厂，主要用来创建线程；

- **handler**：表示当拒绝处理任务时的策略，有以下四种取值：

```java
ThreadPoolExecutor.AbortPolicy:丢弃任务并抛出RejectedExecutionException异常。 
ThreadPoolExecutor.DiscardPolicy：也是丢弃任务，但是不抛出异常。 
ThreadPoolExecutor.DiscardOldestPolicy：丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）
ThreadPoolExecutor.CallerRunsPolicy：由调用线程处理该任务 
```



因为 **ThreadPoolExecutor**继承了 **AbstractExecutorService**，来看一下AbstractExecutorService的实现

```java
public abstract class AbstractExecutorService implements ExecutorService {
 
     
    protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) { };
    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) { };
    public Future<?> submit(Runnable task) {};
    public <T> Future<T> submit(Runnable task, T result) { };
    public <T> Future<T> submit(Callable<T> task) { };
    private <T> T doInvokeAny(Collection<? extends Callable<T>> tasks,
                            boolean timed, long nanos)
        throws InterruptedException, ExecutionException, TimeoutException {
    };
    public <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException {
    };
    public <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                           long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException {
    };
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException {
    };
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                         long timeout, TimeUnit unit)
        throws InterruptedException {
    };
}
```

**AbstractExecutorService**是一个抽象类，实现了**ExecutorService**接口

```java
public interface ExecutorService extends Executor {
 
    void shutdown();
    boolean isShutdown();
    boolean isTerminated();
    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;
    <T> Future<T> submit(Callable<T> task);
    <T> Future<T> submit(Runnable task, T result);
    Future<?> submit(Runnable task);
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;
 
    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;
    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

**ExecutorService**继承了**Executor**接口

```java
public interface Executor {
    void execute(Runnable command);
}
```

​		Executor是一个顶层接口，在它里面只声明了一个方法execute(Runnable)，返回值为void，参数为Runnable类型，从字面意思可以理解，就是用来执行传进去的任务的；

　　然后ExecutorService接口继承了Executor接口，并声明了一些方法：submit、invokeAll、invokeAny以及shutDown等；

　　抽象类AbstractExecutorService实现了ExecutorService接口，基本实现了ExecutorService中声明的所有方法；

　　然后ThreadPoolExecutor继承了类AbstractExecutorService。



**ThreadPoolExecutor** 类中有几个很重要的方法：

```java
execute()
submit()
shutdown()
shutdownNow()
```

​		**execute()**方法实际上是Executor中声明的方法，在ThreadPoolExecutor进行了具体的实现，这个方法是ThreadPoolExecutor的核心方法，通过这个方法可以向线程池提交一个任务，交由线程池去执行。

　　**submit()**方法是在ExecutorService中声明的方法，在AbstractExecutorService就已经有了具体的实现，在ThreadPoolExecutor中并没有对其进行重写，这个方法也是用来向线程池提交任务的，但是它和execute()方法不同，**它能够返回任务执行的结果**，去看submit()方法的实现，会发现它实际上还是调用的execute()方法，只不过它利用了Future来获取任务执行结果（Future相关内容将在下一篇讲述）。

　　**shutdown()和shutdownNow()**是用来关闭线程池的。

　　还有很多其他的方法：

　　比如：getQueue() 、getPoolSize() 、getActiveCount()、getCompletedTaskCount()等获取与线程池相关属性的方法，有兴趣的朋友可以自行查阅API。

### 深入剖析线程池原理

1. #### 线程池状态

   　　在ThreadPoolExecutor中定义了一个volatile变量，另外定义了几个static final变量表示线程池的各个状态：

   ```java
   volatile int runState;
   static final int RUNNING    = 0;
   static final int SHUTDOWN   = 1;
   static final int STOP       = 2;
   static final int TERMINATED = 3;
   ```

   ​		runState表示当前线程池的状态，它是一个volatile变量用来保证线程之间的可见性；

   　　下面的几个static final变量表示runState可能的几个取值。

   　　当创建线程池后，初始时，线程池处于RUNNING状态；

   　　如果调用了shutdown()方法，则线程池处于SHUTDOWN状态，此时线程池不能够接受新的任务，它会等待所有任务执行完毕；

   　　如果调用了shutdownNow()方法，则线程池处于STOP状态，此时线程池不能接受新的任务，并且会去尝试终止正在执行的任务；

   　　当线程池处于SHUTDOWN或STOP状态，并且所有工作线程已经销毁，任务缓存队列已经清空或执行结束后，线程池被设置为TERMINATED状态。

2. #### 任务执行

   在了解将任务提交给线程池到任务执行完毕整个过程之前，我们先来看一下ThreadPoolExecutor类中其他的一些比较重要成员变量：

   ```java
   private final BlockingQueue<Runnable> workQueue;              //任务缓存队列，用来存放等待执行的任务
   private final ReentrantLock mainLock = new ReentrantLock();   //线程池的主要状态锁，对线程池状态（比如线程池大小
                                                                 //、runState等）的改变都要使用这个锁
   private final HashSet<Worker> workers = new HashSet<Worker>();  //用来存放工作集
    
   private volatile long  keepAliveTime;    //线程存货时间   
   private volatile boolean allowCoreThreadTimeOut;   //是否允许为核心线程设置存活时间
   private volatile int   corePoolSize;     //核心池的大小（即线程池中的线程数目大于这个参数时，提交的任务会被放进任务缓存队列）
   private volatile int   maximumPoolSize;   //线程池最大能容忍的线程数
    
   private volatile int   poolSize;       //线程池中当前的线程数
    
   private volatile RejectedExecutionHandler handler; //任务拒绝策略
    
   private volatile ThreadFactory threadFactory;   //线程工厂，用来创建线程
    
   private int largestPoolSize;   //用来记录线程池中曾经出现过的最大线程数
    
   private long completedTaskCount;   //用来记录已经执行完毕的任务个数
   ```

   ​		下面来看一下任务从提交到最终执行完毕经历了哪些过程。

   　　在ThreadPoolExecutor类中，最核心的任务提交方法是execute()方法，虽然通过submit也可以提交任务，但是实际上submit方法里面最终调用的还是execute()方法，所以我们只需要研究execute()方法的实现原理即可：

   ```java
   public void execute(Runnable command) {
       if (command == null)                                              //1
           throw new NullPointerException();
       if (poolSize >= corePoolSize || !addIfUnderCorePoolSize(command)) {//3
           if (runState == RUNNING && workQueue.offer(command)) {         //4
               if (runState != RUNNING || poolSize == 0)                  //5
                   ensureQueuedTaskHandled(command);                     //6
           }
           else if (!addIfUnderMaximumPoolSize(command))                  //8
               reject(command); // is shutdown or saturated               //9
       }
   }
   ```

   - 第1行:提交的任务为null的话直接抛出空指针异常

   - 第3行：接下来这句，是或运算，如果当前线程池的线程数量大于等于核心池大小，就直接进入if语句块；或者当前线程数量小于核心池大小，就执行后半段，如果addIfUnderCorePoolSize(command)返回false也会进入if语句块

     ```java
      if (poolSize >= corePoolSize || !addIfUnderCorePoolSize(command))
     ```

   - 第4行：进入语句块后是与运算，如果当前线程池处于RUNNING状态，并且成功将任务放进任务缓存队列，进入接下来的语句块

     ```
     if (runState == RUNNING && workQueue.offer(command)) 
     ```

   - 第5行：这句判断是为了防止上面第4行执行完以后即将此任务放进任务缓存队列以后，其他线程突然调用了shutdown或shutdownNow方法关闭了线程池的一种应急措施

     ```
     if (runState != RUNNING || poolSize == 0)
     ```

   - 第6行：应急措施就是保证添加进缓存队列的任务可以得到处理

     ```
     ensureQueuedTaskHandled(command);  
     ```

   - 第8行：如果执行addIfUnderMaximumPoolSize(command)失败了，就直接执行reject 拒绝处理任务

     ```
       else if (!addIfUnderMaximumPoolSize(command))  
     ```

     

   分析完以后可以发现其中有两个方法比较重要：**addIfUnderCorePoolSize()**和**addIfUnderMaximumPoolSize()**

   ```java
   private boolean addIfUnderCorePoolSize(Runnable firstTask) {
       Thread t = null;
     	//因为接下来涉及到线程池状态（数量）的变化，所以需要用锁
       final ReentrantLock mainLock = this.mainLock;
       mainLock.lock();
       try {
         	
           if (poolSize < corePoolSize && runState == RUNNING)
               t = addThread(firstTask);        //创建线程去执行firstTask任务   
           } finally {
           mainLock.unlock();
       }
       if (t == null)
           return false;
       t.start();   //会去调用Worker类中的run 方法，因为Worker实现了Runnable接口
       return true;
   }
   ```

   问题：前面execute中已经判断过线程池中线程的个数是小于核心池大小，为什么这里有进行了一次判断？？

   因为先前的判断过程并没有加锁，所以在判断完以后，又有其他线程提交了任务，导致现在线程池中的数量已经大于corePoolSize，对于runState的判断也一样分析

   

   在上面这个方法中，**addThread(firstTask);** 方法很重要，会返回一个线程，如果为空，表示创建线程失败，否则就启动线程。

   ```java
   private Thread addThread(Runnable firstTask) {
       Worker w = new Worker(firstTask);				//用提交的任务创建一个Worker对象
       Thread t = threadFactory.newThread(w);  //用threadFactory创建一个线程，执行任务   
       if (t != null) {
           w.thread = t;            //将创建的线程的引用赋值为w的成员变量       
           workers.add(w);					 //将Worker对象添加到工作集中
           int nt = ++poolSize;     //当前线程数加1       
           if (nt > largestPoolSize)
               largestPoolSize = nt;
       }
       return t;
   }
   ```

   所以需要执行worker的时候，只需要调用worker中的thread的start方法即可，并且可以调用thread的方法来控制worker的状态：

   接下来看一下**Worker类**的实现

   ```java
   private final class Worker implements Runnable {
       private final ReentrantLock runLock = new ReentrantLock();
       private Runnable firstTask;
       volatile long completedTasks;
       Thread thread;
       Worker(Runnable firstTask) {
           this.firstTask = firstTask;
       }
       boolean isActive() {
           return runLock.isLocked();
       }
       void interruptIfIdle() {
           final ReentrantLock runLock = this.runLock;
           if (runLock.tryLock()) {
               try {
           if (thread != Thread.currentThread())
           thread.interrupt();
               } finally {
                   runLock.unlock();
               }
           }
       }
       void interruptNow() {
           thread.interrupt();
       }
    
       private void runTask(Runnable task) {
           final ReentrantLock runLock = this.runLock;
           runLock.lock();
           try {
               if (runState < STOP &&
                   Thread.interrupted() &&
                   runState >= STOP)
               boolean ran = false;
               beforeExecute(thread, task);   //beforeExecute方法是ThreadPoolExecutor类的一个方法，没有具体实现，用户可以根据
               //自己需要重载这个方法和后面的afterExecute方法来进行一些统计信息，比如某个任务的执行时间等           
               try {
                   task.run();
                   ran = true;
                   afterExecute(task, null);
                   ++completedTasks;
               } catch (RuntimeException ex) {
                   if (!ran)
                       afterExecute(task, ex);
                   throw ex;
               }
           } finally {
               runLock.unlock();
           }
       }
    
     //Worker实现了Runnable接口，所以最核心的方法就是run()方法
       public void run() {
           try {
               Runnable task = firstTask;
               firstTask = null;
             //不断通过getTask()从缓存队列里获取新的任务 这个方法是ThreadPoolExecutor类中的方法
               while (task != null || (task = getTask()) != null) {
                   runTask(task);
                   task = null;
               }
           } finally {
               workerDone(this);   //当任务队列中没有任务时，进行清理工作       
           }
       }
   }
   ```

   **线程池的线程复用**就是通过取 Worker 的 firstTask 或者通过 getTask 方法从 workQueue 中不停地取任务，并直接调用 Runnable 的 run 方法来执行任务，这样就保证了每个线程都始终在一个循环中，反复获取任务，然后执行任务，从而实现了线程的复用。

   

   **getTask()**方法实现：

   ```java
   Runnable getTask() {
       for (;;) {
           try {
               int state = runState;
               if (state > SHUTDOWN)
                   return null;
               Runnable r;
               if (state == SHUTDOWN)  // Help drain queue
                   r = workQueue.poll();
               else if (poolSize > corePoolSize || allowCoreThreadTimeOut) //如果线程数大于核心池大小或者允许为核心池线程设置空闲时间，
                   //则通过poll取任务，若等待一定的时间取不到任务，则返回null
                   r = workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS);
               else
                   r = workQueue.take();
               if (r != null)
                   return r;
               if (workerCanExit()) {    //如果没取到任务，即r为null，则判断当前的worker是否可以退出
                   if (runState >= SHUTDOWN) // Wake up others
                       interruptIdleWorkers();   //中断处于空闲状态的worker
                   return null;
               }
               // Else retry
           } catch (InterruptedException ie) {
               // On interruption, re-check runState
           }
       }
   }
   ```

   　	在getTask()中，先判断当前线程池状态，如果runState大于SHUTDOWN（即为STOP或者TERMINATED），则直接返回null。

   　　如果runState为SHUTDOWN或者RUNNING，则从任务缓存队列取任务。

   　　如果当前线程池的线程数大于核心池大小corePoolSize或者允许为核心池中的线程设置空闲存活时间，则调用poll(time,timeUnit)来取任务，这个方法会等待一定的时间，如果取不到任务就返回null。

   ​		然后判断取到的任务r是否为null，为null则通过调用workerCanExit()方法来判断当前worker是否可以退出，我们看一下workerCanExit()的实现：

   ```java
   private boolean workerCanExit() {
       final ReentrantLock mainLock = this.mainLock;
       mainLock.lock();
       boolean canExit;
       //如果runState大于等于STOP，或者任务缓存队列为空了
       //或者  允许为核心池线程设置空闲存活时间并且线程池中的线程数目大于1
       try {
           canExit = runState >= STOP ||
               workQueue.isEmpty() ||
               (allowCoreThreadTimeOut &&
                poolSize > Math.max(1, corePoolSize));
       } finally {
           mainLock.unlock();
       }
       return canExit;
   }
   ```

   ​	如果允许worker退出，则调用interruptIdleWorkers()中断处于空闲状态的worker，我们看一下interruptIdleWorkers()的实现：

   ```java
   void interruptIdleWorkers() {
       final ReentrantLock mainLock = this.mainLock;
       mainLock.lock();
       try {
           for (Worker w : workers)  //实际上调用的是worker的interruptIfIdle()方法
               w.interruptIfIdle();
       } finally {
           mainLock.unlock();
       }
   }
   ```

   ```java
   void interruptIfIdle() {
       final ReentrantLock runLock = this.runLock;
       if (runLock.tryLock()) {    //注意这里，是调用tryLock()来获取锁的，因为如果当前worker正在执行任务，锁已经被获取了，是无法获取到锁的
                                   //如果成功获取了锁，说明当前worker处于空闲状态
           try {
       if (thread != Thread.currentThread())  
       thread.interrupt();
           } finally {
               runLock.unlock();
           }
       }
   ```

   ​		这里有一个非常巧妙的设计方式，假如我们来设计线程池，可能会有一个任务分派线程，当发现有线程空闲时，就从任务缓存队列中取一个任务交给空闲线程执行。但是在这里，并没有采用这样的方式，因为这样会要额外地对任务分派线程进行管理，无形地会增加难度和复杂度，**这里直接让执行完任务的线程去任务缓存队列里面取任务来执行。**

   **总结一下：**

   　	1）首先，要清楚corePoolSize和maximumPoolSize的含义；

   　　2）其次，要知道Worker是用来起到什么作用的；

   　　3）要知道任务提交给线程池之后的处理策略，这里总结一下主要有4点：

   - 如果当前线程池中的线程数目小于corePoolSize，则每来一个任务，就会创建一个线程去执行这个任务；
   - 如果当前线程池中的线程数目>=corePoolSize，则每来一个任务，会尝试将其添加到任务缓存队列当中，若添加成功，则该任务会等待空闲线程将其取出去执行；若添加失败（一般来说是任务缓存队列已满），则会尝试创建新的线程去执行这个任务；
   - 如果当前线程池中的线程数目达到maximumPoolSize，则会采取任务拒绝策略进行处理；
   - 如果线程池中的线程数量大于 corePoolSize时，如果某线程空闲时间超过keepAliveTime，线程将被终止，直至线程池中的线程数目不大于corePoolSize；如果允许为核心池中的线程设置存活时间，那么核心池中的线程空闲时间超过keepAliveTime，线程也会被终止。

   ![img](https://img-blog.csdnimg.cn/20200614223220216.png)

3. #### 线程池中线程的初始化

   默认情况下，创建线程池之后，线程池中是没有线程的，需要提交任务之后才会创建线程。

   　　在实际中如果需要线程池创建之后立即创建线程，可以通过以下两个方法办到：

   - prestartCoreThread()：初始化一个核心线程；
   - prestartAllCoreThreads()：初始化所有核心线程

   　　下面是这2个方法的实现：

   ```java
   public boolean prestartCoreThread() {
       return addIfUnderCorePoolSize(null); //注意传进去的参数是null
   }
    
   public int prestartAllCoreThreads() {
       int n = 0;
       while (addIfUnderCorePoolSize(null))//注意传进去的参数是null
           ++n;
       return n;
   }
   ```

   　注意上面传进去的参数是null，根据第2小节的分析可知如果传进去的参数为null，则最后执行线程会阻塞在getTask方法中的

   ```java
   r = workQueue.take();  //等待任务缓存等列中有任务
   ```

   

4. #### 任务缓存队列及排队策略

   在前面我们多次提到了任务缓存队列，即workQueue，它用来存放等待执行的任务。

   　　workQueue的类型为BlockingQueue<Runnable>，通常可以取下面三种类型：

   　　1）ArrayBlockingQueue：基于数组的先进先出队列，此队列创建时必须指定大小；

   　　2）LinkedBlockingQueue：基于链表的先进先出队列，如果创建时没有指定此队列大小，则默认为Integer.MAX_VALUE；

   　　3）synchronousQueue：这个队列比较特殊，它不会保存提交的任务，而是将直接新建一个线程来执行新来的任务。

5. #### 任务的拒绝策略

   当线程池的任务缓存队列已满并且线程池中的线程数目达到maximumPoolSize，如果还有任务到来就会采取任务拒绝策略，通常有以下四种策略：

   ```java
   ThreadPoolExecutor.AbortPolicy:丢弃任务并抛出RejectedExecutionException异常。
   ThreadPoolExecutor.DiscardPolicy：也是丢弃任务，但是不抛出异常。
   ThreadPoolExecutor.DiscardOldestPolicy：丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）
   ThreadPoolExecutor.CallerRunsPolicy：由调用线程处理该任务
   ```

   

6. #### 线程池的关闭

   　　ThreadPoolExecutor提供了两个方法，用于线程池的关闭，分别是shutdown()和shutdownNow()，其中：

   - shutdown()：不会立即终止线程池，而是要等所有任务缓存队列中的任务都执行完后才终止，但再也不会接受新的任务
   - shutdownNow()：立即终止线程池，并尝试打断正在执行的任务，并且清空任务缓存队列，返回尚未执行的任务

7. #### 线程池容量的动态调整

　   ThreadPoolExecutor提供了动态调整线程池容量大小的方法：setCorePoolSize()和setMaximumPoolSize()，

- setCorePoolSize：设置核心池大小
- setMaximumPoolSize：设置线程池最大能创建的线程数目大小

　　当上述参数从小变大时，ThreadPoolExecutor进行线程赋值，还可能立即创建新的线程来执行任务。



### 使用实例

```java
public class Test {
     public static void main(String[] args) {   
         ThreadPoolExecutor executor = new ThreadPoolExecutor(5, 10, 200, TimeUnit.MILLISECONDS,
                 new ArrayBlockingQueue<Runnable>(5));
          
         for(int i=0;i<15;i++){
             MyTask myTask = new MyTask(i);
             executor.execute(myTask);
             System.out.println("线程池中线程数目："+executor.getPoolSize()+"，队列中等待执行的任务数目："+
             executor.getQueue().size()+"，已执行玩别的任务数目："+executor.getCompletedTaskCount());
         }
         executor.shutdown();
     }
}
 
 
class MyTask implements Runnable {
    private int taskNum;
     
    public MyTask(int num) {
        this.taskNum = num;
    }
     
    @Override
    public void run() {
        System.out.println("正在执行task "+taskNum);
        try {
            Thread.currentThread().sleep(4000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("task "+taskNum+"执行完毕");
    }
}
```

运行结果：

```
正在执行task 0
线程池中线程数目：1，队列中等待执行的任务数目：0，已执行玩别的任务数目：0
线程池中线程数目：2，队列中等待执行的任务数目：0，已执行玩别的任务数目：0
正在执行task 1
线程池中线程数目：3，队列中等待执行的任务数目：0，已执行玩别的任务数目：0
正在执行task 2
线程池中线程数目：4，队列中等待执行的任务数目：0，已执行玩别的任务数目：0
正在执行task 3
线程池中线程数目：5，队列中等待执行的任务数目：0，已执行玩别的任务数目：0
正在执行task 4
线程池中线程数目：5，队列中等待执行的任务数目：1，已执行玩别的任务数目：0
线程池中线程数目：5，队列中等待执行的任务数目：2，已执行玩别的任务数目：0
线程池中线程数目：5，队列中等待执行的任务数目：3，已执行玩别的任务数目：0
线程池中线程数目：5，队列中等待执行的任务数目：4，已执行玩别的任务数目：0
线程池中线程数目：5，队列中等待执行的任务数目：5，已执行玩别的任务数目：0
线程池中线程数目：6，队列中等待执行的任务数目：5，已执行玩别的任务数目：0
正在执行task 10
线程池中线程数目：7，队列中等待执行的任务数目：5，已执行玩别的任务数目：0
正在执行task 11
线程池中线程数目：8，队列中等待执行的任务数目：5，已执行玩别的任务数目：0
正在执行task 12
线程池中线程数目：9，队列中等待执行的任务数目：5，已执行玩别的任务数目：0
正在执行task 13
线程池中线程数目：10，队列中等待执行的任务数目：5，已执行玩别的任务数目：0
正在执行task 14
task 3执行完毕
task 0执行完毕
task 2执行完毕
task 1执行完毕
正在执行task 8
正在执行task 7
正在执行task 6
正在执行task 5
task 4执行完毕
task 10执行完毕
task 11执行完毕
task 13执行完毕
task 12执行完毕
正在执行task 9
task 14执行完毕
task 8执行完毕
task 5执行完毕
task 7执行完毕
task 6执行完毕
task 9执行完毕
```

​		从执行结果可以看出，当线程池中线程的数目大于5时，便将任务放入任务缓存队列里面，当任务缓存队列满了之后，便创建新的线程。如果上面程序中，将for循环中改成执行20个任务，就会抛出任务拒绝异常了。

​		除了使用ThreadPoolExecutor的构造函数创建线程池，还可以使用Executors类中的几个静态方法创建线程池：

```java
Executors.newCachedThreadPool();        //创建一个缓冲池，缓冲池容量大小为Integer.MAX_VALUE
Executors.newSingleThreadExecutor();   //创建容量为1的缓冲池
Executors.newFixedThreadPool(int);    //创建固定容量大小的缓冲池
```

具体实现如下：

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```

　　从它们的具体实现来看，它们实际上也是调用了ThreadPoolExecutor，只不过参数都已配置好了。

​		**newFixedThreadPool**创建的线程池corePoolSize和maximumPoolSize值是相等的，它使用的LinkedBlockingQueue；

　　**newSingleThreadExecutor**将corePoolSize和maximumPoolSize都设置为1，也使用的LinkedBlockingQueue；

　　**newCachedThreadPool**将corePoolSize设置为0，将maximumPoolSize设置为Integer.MAX_VALUE，使用的SynchronousQueue，也就是说来了任务就创建线程运行，当线程空闲超过60秒，就销毁线程。

　　实际中，如果Executors提供的三个静态方法能满足要求，就尽量使用它提供的三个方法，因为自己去手动配置ThreadPoolExecutor的参数有点麻烦，要根据实际任务的类型和数量来进行配置。

　　另外，如果ThreadPoolExecutor达不到要求，可以自己继承ThreadPoolExecutor类进行重写。

### 合理配置线程池大小

​		一般需要根据任务的类型来配置线程池大小：

　　如果是CPU密集型任务，就需要尽量压榨CPU，参考值可以设为 *N*CPU(CPU的核数)+1

　　如果是IO密集型任务，参考值可以设置为2**N*CPU

　　当然，这只是一个参考值，具体的设置还需要根据实际情况进行调整，比如可以先将线程池大小设置为参考值，再观察任务运行情况和系统负载、资源利用率来进行适当调整。

### 面试高频问题

#### 线程池异常处理

![img](https://picb.zhimg.com/80/v2-4a46342775c6f9ad8543654890833b6b_720w.jpg)

#### 线程池有哪几种工作队列？

- ArrayBlockingQueue

ArrayBlockingQueue（有界队列）是一个用数组实现的有界阻塞队列，按FIFO排序量。

- LinkedBlockingQueue

LinkedBlockingQueue（可设置容量队列）基于链表结构的阻塞队列，按FIFO排序任务，容量可以选择进行设置，不设置的话，将是一个无边界的阻塞队列，最大长度为Integer.MAX_VALUE，吞吐量通常要高于ArrayBlockingQueue；newFixedThreadPool线程池使用了这个队列

- DelayQueue

DelayQueue（延迟队列）是一个任务定时周期的延迟执行的队列。根据指定的执行时间从小到大排序，否则根据插入到队列的先后排序。newScheduledThreadPool线程池使用了这个队列。

- PriorityBlockingQueue

PriorityBlockingQueue（优先级队列）是具有优先级的无界阻塞队列；

- SynchronousQueue

SynchronousQueue（同步队列）一个不存储元素的阻塞队列，每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于阻塞状态，吞吐量通常要高于LinkedBlockingQuene，newCachedThreadPool线程池使用了这个队列。

#### 几种常用的线程池

- newFixedThreadPool (固定数目线程的线程池) : 

  keepAliveTime为0

  ```java
   public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
          return new ThreadPoolExecutor(nThreads, nThreads,
                                        0L, TimeUnit.MILLISECONDS,
                                        new LinkedBlockingQueue<Runnable>(),
                                        threadFactory);
      }
  ```

  **使用场景：**

  FixedThreadPool 适用于处理CPU密集型的任务，确保CPU在长期被工作线程使用的情况下，尽可能的少的分配线程，即适用执行长期的任务。

  

- newCachedThreadPool(可缓存线程的线程池)

  corePoolSize：0

  MaximumPoolSize：Integer.MAX_VALUE

  缓存队列：SynchronousQueue

  KeepAliveTime:60秒

  

  **使用场景**

  用于并发执行大量短期的小任务

  

- newSingleThreadExecutor(单线程的线程池)

  corePoolSize：1

  MaximumPoolSize：1

  缓存队列：LinkedBlockingQueue

  KeepAliveTime:0

  **使用场景**

  串行执行任务

  

- newScheduledThreadPool(定时及周期执行的线程池)

  corePoolSize：自己传入的一个参数

  MaximumPoolSize：Integer.MAX_VALUE

  缓存队列：DelayedWorkQueue

  KeepAliveTime:0

  **使用场景**

  周期性执行任务，需要限制线程数量



#### 使用无界缓存队列的线程池会导致内存飙升吗？

**会的，newFixedThreadPool使用了无界的阻塞队列LinkedBlockingQueue，如果线程获取一个任务后，任务的执行时间比较长(比如，上面demo设置了10秒)，会导致队列的任务越积越多，导致机器内存使用不停飙升，** 最终导致OOM。

