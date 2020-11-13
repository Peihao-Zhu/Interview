## BIO

同步阻塞IO，数据的读取写入必须阻塞在一个线程内等待其完成

#### 1.1 传统BIO

一请求一应答模型

服务端由一个Acceptor线程负责监听客户端的连接。通过在while(true)循环中服务端调用`accept()`方法等待接受客户端的连接请求，请求一旦被接受，就建立通信套接字在这个通信套接字上进行读写操作，此时不能再接受其他客户端请求，只能等待当前连接的客户端操作执行完成。

不过可以通过多线程来支持多个客户端的连接，（因为`socket.accept()`、`socket.read()`、`socket.write()` 都是同步阻塞的），如果这个连接不做任何事情，就会造成不必要的线程开销，可以通过**线程池**，降低线程的创建和回收成本。这也就是接下来要讲的伪异步IO

#### 1.2 伪异步IO

后端通过线程池处理多个客户端连接，实现N(客户端数量):M(处理客户端请求的线程数量)，其中N可以远大于M

![img](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20201103183053.png)

将客户端的Socket封装成Task，然后放入线程池里，根据当前线程池中的任务数量以及线程数量， 把任务放到等待队列中。采用这个框架，可以避免为每个请求都创建一个独立线程造成线程资源耗尽问题。

## NIO

是一种同步非阻塞IO模型，NIO提供了与传统BIO的`Socket`和 `ServerSoket`对应的`SocketChannel`和 `ServerSocketChannel`这两种通道都支持阻塞和非阻塞模式。对于低负载、低并发的应用程序，可以使用同步阻塞IO提升开发速率；对于高负载、高并发的应用，可以使用NIO的非阻塞模式。

**流与缓冲区**

I/O 与 NIO 最重要的区别是数据打包和传输的方式，I/O 以流的方式处理数据，而 NIO 以块的方式处理数据。

面向流的 I/O 一次处理一个字节数据：一个输入流产生一个字节数据，一个输出流消费一个字节数据。将数据直接写入或者数据直接读取到Stream对象中，虽然Stream也有Buffer开头的扩展类，但只是流的包装，还是从流读到缓冲区。

面向块的 I/O 一次处理一个缓冲块，按块处理数据比按流处理数据要快得多。但是面向块的 I/O 缺少一些面向流的 I/O 所具有的优雅性和简单性。

最常用的缓冲区是ByteBuffer，一个ByteBuffer提供了一组功能用于操作Byte数组。每一种Java基本类型(除了Boolean)都对应一种缓冲区

**阻塞和非阻塞IO**

- Java IO的各种流是阻塞的。这意味着，当一个线程调用read() 或 write()时，该线程被阻塞，直到有一些数据被读取，或数据完全写入。该线程在此期间不能再干任何事情了。 

- Java NIO的非阻塞模式，使一个线程从某通道发送请求读取数据，但是它仅能得到目前可用的数据，如果目前没有数据可用时，就什么都不会获取。而不是保持线程阻塞，所以直至数据变的可以读取之前，该线程可以继续做其他的事情。 非阻塞写也是如此。一个线程请求写入一些数据到某通道，但不需要等待它完全写入，这个线程同时可以去做别的事情。 线程通常将非阻塞IO的空闲时间用于在其它通道上执行IO操作，所以一个单独的线程现在可以管理多个输入和输出通道（channel）。



**通道**

通道 Channel 是对原 I/O 包中的流的模拟，可以通过它读取和写入数据。

通道与流的不同之处在于，流只能在一个方向上移动(一个流必须是 InputStream 或者 OutputStream 的子类)，而**通道是双向**的，可以用于读、写或者同时用于读写。

通道只能和Buffer交互，因为有Buffer，**通道可以异步的读写**（如果没有Buffer，那写数据到通道的时候，肯定不能读了，读数据的时候，也不能对通道进行写，要保证同步。但是有了Buffer以后，有读Buffer和写Buffer，上述的两个操作可以异步进行）。

通道包括以下类型：

- FileChannel：从文件中读写数据；
- DatagramChannel：通过 UDP 读写网络中数据；
- SocketChannel：通过 TCP 读写网络中数据；
- ServerSocketChannel：可以监听新进来的 TCP 连接，对每一个新进来的连接都会创建一个 SocketChannel。

**缓冲区**

发送给一个通道的所有数据都必须首先放到缓冲区中，同样地，从通道中读取的任何数据都要先读到缓冲区中。也就是说，不会直接对通道进行读写数据，而是要先经过缓冲区。

缓冲区实质上是一个数组，但它不仅仅是一个数组。缓冲区提供了对数据的结构化访问，而且还可以跟踪系统的读/写进程。

缓冲区包括以下类型：

- ByteBuffer
- CharBuffer
- ShortBuffer
- IntBuffer
- LongBuffer
- FloatBuffer
- DoubleBuffer



**缓冲区状态变量**

- capacity：最大容量；如果缓冲区满了，需要读数据，或者清除数据，然后在往里写数据
- position：当前已经读写的字节数；
- limit：还可以读写的字节数。在写模式下limit=capacity，在读模式下limit=position

状态变量的改变过程举例：

① 新建一个大小为 8 个字节的缓冲区，此时 position 为 0，而 limit = capacity = 8。capacity 变量不会改变，下面的讨论会忽略它。

![img](https://cs-notes-1256109796.cos.ap-guangzhou.myqcloud.com/1bea398f-17a7-4f67-a90b-9e2d243eaa9a.png)



② 从输入通道中读取 5 个字节数据写入缓冲区中，此时 position 为 5，limit 保持不变。

![img](https://cs-notes-1256109796.cos.ap-guangzhou.myqcloud.com/80804f52-8815-4096-b506-48eef3eed5c6.png)



③ 在将缓冲区的数据写到输出通道之前，需要先调用 flip() 方法，这个方法将 limit 设置为当前 position，并将 position 设置为 0。

![img](https://cs-notes-1256109796.cos.ap-guangzhou.myqcloud.com/952e06bd-5a65-4cab-82e4-dd1536462f38.png)



④ 从缓冲区中取 4 个字节到输出缓冲中，此时 position 设为 4。

![img](https://cs-notes-1256109796.cos.ap-guangzhou.myqcloud.com/b5bdcbe2-b958-4aef-9151-6ad963cb28b4.png)



⑤ 最后需要调用 clear() 方法来清空缓冲区，此时 position 和 limit 都被设置为最初位置（里面的数据没有清楚，只是这几个变量的值修改了）。

![img](https://cs-notes-1256109796.cos.ap-guangzhou.myqcloud.com/67bf5487-c45d-49b6-b9c0-a058d8c68902.png)

**选择器**

NIO 常常被叫做非阻塞 IO，主要是因为 NIO 在网络通信中的非阻塞特性被广泛使用。

NIO 实现了 IO 多路复用中的 Reactor 模型，一个线程 Thread 使用一个选择器 Selector 通过轮询的方式去监听多个通道 Channel 上的事件，从而让一个线程就可以处理多个事件。

通过配置监听的通道 Channel 为非阻塞，那么当 Channel 上的 IO 事件还未到达时，就不会进入阻塞状态一直等待，而是继续轮询其它 Channel，找到 IO 事件已经到达的 Channel 执行。

因为创建和切换线程的开销很大，因此使用一个线程来处理多个事件而不是一个线程处理一个事件，对于 IO 密集型的应用具有很好地性能。

应该注意的是，只有套接字 Channel 才能配置为非阻塞，而 FileChannel 不能，为 FileChannel 配置非阻塞也没有意义。

![img](https://cs-notes-1256109796.cos.ap-guangzhou.myqcloud.com/093f9e57-429c-413a-83ee-c689ba596cef.png)

```java
//创建选择器
Selector selector = Selector.open();
//将通道注册到选择器上
ServerSocketChannel ssChannel = ServerSocketChannel.open();
ssChannel.configureBlocking(false);  //可以进行异步调用
ssChannel.register(selector, SelectionKey.OP_ACCEPT);
while(true){
  //监听事件
  int num = selector.select();
  //获取到达的事件
  Set<SelectionKey> keys = selector.selectedKeys();
  Iterator<SelectionKey> keyIterator = keys.iterator();
  while (keyIterator.hasNext()) {
      SelectionKey key = keyIterator.next();
      if (key.isAcceptable()) {
          // ...
      } else if (key.isReadable()) {
          // ...
      }
      keyIterator.remove();
  }
}
```

通道必须配置为非阻塞模式，否则使用选择器就没有任何意义了，因为如果通道在某个事件上被阻塞，那么服务器就不能响应其它事件，必须等待这个事件处理完毕才能去处理其它事件，显然这和选择器的作用背道而驰。

在将通道注册到选择器上时，还需要指定要注册的具体事件，主要有以下几类：

- SelectionKey.OP_CONNECT
- SelectionKey.OP_ACCEPT
- SelectionKey.OP_READ
- SelectionKey.OP_WRITE

## AIO

AIO也就是NIO2 是异步非阻塞的IO模型。异步IO是基于事件和回调机制实现的，也就是应用操作之后会返回，不会堵塞在那里。虽然NIO在网络操作中提供了但是NIO的IO还是同步的

# 3 Socket

## 3.1 ★★☆ 五种 IO 模型的特点以及比较。

相关代码参考：

https://github.com/caijinlin/learning-pratice/tree/master/linux/io

可以参考另一篇文章

https://mp.weixin.qq.com/s?__biz=MzUxMzQ0Njc1NQ==&mid=2247486581&idx=1&sn=390f6c6b7dd647e334b34f8f51008cab&chksm=f9544a79ce23c36fcd45d9de55e3652d14ad3bb4d45460a7ece5b916e8b46da79ad37c5e2cbe&mpshare=1&scene=23&srcid=07163oS95nXAnqTIEJJ58rVS&sharer_sharetime=1594901798946&sharer_shareid=0b5259c676795139071f4f7ade041a11%23rd



输入操作包括两个阶段：

1.等待数据准备好（等待数据从网络到达）

2.从内核向进程复制数据（从内核缓冲区复制到应用进程缓冲区）

 

Unix五种I/O模型

- 同步阻塞式I/O

应用进程被阻塞，直到数据从内核缓冲区复制到应用进程缓冲区才返回。

在阻塞过程中，其他应用进程还可以执行，不消耗CPU时间，这种模型的CPU利用率会比较高。

**recvfrom()用于接受Socket传来的数据，并复制到应用进程的缓冲区**

![img](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20200806135239.png)

如果服务端采用单线程，那么这个进程被阻塞后，进程后续的请求都无法被执行，无法处理并发。

如果服务端采用多线程，每次来一个请求都可以开启一个线程来处理，这样可以处理并发， **但是大量线程占用很大的内存，并且线程的切换会带来很大的开销。10000个线程真正发生读写事件的线程数不会超过20%，每次accept都开一个线程也是一种资源浪费**

 

- 同步非阻塞式I/O

应用进程执行一个系统调用后，内核返回一个错误码。应用进程可以继续执行，但是需要不断执行系统调用来获知I/O是否完成，称为轮询。服务器端当accept一个请求后，加入fds集合，每次轮询一遍fds集合recv(非阻塞)数据，没有数据则立即返回错误，`每次轮询所有fd（包括没有发生读写事件的fd）会很浪费cpu`

![img](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20200806135244.png) 

- I/O复用（select和poll）

使用select或者poll等待数据，多个进程的IO可以注册到同一个select函数上，并且可以等待多个套接字的任何一个变为可读。这个过程会被阻塞（因为没有向内核注册信号处理函数），当某个套接字可读时返回，在使用recvfrom把数据从内核复制到进程中。

它可以让单个进程具有处理多个I/O事件的能力。

![img](https://cs-notes-1256109796.cos.ap-guangzhou.myqcloud.com/1492929444818_6.png)

- 信号驱动式I/O(SIGIO)

应用进程使用sigaction系统调用，向内核注册一个信号处理函数，内核立即返回，应用进程可以继续执行，也就是说等待数据阶段进程是非阻塞的。内核在数据到达时向应用进程发送SIGIO信号，应用进程收到后调用recvfrom将数据从内核复制到应用进程。

![img](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20200806135247.png) 

- 异步I/O(AIIO)

应用进程执行aio_read系统调用会立即返回，应用进程可以继续执行，不会被阻塞，内核在所有操作完成后向应用进程发送信号。

![img](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20200806135252.png) 

**五种I/O的比较**

- 同步I/O：将数据从内核缓冲区复制到应用进程缓冲区是同步的，会阻塞（第二阶段）
- 异步I/O；第二阶段不会阻塞

 

同步I/O包括阻塞式I/O、非阻塞式I/O、I/O复用和信号驱动I/O，他们之间主要区别在第一阶段，

非阻塞I/O、信号驱动I/O和异步I/O第一阶段不会阻塞。

 

![img](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20200806135258.png)





#### 有哪些常见的IO模型？

- 同步阻塞IO（Blocking IO）：用户线程发起IO读/写操作之后，线程阻塞，直到可以开始处理数据；对CPU资源的利用率不够；
- 同步非阻塞IO（Non-blocking IO）：发起IO请求之后可以立即返回，如果没有就绪的数据，需要不断地发起IO请求直到数据就绪；不断重复请求消耗了大量的CPU资源；
- IO多路复用
- 异步IO（Asynchronous IO）：用户线程发出IO请求之后，继续执行，由内核进行数据的读取并放在用户指定的缓冲区内，在IO完成之后通知用户线程直接使用。



## 3.2 ★★★ select、poll、epoll 的原理、比较、以及使用场景；epoll 的水平触发与边缘触发。

IO多路复用（IO Multiplexing）是指单个进程/线程就可以同时处理多个IO请求。

实现原理：用户将想要监视的文件描述符（File Descriptor）添加到select/poll/epoll函数中，由内核监视，函数阻塞。一旦有文件描述符就绪（读就绪或写就绪），或者超时（设置timeout），函数就会返回，然后该进程可以进行相应的读/写操作。

<details>
<summary>select/poll/epoll三者的区别？</summary>



- ```select```：将文件描述符放入一个集合中，调用select时，将这个集合从用户空间拷贝到内核空间

  （缺点1：每次都要复制，**开销大**），由内核根据就绪状态修改该集合的内容。

  （缺点2）**集合大小有限制**，32位机默认是1024（64位：2048）；采用水平触发机制。select函数返回后，需要通过遍历这个集合，找到就绪的文件描述符

  （缺点3：**轮询的方式效率较低**），当文件描述符的数量增加时，效率会线性下降；

- ```poll```：和select几乎没有区别，区别在于文件描述符的存储方式不同，poll采用链表的方式存储，没有最大存储数量的限制；

- ```epoll```：通过内核和用户空间共享内存，避免了不断复制的问题；支持的同时连接数上限很高（1G左右的内存支持10W左右的连接数）；epoll_ctl() 用于向内核注册新的描述符或者是改变某个文件描述符的状态。已注册的描述符在内核中会被维护在一棵红黑树上；文件描述符就绪时，采用回调机制，避免了轮询（回调函数将就绪的描述符添加到一个链表中，执行epoll_wait时，返回这个链表）；支持水平触发和边缘触发，采用边缘触发机制时，只有活跃的描述符才会触发回调函数。

  （缺点：只能工作在Linux下）

#### select/poll/epoll的区别



|            | select                      | poll             | epoll                                             |
| ---------- | --------------------------- | ---------------- | ------------------------------------------------- |
| 数据结构   | bitmap                      | 数组             | 红黑树                                            |
| 最大连接数 | 1024(32位机)/2048（64位机） | 无上限           | 无上限                                            |
| fd拷贝     | 每次调用select拷贝          | 每次调用poll拷贝 | fd首次调用epoll_ctl拷贝，每次调用epoll_wait不拷贝 |
| 工作效率   | 轮询：O(n)                  | 轮询：O(n)       | 回调：O(1)                                        |



#### 什么时候使用select/poll，什么时候使用epoll？

Epoll:当连接数较多并且有很多的不活跃连接时.但是当连接数较少并且都十分活跃的情况下，由于epoll需要很多回调，因此性能可能低于其它两者。nginx/redis都使用epoll

Select: select 的 timeout 参数精度为微秒，而 poll 和 epoll 为毫秒，因此 select 更加适用于实时性要求比较高的场景，比如核反应堆的控制。

Poll:



#### 什么是文件描述符？

文件描述符在形式上是一个非负整数。实际上，它是一个索引值，用以标明每一个被进程所打开的文件和socket。第一个打开的文件是0，第二个是1，依此类推。一个现有文件或者创建一个新文件时，内核向进程返回一个文件描述符。

**内核通过文件描述符来访问文件。文件描述符指向一个文件。**



#### 什么是水平触发？什么是边缘触发？

https://blog.csdn.net/lihao21/article/details/67631516

- 水平触发（LT，Level Trigger）模式下，只要还有FD就绪,每次调用epoll_wait()都会通知用户进程去处理对应的FD。是默认的一种模式，支持阻塞和非阻塞；

- 边缘触发（ET，Edge Trigger）模式下，当FD从未就绪变为就绪时通知一次，之后不会再通知，直到又有FD从未就绪变为就绪（缓冲区从不可读/写变为可读/写），所以每次read一个fd的时候要把他的buffer读完，仅支持非阻塞。

  

  **水平触发**

  1. 对于读操作
     只要缓冲内容不为空，LT模式返回读就绪。

  2. 对于写操作
     只要缓冲区还不满，LT模式会返回写就绪。

  **边缘触发**

  1. 对于读操作

    （1）当缓冲区由不可读变为可读的时候，即缓冲区由空变为不空的时候。

  （2）当有新数据到达时，即缓冲区中的待读数据变多的时候。

  （3）当缓冲区有数据可读，且应用进程对相应的描述符进行EPOLL_CTL_MOD 修改EPOLLIN事件时。

  2. 对于写操作
     （1）当缓冲区由不可写变为可写时。

  （2）当有旧数据被发送走，即缓冲区中的内容变少的时候。

    （3）当缓冲区有空间可写，且应用进程对相应的描述符进行EPOLL_CTL_MOD 修改EPOLLOUT事件时。

  **区别：**边缘触发效率更高，减少了被重复触发的次数，函数不会返回大量用户程序可能不需要的文件描述符。

  

  **为什么边缘触发一定要用非阻塞（non-block）IO：**因为边缘触发当描述符就绪时就会立即处理，避免由于一个描述符的阻塞读/阻塞写操作让处理其它描述符的任务出现饥饿状态（如果短时间内很多描述符都就绪了，如果采用阻塞IO的话就会）。



## 3.3 ★★★ Reactor模型和Proactor模型

https://www.jianshu.com/p/5fe6c59e5c00

### Reactor

PPC和TPC模式无法支持高并发，每来一个连接就创建一个进程或线程，连接结束就销毁，浪费太大。可以实现资源的复用，创建线程池，一个线程处理多个连接业务。

引入**资源池**的处理方式后，会引出一个新的问题：进程如何才能高效地处理多个连接的业务？当**一个连接一个进程**时，进程可以采用“**read -> 业务处理 -> write”的处理流程**，如果当前连接没有数据可以读，则进程就**阻塞在 read 操作上**。这种阻塞的方式在一个连接一个进程的场景下没有问题，但如果一个进程处理多个连接，进程阻塞在某个连接的 read 操作上，此时即使其他连接有数据可读，进程也无法去处理，很显然这样是**无法做到高性能**的。

最简单的方式就是将 **read操作改成非阻塞，然后进程不断轮询多个连接**，但是**轮询是要消耗 CPU** 的；其次，如果一个进程处理几千上万的连接，则**轮询的效率是很低**的。

为了能够更好地解决上述问题，很容易可以想到，只有当**连接上有数据**的时候进**程才去处理**，这就是 **I/O 多路复用技术**的来源。

当某条连接有新的数据可以处理时，操作系统通知进程，进程从阻塞状态返回，开始进行业务处理。简单点说就是IO多路复用统一监听事件、收到事件后分配给某个进程来处理。核心组件：Reactor和处理资源池（**进程池或线程池**）实际上Reactor模式可以灵活多变，根据Reactor和资源池中线程的数量可以分为：

**单 Reactor 单进程 / 线程。**

**单 Reactor 多线程。**

**多 Reactor 多进程 / 线程。**



##### 单 Reactor 单进程 / 线程。

![WeChat908beb836cec9be8bcf63ca6383d791a](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20200806135613.png)

1）Reactor 对象通过 select 监控连接事件，收到事件后通过 **dispatch 进行分发**。

2）如果是**连接建立的事件**，则由 Acceptor 处理，**Acceptor 通过 accept 接受连接，并创建一个 Handler 来处理连接后续的各种事件**。

如果不是连接建立事件，则 **Reactor** 会**调用**连接对应的 **Handler**（第 2 步中创建的 Handler）来进行响应。

3）Handler 会完成 **read**-> 业务处理 ->send 的完整业务流程。

Handler 在处理某个连接上的业务时，整个进程无法处理其他连接的事件，很容易导致性能瓶颈。

因此，单 Reactor 单进程的方案在实践中应用场景不多，**只适用于业务处理非常快速的场景**，目前比较著名的开源软件中使用单 **Reactor 单进程的是 Redis**。



##### 单 Reactor 多线程。

![WeChateac4ba648d26377edc1d2b33d92cde60](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20200806135725.png)

与单线程不同的地方在于，Handler只负责的响应事件，不负责业务处理。业务处理交给子线程的Processor来处理。

 Reator 多线程方案能够充分利用**多核多 CPU** 的处理能力。但是**Reactor** 承担所有事件的**监听和响应**，只在主线程中运行，瞬间**高并发时会成为性能瓶颈**；多线程数据共享和访问比较复杂。



##### 多 Reactor 多进程 / 线程。

![WeChat4fbd25d88e335e30b8b5d2f5da573404](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20200806135759.png)

父进程中 mainReactor 对象通过 select 监控连接建立事件，收到事件后通过 Acceptor 接收，将新的连接分配给某个子进程。

子进程的 subReactor 将 mainReactor 分配的连接加入连接队列进行监听，并创建一个 Handler 用于处理连接的各种事件。

当有新的事件发生时，subReactor 会调用连接对应的 Handler（即第 2 步中创建的 Handler）来进行响应。

Handler 完成 read→业务处理→send 的完整业务流程。

父进程和子进程的职责非常明确，**父进程只负责接收新连接，子进程负责完成后续的业务处理**。

父进程和子进程的**交互很简单**，父进程只需要把新连接传给子进程，子进程无须返回数据。

子进程之间是互相独立的，无须同步共享之类的处理（这里仅限于网络模型相关的 select、read、send 等无须同步共享，“业务处理”还是有可能需要同步共享的）。

目前著名的开源系统 **Nginx** 采用的是**多 Reactor 多进程**，采用多 Reactor 多线程的实现有 **Memcache 和 Netty**。



### Proactor

异步IO，有新的连接，操作系统将数据读写到应用传递进来的缓冲区，然后通知用户进程进行处理



Proactor Initiator 负责创建 Proactor 和 Handler，并将 Proactor 和 Handler 都通过 Asynchronous Operation Processor 注册到内核。

Asynchronous Operation Processor 负责处理注册请求，并完成 I/O 操作。完成 I/O 操作后通知 Proactor。

Proactor 根据不同的事件类型回调不同的 Handler 进行业务处理。

Handler 完成业务处理，Handler 也可以注册新的 Handler 到内核进程。



## 3.4 ★★★ socket位于哪一层

https://blog.csdn.net/jia12216/article/details/82702960

socket编程是应用层开发，利用socket接口去实现自己的业务和协议。在OSI的七层模型中，表示层主要处理应用数据到网络服务的转变包括字符的编码，数据压缩和加密等操作。会话层则主要管理两个节点之间的信息交换，所以socket是会话层的。



