### Kafka 消费者组（consumer group）详解

https://www.cnblogs.com/huxi2b/p/6223228.html

包含一下内容：

- 消费者组

- 消费者位置

  ​	消费者消费的信息用位移来表示。很多消息引擎都把这部分信息放在服务器端，好处是实现简单，但是存在在三个问题：1.broker从此变成有状态，影响伸缩性；2.需要引入应答机制确认消费成功；3.由于要保存很多consumer 的offset，会引入复杂的数据结构，造成资源浪费。

  ​	所以kafka让每个consumer group保存自己的位移信息，只需要一个整数表示位置就行；在引入checkpoint机制定期持久化，简化了应答机制（是不是和MySQL的redo log很像。

- 位移管理

  ![img](https://images2015.cnblogs.com/blog/735367/201612/735367-20161226175429711-638862783.png)

  会记录消费者组里 各个消费者消费的主题-分区 以及offset  用map的形式存储

  **位移提交**

  老版本的位移提交到zookeeper中，目录结构/consumers/<[group.id](http://group.id/)>/offsets/<topic>/<partitionId>

  但是zookeeper不适合进行大量读写操作，所以新版本kafka新增了一个主题：__consumer_offsets  ，将offset信息写入这个topic，格式如下：

![img](https://images2015.cnblogs.com/blog/735367/201612/735367-20161226175522507-1842910668.png)

每个goup会被保存在那个__consumer_offsets 分区可以看 http://www.cnblogs.com/huxi2b/p/6061110.html

- Rebalance
  - 定义：分配消费者下所有消费者订阅topic的各个分区
  - 如何进行组内分区分配：range、round-robin
  - 谁来执行rebalance和consumer group管理：coordinator管理 组成员、位移提交保护机制等。每一个consumer会coordinator进行协调通信
  - 如何确定coordinator：1.确定consumer group位移信息写入__consumers_offsets 的哪个分区 2.该分区leader所在的broker就是coordinator
  - Rebalance Generation
  - protocol：有五个协议Heartbeat、LeaveGroup、SyncGroup、JoinGroup、DescribeGroup

## Kafka 系列文章

Spring-Kafka（三）—— 操作Topic以及Kafka Tool 2的使用

https://www.jianshu.com/p/aa196f24f332

Spring-Kafka（四）—— KafkaTemplate发送消息及结果回调

https://www.jianshu.com/p/9bf9809b7491

Spring-Kafka（五）—— 使用Kafka事务的两种方式

https://www.jianshu.com/p/59891ede5f90

Spring-Kafka（六）—— @KafkaListener的花式操作

https://www.jianshu.com/p/a64defb44a23

Spring-Kafka（八）—— KafkaListener定时启动（禁止自启动）

https://www.jianshu.com/p/2447592ca5a9

