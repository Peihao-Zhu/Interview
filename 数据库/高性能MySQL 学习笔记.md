## 1.服务器性能剖析

### 1.1剖析MySQL查询

​		先剖析整个数据库服务器，分析出哪些查询时主要的压力来源。定位到具体需要优化的查询后，可以钻取下去对这些查询进行单独的剖析，分析哪些子任务是响应时间的主要消耗者。

- 剖析服务器负载

**慢查询日志**的功能原本知识捕获比较慢的查询，在MySQL5.0之前的版本中，慢查询日志的响应时间单位是s，粒度太粗了。在MySQL5.1以后可以通过设置long_query_time为0来捕获所有查询，而且查询单位是微妙。

在MySQL当前版本中，慢查询日志是开销最低、精度最高的测量查询时间的工具。

**分析查询日志**

不要直接打开整个慢查询日志进行分析，首先生成一个剖析报告，这里建议使用pt-query-digest。一般情况下，只需要将满查询日志文件作为参数传递给pt-query-digest，就可以工作了。它会讲查询报告打印出来，并切能够选择将“重要的查询”逐条打印出更详细的信息![image-20200821104937297](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20200821111636.png)

- 剖析单条查询

**使用SHOW PROFILE**

该命令在MySQL 5.1以后引入，默认是禁用的,可以通过以下命令开启

```mysql
mysql>SET profiling=1
```

当执行一条查询语句时，在使用show profiles命令，返回如下：

```mysql
mysql> select * from book;
mysql> show profiles;
+----------+------------+--------------------+
| Query_ID | Duration   | Query              |
+----------+------------+--------------------+
|        1 | 0.00551700 | select * from book |
+----------+------------+--------------------+
1 row in set, 1 warning (0.00 sec)

```

还可以使用下面的查询语句,查看更详细的细节。但是这存在一个问题，输出是按照执行顺序排序，并不是按照花费时间排序。

```mysql
mysql> show profile for query 1;
+--------------------------------+----------+
| Status                         | Duration |
+--------------------------------+----------+
| starting                       | 0.000486 |
| Executing hook on transaction  | 0.000019 |
| starting                       | 0.000009 |
| checking permissions           | 0.000005 |
| Opening tables                 | 0.000286 |
| init                           | 0.000264 |
| System lock                    | 0.000279 |
| optimizing                     | 0.000019 |
| statistics                     | 0.000019 |
| preparing                      | 0.000019 |
| executing                      | 0.004036 |
| end                            | 0.000014 |
| query end                      | 0.000004 |
| waiting for handler commit     | 0.000008 |
| closing tables                 | 0.000010 |
| freeing items                  | 0.000017 |
| cleaning up                    | 0.000023 |
+--------------------------------+----------+
17 rows in set, 1 warning (0.00 sec)

```

为了解决上面这个可以直接查询 INFORMATION_SCHEMA对应的表![image-20200821111624011](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20200821111624.png)

可以发现查询时间太长主要是因为花了一大半时间将数据复制到临时表，“发送数据”也消耗很多时间，可能是不同的服务器活动，包括在关联时搜索匹配的行记录等。

但要分析为什么复制数据到临时表要话费这么多时间需要进一步剖析子任务

**SHOW STATUS**

该命令返回的都是计数器，显示某些活动如读索引的频繁程度，但无法给出消耗多少时间。既有服务器级别的全局计数器，也有基于某个连接的会话级别的计数器。

SHOW STATUS可以猜测哪些操作代价较高或者消耗时间较多。最有用的计数器包括句柄计数器、临时文件和表计数器等。下面例子演示了如何将会话级别的计数器重置为0，然后查询前面提到的视图，在检查计数器的结果：

![image-20200821112905334](/Users/zhupeihao/Library/Application Support/typora-user-images/image-20200821112905334.png)

从结果可以看到，该查询使用了三个临时表，其中两个是磁盘临时表，并且有很多没有用到索引的读操作（Handler_read_rnd_next）

### 1.2诊断间歇性问题

间歇性问题比如系统偶尔停顿或者慢查询，很难诊断。如果一时无法定位问题，可能是测量的方式不正确，或者测量的点选择有误，或者使用的工具不合适。

- 单条查询问题还是服务器问题

单条查询问题：服务器整体运行没有问题，知识某条查询变慢

服务器问题：服务器所有的程序都突然变慢，又突然变好，每一条查询也都变慢了。

如何判断是哪种问题？

1. SHOW GLOBAL STATUS命令，每秒一次执行通过返回的计数器来分析

![image-20200821125233445](/Users/zhupeihao/Library/Application Support/typora-user-images/image-20200821125233445.png)

这个命令每秒捕获一次SHOW GLOBAL STATUS，输出给awk计算并输出每秒的查询数、Threads_connected和Threads_running。这三个数据的趋势对于服务器级别偶尔停顿的敏感性很高

   2.SHOW PROCESSLIST，不停捕获该命令的输出，来观察是否有大量线程处于不正常的状态或有其他不正常的特征。



## 2. Schema与数据类型优化

### 2.1选择优化的数据类型

1. 更小的通常更好：一般情况，尽量使用正确存储数据的最小数据类型。
2. 简单就好：简单数据类型通常需要更少的CPU周期
3. 尽量避免NULL：NULL会使的索引、索引统计和值都更复杂，所以把列改为NOT NULL。

**2.1.1整数类型**

TINYINT、SMALLINT、MEDIUMINT、INT、BIGINT分别使用8，16，24，32，64位存储空间。整数类型又可选的UNSIGNED属性，表示不允许负值，也就是第一位不是符号位。这样的话可以是正数的上限提高一倍。

**2.1.2实数类型**

实数是带有小数部分的数字。MySQL支持精确类型（DECIMAl），也支持不精确类型（FLOAT、DOUBLE）。

浮点类型存储同样范围的值，通常比DECIMAL使用更少的空间。FLOAT占4字节，DOUBLE占8字节相比FLOAT有更高的精度和更大的范围。

因为需要额外的空间和计算开销，应该尽量只在对小数进行精确计算时才使用DECIMAL，但在数据量大的时候，可以考虑使用BIGINT代替DECIMAL，乘小数的位数，

**2.1.3字符串类型**

**VARCHAR**

VARCHAR用于存储可变长字符串，仅使用必要的空间，需要使用1或2个额外字节记录字符串长度：如果长度小于或等于255，只使用1个字节表示，否则使用2个字节。

VARCHAR节省存储空间，对性能有帮助。但是，由于行是变成的，在UPDATE以后可能行变得很长，如果页内没有更多的空间存储，此时不同的存储引擎的处理方式是不一样的。MyISAM会讲行拆成不同的片段存储，InnoDB需要分裂页来使行可以放进页内。

**CHAR**

CHAR是定长的，MYSQL在服务器层灰删除所有末尾的空格。适合存储很短的字符串，因为CAHR（1）只有1个字节，而VARCHAR（1）需要2个字节，因为额外1个字节用来存储长度。

**BLOB和TEXT**

两者都是存储很大的数据而设计的字符串数据类型，分别采用二进制和字符方式存储。

字符类型有TINYTEXT，SMALLTEXT，TEXT，MEDIUMTEXT，LONGTEXT；对应的二进制类型是TINYBLOB，SMALLBLOB，BLOB，MEDIUMBLOB，LONGBLOB。

MySQL对BLOB和TEXT列进行排序与其他类型不同：它只对每个列的最前max_sort_length字节而不是整个字符串做排序。

**2.1.4日期和时间类型**

MySQL可以使用YEAR、DATE等类型来保存日期和时间值。MySQL能存储的最小时间粒度是s（MariaDB支持微妙级别）。

**DATETIME**

能保存大范围的值，从1001年到9999年，精度为s，使用8字节的存储空间。

**TIMESTAMP**

保存从1970年1月1日到2038奶奶，4个字节存储。比DATETIME空间效率更高

### 2.2MySQL schema设计的陷阱

**太多的列**

MySQL的存储引擎API工作时需要在服务器层和存储引擎层之间通过行缓冲格式拷贝数据，然后在服务器层将缓冲内容解码成各个列。从行缓冲中将编码过的列转换成行数据结构的操作代价是非常高的。MyISAM的定长行结构实际上与服务器层的行结构正好匹配，所以不需要转换。然而MyISAM变长行结构和InnoDB的行结构则总是需要转换，转换的代价需要依赖列的数量。

**太多的关联**

单个查询最好在12个表以内做关联

**NULL值设定**

尽可能考虑替代NULL，比如用0、空字符串等，但是也不要走极端，

有时候用NULL可能效果反倒要好。

### 2.3范式和反范式

​		**范式优点**

- 更新操作比反范式化更快

- 数据较好的范式化，只有很少或者没有重复数据，修改数据量也更小

- 范式化的表通常更小，更好的放在内存，执行操作更快。

- 冗余的数据更少，所以更少的需要使用DISTINCT或者GROUP BY字段。

  **范式缺点**

查询的时候经常要关联表

所以在实际开发中可以混用范式化和反范式化。

### 2.4加快ALTER TABLE操作的速度

MySQL执行大部分修改表结构操作的方法使用心得结构创建一个空表，从旧表查处所有数据插入新表，然后删除旧表。一般而言，大部分ALTER TABLE操作会导致MySQL服务中断，我们会使用两个技巧来解决：一种是先在一台不提供服务的机器上执行ALTER TABLE操作，然后和提供服务的主库进行切换；另外一种技巧是影子拷贝，用要求的表结构创建一张和原表无关的新表，然后通过重命名和删表操作交换两张表。

但不是所有的 ALTER TABLE操作都会引起表重建，如果知识修改列字段的默认值，会直接修改.frm文件而不设计表数据，所以操作非常快。

**只修改.frm文件**

1. 创建一张有相同结构的空表，并进行所需要的修改
2. 执行FLUSH TABLES WITH READ LOCK。关闭所有正在使用的表，禁止任何表被打开
3. 交换.frm文件
4. 执行UNLOCK TABLES

## 3创建高性能的索引

### 3.1索引基础

#### 3.1.1索引的类型

索引是在存储引擎层而不是服务器层实现的，所以，没有统一的索引标准。

**B-Tree**

在没有特别指明什么索引类型的时候，基本上都是B-Tree索引，InnoDB使用B+Tree，NDB使用T-Tree作为索引。

存储引擎以不同的方式使用B-Tree索引，性能也各不相同。例如：MyISAM使用前缀压缩技术使得索引更小，但InnoDB按照原数据格式进行存储。再如MyISAM索引通过数据的物理位置引用被索引的行，而InnoDB则根据主键引用被索引的行。

使用B-Tree索引的查询类型，适用于全键值、键值范围或键前缀查找。其中键前缀查找只适用于根据最左前缀的查找。

建立索引（姓，名，生日）

- 全值匹配
- 匹配最左前缀
- 匹配列前缀：匹配某一列的值的开头部分
- 匹配范围值：查找姓在一个范围之间的人
- 精确匹配某一列并范围匹配另一列
- 只访问索引的查询

如果不是上面几种情况，就无法使用索引，比如说知识查询名为Bill的人，或生日为...的人，类似的这些都不符合最左数据列。如果某个列卫范围查询，那么其右边所有列都无法使用索引优化查找。

### 3.2聚簇索引

聚簇索引是一种数据存储方式，数据存放在索引的叶子节点中。一些数据库服务器允许选择哪个索引作为聚簇索引，但是MySQL内建的存储引擎还没有支持这一点。InnoDB通过主键聚集数据。如果没有定义主键，InnoDB回选择一个唯一的非空索引代替，如果没有这样的索引，InnoDB回隐式定义一个主键来作为聚簇索引。

聚集数据的好处：

- 把相关数据保存在一起。如：实现电子邮箱时，根据用户ID聚集数据，这样要获取用户的全部邮件，只需要从磁盘读取少量数据页就可以，否则需要每封邮件都进行一次磁盘IO
- 数据访问更快。将索引和数据都保存在同一个B-Tree中，因此聚簇索引中获取数据通常比非聚簇索引快
- 使用覆盖索引扫描的查询可以直接使用页节点的主键值

聚簇索引的缺点

- 插入速度严重依赖于插入顺序。按照主键顺序插入时加载数据到InnoDB表中速度最快的方式。但如果不是按照主键顺序加载数据，那么加载完成后最好使用OPTIMIZE TABLE命令重新组织表。
- 更新聚簇索引列的代价很高，导致每个被更新的行都移动位置
- 基于聚簇索引的表在插入新行，或者主键被更新导致需要移动行是，可能面临页分裂，页分裂会导致表占用更多磁盘空间。
- 聚簇索引可能导致全表扫描更慢
- 耳机索引可能比想象要更大

**InnoDB和MyISAM的数据分布区别**

MyISAM按照数据插入顺序存储在磁盘上，新插入的一行数据就存储在下一个磁盘空间。建立索引，是针对索引进行排序（这是B+Tree的特性）。关于索引，MyISAM没有分聚簇和二级索引，所有索引的分布方式都一样。

InnoDB的索引存储方式不同

![image-20200823154723903](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20200823154724.png)

虽然也是按照主键顺序进行排序，但是叶子节点存储了整个表的数据，包含了主键值、事务ID、用于事务和MVCC的回滚指针以及其他列。如果主键是一个列前缀索引，InnoDB也会抱憾完整的主键列和剩余列。

**在InnoDB表中按主键顺序插入行**

如果使用InnoDB表，没有什么数据需要聚集，可以使用自增列，作为主键，这样可以保证数据行是按顺序写入。

最好避免随机聚簇索引，例如：用UUID作为聚簇索引会很糟糕，是的索引的插入变得完全随机。

为了演示，测试两张表，只有主键有区别（一个是自增ID，一个是UUID），分别网两个表插入100万条记录。然后向这两个表继续插入300万条记录。

![image-20200823155724418](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20200823162018.png)

可以发现使用了uuid的表不仅插入时间更长，而且建立的索引也很大，原因在于页分裂和碎片。

向聚簇索引中插入顺序的索引值时，InnoDB把每一条记录存储到上一条记录的后面。当达到页的最大填充因子时，下一条记录就会写入新的页中。

![image-20200823162008163](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20200823162008.png)

那么如果插入的是无序的值？

![image-20200823212204779](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20200823212204.png)

因为新行的主键值不一定比前一条记录大，所以会产生以下缺点：

- 写入的目标页可能已经刷到磁盘上并从缓存中移除，InnoDB在插入之前需要先找到磁盘中的目标页读到内存中。这将导致大量的随机IO
- 因为写入的乱序的，InnoDB不得不频繁的做页分裂操作，以便为新的行分配空间
- 由于频繁的页分裂，页会变得稀疏，最终数据会有碎片



### 3.3覆盖索引

索引不仅是查找数据的高效方式，还可以直接用来获取列的数据。如果索引包含需要查询的字段值，就称为“覆盖索引”。

- 索引条目通常远小于数据行大小，所以只读取索引，MySQL可以极大键撒后数据访问量。这对缓存的负载非常重要，因为这种情况下，响应时间大部分花在拷贝上。覆盖索引对与IO密集型的应用也有帮助，因为索引比数据更小，更容易全部放入内存。
- 因为索引是按照列值顺序存储（至少单个页内如此），对于IO密集型的范围查询会比随机从磁盘中读取每一行数据的IO要少得多。
- 一些存储引擎如MyISAM在内存中只缓存索引，数据则依赖于操作系统来缓存，因此访问数据需要一次系统调用。这对于频繁访问数据的系统会导致严重的性能问题。
- 在InnoDB中，二级索引在叶子节点只保存了行的主键值，所以如果二级主键能覆盖查询，就可以避免对主键索引的二次查询。

不是所有类型的索引都可以成为覆盖索引。覆盖索引必须要存储索引列的值，而哈希索引、空间索引和全文索引都不存储索引列的值，所以只能用B-Tree索引做覆盖索引。

在EXPLAIN的Extra列可以看到，如果表inventory又一个多列索引（store_id,film_id）

![image-20200823222547201](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20200823222547.png)

MySQL查询优化器会在执行查询前判断是否有一个索引能进行覆盖。假设索引覆盖了where条件中的字段，但不是整个查询涉及的字段，就会去了回表，尽管where条件为false。

![image-20200823232148589](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20200823232148.png)

无法覆盖的原因：

- 查询返回所有列，没有任何索引可以覆盖所有列
- MySQL不能再索引中执行LIKE操作。这是底层存储引擎API的限制，MySQL5.5只允许在索引中做简单比较操作（等于、不等于以及大于），MySQL能在索引中做最左前缀的LIKE比较。

### 3.4使用索引扫描做排序

MySQL有两种方式生成有序结果：通过排序操作；或按索引顺序扫描。如果EXPLAIN的type值为“index”，说懵MySQL使用了索引扫描做排序。

扫描索引本身是很快的，因为只需要从一条索引记录移动到紧接着的吓一条记录就可以了。但如果索引不能覆盖查询所需的全部列，那就不得不没扫描一条索引记录就回表查询一次，这都是随机IO，所以索引顺序读取数据的速度要比顺序的全表扫描慢。

MySQL可以用同一个索引既满足排序，又用于查找行。当索引的列顺序和order by子句顺序完全一致，并且所有列的排序方向都一样。order by子句和查询都要满足最左前缀的要求

### 3.5压缩索引

MyISAM使用前缀压缩来减少索引的大小，从而让更多的索引可以放入内存中。默认只压缩字符串，但可以通过参数设置对整数做压缩。MyISAM压缩每个索引块的方法是，先完全报错索引块中的第一个值，然后将其他值和第一个值进行比较的到相同前缀的字节数和不同后缀部分，把这部分存储起来。例如：第一个值为“perform”，第二个位“performance”，那么第二个值的前缀压缩后存储位类似“7，ance”的形式。MyISAM对行指针也采用类似的前缀压缩方式。

压缩块使用更少的空间，代价是某些操作可能更慢，因为每个值的压缩前缀都依赖前面的值，所以无法使用二分查找而只能从头开始扫描。正序的扫描速度还不错，但如果是倒叙扫描--order by DESC，查找某一行的操作平均都需要扫描半个索引块。 

### 3.6冗余索引和重复索引

对于重复索引，很可能是主键--唯一这些性质，这两个本质上都是通过索引来实现的。

冗余索引：（A，B） 和（A）就是冗余索引，（A，B）和（B）并不是。

大多数情况下不需要冗余索引，应该尽量扩展已有的索引而不是创建新索引。但也有时候出于性能考虑需要冗余索引。例如，本来在整数列上有一个索引，现在需要额外增加一个很长的VARCHAR来扩展索引，性能就会急剧下降。

![image-20200901095518574](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20200901095518.png)

![image-20200901095549752](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20200901095549.png)

### 3.7索引和锁

索引可以让查询锁定更少的行，但只有当InnoDB在存储引擎层过滤掉所有不需要的行才有效，如果索引无法过滤掉无效的行，那么当数据返回给服务器层后，MySQL服务器只能应用where子句，此时必须锁定行了。MySQL5.1以后版本InnoDB可以在服务器端过滤掉行后就释放锁，之前的版本只有在事务提交后才释放锁。

![image-20200901101054066](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20200901101054.png)

虽然只返回2-4之间的行，但实际会获取1～4之间的行的排他锁，因为MySQL为该查询选择的执行计划是索引范围扫描，存储引擎做的就是从索引开头获取满足条件actor_id<5的记录，之后就返回给服务器层。

### 3.8避免多个范围条件

对于 ID>20或者 ID IN(2,3,4)这两个用EXPLAIN输出 ，他们的type类型都是range，但是IN()实质上是多个等值条件查询，并不算范围查询，所以MySQL无法在使用范围列后面的其他索引列，但是对于“多个等值条件查询”没有这个限制。

## 4 查询性能优化

### 4.1 为什么查询速度会慢

如果把查询看作一个任务，那么它是由一系列子任务组成的。通常查询的生命周期来看，从客户端，到服务器，然后在服务器上进行解析，生成执行计划，执行，并返回结果给客户端。其中“执行”可以认为是整个生命周期中最重要的阶段，这其中包括大量为了检索数据到存储引擎的调用以及调用后的数据处理，包括排序、分组等。

查询需要在不同的地方花费时间，包括网络，CPU计算，生成统计信息和执行计划、锁等待等操作。

### 4.2 慢查询基础：优化数据访问

查询性能低下的最基本原因是访问数据太多。某些查询不可避免的需要筛选大量数据，但可以通过减少访问数据量的方式进行优化。可以通过以下两个步骤分析：

1. 确认应用程序是否在检索大量超过需要的数据。可能访问了太多的行，也可能访问了太多的列
2. 确认MySQL服务器是否在分析大量超过需要的数据行

#### 4.2.1是否向数据库请求不需要的数据

- 查询不需要的记录

- 多表关联时返回全部列
- 总是取出全部列：不要出现select *，优化器无法完成覆盖扫描，还会为服务器带来额外的IO、内存和CPU的消耗。 但是如果遇到代码的复用性考虑，需要返回多余原先需要的数据，也是可以的。
- 重复查询相同的数据：使用缓存

#### 4.2.2 MySQL是否在扫描额外的记录

可以通过三个指标查看：响应时间、扫描的行数、返回的行数。

**响应时间**

响应时间是两个部分之和：服务时间和排队时间。前者指数据库处理这个查询真正花了多长时间；后者指服务器因为等待某些资源而没有真正执行查询的时间，可能是等IO操作完成，也可能是等待行锁。

**扫描的行数和返回的行数**

理想情况下扫描的行数和返回的行数应该是相同的，但实际情况不大可能，在做一个关联查询时，服务器要扫描多行才能生成结果集中的一行。扫描的行数对返回的行数比率通常很小，一般在1:1和1:10之间，不过有时候也可能很大。

**扫描的行数和访问类型**

从一个表找到某一行数据的成本，可以通过EXPLAIN 的type查看访问类型。访问类型有多种，全表扫描、索引扫描、范围扫描、唯一索引查询、常数引用等。

一般MySQL能够使用三种方式应用WHERE条件，从好到坏一次为：

- 在索引中使用WHERE条件过滤不匹配的记录。在存储引擎层完成。
- 使用索引覆盖扫描（Extra列中出现了 Using index）来返回记录，直接从索引中过滤不需要的记录并返回命中的结果。这是在MySQL服务器层完成的，但无需在回表查询。
- 从数据表返回数据，然后过滤不满足条件的记录（Extra中出现Using Where）。在MySQL服务器层完成，先从数据表读出记录然后过滤。

如果发现查询需要扫描大量的数据但只返回少数行，可以使用以下技巧优化：

1. 使用索引覆盖
2. 改变库表结构
3. 重写这些这个复杂的查询

### 4.3重构查询方式

#### 4.3.1一个复杂查询还是多个简单查询

MySQL内部每秒能扫描内存中上百万行数据，相比之下，MySQL响应数据给客户端就慢得多了。有时候将一个大查询分解为多个小查询是有必要的。

#### 4.3.2 切分查询

对一个大查询分成小查询，每个查询功能一样，只完成一小部分，每次只返回小部分查询结果。

删除旧数据为例，如果用一个大的语句一次性完成的话，可能会锁住很多数据、沾满整个事务日志、耗尽系统资源、阻塞很多小但是重要的查询。

![image-20200901145700426](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20200901145700.png)

一次删除一万行数据一般是比较高效，对服务器影响最小的做法。每次删除数据后们都暂停一会再做下一次删除，可以将服务器上原本的压力分散到很长一段时间，减少删除时锁的持有时间。

#### 4.3.3分解关联查询

![image-20200901165207038](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20200901165207.png)

这样做有以下优势：

- 让缓存的效率更高。
- 将查询分解后，执行单个查询可以减少锁的竞争
- 在应用层做关联，可以更容易对数据库进行拆分，更容易做到高性能和可扩展。
- 查询本身效率会有所提升。使用IN代替关联查询，可以让MySQL按照ID顺序进行查询，比随机关联更高效。
- 减少冗余记录的查询
- 相当于在应用中实现了哈希关联，而不是使用MySQL的嵌套循环关联。

### 4.4 查询执行的基础

先来了解胰腺癌MySQL是如何优化和执行查询的。

![image-20200901170612658](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20200901170612.png)

#### 4.4.1 MySQL客户端/服务器通信协议

半双工通信，所以服务器向客户端发送数据，和客户端像服务器发送数据不可能同时发生。这个协议让MySQL没法进行流量控制，一旦一段开始发送消息，另一端要接受完整个消息才能响应。

一般服务器响应给用户的数据通常很多，由多个数据包组成。当服务器开始响应客户端请求时，客户端必须完整的接受整个返回结果。

多数连接MySQL的库函数都可以获得全部结果集并缓存到内存里，还可以逐行获取需要的数据。默认一般是获得全部结果集并缓存到内存中。MySQL通常是等所有的数据都发送给客户端才释放这条查询所占有的资源，所以接受全部结果并缓存可以减少服务器的压力。

当使用多数连接MySQL的库函数从MySQL获取数据时，实际上是从这个库函数的缓存获取数据。这多数情况没什么问题，但是如果需要返回很大的结果集，库函数会花很多时间和内存来存储所有的结果集。如果能尽早开始处理这些结果集，就能大大减少内存的消耗。

**查询状态**

任何一个MySQL连接，任何时刻都有一个状态。可以使用SHOW FULL PROCESSLIST命令查看

Sleep：线程正在等待客户端发送新的请求

Query：线程正在执行查询或正在讲结果发送给客户端

Locked：在MySQL服务器层，线程正在等待表锁。在存储引擎实现的锁，例如InnoDB的行锁，不会体现在线程状态中。

Analyzing and statistics：线程正在收集存储引擎的统计信息，并生成查询的执行计划

Copying to tmp table [on disk]：线程正在执行查询，并将结果集都复制到一个临时表，这种情况一般要么是做group by，要么是文件排序才做，或者是union操作。如果后面有on disk标记，表示MySQL正在将一个内存临时表放到磁盘上。

Sorting result：线程正在对结果集进行排序

Sending data：有多种情况可能再多个状态之间传送数据，或者生成结果集，或者在向客户端返回数据

#### 4.4.2 查询缓存

在解析一个查询语句之前，如果查询缓存是打开的，MySQL会优先检查这个查询是否命中缓存中的值。这个检查使用大小写敏感的哈希查找。如果命中了查询缓存，在返回结果之前，MySQL会检查一次用户权限。

#### 4.4.3 查询优化处理

将SQL转换成一个执行计划，MySQL在依照这个执行计划和存储引擎进行交互。

**语法解析器和预处理**

MySQL通过关键字将SQL语句进行解析，并生成一颗“解析树”。解析器会使用MySQL语法规则验证和解析查询，如：验证是否使用错误的关键字，或者使用关键字的顺序是否正确。

预处理器根据MySQL规则进一步检查解析数是否合法，如：检查数据表和数据列是否存在

下一步预处理器会验证权限

**查询优化器**

由优化器将语法树转化成执行计划，一条查询可以有很多种执行方式，最后都返回相同的结果。优化器找其中最好的执行计划。

有很多原因导致MySQL优化器选择错误的执行计划：

- 统计信息不准确。MySQL依赖存储引擎提供的统计信息评估成本，但是有的存储引擎提供的信息是准确的没有的偏差会非常大。
- 执行计划中的成本估算不等同于实际执行的成本。有时候某个执行计划孙然读取更多的页面，但是它的成本却更小，因为这些页面都是顺序读或者都存在内存中。
- MySQL的最优可能和我们想象的最优不一样，我们希望执行时间短，但是MySQL只是基于成本模型
- MySQL不是任何时候都基于成本的优化

MySQL的查询优化器是非常复杂的部件，有很多优化策略，简单分为两种，静态优化和动态优化。静态优化可以直接对解析树进行分析，动态优化和查询的上下文有关，也可能和其他因素有关，例如where条件的取值、索引中条目对应的数据行数等。

MySQL对查询的静态优化只做一次，但动态优化在每次执行时都重新评估一次。

下面有一些MySQL能够处理的优化类型：

1. 重新定义关联表的顺序：数据表的关联并补总是按照查询中指定的顺序进行，会由优化器来进行决定。
2. 将外连接转化成内连接：并不是所有的OUTER JOIN语句都必须以外连接的方式执行。
3. 使用等价变换规则：MySQL可以使用一些等价变化来简化并规范表达式，可以合并和减少一些比较，还可以移除一些恒成立和一些恒不成立的判断
4. 优化COUNT()、MIN()、MAX()：索引和列是否可为空通常可以帮助MySQL优化这类表达式。例如：要找到某一列的最小值，只需要查询对于B-Tree索引最左端的记录，MySLQ可以直接获取索引的第一条记录，不需要去表里统计。
5. 预估并转化为常数表达式：当MySQL监测到一个表达式可以转化为常数的时候，会一直把该表达式作为常数进行优化处理，数学表达式就是一个很典型的例子。甚至有时一个查询也能转化为一个常数。
6. 覆盖索引扫描：
7. 子查询优化：MySQL在某些情况下可以讲子查询转换一种效率更高的形式，从而减少多个查询多次对数据进行访问。
8. 提前终止查询：LIMIT
9. 等值传播：如果两个列的值通过等式关联，那么MySQL能够把其中一个列的WHERE条件传递到另一列上。
10. 列表IN()的比较：在很多数据库系统中，IN()等同于多个or条件的子句，因为这两者是完全等价的。但在MySQL中不成立，它会先对列表的数据进行排序，然后通过二分查找的方式来确定列表中的值是否满足条件，这是一个O（logn）操作

**数据和索引的统计信息**

查询优化器在服务器层，但该层没有保存数据和索引的统计信息，统计信息由存储引擎实现。所以MySQL查询优化器在生成查询的执行计划时，需要想存储引擎获取相应的统计信息。存储引擎提供给优化器对应的统计信息，包括：每个表或者索引有多少页、每个表的每个索引的基数是多少、数据行和索引长度等。

**MySQL如何执行关联查询**

MySQL认为任何一次查询都是’关联‘。所以需要理解MySQL如何执行关联查询，UNION查询例子，MySQL先将一系列的单个查询结果放到一个临时表，然后在重新读出临时表数据来完成UNION查询。所以读取结果临时表也是一次关联。

MySQL对任何关联都执行嵌套循环关联操作，即MySQL先在一个表中循环取出单条数据，然后在嵌套循环到下一个表中寻找匹配的行，一次执行下去。

![image-20200912132215058](/Users/zhupeihao/Library/Application Support/typora-user-images/image-20200912132215058.png)

**关联查询优化器**

MySQL优化器最重要的一部分就是关联查询优化，它决定了多个表关联时的顺序。通常多表关联的时候，可以有多种不同的关联顺序来获得相同的执行结果。关联查询优化器则通过评估不同顺序时的成本来选择一个代价小的关联顺序。

不过有时候，优化器给出的并不是最优的关联顺序，这时可以用STRAIGHT_JOIN关键字重写查询。

关联优化器会尝试在所有的关联顺序中选择一个成本最小的生成执行计划数，如果有超过n个表的关联，需要检查n阶乘。当搜索空间很大时，优化器会使用“贪婪”搜索查找最优的关联顺序。实际上， 当关联的表超过optimizer_search_depth的限制时，就会使用这个模式。

**排序优化**

可以通过MySQL索引进行排序，当不能使用索引排序的时候，MySQL会自己进行排序，如果数据量小则在内存中进行，如果数据量大则需要使用磁盘，不过MySQL将这个过程统一称为文件排序(filesort) 。

如果需要排序的数据量小于“排序缓冲区”，MySQL使用内存快排，如果内存不够，就先将数据分块，对每个独立的块使用快排，并将各个块的排序结果存放在磁盘上，然后将各个块进行合并。

MySQL在进行文件排序的时候要用临时存储空间比原表大很多，因为MySQL会对每一个排序记录都分配一个足够长的定长空间来存放。

在关联查询的时候如果需要排序，MySQL会分两种情况处理：如果ORDER BY的列都来自关联的第一个表，那么在处理第一个表的时候就会进行文件排序，这样EXPLAIN的Extra字段会显示'Using filesort'。除此之外的情况，MySQL会先将关联的结果存放到一个临时表，然后在所有的关联都结束后，在进行文件排序。此时EXPLAIN的Extra字字段会显示'Using temporary, Using filesort'。如果使用LIMIT，MySQL不会多所有的结果进行排序，而是根据实际情况，抛弃不满足条件的结果。