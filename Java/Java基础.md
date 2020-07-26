## 重写和重载的区别

1. **重写**

   在继承体系中，子类实现了父类中申明的方法

   为了满足里式替换原则，重写有以下三个限制：

   - 子类方法的访问权限必须大于等于父类方法；
   - 子类方法的返回类型必须是父类方法返回类型或为其子类型。
   - 子类方法抛出的异常类型必须是父类抛出异常类型或为其子类型。

   使用@Override注解，可以让编译器帮忙检查是否满足上面三个条件。

2. **重载**

   存在于同一个类中，指一个方法与已经存在的方法名称，返回类型相同，但是参数类型、个数、顺序至少有一个不同。

## 包装类型与缓存池

每一个基本类型都有一个包装类型与之对应

```java
Integer x = 2;     // 装箱 调用了 Integer.valueOf(2)
int y = x;         // 拆箱 调用了 X.intValue()

```

**缓存池**

new Integer(123) 与 Integer.valueOf(123) 的区别在于：

- new Integer(123) 每次都会新建一个对象；
- Integer.valueOf(123) 会使用缓存池中的对象，多次调用会取得同一个对象的引用。

Integer 缓存池的大小默认为 -128~127。

编译器在自动装箱的过程中调用的是valueOf()，所以

```java
Integer m = 123;
Integer n = 123;
System.out.println(m == n); // true

```

## 字符串常量池

String Pool 保存着所有字符串字面量（literal strings），这些字面量在编译时期就确定。不仅如此，还可以使用 String 的 intern() 方法在运行过程将字符串添加到 String Pool 中。

当一个字符串调用 intern() 方法时，如果 String Pool 中已经存在一个字符串和该字符串值相等（使用 equals() 方法进行确定），那么就会返回 String Pool 中字符串的引用；否则，就会在 String Pool 中添加一个新的字符串，并返回这个新字符串的引用。

```java
//下面两个是字面量形式
String s5 = "bbb";
String s6 = "bbb";
System.out.println(s5 == s6);  // true
 
String s1 = new String("aaa");
String s2 = new String("aaa");
System.out.println(s1 == s2);           // false

```

String Pool 在Java 7之前被放在运行时常量池，但是之后被移到了堆。因为永久代的空间有限，在大量食用字符串的场景下会导致OOM错误

**new String("aaa")**

会创建两个字符串对象（如果String Pool中没有这个字符串的话）

- 首先编译器，会在String Pool中创建一个字符串对象，
- 使用new 在堆中创建一个字符串对象



### Java 参数是传值 不是传引用

https://blog.csdn.net/qq_39751320/article/details/106193277



### HashMap

cnblogs.com/Young111/p/11519952.html?utm_source=gold_browser_extension

- **HashMap的工作原理**

HashMap 底层是 hash 数组和单向链表实现，数组中的每个元素都是链表，由 Node 内部类（实现 Map.Entry接口）实现，HashMap 通过 put & get 方法存储和获取。

存储对象时，将 K/V 键值传给 put() 方法：

①、调用 hash(K) 方法计算 K 的 hash 值，然后结合数组长度，计算得数组下标；

②、调整数组大小（当容器中的元素个数大于 capacity * loadfactor 时，容器会进行扩容resize 为 2n）；

③、
i.如果 K 的 hash 值在 HashMap 中不存在，则执行插入，若存在，则发生碰撞；

ii.如果 K 的 hash 值在 HashMap 中存在，且它们两者 equals 返回 true，则更新键值对；

iii. 如果 K 的 hash 值在 HashMap 中存在，且它们两者 equals 返回 false，则插入链表的尾部（尾插法）或者红黑树中（树的添加方式）。（JDK 1.7 之前使用头插法、JDK 1.8 使用尾插法）（注意：当碰撞导致链表大于 TREEIFY_THRESHOLD = 8 时，就把链表转换成红黑树）

获取对象时，将 K 传给 get() 方法：①、调用 hash(K) 方法（计算 K 的 hash 值）从而获取该键值所在链表的数组下标；②、顺序遍历链表，equals()方法查找相同 Node 链表中 K 值对应的 V 值。

<u>hashCode 是定位的，存储位置；equals是定性的，比较两者是否相等。</u>

```java
 public final boolean equals(Object o) {
            if (o == this)
                return true;
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>)o;
                if (Objects.equals(key, e.getKey()) &&
                    Objects.equals(value, e.getValue()))
                    return true;
            }
            return false;
        }
```



- **当两个对象的 hashCode 相同会发生什么？**

因为 hashCode 相同，不一定就是相等的（equals方法比较），所以两个对象所在数组的下标相同，"碰撞"就此发生。又因为 HashMap 使用链表存储对象，这个 Node 会存储到链表中。

- **你知道 hash 的实现吗？为什么要这样实现？**

JDK 1.8 中，是通过 hashCode() 的高 16 位异或低 16 位实现的：(h = k.hashCode()) ^ (h >>> 16)，主要是从速度，功效和质量来考虑的，减少系统的开销，也不会造成因为高位没有参与下标的计算，从而引起的碰撞。

```java
static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

**这里可能会有疑问，为什么要将hashcode右移16位？**

因为在后续取模运算时，使用了以下代码，可以通过位运算，快速的获取低位的1，也就是余数。所以说真真参与运算的，往往是低位的数，很可能会产生碰撞，所以通过将高16位的树右移在异或，可以有效的打乱低16位，避免碰撞。

```java
h & (length-1)
  
  举例子
y       : 10110010
x-1     : 00001111    //x为16
y&(x-1) : 00000010		//仅获取低4位的值，就是余数

```

**为什么要使用异或运算呢？**

保证了对象的 hashCode 的 32 位值只要有一位发生改变，整个 hash() 返回值就会改变。尽可能的减少碰撞。



- **HashMap 的 table 的容量如何确定？loadFactor 是什么？该容量如何变化？这种变化会带来什么问题？**

①、table 数组大小是由 capacity 这个参数确定的，默认是16，也可以构造时传入，最大限制是1<<30；

②、loadFactor 是装载因子，主要目的是用来确认table 数组是否需要动态扩展，默认值是0.75，比如table 数组大小为 16，装载因子为 0.75 时，threshold 就是12，当 table 的实际大小超过 12 时，table就需要动态扩容；

③、扩容时，调用 resize() 方法，将 table 长度变为原来的两倍（注意是 table 长度，而不是 threshold）

④、如果数据很大的情况下，扩展时将会带来性能的损失，在性能要求很高的地方，这种损失很可能很致命。

- **put方法的过程**

答：“调用哈希函数获取Key对应的hash值，再计算其数组下标；

如果没有出现哈希冲突，则直接放入数组；如果出现哈希冲突，则以链表的方式放在链表后面；

如果链表长度超过阀值( TREEIFY THRESHOLD==8)，就把链表转成红黑树，链表长度低于6，就把红黑树转回链表;

如果结点的key已经存在，则替换其value即可；

如果集合中的键值对大于12，调用resize方法进行数组扩容。”

- **数组扩容的过程？**

创建一个新的数组，其容量为旧数组的两倍，并重新计算旧数组中结点的存储位置（非一致性哈希）。结点在新数组中的位置只有两种，原下标位置或原下标+旧数组的大小。

- **拉链法导致的链表过深问题为什么不用二叉查找树代替，而选择红黑树？为什么不一直使用红黑树？**

之所以选择红黑树是为了解决二叉查找树的缺陷，二叉查找树在特殊情况下会变成一条线性结构（这就跟原来使用链表结构一样了，造成很深的问题），遍历查找会非常慢。

而红黑树在插入新数据后可能需要通过左旋，右旋、变色这些操作来保持平衡，引入红黑树就是为了查找数据快，解决链表查询深度的问题，我们知道红黑树属于平衡二叉树，但是为了保持“平衡”是需要付出代价的，但是该代价所损耗的资源要比遍历线性链表要少，所以当长度大于8的时候，会使用红黑树，如果链表长度很短的话，根本不需要引入红黑树，引入反而会慢。

- **说说你对红黑树的见解？**

1. 每个节点非红即黑
2. 根节点总是黑色的
3. 如果节点是红色的，则它的子节点必须是黑色的（反之不一定）
4. 每个叶子节点都是黑色的空节点（NIL节点）
5. 从根节点到叶节点或空子节点的每条路径，必须包含相同数目的黑色节点（即相同的黑色高度）

- **jdk8中对HashMap做了哪些改变？**

在java 1.8中，如果链表的长度超过了8，那么链表将转换为红黑树。（桶的数量必须大于64，小于64的时候只会扩容）

发生hash碰撞时，java 1.7 会在链表的头部插入，而java 1.8会在链表的尾部插入

在java 1.8中，Entry被Node替代(换了一个马甲)。

- **HashMap，LinkedHashMap，TreeMap 有什么区别？**

LinkedHashMap 保存了记录的插入顺序，在用 Iterator 遍历时，先取到的记录肯定是先插入的；遍历比 HashMap 慢；

TreeMap 实现 SortMap 接口，能够把它保存的记录根据键排序（默认按键值升序排序，也可以指定排序的比较器）

**使用场景：**

HashMap：使用最广泛

LinkedHashMap：需要输出的顺序和插入的顺序相同情况下

TreeMap：按自然顺序或自定义顺序遍历键

- **HashMap 和 HashTable 有什么区别？**

①、HashMap 是线程不安全的，HashTable 是线程安全的；

②、由于线程安全，所以 HashTable 的效率比不上 HashMap；

③、HashMap最多只允许一条记录的键为null，允许多条记录的值为null，而 HashTable不允许；

④、HashMap 默认初始化数组的大小为16，HashTable 为 11，前者扩容时，扩大两倍，后者扩大两倍+1；

⑤、HashMap 需要重新计算 hash 值，而 HashTable 直接使用对象的 hashCode

- **HashMap & ConcurrentHashMap 的区别？**

除了加锁，原理上无太大区别。另外，HashMap 的键值对允许有null，但是ConCurrentHashMap 都不允许。

- **为什么 ConcurrentHashMap 比 HashTable 效率要高？**

HashTable 使用一把锁synchronized（锁住整个链表结构）处理并发问题，多个线程竞争一把锁，容易阻塞，当一个线程put元素的时候，其他线程不仅能put，也不能get；

ConcurrentHashMap

1. JDK 1.7 中使用分段锁（ReentrantLock + Segment + HashEntry），相当于把一个 HashMap 分成多个段，每段分配一把锁，这样支持多线程访问。锁粒度：基于 Segment（锁），包含多个 HashEntry（桶）。默认分配16个segment，比hashtable效率提高16倍。

   ![img](https://cs-notes-1256109796.cos.ap-guangzhou.myqcloud.com/image-20191209001038024.png)

1. JDK 1.8 中使用 CAS + synchronized + Node + 红黑树。锁粒度：Node（首结点）（实现 Map.Entry）。锁粒度降低了。 JDK8中的实现也是锁分离的思想，它把锁分的比segment（JDK1.5）更细一些，只要hash不冲突，就不会出现并发获得锁的情况。它首先使用无锁操作CAS插入头结点，如果插入失败，说明已经有别的线程插入头结点了，再次循环进行操作。如果头结点已经存在，则通过synchronized获得头结点锁，进行后续的操作。性能比segment分段锁又再次提升。

   

- **ConcurrentHashMap 简单介绍？**

①、重要的常量：

private transient volatile int sizeCtl;

当为负数时，-1 表示正在初始化，-N 表示 N - 1 个线程正在进行扩容；

当为 0 时，表示 table 还没有初始化；

当为其他正数时，表示初始化或者下一次进行扩容的大小。

②、数据结构：

Node 是存储结构的基本单元，继承 HashMap 中的 Entry，用于存储数据；

TreeNode 继承 Node，但是数据结构换成了二叉树结构，是红黑树的存储结构，用于红黑树中存储数据；

TreeBin 是封装 TreeNode 的容器，提供转换红黑树的一些条件和锁的控制。

③、存储对象时（put() 方法）：

如果没有初始化，就调用 initTable() 方法来进行初始化；

如果没有 hash 冲突就直接 CAS 无锁插入；

如果需要扩容，就先进行扩容；

如果存在 hash 冲突，就加锁来保证线程安全，两种情况：一种是链表形式就直接遍历到尾端插入，一种是红黑树就按照红黑树结构插入；

如果该链表的数量大于阀值 8，就要先转换成红黑树的结构，break 再一次进入循环

如果添加成功就调用 addCount() 方法统计 size，并且检查是否需要扩容。

④、扩容方法 transfer()：默认容量为 16，扩容时，容量变为原来的两倍。

helpTransfer()：调用多个工作线程一起帮助进行扩容，这样的效率就会更高。

⑤、获取对象时（get()方法）：

计算 hash 值，定位到该 table 索引位置，如果是首结点符合就返回；

如果遇到扩容时，会调用标记正在扩容结点 ForwardingNode.find()方法，查找该结点，匹配就返回；

以上都不符合的话，就往下遍历结点，匹配就返回，否则最后就返回 null。

- **ConcurrentHashMap 的并发度是什么？**

**程序运行时能够同时更新 ConccurentHashMap 且不产生锁竞争的最大线程数。**默认为 16，且可以在构造函数中设置。

当用户设置并发度时，ConcurrentHashMap 会使用大于等于该值的最小2幂指数作为实际并发度（假如用户设置并发度为17，实际并发度则为32）

- HashMap线程不安全的地方

https://www.jianshu.com/p/e2f75c8cce01

有两个场景

1. put的时候导致多线程的数据不一致

   这个问题比较好想象，比如有两个线程A和B，首先A希望插入一个key-value对到HashMap中，首先计算记录所要落到的桶的索引坐标，然后获取到该桶里面的链表头结点，此时线程A的时间片用完了，而此时线程B被调度得以执行，和线程A一样执行，只不过线程B成功将记录插到了桶里面，假设线程A插入的记录计算出来的桶索引和线程B要插入的记录计算出来的桶索引是一样的，那么当线程B成功插入之后，线程A再次被调度运行时，它依然持有过期的链表头但是它对此一无所知，以至于它认为它应该这样做，如此一来就覆盖了线程B插入的记录，这样线程B插入的记录就凭空消失了，造成了数据不一致的行为。

   

2. get操作可能因为resize引起死循环

   ```java
   void transfer(Entry[] newTable, boolean rehash) {  
           int newCapacity = newTable.length;  
           for (Entry<K,V> e : table) {  
     
               while(null != e) {  
                   Entry<K,V> next = e.next;           
                   if (rehash) {  
                       e.hash = null == e.key ? 0 : hash(e.key);  
                   }  
                   int i = indexFor(e.hash, newCapacity);   
                 //头插法
                   e.next = newTable[i];  
                   newTable[i] = e;  
                   e = next;  
               } 
           }  
       }  
   ```

   图片地址：

https://upload-images.jianshu.io/upload_images/7853175-ab75cd3738471507.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp

我们假设有两个线程同时需要执行resize操作，我们原来的桶数量为2，记录数为3，需要resize桶到4，原来的记录分别为：[3,A],[7,B],[5,C]，在原来的map里面，我们发现这三个entry都落到了第二个桶里面。
 假设线程thread1执行到了transfer方法的Entry next = e.next这一句，然后时间片用完了，此时的e = [3,A], next = [7,B]。线程thread2被调度执行并且顺利完成了resize操作，需要注意的是，此时的[7,B]的next为[3,A]。此时线程thread1重新被调度运行，此时的thread1持有的引用是已经被thread2 resize之后的结果。线程thread1首先将[3,A]迁移到新的数组上，然后再处理[7,B]，而[7,B]被链接到了[3,A]的后面，处理完[7,B]之后，就需要处理[7,B]的next了啊，而通过thread2的resize之后，[7,B]的next变为了[3,A]，此时，[3,A]和[7,B]形成了环形链表，在get的时候，如果get的key的桶索引和[3,A]和[7,B]一样，那么就会陷入死循环。

如果在取链表的时候从头开始取（现在是从尾部开始取）的话，则可以保证节点之间的顺序，那样就不存在这样的问题了。

综合上面两点，可以说明HashMap是线程不安全的。

