[TOC]



## 1 ★★☆ 字典和跳跃表原理分析。

#### 字典

hashtable：1个dict结构、2个dictht、1个dictEntry指针数组和多个dictEntry构成

![img](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20200806142634.png)

由底层向上介绍各部分：

**dictEntry**

保存具体的键值对

```c
typedef struct dictEntry{
    void *key;
    union{
        void *val;
        uint64_tu64;
        int64_ts64;
    }v;
    struct dictEntry *next;
}dictEntry;
```

- val: value 用union实现，存储的内容既可以是指向值的指针，也可以是64位整型或无符号64位整型。

- next：指向下一个dictEntry，用于解决hash冲突。使用头插法，O（1）时间

  在64位系统中，一个dictEntry对象占24字节（key/value/next各占8字节）

**bucket**

是一个数组，每个元素都是指向dictEntry结构的指针。redis中bucket数组大小计算规则：大于dictEntry的，最小的2^n.比如有1000个dictEntry，那么bucket就是1024

**dictht**

```c
typedef struct dictht{
    dictEntry **table;
    unsigned long size;
    unsigned long sizemask;
    unsigned long used;
}dictht;
```

- table是一个指针，指向bucket
- size记录了哈希表的大小，即bucket大小
- sizemask值总是size-1，这个属性和哈希值一起决定索引值（key在table中存储的位置）-------index=hash&sizemask
- used表示已经使用的dictEntry的数量

**dict**

一般来说 dictht和dictEntry就能实现哈希表功能，但是在Redis中，在dictht上面还有一层dict。

```c
typedef struct dict{
		//类型特定的函数
    dictType *type;
  	//似有数据
    void *privdata;
  	//哈希表
    dictht ht[2];
  	//索引，当不执行rehash时，值为-1
    int rehashidx;
} dict;
```

​		type和privatedata属性是针对不同类型的键值对，为创建多态字典而设置的：

- type属性是一个只想dictType结构的指针，每个dictType结构保存了一组用于操作特定类型键值对的函数，Redis会为用途不同的字典设置不同的类型特定函数。
- privatedata属性保存了需要传给那些类型特定函数的可选参数。

​        ht属性和trehashidx属性用于rehash，即当哈希表需要扩展或收缩时使用。ht是一个包含两个项的数组，每项都指向一个dictht结构，这也是Redis的哈希会有1个dict、2个dictht结构的原因。通常情况下，所有的数据都是存放在dict的ht[0]中，ht[1]只在rehash的时候使用。

**rehash**

1）先为ht[1]分配空间，他的大小由ht[0].used（已使用的数量）和执行的操作决定

- 扩展操作，ht[1]的大小为第一个大于等于ht[0].used*2的2^n
- 收缩操作，ht[1]的大小为第一个大于等于ht[0].used的2^n

2）将ht[0]中的所有键值对rehash到ht[1]中

3）释放ht[0]（因为已经是空表），将ht[1]设置为ht[0]，并在ht[1]新创建一个空白哈希表，为下一次哈希做准备

**哈希表的扩展和收缩**

​		当以下任意一个条件被满足时都会触发哈希表扩展

- 服务器没有执行bgsave或bgrewriteaof命令，并且哈希表的负载因子大于等于1

- 服务器正在执行bgsave或bgrewriteaof命令，并且哈希表的负载因子大于等于5

  负载因子的计算为：哈希表已保存节点数量（ht[0].used）/哈希表大小(ht[0].size)

  为什么执行bgsave或bgrewriteaof命令的时候，执行扩展操作的负载因子变得很大？------因为在执行这两个命令的时候，Redis会创建子进程来执行写时复制操作，为了提高持久化的效率，要避免在这个时期进行哈希表的扩展操作，所以负载因子为5.

  当哈希表的负载因子小于0.1时，程序会对哈希表执行收缩操作。

**渐进rehash**

​		这里rehash并不是一下子完成的，而是渐进方式，避免一次性执行过多rehash给服务器带来过大负担。将rehashidx设为0表示rehash正式开始，渐进rehash通过将ht[0]上table[rehashidx]上的键值对rehash到ht[1]上（具体table的哪个位置需要重新计算），然后rehashidx++.

在 rehash 期间，每次对字典执行添加、删除、查找或者更新操作时，都会执行一次渐进式 rehash。添加操作都会直接将键值对放入到ht[1]中。

采用渐进式 rehash 会导致字典中的数据分散在两个 dictht 上，因此两个表都会进行相关操作。

#### 跳跃表

是zset的底层实现之一，集群节点中用作内部数据结构。

支持平均O(logN),最坏O(N)的节点查找

**跳跃表的实现**

​		由redis.h/zskiplistNode和redis.h/zskiplist两个结构定义，前者用于表示跳跃表节点，后者保存跳跃表节点的相关信息。

![A187B68E9115D7C4BDD78235E1A2AB7F](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20200806142654.jpg)

​		上图最左边是zskiplist结构，包括如下属性：

- header：指向跳跃表的表头节点

- tail：指向表尾节点

- level：记录目前跳跃表内，层数最大的那个节点的层数。（表头节点的层数不算在内）

- length：跳跃表的长度

  zskiplistNode结构是右边四个，包括如下属性：

  - 层（level）：L1代表第一层。每个层有两个属性：前进指针和跨度，前进指针指向表尾方向的其他节点，而跨度记录了前进指针指向的节点和当前节点的距离。其中跨度是用来计算排位（rank）：在查找某个节点时，将沿途访问过的所有层的跨度累计起来
  - 后退指针（backward）：指向当前节点的前一个节点
  - 分值（score）：各个节点1.0，2.0都是对应的分值，double类型，节点按照所保存的分值从小到大排列。
  - 成员对象：o1,o2是节点保存的成员对象，指向一个字符串对象（SDS）

  注意：表头节点和其他节点的构造一样，但是后退指针、分值和成员对象都用不到所以没有显示

跳跃表是基于多指针有序链表实现的，可以看成多个有序链表。

![img](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20200806142708.png)

在查找时，从上层指针开始查找，找到对应的区间之后再到下一层去查找。下图演示了查找 22 的过程。

![img](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20200806142721.png)

与红黑树等平衡树相比，跳跃表具有以下优点：

- 插入速度非常快速，因为不需要进行旋转等操作来维护平衡性；
- 更容易实现；
- 支持无锁操作。

## 2 ★★★ 使用场景。

**计数器**

可以对 String 进行自增自减运算，从而实现计数器功能。

Redis 这种内存型数据库的读写性能非常高，很适合存储频繁读写的计数量。

**缓存**

将热点数据放到内存中，设置内存的最大使用量以及淘汰策略来保证缓存的命中率。

**查找表**

例如 DNS 记录就很适合使用 Redis 进行存储。

查找表和缓存类似，也是利用了 Redis 快速的查找特性。但是查找表的内容不能失效，而缓存的内容可以失效，因为缓存不作为可靠的数据来源。

**消息队列**

List 是一个双向链表，可以通过 lpush 和 rpop 写入和读取消息

不过最好使用 Kafka、RabbitMQ 等消息中间件。

**会话缓存**

可以使用 Redis 来统一存储多台应用服务器的会话信息。

当应用服务器不再存储用户的会话信息，也就不再具有状态，一个用户可以请求任意一个应用服务器，从而更容易实现高可用性以及可伸缩性。

**分布式锁实现**

在分布式场景下，无法使用单机环境下的锁来对多个节点上的进程进行同步。

可以使用 Redis 自带的 SETNX 命令实现分布式锁，除此之外，还可以使用官方提供的 RedLock 分布式锁实现。

**其它**

Set 可以实现交集、并集等操作，从而实现共同好友等功能。

ZSet 可以实现有序性操作，从而实现排行榜等功能。	

## 3 ★★★ 与 Memchached 的比较。

**数据类型**

Memcached 仅支持字符串类型，而 Redis 支持五种不同的数据类型，可以更灵活地解决问题。

**数据持久化**

Redis 支持两种持久化策略：RDB 快照和 AOF 日志，而 Memcached 不支持持久化。

**分布式**

Memcached 不支持分布式，只能通过在客户端使用一致性哈希来实现分布式存储，这种方式在存储和查询时都需要先在客户端计算一次数据所在的节点。

Redis Cluster 实现了分布式的支持。

**内存管理机制**

- 在 Redis 中，并不是所有数据都一直存储在内存中，可以将一些很久没用的 value 交换到磁盘，而 Memcached 的数据则会一直在内存中。
- Memcached 将**内存分割成特定长度的块来存储数据**，以完全解决内存碎片的问题。但是这种方式会使得内存的利用率不高，例如块的大小为 128 bytes，只存储 100 bytes 的数据，那么剩下的 28 bytes 就浪费掉了。

## 4 ★☆☆ 数据淘汰机制。

可以设置内存最大使用量，当内存使用量超出时，会施行数据淘汰策略。

Redis 具体有 6 种淘汰策略：

| 策略            | 描述                                                 |
| --------------- | ---------------------------------------------------- |
| volatile-lru    | 从已设置过期时间的数据集中挑选最近最少使用的数据淘汰 |
| volatile-ttl    | 从已设置过期时间的数据集中挑选将要过期的数据淘汰     |
| volatile-random | 从已设置过期时间的数据集中任意选择数据淘汰           |
| allkeys-lru     | 从所有数据集中挑选最近最少使用的数据淘汰             |
| allkeys-random  | 从所有数据集中任意选择数据进行淘汰                   |
| noeviction      | 禁止驱逐数据                                         |

作为内存数据库，出于对性能和内存消耗的考虑，Redis 的淘汰算法实际实现上并非针对所有 key，而是抽样一小部分并且从中选出被淘汰的 key。

使用 Redis 缓存数据时，为了提高缓存命中率，需要保证缓存数据都是热点数据。可以将内存最大使用量设置为热点数据占用的内存量，然后启用 allkeys-lru 淘汰策略，将最近最少使用的数据淘汰。

Redis 4.0 引入了 volatile-lfu 和 allkeys-lfu 淘汰策略，LFU 策略通过统计访问频率，将访问频率最少的键值对淘汰。

## 5 ★★☆ RDB 和 AOF 持久化机制。

https://www.cnblogs.com/kismetv/p/9137897.html

Redis持久化分为RDB持久化和AOF持久化**：前者将当前数据保存到硬盘，后者则是将每次执行的写命令保存到硬盘（类似于MySQL的binlog）；**

#### RDB持久化

将某个时间点的所有数据都生成快照保存放到硬盘上。

**触发条件**

**1）手动触发**

​		save命令会阻塞redis主进程，然后去创建rdb文件

​		bgsave命令会创建一个子进程，让子进程去生成rdb文件。只有在fork子进程的时候会阻塞主进程

**2）自动触发**

​		save m n 在redis.conf中进行配置，当m秒内发生n次变化就会自动触发bgsave。

​		**save m n的实现原理**：serverCron函数、dirty计数器、和lastsave时间戳来实现的

​		serverCron是Redis服务器的周期性操作函数，默认每隔100ms执行一次；该函数对服务器的状态进行维护，其中一项工作就是检查 save m n 配置的条件是否满足，如果满足就执行bgsave。这里Redis使用了saveparams数组来保存配置文件中的属性：seconds存储m的值，changes存储n的值，所以就是通过遍历这个数组来判断的。

​		dirty计数器是Redis服务器维持的一个状态，记录了上一次执行bgsave/save命令后，服务器状态进行了多少次修改(包括增删改)；而当save/bgsave执行完成后，会将dirty重新置为0。

​		例如，如果Redis执行了set mykey helloworld，则dirty值会+1；如果执行了sadd myset v1 v2 v3，则dirty值会+3；注意dirty记录的是服务器进行了多少次修改，而不是客户端执行了多少修改数据的命令。

​		lastsave时间戳也是Redis服务器维持的一个状态，记录的是上一次成功执行save/bgsave的时间。

​		save m n的原理如下：每隔100ms，执行serverCron函数；在serverCron函数中，遍历save m n配置的保存条件，只要有一个条件满足，就进行bgsave。对于每一个save m n条件，只有下面两条同时满足时才算满足：

（1）当前时间-lastsave > m

（2）dirty >= n

​	**其他自动触发机制**

- 在主从复制场景下，如果从节点执行全量复制操作，则主节点会执行bgsave命令，并将rdb文件发送给从节点
- 执行shutdown命令时，自动执行rdb持久化，如下图所示：

![img](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20200806142732.png)



**bgsave命令执行时的服务器状态**

执行bgsave命令以后会产生一个子进程来创建RDB过程，Redis主进程仍然能处理客户端请求，但这时如果要处理save、bgsave、bgrewriteaof三个命令时会有所区别：

- save：客户端发送save命令会被服务器拒绝，因为避免父子进程同时执行两个rdbSave调用，产生竞争条件

- bgsave：客户端发送的bgsave命令会被拒绝，因为执行两个bgsave命令会产生竞争

- bgrewriteaof：和bgsave命令不能同时执行

  - 如果当前是bgsave正在执行，那么客户端发送的bgrewriteaof会被延迟到bgsave执行完毕
  - 如果当前bgrewriteaof正在执行，那么客户发送的bgsave命令会被服务器拒绝

  因为两个命令都是在子进程中运行，所以没有什么冲突，不能同时执行仅仅是性能方面的考虑。

**RDB文件载入**

RDB文件会在服务器启动的时候，自动载入，没有载入相关的命令。然后在服务器载入期间，会一直处于阻塞状态，知道载入完成以后。

**RDB文件**

RDB文件是经过压缩的二进制文件

​	**存储路径**

​		配置文件中配置：dir--路径 dbfilename--文件名（默认是dump.rdb）

​		动态设定：config set dir {newdir}

​	**RDB文件格式**

​		![img](https://images2018.cnblogs.com/blog/1174710/201806/1174710-20180605090115749-1746859283.png)

1) REDIS：常量，保存着”REDIS”5个字符，长度5字节。

2) db_version：长度4字节，RDB文件的版本号，注意不是Redis的版本号，“0006”代表了RDB文件的版本为第六版。

3) SELECTDB 0 pairs：表示一个完整的数据库(0号数据库)，同理SELECTDB 3 pairs表示完整的3号数据库；只有当数据库中有键值对时，RDB文件中才会有该数据库的信息(上图所示的Redis中只有0号和3号数据库有键值对)；如果Redis中所有的数据库都没有键值对，则这一部分直接省略。其中：SELECTDB是一个常量，1个字节，代表后面跟着的是数据库号码；0和3是数据库号码；pairs则存储了具体的键值对信息，包括key、value值，及其数据类型、内部编码、过期时间、压缩信息等等。

4) EOF：常量，1个字节，标志RDB文件正文内容结束。

5) check_sum：8字节的无符号整数，前面所有内容的校验和；Redis在载入RBD文件时，会计算前面的校验和并与check_sum值比较，判断文件是否损坏。



**RDB常用配置**

- save m n：bgsave自动触发的条件；如果没有save m n配置，相当于自动的RDB持久化关闭，不过此时仍可以通过其他方式触发
- stop-writes-on-bgsave-error yes：当bgsave出现错误时，Redis是否停止执行写命令；设置为yes，则当硬盘出现问题时，可以及时发现，避免数据的大量丢失；设置为no，则Redis无视bgsave的错误继续执行写命令，当对Redis服务器的系统(尤其是硬盘)使用了监控时，该选项考虑设置为no
- rdbcompression yes：是否开启RDB文件压缩
- rdbchecksum yes：是否开启RDB文件的校验，在写入文件和读取文件时都起作用；关闭checksum在写入文件和启动文件时大约能带来10%的性能提升，但是数据损坏时无法发现
- dbfilename dump.rdb：RDB文件名
- dir ./：RDB文件和AOF文件所在目录



#### AOF持久化

与RDB持久化通过保存数据库中的键值对状态不同，AOF持久化是保存Redis服务器执行的写命令，将写命令添加到 AOF 文件（Append Only File）的末尾。在配置文件中：appendonly yes表示开启AOF，默认是关闭的。

**执行流程/实现方式**

- 命令追加(append)：将Redis的写命令追加到缓冲区aof_buf；
- 文件写入(write)和文件同步(sync)：根据不同的同步策略将aof_buf中的内容同步到硬盘；
- 文件重写(rewrite)：定期重写AOF文件，达到压缩的目的。

**1）命令追加**

​	Redis先将写命令追加到缓冲区，而不是直接写入文件，主要是为了避免每次有写命令都直接写入硬盘，导致硬盘IO成为Redis负载的瓶颈。

**2）文件写入(write)和文件同步(sync)**

​		Redis服务器进程是一个事件循环，循环中 文件事件负责接受客户端的命令请求，向客户端发送命令；时间事件负责执行像serverCron函数这样需要定时执行的函数。

​		因为服务器处理文件事件时可能会执行写命令，向aof_buf中写入数据，所以在服务器结束一个事件循环之前，会调用flushAppendOnlyFile函数，考虑是否将aof_buf缓冲区的内容写入和保存到AOF文件里面

```java
while (True){
  //处理文件事件
	processFileEvent();
  //处理时间事件
	processTimeEvent();
	//考虑是否将aof_buf缓冲区中的内容写入和保存到AOF文件
	flushAppendOnlyFile();
}
```

​		flushAppendOnlyFile函数的行为是由配置文件的appendfsync来决定的

appendfsync有以下选项：

| 选项     | 同步频率                 |
| -------- | ------------------------ |
| always   | 每个写命令都同步         |
| everysec | 每秒同步一次             |
| no       | 让操作系统来决定何时同步 |

- always 命令写入缓冲区后立即调用系统fsync操作同步到AOF文件，fsync完成后线程返回。这种情况下，每次有写命令都要同步到AOF文件，硬盘IO成为性能瓶颈，Redis只能支持大约几百TPS写入，严重降低了Redis的性能；即便是使用固态硬盘（SSD），每秒大约也只能处理几万个命令，而且会大大降低SSD的寿命。但是它是最安全的，即使服务器停机，也只会丢失一个事件循环产生的命令数据。

- everysec 选项比较合适，可以保证系统崩溃时只会丢失一秒左右的数据，并且 Redis 每秒执行一次同步对服务器性能几乎没有任何影响；

- no 选项：在每个时间循环结束后会把aof_buf中的数据写入到缓冲区，但是不会马上同步。并不能给服务器性能带来多大的提升，而且也会增加系统崩溃时数据丢失的数量。

  `文件写入（write），操作系统为了提高写入效率，都会先将写入数据暂时保存在一个内存缓冲区里面，等到缓冲区的数据填满，或者超过一个指定的时间，在将缓冲区中的数据写入到磁盘中。这样虽然提高了写入效率，但是会有安全问题，如果计算机发生停记，会导致缓冲区中的数据都丢失`

  `因此系统同时提供了fsync、fdatasync等同步函数，可以强制操作系统立刻将缓冲区中的数据写入到硬盘里，从而确保数据的安全性。`

**3）文件重写**

随着时间流逝，Redis服务器执行的写命令越来越多，AOF文件也会越来越大；过大的AOF文件不仅会影响服务器的正常运行，也会导致数据恢复需要的时间过长。

文件重写是指定期重写AOF文件，减小AOF文件的体积。需要注意的是，**AOF重写是把当前Redis服务器内的数据转化为写命令，同步到新的AOF文件；不会对旧的AOF文件进行任何读取、写入操作!**

关于文件重写需要注意的另一点是：对于AOF持久化来说，文件重写虽然是强烈推荐的，但并不是必须的；即使没有文件重写，数据也可以被持久化并在Redis启动的时候导入；因此在一些实现中，会关闭自动的文件重写，然后通过定时任务在每天的某一时刻定时执行。



文件重写之所以能够压缩AOF文件，原因在于：

- 过期的数据不再写入文件

- 无效的命令不再写入文件：如有些数据被重复设值(set mykey v1, set mykey v2)、有些数据被删除了(sadd myset v1, del myset)等等

- 多条命令可以合并为一个：如sadd myset v1, sadd myset v2, sadd myset v3可以合并为sadd myset v1 v2 v3。

  **文件重写触发**

  手动触发：bgrewriteaof，和bgsave很类似都会fork子进程进行具体工作

  自动触发：根据auto-aof-rewrite-min-size和auto-aof-rewrite-percentage参数，以及aof_current_size和aof_base_size状态确定触发时机，只有当下面两个参数都满足时才会触发。

  - auto-aof-rewrite-min-size：执行AOF重写时，文件的最小体积，默认值为64MB。
  - auto-aof-rewrite-percentage：执行AOF重写时，当前AOF大小(即aof_current_size)和上一次重写时AOF大小(aof_base_size)的比值。

  文件重写的流程

  ![img](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20200806142741.png)

  1) Redis父进程首先判断当前是否存在正在执行 bgsave/bgrewriteaof的子进程，如果存在则bgrewriteaof命令直接返回，如果存在bgsave命令则等bgsave执行完成后再执行。前面曾介绍过，这个主要是基于性能方面的考虑。

  2) 父进程执行fork操作创建子进程，这个过程中父进程是阻塞的。

  3.1) 父进程fork后，bgrewriteaof命令返回”Background append only file rewrite started”信息并不再阻塞父进程，并可以响应其他命令。**Redis的所有写命令依然写入AOF缓冲区，并根据appendfsync策略同步到硬盘，保证原有AOF机制的正确。**

  3.2) 由于fork操作使用写时复制技术，子进程只能共享fork操作时的内存数据。**由于父进程依然在响应命令，因此Redis使用AOF重写缓冲区(图中的aof_rewrite_buf)保存这部分数据，防止新AOF文件生成期间丢失这部分数据。也就是说，bgrewriteaof执行期间，Redis的写命令同时追加到aof_buf和aof_rewirte_buf两个缓冲区。**

  4) 子进程根据内存快照，按照命令合并规则写入到新的AOF文件。

  5) 子进程写完新的AOF文件后，向父进程发信号，父进程会执行一个信号处理函数。

  ​	5.1) 父进程把AOF重写缓冲区的数据写入到新的AOF文件，这样就保证了新AOF文件所保存的数据库状态和服务器当前状态一致。

  ​	5.2) 使用新的AOF文件替换老文件，完成AOF重写。
  
  在整个重写过程中，只有在父进程执行信号处理函数的时候会阻塞父进程，其他时候都不会。

**AOF常用配置**

- appendonly no：是否开启AOF
- appendfilename "appendonly.aof"：AOF文件名
- dir ./：RDB文件和AOF文件所在目录
- appendfsync everysec：fsync持久化策略
- no-appendfsync-on-rewrite no：AOF重写期间是否禁止fsync；如果开启该选项，可以减轻文件重写时CPU和硬盘的负载（尤其是硬盘），但是可能会丢失AOF重写期间的数据；需要在负载和安全性之间进行平衡
- auto-aof-rewrite-percentage 100：文件重写触发条件之一
- auto-aof-rewrite-min-size 64mb：文件重写触发提交之一
- aof-load-truncated yes：如果AOF文件结尾损坏，Redis启动时是否仍载入AOF文件

## 6 ★★☆ 事件驱动模型。

https://cloud.tencent.com/developer/article/1353761

https://www.jianshu.com/p/fda1d6096d48

[Redis](https://cloud.tencent.com/product/crs?from=10680) 是一个事件驱动的内存数据库，服务器需要处理两种类型的事件---文件事件和时间事件。

#### 文件事件

Redis 服务器通过 socket 实现与客户端（或其他redis服务器）的交互,文件事件就是服务器对 socket 操作的抽象。 Redis 服务器，通过监听这些 socket 产生的文件事件并处理这些事件，实现对客户端调用的响应。

**Reactor**

Redis 基于 Reactor 模式开发了自己的文件事件处理器。

![img](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20200806142755.png)

“I/O 多路复用模块”会监听多个 FD ，当这些FD产生，accept，read，write 或 close 的文件事件。会向“文件事件分发器（dispatcher）”传送事件。尽管多个文件事件可能同时出现，IO多路复用程序会讲产生事件的套接字放到一个队列里，然后通过这个队列有序、同步、每次一个套接字的方式向文件事件分派器传送套接字。

文件事件分发器（dispatcher）在收到事件之后，会根据事件的类型将事件分发给对应的 handler。

**IO多路复用模块**

Redis 的IO多路复用就是封装了操作系统提供的select、epoll等函数。

epoll提供的三个方法：

```c
/*
 * 创建一个epoll的句柄，size用来告诉内核这个监听的数目一共有多大
 */
int epoll_create(int size)；

/*
 * 可以理解为，增删改 fd 需要监听的事件
 * epfd 是 epoll_create() 创建的句柄。
 * op 表示 增删改
 * epoll_event 表示需要监听的事件，Redis 只用到了可读，可写，错误，挂断 四个状态
 */
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；

/*
 * 可以理解为查询符合条件的事件
 * epfd 是 epoll_create() 创建的句柄。
 * epoll_event 用来存放从内核得到事件的集合
 * maxevents 获取的最大事件数
 * timeout 等待超时时间
 */
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```



```c
/*
 * 事件状态
 */
typedef struct aeApiState {
    // epoll_event 实例描述符
    int epfd;
    // 事件槽
    struct epoll_event *events;

} aeApiState
  /*
 * 创建一个新的 epoll 
 */
static int  aeApiCreate(aeEventLoop *eventLoop)
/*
 * 调整事件槽的大小
 */
static int  aeApiResize(aeEventLoop *eventLoop, int setsize)
/*
 * 释放 epoll 实例和事件槽
 */
static void aeApiFree(aeEventLoop *eventLoop)
/*
 * 关联给定事件到 fd
 */
static int  aeApiAddEvent(aeEventLoop *eventLoop, int fd, int mask)
/*
 * 从 fd 中删除给定事件
 */
static void aeApiDelEvent(aeEventLoop *eventLoop, int fd, int mask)
/*
 * 获取可执行事件
 */
static int  aeApiPoll(aeEventLoop *eventLoop, struct timeval *tvp)
```

- `aeApiCreate()` 是对 `epoll.epoll_create()` 的封装。
- `aeApiAddEvent()`和`aeApiDelEvent()` 是对 `epoll.epoll_ctl()`的封装。
- `aeApiPoll()` 是对 `epoll_wait()`的封装。

在看看ea.c是怎么封装的？

```c
typedef struct aeFileEvent {
    // 监听事件类型掩码，
    // 值可以是 AE_READABLE 或 AE_WRITABLE ，
    // 或者 AE_READABLE | AE_WRITABLE
    int mask; 
    /* one of AE_(READABLE|WRITABLE) */
    // 读事件处理器
    aeFileProc *rfileProc;
    // 写事件处理器
    aeFileProc *wfileProc;
    // 多路复用库的私有数据
    void *clientData;
} aeFileEvent;
```

**事件的类型**

​		IO多路复用程序可以监听多个套接字的AR_READABLE事件和AE_WRITEABLE事件

- 当套接字变得可读时（客户端对套接字执行了write操作，或者执行close操作），或者有新的accpetable套接字出现时，套接字产生AR_READABLE事件

- 当套接字变得可写时（客户端对套接字执行了read操作）套接字产生AE_WRITEABLE事件

  IO多路复用程序运行服务器同时监听套接字的AR_READABLE事件和AE_WRITEABLE事件,如果两个事件同时发生，会优先处理读套接字，然后写套接字。

**事件分发器**

既会处理文件事件还会处理时间事件，根据mask不同调用不同的事件处理器。

**文件事件处理器的类型**

Redis 有大量的事件处理器类型，用于实现不同的网络通信需要，我们就讲解处理一个简单命令涉及到的三个处理器：

- acceptTcpHandler 连接应答处理器：当Redis服务器初始化的时候，程序会将连接应答处理器和服务器监听套接字的AE_READABLE事件关联起来，当有client 调用connect函数连接到Redis的时候，套接字就会产生 AE_READABLE 事件，引发连接应答处理器执行。

- readQueryFromClinet 命令请求处理器：当服务器已经和一个客户端连接后，服务器会将客户端套接字的AE_READABLE事件和命令请求处理器关联起来，当客户端向服务器发送命令请求时，套接字就会产生AE_READABLE事件，引发命令请求处理器执行。

- sendReplyToClient 命令回复处理器：服务器会将客户端套接字的AE_WRITEABLE事件和命令回复处理器关联起来，当客户端准备好接收服务器传回的命令时，就会产生 AE_WRITEABLE 事件，调用命令回复处理器将数据回复给 client。

  当命令回复完毕以后，会解除命令回复处理器和客户端套接字AE_WRITEABLE事件的关联。

#### 时间事件

Redis的时间事件分以下两类：

- 定时事件：在指定时间之后执行一次
- 周期性事件：每隔一段时间就执行一次

```c
/* Time event structure
 *
 * 时间事件结构
 */
typedef struct aeTimeEvent {
    // 时间事件的唯一标识符，新来的时间事件ID增大
    long long id; 
    /* time event identifier. */
    // 事件的到达时间
    long when_sec; 
    /* seconds */  
    long when_ms;
    /* milliseconds */
    // 事件处理函数
    aeTimeProc *timeProc;
    // 事件释放函数
    aeEventFinalizerProc *finalizerProc;
    // 多路复用库的私有数据
    void *clientData;
    // 指向下个时间事件结构，形成链表
    struct aeTimeEvent *next;
} aeTimeEvent;
```

目前的Redis版本只使用周期性事件，而没有使用定时事件。当一个时间事件到达后，服务器会根据处理器返回的值对时间事件的when属性进行更新。

所有的时间事件都放在一个无序链表中（不是按照when的顺序），每次新来一个时间事件就会插入到链表表头。所以每次时间事件处理器都要遍历整个链表判断是否有已经到达的时间事件被处理。

**processTimeEvent**

Redis 使用这个函数处理所有的时间事件，我们整理一下执行思路：

1. 记录最新一次执行这个函数的时间，用于处理系统时间被修改产生的问题。
2. 遍历链表找出所有 when_sec 和 when_ms 小于等于现在时间的事件。
3. 执行事件对应的处理函数。
4. 检查事件类型，如果是周期事件则刷新该事件下一次的执行事件。
5. 否则从列表中删除事件。

**综合调度器（aeProcessEvents）**

```c
// 1. 获取离当前时间最近的时间事件
shortest = aeSearchNearestTimer(eventLoop);

// 2. 获取间隔时间
timeval = shortest - nowTime;
// 如果timeval 小于 0，说明已经有需要执行的时间事件了。
if(timeval < 0){
    timeval = 0
}

// 3. 阻塞并等待文件事件产生，最大阻塞时间由timeval决定，如果timeval为0就不阻塞，立即返回。如果有文件事件产生就先去执行文件事件，执行完后如果timeval仍然>0就继续阻塞等待文件事件。
numevents = aeApiPoll(eventLoop, timeval);

// 4.根据文件事件的类型指定不同的文件处理器
if (AE_READABLE) {
    // 读事件
    rfileProc(eventLoop,fd,fe->clientData,mask);
    }
    // 写事件
    if (AE_WRITABLE) {
        wfileProc(eventLoop,fd,fe->clientData,mask);
    }
}
//5.处理已到达的时间事件
processTimeEvent()
```

​		将aeProcessEvents函数放一个循环里面，加上初始化函数和清理函数就构成一个Redis主函数了

```java
main(){
	init_server();
  /一直处理事件
	while(server is not shutdown){
		aeProcessEvents();
	}
	clean_server();	
}
```

​		所以我们说Redis是一个事件驱动的程序,并且没fork任何线程,所以也成为基于事件驱动的单线程应用。

​		**事件的调度和执行规则**

1. aeApiPoll函数最大阻塞时间由到达时间最接近当前时间的时间事件决定，这样可以避免服务器对时间事件进行频繁的轮询，也可以确保该函数不会阻塞过长时间。
2. 因为文件事件时随机出现的，如果等待并处理完一次文件事件后，仍然没有时间事件到达的会，会再次等待和处理文件事件。
3. 对文件事件和时间事件的处理都是同步、有序、原子的执行。服务器不会中途中断，也不会对事件进行抢占，但是不管是文件事件处理器还是时间事件处理器会在有需要的时候让出执行权，降低事件饥饿的可能性。
4. 因为时间事件都是在文件事件之后执行，并且事件之间不会出现抢占，所以时间事件实际处理事件，通常会比设定的时间晚一些。



问1：为啥这么快？
 答：

1. 完全基于内存,绝大部分请求是纯粹的内存操作，数据在内存的字典中,字典的结构类似于HashMap,查找和操作的时间复杂度都是`O(1)`
2. 数据结构简单,对数据结构的操作也简单,Redis基于C语言精心设计了很多数据结构。
3. 采用了单线程,避免了不必要的上下文切换和竞态条件，也不存在多线程切换消耗CPU，不用考虑各种并发锁的问题，也不会出现死锁导致性能的消耗。
4. 使用I/O多路复用模型，非堵塞的IO

问2： 说一说I/O多路复用:
 答：
       多路指的是多个网络连接，复用指的是复用同一个线程。采用多路I/O复用可以让单个线程高效处理多个连接请求，且Redis内存操作速度非常快，也就是说CPU的操作不会成为影响`Redis`性能的瓶颈,所以它具备很高的吞吐量。

问3:为什么Redis是单线程的？
 答：
 		Redis是基于内存的操作，CPU不是Redis的瓶颈，Redis的瓶颈最有可能是机器内存的大小或者网络带宽。既然单线程容易实现，而且CPU不会成为瓶颈，那就顺理成章地采用单线程的方案了（毕竟采用多线程会有很多麻烦！）。但是有个缺点，我们使用单线程的方式是无法发挥多核CPU 性能，不过我们可以通过在单机开多个Redis 实例来完善！而且这个单线程指的是接收网络请求只用一个线程处理，实际上还会有一些子线程的，比如刷盘、主从同步、交换心跳信息等

## 7 ★☆☆ 主从复制原理。

https://www.cnblogs.com/kismetv/p/9236731.html

主从复制过程大体可以分为3个阶段：连接建立阶段（即准备阶段）、数据同步阶段、命令传播阶段；下面分别进行介绍。

#### 1. 连接建立阶段

该阶段的主要作用是在主从节点之间建立连接，为数据同步做好准备。

**步骤1：保存主节点信息**

从节点服务器内部维护了两个字段，即masterhost和masterport字段，用于存储主节点的ip和port信息。 

主从复制的开启，完全是在从节点发起的；不需要我们在主节点做任何事情，在从节点输入如下命令：

```
slaveof <masterip> <masterport>
```

需要注意的是，**slaveof是异步命令，从节点完成主节点ip和port的保存后，向发送slaveof命令的客户端直接返回OK，实际的复制操作在这之后才开始进行。**

**步骤2：建立socket连接**

从节点每秒1次调用复制定时函数replicationCron()，如果发现了有主节点可以连接，便会根据主节点的ip和port，创建socket连接。如果连接成功，则：

从节点：为该socket建立一个专门处理复制工作的文件事件处理器，负责后续的复制工作，如接收RDB文件、接收命令传播等。

主节点：接收到从节点的socket连接后（即accept之后），为该socket创建相应的客户端状态，**并将从节点看做是连接到主节点的一个客户端，后面的步骤会以从节点向主节点发送命令请求的形式来进行。**

**步骤3：发送ping命令**

从节点成为主节点的客户端之后，发送ping命令进行首次请求，目的是：检查socket连接是否可用，以及主节点当前是否能够处理请求。

从节点发送ping命令后，可能出现3种情况：

（1）返回pong：说明socket连接正常，且主节点当前可以处理请求，复制过程继续。

（2）超时：一定时间后从节点仍未收到主节点的回复，说明socket连接不可用，则从节点断开socket连接，并重连。

（3）返回pong以外的结果：如果主节点返回其他结果，如正在处理超时运行的脚本，说明主节点当前无法处理命令，则从节点断开socket连接，并重连。

**步骤4：身份验证**

如果从节点中设置了masterauth选项，则从节点需要向主节点进行身份验证；没有设置该选项，则不需要验证。从节点进行身份验证是通过向主节点发送auth命令进行的，auth命令的参数即为配置文件中的masterauth的值。

如果主节点设置密码的状态，与从节点masterauth的状态一致（一致是指都存在，且密码相同，或者都不存在），则身份验证通过，复制过程继续；如果不一致，则从节点断开socket连接，并重连。

**步骤5：发送从节点端口信息**

身份验证之后，从节点会向主节点发送其监听的端口号（前述例子中为6380），主节点将该信息保存到该从节点对应的客户端的slave_listening_port字段中；该端口信息除了在主节点中执行info Replication时显示以外，没有其他作用。

#### 2. 数据同步阶段

主从节点之间的连接建立以后，便可以开始进行数据同步，该阶段可以理解为从节点数据的初始化。具体执行的方式是：从节点向主节点发送psync命令（Redis2.8以前是sync命令），开始同步。

数据同步阶段是主从复制最核心的阶段，根据主从节点当前状态的不同，可以分为全量复制和部分复制。

需要注意的是，在数据同步阶段之前，从节点是主节点的客户端，主节点不是从节点的客户端；而到了这一阶段及以后，主从节点互为客户端。原因在于：在此之前，主节点只需要响应从节点的请求即可，不需要主动发请求，而在数据同步阶段和后面的命令传播阶段，主节点需要主动向从节点发送请求（如推送缓冲区中的写命令），才能完成复制。

**全量复制**

用于初次复制或其他无法进行部分复制的情况，将主节点中的所有数据都发送给从节点，是一个非常重型的操作。

Redis通过psync命令进行全量复制的过程如下：

（1）从节点判断无法进行部分复制，向主节点发送全量复制的请求；或从节点发送部分复制的请求，但主节点判断无法进行部分复制；具体判断过程需要在讲述了部分复制原理后再介绍。

（2）主节点收到全量复制的命令后，执行bgsave，在后台生成RDB文件，并使用一个缓冲区（称为复制缓冲区）记录从现在开始执行的所有写命令

（3）主节点的bgsave执行完成后，将RDB文件发送给从节点；**从节点首先清除自己的旧数据，然后载入接收的RDB文件**，将数据库状态更新至主节点执行bgsave时的数据库状态

（4）主节点将前述复制缓冲区中的所有写命令发送给从节点，从节点执行这些写命令，将数据库状态更新至主节点的最新状态

（5）如果从节点开启了AOF，则会触发bgrewriteaof的执行，从而保证AOF文件更新至主节点的最新状态



通过全量复制的过程可以看出，全量复制是非常重型的操作：

​	1 主节点通过bgsave命令fork子进程进行RDB持久化，该过程是非常消耗CPU、内存(页表复制)、硬盘IO的；关于bgsave的性能问题，可以参考 [深入学习Redis（2）：持久化](https://www.cnblogs.com/kismetv/p/9137897.html)

​	2 主节点通过网络将RDB文件发送给从节点，对主从节点的带宽都会带来很大的消耗

​	3 从节点清空老数据、载入新RDB文件的过程是阻塞的，无法响应客户端的命令；如果从节点执行bgrewriteaof，也会带来额外的消耗

**部分复制**

用于网络中断等情况后的复制，只将中断期间主节点执行的写命令发送给从节点，与全量复制相比更加高效。需要注意的是，如果网络中断时间过长，导致主节点没有能够完整地保存中断期间执行的写命令，则无法进行部分复制，仍使用全量复制。

**（1）复制偏移量（replication offset）**

主从服务器都会维持一个复制偏移量：

- 主服务器发送N个字节的数据，自己的复制偏移量+N
- 从服务器接收N个字节的数据，自己的复制偏移量+N

offset用于判断主从节点的数据库状态是否一致：如果二者offset相同，则一致；如果offset不同，则不一致，此时可以根据两个offset找出从节点缺少的那部分数据。例如，如果主节点的offset是1000，而从节点的offset是500，那么部分复制就需要将offset为501-1000的数据传递给从节点。而offset为501-1000的数据存储的位置，就是下面要介绍的复制积压缓冲区。

**（2）复制积压缓冲区（replication backlog）**

复制积压缓冲区是由主节点维护的、固定长度的、先进先出(FIFO)队列，默认大小1MB；当主节点开始有从节点时创建，其作用是备份主节点最近发送给从节点的数据。注意，无论主节点有一个还是多个从节点，都只需要一个复制积压缓冲区。

![430D1E3275277905EFE04C3B1E813DF6](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20200806142808.jpg)

在命令传播阶段，主节点除了将写命令发送给从节点，还会发送一份给复制积压缓冲区，作为写命令的备份；除了存储写命令，复制积压缓冲区中还存储了其中的每个字节对应的复制偏移量（offset）。由于复制积压缓冲区定长且是先进先出，所以它保存的是主节点最近执行的写命令；时间较早的写命令会被挤出缓冲区。

由于该缓冲区长度固定且有限，因此可以备份的写命令也有限，当主从节点offset的差距过大超过缓冲区长度时，将无法执行部分复制，只能执行全量复制。反过来说，为了提高网络中断时部分复制执行的概率，可以根据需要增大复制积压缓冲区的大小(通过配置repl-backlog-size)；例如如果网络中断的平均时间是60s，而主节点平均每秒产生的写命令(特定协议格式)所占的字节数为100KB，则复制积压缓冲区的平均需求为6MB，保险起见，可以设置为12MB，来保证绝大多数断线情况都可以使用部分复制。

从节点将offset发送给主节点后，主节点根据offset和缓冲区大小决定能否执行部分复制：

- 如果offset偏移量之后的数据，仍然都在复制积压缓冲区里，则执行部分复制；
- 如果offset偏移量之后的数据已不在复制积压缓冲区中（数据已被挤出），则执行全量复制。

**（3）服务器运行ID(runid)**

每个Redis节点(无论主从)，在启动时都会自动生成一个随机ID(每次启动都不一样)，由40个随机的十六进制字符组成；runid用来唯一识别一个Redis节点。通过info Server命令，可以查看节点的runid：

![img](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20200806142816.png)

主从节点初次复制时，主节点将自己的runid发送给从节点，从节点将这个runid保存起来；当断线重连时，从节点会将这个runid发送给主节点；主节点根据runid判断能否进行部分复制：

- 如果从节点保存的runid与主节点现在的runid相同，说明主从节点之前同步过，主节点会继续尝试使用部分复制(到底能不能部分复制还要看offset和复制积压缓冲区的情况)；
- 如果从节点保存的runid与主节点现在的runid不同，说明从节点在断线前同步的Redis节点并不是当前的主节点，只能进行全量复制。

#### 3. 命令传播阶段

数据同步阶段完成后，主从节点进入命令传播阶段；在这个阶段主节点将自己执行的写命令发送给从节点，从节点接收命令并执行，从而保证主从节点数据的一致性。

在命令传播阶段，除了发送写命令，主从节点还维持着心跳机制：PING和REPLCONF ACK。由于心跳机制的原理涉及部分复制，因此将在介绍了部分复制的相关内容后单独介绍该心跳机制。

**延迟与不一致**

需要注意的是，命令传播是异步的过程，即主节点发送写命令后并不会等待从节点的回复；因此实际上主从节点之间很难保持实时的一致性，延迟在所难免。数据不一致的程度，与主从节点之间的网络状况、主节点写命令的执行频率、以及主节点中的repl-disable-tcp-nodelay配置等有关。

repl-disable-tcp-nodelay no：该配置作用于命令传播阶段，控制主节点是否禁止与从节点的TCP_NODELAY；默认no，即不禁止TCP_NODELAY。当设置为yes时，TCP会对包进行合并从而减少带宽，但是发送的频率会降低，从节点数据延迟增加，一致性变差；具体发送频率与Linux内核的配置有关，默认配置为40ms。当设置为no时，TCP会立马将主节点的数据发送给从节点，带宽增加但延迟变小。

一般来说，只有当应用对Redis数据不一致的容忍度较高，且主从节点之间网络状况不好时，才会设置为yes；多数情况使用默认值no。

#### PSYNC命令的实现

psync的调用方法有两种：

- 如果从服务器以前没有没有复制过任何主服务器，或者之前执行过slaveof no one 命令，那么从服务器在开始执行复制的时候发送PSYNC ？-1命令，主动请求全量复制
- 如果从服务器已经复制过某个主服务器了，就会发送PSYNC <runid> <offset> ：其中runid是上次复制的主服务器id，offset是从服务器的offset。主服务器会根据这两个参数来判断执行哪种复制

主服务器会返回以下几种回复

- +FULLRESYNC <runid> <offset>，表示主服务器将执行全量复制，runid是主节点的id，offset是主节点当前的偏移量，从节点会将这个值作为自己的初始化偏移量。
- +CONTINUE，表示主服务器将进行部分复制
- -ERR，表示主服务器的版本低于Redis 2.8，识别不了PSYNC命令，从服务器将发送SYNC命令，然后主服务器就会执行全量复制。

#### 心跳检测

在命令传播阶段，从服务器默认每秒一次向主服务器发送 REPLCONF ACK <replication_offset>命令，其中偏移量是从服务器的偏移量

发送REPLCONF ACK 命令有三个作用：

- **检测主从服务器的网络连接状态**：如果主服务器超过1s没有收到从服务器发送过来的REPLCONF ACK 命令，主服务器就知道主从服务器之间的连接出现了问题

- **辅助实现min-slaves配置选项**：Redis的min-slaves-to-write和min-slaves-to-max-lag配置选项可以防止主服务器在不安全的情况下执行写命令

  例如：

  min-slaves-to-write 3

  min-slaves-to-max-lag 10

  那么在从服务器数量少于3个，或者三个从服务器的延迟都大于10s时，主服务器拒绝执行写命令

- **检测命令丢失**：因为网络故障，主服务器传播给从服务器的写命令在网络中丢失，那么从服务器发送REPLCONF ACK 命令时，主从的偏移量会不一致，所以主服务器会根据从服务器的偏移量到复制积压缓冲区中去查找，如果有缺失的数据，就发送给从服务器。

  这个过程与部分复制的原理类似，区别就是部分复制时发生了主从服务器的断线然后重连，这里只是网络拥挤等原因。



## 8 ★★★ 集群与分布式。

https://www.cnblogs.com/kismetv/p/9853040.html

集群由多个节点(Node)组成，Redis的数据分布在这些节点中。集群中的节点分为主节点和从节点：只有主节点负责读写请求和集群信息的维护；从节点只进行主节点数据和状态信息的复制。

集群的作用，可以归纳为两点：

1、数据分区：数据分区(或称数据分片)是集群最核心的功能。

集群将数据分散到多个节点，一方面突破了Redis单机内存大小的限制，存储容量大大增加；另一方面每个主节点都可以对外提供读服务和写服务，大大提高了集群的响应能力。

Redis单机内存大小受限问题，在介绍持久化和主从复制时都有提及；例如，如果单机内存太大，bgsave和bgrewriteaof的fork操作可能导致主进程阻塞，主从环境下主机切换时可能导致从节点长时间无法提供服务，全量复制阶段主节点的复制缓冲区可能溢出……。

2、高可用：集群支持主从复制和主节点的自动故障转移（与哨兵类似）；当任一节点发生故障时，集群仍然可以对外提供服务。



#### 数据分区（分片）

数据分区有顺序分区、哈希分区等，其中哈希分区由于其天然的随机性，使用广泛；集群的分区方案便是哈希分区的一种。

哈希分区的基本思路是：对数据的特征值（如key）进行哈希，然后根据哈希值决定数据落在哪个节点。常见的哈希分区包括：哈希取余分区、一致性哈希分区、带虚拟节点的一致性哈希分区等。

衡量数据分区方法好坏的标准有很多，其中比较重要的两个因素是(1)数据分布是否均匀(2)增加或删减节点对数据分布的影响。由于哈希的随机性，哈希分区基本可以保证数据分布均匀；因此在比较哈希分区方案时，重点要看增减节点对数据分布的影响。

（1）哈希取余分区

哈希取余分区思路非常简单：计算key的hash值，然后对节点数量进行取余，从而决定数据映射到哪个节点上。该方案最大的问题是，当新增或删减节点时，节点数量发生变化，系统中所有的数据都需要重新计算映射关系，引发大规模数据迁移。

（2）一致性哈希分区

一致性哈希算法将整个哈希值空间组织成一个虚拟的圆环，如下图所示，范围为0-2^32-1；对于每个数据，根据key计算hash值，确定数据在环上的位置，然后从此位置沿环顺时针行走，找到的第一台服务器就是其应该映射到的服务器。

![img](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20200806142827.png)

与哈希取余分区相比，一致性哈希分区将增减节点的影响限制在相邻节点。以上图为例，如果在node1和node2之间增加node5，则只有node2中的一部分数据会迁移到node5；如果去掉node2，则原node2中的数据只会迁移到node4中，只有node4会受影响。

一致性哈希分区的主要问题在于，当节点数量较少时，增加或删减节点，对单个节点的影响可能很大，造成数据的严重不平衡。还是以上图为例，如果去掉node2，node4中的数据由总数据的1/4左右变为1/2左右，与其他节点相比负载过高。

（3）带虚拟节点的一致性哈希分区

该方案在一致性哈希分区的基础上，引入了虚拟节点的概念。**Redis集群使用的便是该方案，其中的虚拟节点称为槽（slot）。**槽是介于数据和实际节点之间的虚拟概念；每个实际节点包含一定数量的槽，每个槽包含哈希值在一定范围内的数据。引入槽以后，数据的映射关系由数据hash->实际节点，变成了数据hash->槽->实际节点。

**在使用了槽的一致性哈希分区中，槽是数据管理和迁移的基本单位。槽解耦了数据和实际节点之间的关系，增加或删除节点对系统的影响很小。**仍以上图为例，系统中有4个实际节点，假设为其分配16个槽(0-15)； 槽0-3位于node1，4-7位于node2，以此类推。如果此时删除node2，只需要将槽4-7重新分配即可，例如槽4-5分配给node1，槽6分配给node3，槽7分配给node4；可以看出删除node2后，数据在其他节点的分布仍然较为均衡。

槽的数量一般远小于2^32，远大于实际节点的数量；在Redis集群中，槽的数量为16384。

下面这张图很好的总结了Redis集群将数据映射到实际节点的过程：

![img](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20200806142834.png)

#### 节点通信机制

（1）Redis对数据的特征值（一般是key）计算哈希值，使用的算法是CRC16。

（2）根据哈希值，计算数据属于哪个槽。

（3）根据槽与节点的映射关系，计算数据属于哪个节点。

在哨兵系统中，节点分为数据节点和哨兵节点：前者存储数据，后者实现额外的控制功能。在集群中，没有数据节点与非数据节点之分：所有的节点都存储数据，也都参与集群状态的维护。为此，集群中的每个节点，都提供了两个TCP端口：

- 普通端口：即我们在前面指定的端口(7000等)。普通端口主要用于为客户端提供服务（与单机节点类似）；但在节点间数据迁移时也会使用。
- 集群端口：端口号是普通端口+10000（10000是固定值，无法改变），如7000节点的集群端口为17000。集群端口只用于节点之间的通信，如搭建集群、增减节点、故障转移等操作时节点间的通信；不要使用客户端连接集群接口。为了保证集群可以正常工作，在配置防火墙时，要同时开启普通端口和集群端口。

#### 建立集群，增减节点

1）建立集群

./redis-trib.rb create --replicas 1 ip:port 

1表示每个主节点都有个从节点

2）增加节点

使用redis-trib.rb的add-node ip:port新增节点

redis-trib.rb reshard ip:port (ip和port可以是集群中的任一节点)

3）减少节点

使用reshard将带删除的节点哪中的槽，迁移到其他节点中

使用redis-trib.rb del-node ip/port 删除节点

## 9 ★★☆ 事务原理。	

一个事务包含了多个命令，服务器在执行事务期间，不会改去执行其它客户端的命令请求。

事务中的多个命令被一次性发送给服务器，而不是一条一条发送，这种方式被称为流水线，它可以减少客户端与服务器之间的网络通信次数从而提升性能。

Redis 最简单的事务实现方式是使用 MULTI 和 EXEC 命令将事务操作包围起来。

## 10 ★★★ 线程安全问题。

https://blog.csdn.net/diweikang/article/details/90264993

Redis是个单线程程序，所以它是线程安全的。



**为什么Redis是单线程的？**

**1.官方答案**

因为Redis是基于内存的操作，CPU不是Redis的瓶颈，Redis的瓶颈最有可能是机器内存的大小或者网络带宽。既然单线程容易实现，而且CPU不会成为瓶颈，那就顺理成章地采用单线程的方案了。

**2.性能指标**

关于Redis的性能，官方网站也有，普通笔记本轻松处理每秒几十万的请求。

**3.详细原因**

**1）不需要各种锁的性能消耗**

Redis的数据结构并不全是简单的Key-Value，还有list，hash等复杂的结构，这些结构有可能会进行很细粒度的操作，比如在很长的列表后面添加一个元素，在hash当中添加或者删除一个对象。这些操作可能就需要加非常多的锁，导致的结果是同步开销大大增加。

**总之，在单线程的情况下，就不用去考虑各种锁的问题，不存在加锁释放锁操作，没有因为可能出现死锁而导致的性能消耗。**

**2）单线程多进程集群方案**

单线程的威力实际上非常强大，核心效率也非常高，多线程自然是可以比单线程有更高的性能上限，但是在今天的计算环境中，即使是单机多线程的上限也往往不能满足需要了，需要进一步摸索的是多服务器集群化的方案，这些方案中多线程的技术照样是用不上的。

**所以单线程、多进程的集群不失为一个时髦的解决方案。**

**3）CPU消耗**

采用单线程，避免了不必要的上下文切换和竞争条件，也不存在多进程或者多线程导致的切换而消耗 CPU。

但是如果CPU成为Redis瓶颈，或者不想让服务器其他CPU核闲置，那怎么办？

可以考虑多起几个Redis进程，Redis是key-value数据库，不是关系数据库，数据之间没有约束。只要客户端分清哪些key放在哪个Redis进程上就可以了。

 

## 11 ★★★ 字符串底层结构（SDS）

Redis没有使用C语言传统的字符串表示，而是采用了简单动态字符串（SDS），用其作为Redis默认的字符串表示。除了用来作为数据库中的**字符串值**外，SDS还用做**缓冲区**：AOF模块中的AOF缓冲区，以及客户端状态中的输入缓冲区都适用SDS实现的。

#### 1）SDS的定义

```c
struct sdshdr{
	//记录buf数组中已使用的字节数量
	//也就是SDS所保存的字符串长度
	int len;
	//记录buf数组中还未使用的字节数量
	int free;
	//用来存放字符串
	char buf[];
}
```

如果存储了一个字符串“Redis” 那么他的len为5，buf数组为‘R’｜‘e’｜‘d’｜‘i’｜ ‘s’ ｜‘\0’,最后一个字符保存空字符'\0'，但这个空字符不算在len的长度里

#### 2）SDS和C字符串的区别

C字符串就是用长度为N+1的字符数组来存储长度为N的字符串.

**2.1常数复杂度获取字符串长度**

C字符串如果要获取其长度的话都要通过遍历整个字符串获得，而SDS因为有len属性，所以O(1)时间就能得到。

**2.2杜绝缓冲区溢出**

C字符串不记录自身长度还容易造成缓冲区溢出。举例：有一个strcat函数可以讲src字符串中的内容拼接到dest字符串末尾，当用户使用这个函数时，如果dest字符串没有分配足够的内存，不足以容纳src所有的内容，就会产生缓冲区溢出。

而SDS空间分配策略完全杜绝了这个可能：当需要对SDS进行修改时，API会首先检查SDS空间是否满足修改所需的要求，如果不满足，API会自动将SDS空间扩展至执行修改所需的大小，然后才执行修改操作

**2.3减少修改字符串带来的内存重分配次数**

每次增长或缩短一个C字符串，都要对保存这个C字符串的数组进行一次内存重分配操作：

- 如果增长一个字符串，需要通过内存分配来扩展底层数组的空间大小--否则会产生缓冲区溢出
- 如果缩短字符串，需要通过内存分配来释放不再使用的空间--否则会产生内存泄露

内存重分配设计复杂的算法，并且可能要执行系统调用，是一个比较耗时的操作，所以SDS通过未使用空间接触了字符串长度和底层数组长度之间的关联：在SDS中，buf数组的长度不一的就是字符串数量+1。通过未使用空间，SDS实现了空间预分配和惰性空间释放两种优化策略。

1. 空间预分配

   用于优化SDS的字符串增长操作，当要对SDS进行修改，并且需要对SDS空间进行扩展时，程序不仅会分配修改所需要的空间，还会为SDS分配额外的未使用空间。

   对于额外空间的分配使用一下公示

   - 对SDS进行修改之后，SDS的长度（len的值）小于1MB，那么程序会分配和len属性同样大小的未使用空间。所以free=len，buf数组的长度为len+free+1=2*len+1
   - 如果len的长度大于1MB，那么程序会分配1MB的未使用空间

   通过这种预分配策略，SDS将连续增长N次字符串所需的内存重分配次数从至少N次降低为最多N次。

2. 惰性空间释放

   ​		当SDS的API缩短SDS的长度时，程序不会立即用内存重分配来回收缩短后多出来的字节，而是用free属性来记录，等下一次拼接操作使用。

   ​		与此同时，SDS也提供了相应的API，可以再有需要的时候真正释放SDS的未使用空间，不用单行惰性空间释放策略会造成内存浪费问题。

**2.4二进制安全**

​		C字符串是根据空字符'\0'判断字符串是否结束，所以空字符不能包括在字符串里面，并且C字符串的字符必须符合某种编码，这些限制使C字符串只能保存文本数据，不能保存像图片、音频等二进制数据。

​		虽然数据库一般用来保存文本数据，但是保存二进制数据的场景也不少见，所以SDS的API都是二进制安全的，所有SDS API都会以处理二进制的方式来处理SDS存放在buf数组里的数据。

​		所以称buf数组为字节数组，而不是字符数组。

**2.5兼容部分C字符串函数**

​		虽然SDS的API都是二进制安全的，但是一样遵循C字符串以空字符结尾的惯例，这样如果SDS存的是文本数据，那么也可以重用一些<string.h>中的函数。

## 12 ★★★ 哨兵系统(Sentinel System) 

为了实现Redis的高可用性，可以由一个或多个Sentinel实例组成的Sentinel系统监视多个主服务器，以及这些主服务器的从服务器。在被监视的主服务器进入下线状态后，从该主服务器属下的某个从服务器升级为新的主服务器，然后等原来的主服务器上线以后变成从服务器。

![6CAF77397DC6C18676768A1B6E56B850](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20200806142917.jpg)

Sentinel与被监视的主服务器建立两个异步连接：

- 命令连接，专门用于向主服务器发送命令。
- 订阅连接，专门用于订阅主服务器的 _ sentinel _:hello频道，接受消息。因为Redis的发布与订阅功能中，被发送的消息是不会保存在Redis服务器里，如果接收消息的客户端下线，这个客户端就会丢失这个消息，所以Sentinel订阅这个频道。

不同的Sentinel如果监听同一个主服务器，会互相之间建立命令连接，但是不会创建订阅连接。Sentinel与主从服务器建立订阅连接就是用来从频道信息发现未知的新Sentinel，相互已知的Sentinel使用命令连接进行通信就足够了。

#### 主观下线

​		默认情况下，Sentinel每秒一次向与他创建命令连接的实例（包括主从服务器，和其他Sentinel在内）发送Ping命令，通过返回值判断实例是否在线。

- 有效回复：实例返回+Pong、-LOADING、-MASTERDOWN三种中的一种

- 无效回复：其他命令

  Sentinel配置文件中的down-after-milliseconds制定了Sentinel判断实例进入主观下线所需要的时间长度

  多个Sentinel设置的主观下线时常可能不同

#### 客观下线

​		当Sentinel将一个主服务器判断为主观下线以后，为了确认这个主服务器是否真的下线，会想同样监视这个主服务器的其他Sentinel进行询问，看他们是否也认为主服务器已经进入下线状态（可以是主观也可以是客观下线）。当Sentinel从其他Sentinel获得了足够数量的已下线判断后，Sentinel会将从服务器判定为客观下线，并执行故障转移操作。

​		判断是否是客观下线的值有Sentinel配置里quorum参数来决定，每个Sentinel可以设置不同的值。

```c
sentinel monitor master 127.0.0.1 6379 2
```

那么包括当前Sentinel在内，只要有两个Sentinel认为主服务器已经进入下线状态，当前Sentinel就将主服务器判断为客观下线。

#### 选举领头Sentinel

当一个主服务器被判断为客观下线时，监视这个主服务器的各个Sentinel会进行协商，选举出一个领头Sentinel，并由这个领头Sentinel对下线主服务器执行故障转移操作。

## 13 ★★★ 分布式锁

https://www.jianshu.com/p/a1ebab8ce78a

为了防止分布式系统中多个进程之间相互干扰，需要使用分布式锁对这些进程进行调度。

分布式锁应该具备哪些条件：

- 在分布式环境下，一个方法在同一时间内只能被一个机器的一个线程执行
- 高可用、高性能的获取锁与释放锁
- 具备可重入特性
- 具备锁失效机制，防止死锁
- 具备非阻塞锁特性，即没有获取锁将直接返回获取锁失败

**分布式锁实现有哪些**

Memcached：利用 Memcached 的 `add` 命令。此命令是原子性操作，只有在 `key` 不存在的情况下，才能 `add` 成功，也就意味着线程得到了锁。

Redis：和 Memcached 的方式类似，利用 Redis 的 `setnx` 命令。此命令同样是原子性操作，只有在 `key` 不存在的情况下，才能 `set` 成功。

Zookeeper：利用 Zookeeper 的顺序临时节点，来实现分布式锁和等待队列。Zookeeper 设计的初衷，就是为了实现分布式锁服务的。

Chubby：Google 公司实现的粗粒度分布式锁服务，底层利用了 Paxos 一致性算法。



**介绍一下Redis实现分布式锁的概念**

Redis有三种方法实现分布式锁：

**1.通过Redis事务**

先回顾一下Redis事务中的几个命令

MULTI：开启事务

EXEC：执行事务

DISCARD：放弃事务，从事务状态转到非事务状态

WATCH： 监视某个key，如果这个key 被其他客户端访问并且修改了，那么这个事务就执行不成功

可以发现这个WATCH的功能很想乐观锁，判断这个值在锁住的阶段是否有更改，如果有更改就放弃这个操作。

用Redis事务实现一个秒杀系统的库存扣减

（1）用线程池初始化5000个客户端

```java
public static void intitClients() {
 ExecutorService threadPool= Executors.newCachedThreadPool();
 for (int i = 0; i < 5000; i++) {
  threadPool.execute(new Client(i));
 }
 threadPool.shutdown();
 
 while(true){ 
         if(threadPool.isTerminated()){  
             break;  
         }  
     }  
}
```

(2)初始化商品的库存数为1000

```java
public static void initPrductNum() {
  Jedis jedis = RedisUtil.getInstance().getJedis();
  jedisUtils.set("produce", "1000");// 初始化商品库存数
  RedisUtil.returnResource(jedis);// 返还数据库连接
 }
}
```

（3）库存扣减

```java
/**
 * 顾客线程
 * 
 * @author linbingwen
 *
 */
class client implements Runnable {
 Jedis jedis = null;
 String key = "produce"; // 商品数量的主键
 String name;
 
 public ClientThread(int num) {
  name= "编号=" + num;
 }
 
 public void run() {
 
  while (true) {
   jedis = RedisUtil.getInstance().getJedis();
   try {
    jedis.watch(key);//监视商品数量
    int num= Integer.parseInt(jedis.get(key));// 当前商品个数
    if (num> 0) {
     Transaction ts= jedis.multi(); // 开始事务
     ts.set(key, String.valueOf(num - 1)); // 库存扣减
     List<Object> result = ts.exec(); // 执行事务
     if (result == null || result.isEmpty()) {
      System.out.println("抱歉，您抢购失败，请再次重试");
     } else {
      System.out.println("恭喜您，抢购成功");
      break;
     }
    } else {
     System.out.println("抱歉，商品已经卖完");
     break;
    }
   } catch (Exception e) {
    e.printStackTrace();
   } finally {
    jedis.unwatch(); // 解除被监视的key
    RedisUtil.returnResource(jedis);
   }
  }
 }
}
```

如果客户端发现商品数量被其他客户端修改了，就会不断自旋进行抢购

**2.setnx、getset、expire、del等命令实现**

1. `setnx`：命令表示如果key不存在，就会执行set命令，返回1，若是key已经存在，不会执行任何操作，返回0。
2. `getset`：将key设置为给定的value值，并返回原来的旧value值，若是key不存在就会返回返回nil 。
3. `expire`：设置key生存时间，当当前时间超出了给定的时间，就会自动删除key。
4. `del`：删除key，它可以删除多个key，语法如下：`DEL key [key …]`，若是key不存在直接忽略。释放对应的锁，其他线程就可以通过setnx来获得锁

综合伪代码如下：

```java
if（setnx（lock_sale_商品ID，1） == 1）{
    expire（lock_sale_商品ID，30）
    try {
        do something ......
    } finally {
        del（lock_sale_商品ID）
    }
}
```

但上面代码存在一个问题，**setnx和expire是非原子性**的，可能存在一种情况，一个线程执行setnx成功得到锁以后，还没来得及执行expire就挂掉了，这时候这把锁还没有设置过期时间，变成死锁了。

**如何解决：**setnx本身并不支持传入超时时间，set指令可以

```java
set（lock_sale_商品ID，value，30，NX）
```

这时候在联想一个场景，一个线程A得到了锁，设置了锁的有效期是30s，但是本身线程执行该代码的时间很慢，30s还没执行完，此时锁过期自动释放了，线程B得到了这个锁。

随后A运行完了代码，执行del指令来释放锁。但是线程B还没来得及执行，所以相当于线程A把线程B得到的锁给释放了。

**解决方法：**可以在释放锁之前先判断，当前持有锁的是不是自身线程，所以可以在setnx的时候将自身线程ID设为value，然后

```dart
String threadId = Thread.currentThread().getId()
set（key，threadId ，30，NX）
```

```java
if（threadId .equals(redisClient.get(key))）{
    del(key)
}
```

这样又产生了新的问题，判断和释放锁是两个独立的操作，不是原子性。所以归根结底，就是我们不想让一个线程在运行的时候，其他线程也来访问这个代码块。此时可以让获得锁的线程开启一个守护线程，来给快要过期的锁续航，如果快到过期时间了，线程还没执行完，守护线程会执行expire指令，续航20s，之后每20s执行一次，直到线程运行完了这块代码。当线程完成任务以后，会显示的关掉守护线程，即使节点1突然挂掉了，守护线程只是续航了20s，之后锁仍然会超时，自动释放

**3.Redisson实现**

![img](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20200904195117.png)

可以发现这个看门狗和上面所说的守护线程很像，使用方法如下：

```java
RLock lock = redisson.getLock("lockName");
lock.locl();
lock.unlock();
```

Redisson 中加锁机制是通过lua脚本实现，Redisson会首先通过hash算法选择Redis Cluster集群中的一个节点，然后会把一个lua脚本发送到Redis中。

lua脚本底层实现如下：

```lua
returncommandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, command,
 "if (redis.call('exists', KEYS[1]) == 0) then " +
       "redis.call('hset', KEYS[1], ARGV[2], 1); " +
       "redis.call('pexpire', KEYS[1], ARGV[1]); " +
       "return nil; " +
   "end; " +
   "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
       "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
       "redis.call('pexpire', KEYS[1], ARGV[1]); " +
       "return nil; " +
   "end; " +
   "return redis.call('pttl', KEYS[1]);",
     Collections.<Object>singletonList(getName()), internalLockLeaseTime, getLockName(threadId));
```

Redis.call()的第一个参数表示要执行的命令，KEYS[1]表示要加锁的key值，ARGV[1]表示key的有效期,默认是30s,ARGV[2]表示加锁的客户端的ID

lua脚本封装了执行的业务逻辑代码，它能保证执行业务代码的原子性，通过hset lockName进行加锁。

如果第一个客户端已经通过hset命令加锁，第二个客户端继续执行lua脚本，会发现锁被占用，会通过pttl myLock返回第一个客户端持有的锁的生存时间。若还有生存时间，第二个客户端会不停自旋尝试获取锁。

Redisson中可重入锁的实现是通过incrby lockName来实现，重入计数器会+1，释放一次计数-1.最后用完锁执行 del lockName可以直接释放锁。

多窗口抢票的例子

```java
public class SellTicket implements Runnable {
    private int ticketNum = 1000;
    RLock lock = getLock();
    // 获取锁 
    private RLock getLock() {
        Config config = new Config();
        config.useSingleServer().setAddress("redis://localhost:6379");
        Redisson redisson = (Redisson) Redisson.create(config);
        RLock lock = redisson.getLock("keyName");
        return lock;
    }
 
    @Override
    public void run() {
        while (ticketNum>0) {
            // 获取锁,并设置超时时间
            lock.lock(1, TimeUnit.MINUTES);
            try {
                if (ticketNum> 0) {
                    System.out.println(Thread.currentThread().getName() + "出售第 " + ticketNum-- + " 张票");
                }
            } finally {
                lock.unlock(); // 释放锁
            }
        }
    }
}
```

```java
public class Test {
    public static void main(String[] args) {
        SellTicket sellTick= new SellTicket();
        // 开启五条线程，模拟5个窗口
        for (int i=1; i<=5; i++) {
            new Thread(sellTick, "窗口" + i).start();
        }
    }
}
```



## 13 ★★★ 如何统计网站的UV

数据量少的话用set 或者 hashmap啥的都行，如果用户量很大，那用前两者就会导致内存不够用的现象，可以使用hyperLogLog，具体原理在下文中介绍

https://blog.justwe.site/post/redis-hyperloglog/

