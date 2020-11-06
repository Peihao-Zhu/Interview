## BIO

同步阻塞IO，数据的读取写入必须阻塞在一个线程内等待其完成

#### 1.1 传统BIO

一请求一应答模型

服务端由一个Acceptor线程负责监听客户端的连接。通过在while(true)循环中服务端调用`accept()`方法等待接受客户端的连接请求，请求一旦被接受，就建立通信套接字在这个通信套接字上进行读写操作，此时不能再接受其他客户端请求，只能等待当前连接的客户端操作执行完成。

不过可以通过多线程来支持多个客户端的连接，（因为`socket.accept()`、`socket.read()`、`socket.write()` 都是同步阻塞的），如果这个连接不做任何事情，就会造成不必要的线程开销，可以通过**线程池**，降低线程的创建和回收成本。这也就是接下来要讲的伪异步IO

#### 1.2 伪异步IO

后端通过线程池处理多个客户端连接，实现N(客户端数量):M(处理客户端请求的线程数量)，其中N可以远大于M

![img](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20201103183053.png)

将客户端的Socket封装成Task，然后放入线程池里，根据当前线程池中的人物数量以及线程数量， 把任务放到等待队列中。采用这个框架，可以避免为每个请求都创建一个独立线程造成线程资源耗尽问题。

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

通道只能和Buffer交互，因为有Buffer，**通道可以异步的读写**。

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

