[TOC]



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

## Java常量池

什么是常量：

- final修饰的成员变量是常量，值一旦给定就无法修改
- final修饰的变量有三种：静态变量、实例变量和局部变量

Java内存分配中，有三种常量池：字符串常量池、class文件常量池、运行时常量池

#### 字符串常量池

String Pool 在Java 7之前被放在运行时常量池，但是之后被移到了堆。因为永久代的空间有限，在大量使用字符串的场景下会导致OOM错误

实现String Pool 功能的是一个StringTable类，它是一个Hash表，默认大小1009；**存储的是字符串的引用（不是字符串本身）**。

StringTable本质就是一个HashSet<String>，如果String Pool中的String很多，但是Hash表设置的太小，容易造成Hash冲突，导致链表过长，那么使用String/intern()方法是会去链表中一个个找，效率低下。

所以可以通过参数指定StringTable的大小

```java
-XX:StringTableSize=66666
```



```java
String s1 = "abc";  // 1
String s2 = "abc";	// 2
String s3 = "xxx";	// 3
```

![img](https://pic2.zhimg.com/80/v2-5d7f1fddd6684c628ce87a6145ab3dcc_720w.jpg)

- 执行第一行，解析时在字符串常量池中没有发现"abc"的引用，所以去堆里创建一个"abc"对象，然后把这个对象的引用存在字符串常量池中，最后把这个引用返回给s1
- 执行第二行，解析时发现字符串常量池中已经有"abc"的引用，所以直接返回引用给s2
- 执行第三行，分析过程和第一行一样

还可以使用 String 的 intern() 方法在运行过程将字符串引用添加到 String Pool 中。

当一个字符串调用 intern() 方法时，如果 String Pool 中已经存在一个字符串和该字符串值相等（使用 equals() 方法进行确定），那么就会返回 String Pool 中字符串的引用；否则，就会在 String Pool 中添加该字符串的引用，并返回这个引用。

```java
String s1 = "ab";//#1
String s2 = new String(s1+"d");//#2
s2.intern();//#3
String s4 = "xxx";//#4
String s3 = "abd";//#5
System.out.println(s2 == s3);//true
```

![img](https://pic1.zhimg.com/80/v2-038133505bc57cac2950a49f34877f3d_720w.jpg)

- 第二行，内部创建一个StringBuilder对象执行append()操作，最后执行toString（）方法创建一个新的String对象"abd"，然后直接返回给s2。**注意，此时并没有把引用放入字符串常量池中**
- 第三行，intern()首先会去字符串常量池中查找是否有"abd"的引用，发现没有，就把堆里"abd"对象的引用存入到字符串常量池中，并返回这个引用。
- 执行第五行时，因为常量池中已经有这个"abd"对象的引用，所以直接返回给s3

**new String("aaa")**

会创建两个字符串对象（如果String Pool中没有这个字符串的话）

- 首先编译器，会在String Pool中创建一个字符串对象，
- 使用new 在堆中创建一个字符串对象

**String str1=new String("A"+"B")；会创建多少个对象**

"A"+"B"在编译的时候会被JVM直接认为是"AB"，所以实际上就是判断new String("AB")会创建多少对象，和上面一样。

**String str2=new String("ABC")+"ABC"；会创建多少个对象**

字符串常量池中："ABC"

堆中：new String("ABC")一个，以及"ABCABC"

```java
String str1 = "abc";
String str2 = new String(str1);
System.out.println(str1==str2);
System.out.println(str1.hashCode());
System.out.println(str2.hashCode());
```

输出结果
false
96354
96354

str1和str2的hashcode还有指向的内部字符串数组都相同，但是两个不是同一个对象

#### class文件常量池

class 文件中除了包含类的版本、字段、方法、接口等描述信息外，还有一项信息就是常量池(constant pool table)，用于存放编译器生成的`各种字面量(Literal)和符号引用(Symbolic References)`。



#### 运行时常量池

方法区的一部分，当Java文件被编译成class文件之后，会生成class文件常量池。

JVM执行某个类的时候，需要经过加载、连接、初始化过程。当class文件加载到内存后，JVM会将class文件常量池中的内容存放到运行时常量池中。class常量池中存的是字面量和符号引用，也就是说他们存储的并不是对象实例，而是对象的符号引用。那么经过连接（验证，准备，解析）里面的解析后，会将符号引用替换为直接引用

### Java 参数是传值 不是传引用

https://blog.csdn.net/qq_39751320/article/details/106193277



## HashMap

cnblogs.com/Young111/p/11519952.html?utm_source=gold_browser_extension

- **HashMap的工作原理**

HashMap 底层是 hash 数组和单向链表实现，数组中的每个元素都是链表，由 Node 内部类（实现 Map.Entry接口）实现，HashMap 通过 put & get 方法存储和获取。



**为什么采用尾插法**

因为头插法在resize的时候可能会产生环形链表

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

因为在后续取模运算时，使用了以下代码，可以通过位运算，快速的获取低位的1，也就是余数。所以说真正参与运算的，往往是低位的数，很可能会产生碰撞，所以通过将高16位的数右移在异或，可以有效的打乱低16位，避免碰撞。

```java
h & (length-1)
  
  举例子
y       : 10110010
x-1     : 00001111    //x为16
y&(x-1) : 00000010		//仅获取低4位的值，就是余数

```

> 所以说一般我们使用hashmap，容量不会超过2^16，所以说每次都是取低16位的值。通过将高16位右移能有效打乱低16位，避免碰撞。

**为什么要使用异或运算呢？**

保证了对象的 hashCode 的 32 位值只要有一位发生改变，整个 hash() 返回值就会改变。尽可能的减少碰撞。



- **HashMap 的 table 的容量如何确定？loadFactor 是什么？该容量如何变化？这种变化会带来什么问题？**

①、table 数组大小是由 capacity 这个参数确定的，默认是16，也可以构造时传入，最大限制是1<<30；

②、loadFactor 是装载因子，主要目的是用来确认table 数组是否需要动态扩展，默认值是0.75，比如table 数组大小为 16，装载因子为 0.75 时，threshold 就是12，当 table 的实际大小超过 12 时，table就需要动态扩容；

③、扩容时，调用 resize() 方法，将 table 长度变为原来的两倍（注意是 table 长度，而不是 threshold）

④、如果数据很大的情况下，扩展时将会带来性能的损失，在性能要求很高的地方，这种损失很可能很致命。

- **put方法的过程**

①、调用 hash(K) 方法计算 K 的 hash 值，然后结合数组长度，计算得数组下标；

②、调整数组大小（当容器中的元素个数大于 capacity * loadfactor 时，容器会进行扩容resize 为 2*capacity）；

③、
i.如果 K 的 hash 值在 HashMap 中不存在，则执行插入，若存在，则发生碰撞；

ii.如果 K 的 hash 值在 HashMap 中存在，且它们两者 equals 返回 true，则更新键值对；

iii. 如果 K 的 hash 值在 HashMap 中存在，且它们两者 equals 返回 false，则插入链表的尾部（尾插法）或者红黑树中（树的添加方式）。（JDK 1.7 之前使用头插法、JDK 1.8 使用尾插法）（注意：当碰撞导致链表大于 TREEIFY_THRESHOLD = 8 时，就把链表转换成红黑树）

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

- **为什么HashMap的负载因子是0.75**

https://www.jianshu.com/p/eb71278b641e

核心就是：要保证空的桶数量多于一半，因为一旦空的桶少于一半，意味着一个新的数据进来有超过50%的概率会进入放有数据的桶（即产生Hash冲突），但是负载因子是针对放入HashMap的元素个数，而不是针对空桶的个数，所以要计算的是当一半桶被占用的时候，实际存储的数据量是大于0.5n

![image-20201031141342956](https://cdn.jsdelivr.net/gh/Peihao-Zhu/blogImage@master/data/20201031141343.png)

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

  https://www.jianshu.com/p/a7767e6ff2a2

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



### HashMap 8为什么先插入元素在判断扩容？

https://blog.csdn.net/LovePluto/article/details/79755496



### ConcurrentHashMap1.8 - 扩容详解

https://blog.csdn.net/ZOKEKAI/article/details/90051567

### HashMap和ConcurrentHashMap知识点总结

https://www.jianshu.com/p/a7767e6ff2a2

## ArrayList/LinkedList 的区别

**ArrayList**

基于数组实现（需要连续的存储空间），默认大小是10。添加元素时如果数组容量不够，就使用grow（）方法进行扩容，新容量的大小是原大小的1.5倍。扩容操作需要调用Arrays.copyOf()把原数组整个复制到新数组中。可见扩容的代价很高

```java
oldCapacity + (oldCapacity >> 1)
```



**Fail-Fast**

modeCount用来记录ArrayList结构发生变化的次数，在进行序列化或者迭代等操作时，需要比较前后的modCount是否改变，如果改变了会抛出ConcurrentModificationException



**LinkedList**

双向链表实现，不连续的存储空间



**两者比较**

- 查找

如果是查找第n个元素，`ArrayList`更有优势，因为是连续的内存空间，可以通过起始地址+偏移量获得第n个元素的内存地址，但是`LinkedList`不连续，无法计算偏移量，只能一个一个找。

如果是查找具体的‘ele' 那么都需要一个一个遍历，两者差不多

- 插入

如果从中间或者开头插入，那`ArrayList` 会很慢，因为需要将后面的大量数据通过调用Systems.arraycopy()往后移动，而`LinkedList`只需要改变指针就好了

如果往末尾插入元素，`ArrayList` 可以直接计算出内存地址，然后插入；`LinkedList`也有指针可以直接指向尾部

- 删除

和插入类似

## final/static关键字

### final

https://blog.csdn.net/zhaotengfei36520/article/details/45098977

**1.数据**

声明数据为常量，可以是编译时常量，也可以是在运行时被初始化后不能被改变的常量。

- 对于基本类型，final 使数值不变；
- 对于引用类型，final 使引用不变，也就是不能引用其它对象，但是被引用的对象本身是可以修改的。

```java
final int x = 1;
// x = 2;  // cannot assign value to final variable 'x'
final A y = new A();
y.a = 1;
```

**2.方法**

声明方法不能被子类重写，但是可以被继承。

**private 方法隐式地被指定为 final**，如果在子类中定义的方法和基类中的一个 private 方法相同，此时子类的方法不是重写基类方法，而是在子类中定义了一个新的方法。

**3.类**

声明类不允许被继承,final类中的成员方法被隐式的指定为final方法

**注意：final不能修饰抽象类和接口**

说明：使用final的原因。

- 把方法锁定，防止任何继承类修改它的含义
- 提升效率，会讲final方法转为内嵌调用。

### static

**1.静态变量**

- 静态变量：又称为类变量，也就是说这个变量属于类的，类所有的实例都共享静态变量，可以直接通过类名来访问它。静态变量在内存中只存在一份（在方法区）。
- 实例变量：每创建一个实例就会产生一个实例变量，它与该实例同生共死。

**2.静态方法**

静态方法在类加载的时候就存在了，它不依赖于任何实例。所以静态方法必须有实现，也就是说它不能是抽象方法。

```java
public abstract class A {
    public static void func1(){
    }
    // public abstract static void func2();  // Illegal combination of modifiers: 'abstract' and 'static'
}

```

只能访问所属类的静态字段和静态方法，方法中不能有 this 和 super 关键字，因此这两个关键字与具体对象关联。

**3.静态语句块**

被定义在类中方法外。静态语句块在类初始化时运行一次。

**4.静态内部类**

非静态内部类在编译完成后会隐含的保存着一个引用，指向创建它的外围类，但静态内部类没有。这意味着：

1.非静态内部类依赖于外部类的实例，也就是说需要先创建外部类实例，才能用这个实例去创建非静态内部类。而静态内部类不需要。

2.静态内部类不能访问外部类的非静态的变量和方法。

**5.静态导包**

`import static` 导入某个类中的指定静态资源，并且不需要使用类名调用类中静态成员

**6.初始化顺序**

静态变量和静态语句块优先于实例变量和普通语句块，静态变量和静态语句块的初始化顺序取决于它们在代码中的顺序。

最后才是构造函数的初始化

在继承的情况下，初始化顺序为：

- 父类（静态变量、静态语句块）
- 子类（静态变量、静态语句块）
- 父类（实例变量、普通语句块）
- 父类（构造函数）
- 子类（实例变量、普通语句块）
- 子类（构造函数）

**静态代码块，非静态代码块以及静态方法的区别**

静态代码块可能在第一次new执行，也可能在反射调用的时候执行，非静态代码块每new一次都会执行。

静态代码块是自动执行，而静态方法是被调用才会执行。

**非静态代码与构造函数的区别**

非静态代码块是给所有对象进行统一初始化，而构造函数是对应的对象初始化。

## 继承

### 访问权限

Java 中有三个访问权限修饰符：private、protected 以及 public，如果不加访问修饰符，表示包级可见。

可以对类或类中的成员（字段和方法）加上访问修饰符。

- 类可见表示其它类可以用这个类创建实例对象。
- 成员可见表示其它类可以用这个类的实例对象访问到该成员；

如果子类的方法重写了父类的方法，那么子类中该方法的访问级别不允许低于父类的访问级别。这是为了确保可以使用父类实例的地方都可以使用子类实例去代替，也就是确保满足里氏替换原则。

字段绝不能是公有的，因为这么做的话就失去了对这个字段修改行为的控制，客户端可以对其随意修改。例如下面的例子中，AccessExample 拥有 id 公有字段，如果在某个时刻，我们想要使用 int 存储 id 字段，那么就需要修改所有的客户端代码。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190212171052701.png)

- 类中不加任何修饰符就是default，默认对于同一个包中的其他类是公开的。
- protected修饰的类、方法、属性对于同一个包中的类，或者不同包中的子类相当于公开的。

讲述的时候建议有逻辑性一层一层的来，

```java
    1、private，私有的，被private修饰的类、方法、属性、只能被本类的对象所访问。----- 我什么都不跟别人分享。只有自己知道。

    2、default，默认的，在这种模式下，只能在同一个包内访问。----我的东西可以和跟我一块住的那个人分享。

    3、protected，受保护的，被protected修饰的类、方法、属性、只能被本类、本包、不同包的子类所访问。-----我的东西我可以和跟我一块住的那个人分享。另外也可以跟不在家的儿子分享消息，打电话

    4、public，公共的，被public修饰的类、方法、属性、可以跨类和跨包访问。----- 我的东西大家任何人都可以分享。

```

**注意：Java中外部类的修饰符之能是default或public，类的成员或内部类可以是以上四种**

### 抽象类和接口

**抽象类**

抽象类和抽象方法都使用 abstract 关键字进行声明。如果一个类中包含抽象方法，那么这个类必须声明为抽象类。

- 抽象类和普通类最大的区别是，抽象类不能被实例化，只能被继承。但是抽象类可以有构造方法，供子类创建对象，初始化父类使用。

- 子类如果实现抽象类，就必须实现父类所有的抽象方法

  

**接口**

接口是抽象类的延伸，在 Java 8 之前，它可以看成是一个完全抽象的类，也就是说它不能有任何的方法实现。

从 Java 8 开始，接口也可以拥有默认的方法实现，这是因为不支持默认方法的接口的维护成本太高了。在 Java 8 之前，如果一个接口想要添加新的方法，那么要修改所有实现了该接口的类，让它们都实现新增的方法。

接口的成员（字段 + 方法）默认都是 public 的，并且不允许定义为 private 或者 protected。

接口的字段默认都是 static 和 final 的（可以直接通过接口名来访问）。

**比较**

- 从设计层面上看，抽象类提供了一种 IS-A 关系，需要满足里式替换原则，即子类对象必须能够替换掉所有父类对象。而接口更像是一种 LIKE-A 关系，它只是提供一种方法实现契约，并不要求接口和实现接口的类具有 IS-A 关系。
- 从使用上来看，一个类可以实现多个接口，但是不能继承多个抽象类。
- 接口的字段只能是 static 和 final 类型的，而抽象类的字段没有这种限制。
- 接口的成员只能是 public 的，而抽象类的成员可以有多种访问权限。

**使用选择**

使用接口：

- 需要让不相关的类都实现一个方法，例如不相关的类都可以实现 Comparable 接口中的 compareTo() 方法；
- 需要使用多重继承。

使用抽象类：

- 需要在几个相关的类中共享代码。
- 需要能控制继承来的成员的访问权限，而不是都为 public。
- 需要继承非静态和非常量字段。

在很多情况下，接口优先于抽象类。因为接口没有抽象类严格的类层次结构要求，可以灵活地为一个类添加行为。并且从 Java 8 开始，接口也可以有默认的方法实现，使得修改接口的成本也变的很低。

### super

- 访问父类的构造函数：可以使用 super() 函数访问父类的构造函数，从而委托父类完成一些初始化的工作。应该注意到，子类一定会调用父类的构造函数来完成初始化工作，一般是调用父类的默认构造函数，如果子类需要调用父类其它构造函数，那么就可以使用 super() 函数。
- 访问父类的成员：如果子类重写了父类的某个方法，可以通过使用 super 关键字来引用父类的方法实现。



## 反射

每个类都有一个 **Class** 对象，包含了与类有关的信息。当编译一个新类时，会产生一个同名的 .class 文件，该文件内容保存着 Class 对象。

类加载相当于 Class 对象的加载，类在第一次使用时才动态加载到 JVM 中。也可以使用 `Class.forName("com.mysql.jdbc.Driver")` 这种方式来控制类的加载，该方法会返回一个 Class 对象。

反射可以提供运行时的类信息，并且这个类可以在运行时才加载进来，甚至在编译时期该类的 .class 不存在也可以加载进来。

Class 和 java.lang.reflect 一起对反射提供了支持，java.lang.reflect 类库主要包含了以下三个类：

- **Field** ：可以使用 get() 和 set() 方法读取和修改 Field 对象关联的字段；
- **Method** ：可以使用 invoke() 方法调用与 Method 对象关联的方法；
- **Constructor** ：可以用 Constructor 的 newInstance() 创建新的对象。

**反射的优点：**

- **可扩展性** ：应用程序可以利用全限定名创建可扩展对象的实例，来使用来自外部的用户自定义类。
- **类浏览器和可视化开发环境** ：一个类浏览器需要可以枚举类的成员。可视化开发环境（如 IDE）可以从利用反射中可用的类型信息中受益，以帮助程序员编写正确的代码。
- **调试器和测试工具** ： 调试器需要能够检查一个类里的私有成员。测试工具可以利用反射来自动地调用类里定义的可被发现的 API 定义，以确保一组测试中有较高的代码覆盖率。

**反射的缺点：**

尽管反射非常强大，但也不能滥用。如果一个功能可以不用反射完成，那么最好就不用。在我们使用反射技术时，下面几条内容应该牢记于心。

- **性能开销** ：反射涉及了**动态类型的解析**，所以 JVM 无法对这些代码进行优化。因此，反射操作的效率要比那些非反射操作低得多。我们应该避免在经常被执行的代码或对性能要求很高的程序中使用反射。
- **安全限制** ：使用反射技术要求程序必须在一个没有安全限制的环境中运行。如果一个程序必须在有安全限制的环境中运行，如 Applet，那么这就是个问题了。
- **内部暴露** ：由于反射允许代码执行一些在正常情况下不被允许的操作（比如访问私有的属性和方法），所以使用反射可能会导致意料之外的副作用，这可能导致代码功能失调并破坏可移植性。反射代码破坏了抽象性，因此当平台发生改变的时候，代码的行为就有可能也随着变化。

### Class类详解

https://blog.csdn.net/dufufd/article/details/80537638

 在java世界里，一切皆对象。从某种意义上来说，java有两种对象：实例对象和Class对象。每个类的运行时的**类型信息**就是用Class对象表示的。它包含了与类有关的信息。其实我们的**实例对象就通过Class对象来创建**的。Java使用Class对象执行其RTTI（运行时类型识别，Run-Time Type Identification），多态是基于RTTI实现的。

  每一个类都有一个Class对象，每当编译一个新类就产生一个Class对象，基本类型 (boolean, byte, char, short, int, long, float, and double)有Class对象，数组有Class对象，就连关键字void也有Class对象（void.class）。Class对象对应着java.lang.Class类，如果说类是对象抽象和集合的话，那么Class类就是对类的抽象和集合。

  Class类没有公共的构造方法，Class对象是在类加载的时候由**Java虚拟机**以及通过调用类加载器中的 defineClass 方法自动构造的，因此不能显式地声明一个Class对象。一个类被加载到内存并供我们使用需要经历如下三个阶段：

1. **加载**，这是由类加载器（ClassLoader）执行的。通过一个类的全限定名来获取其定义的二进制字节流（Class字节码），将这个字节流所代表的静态存储结构转化为方法区的运行时数据接口，根据字节码在java堆中生成一个代表这个类的java.lang.Class对象。
2. **链接**。在链接阶段将验证Class文件中的字节流包含的信息是否符合当前虚拟机的要求，为静态域分配存储空间并设置类变量的初始值（默认的零值），并且如果必需的话，将常量池中的符号引用转化为直接引用。
3. **初始化**。到了此阶段，才真正开始执行类中定义的java程序代码。用于执行该类的静态初始器和静态初始块，如果该类有父类的话，则优先对其父类进行初始化。


  **所有的类都是在对其第一次使用时，动态加载到JVM中的（懒加载）。**当程序创建第一个对类的静态成员的引用时，就会加载这个类。使用new创建类对象的时候也会被当作对类的静态成员的引用。**因此java程序在它开始运行之前并非被完全加载，其各个类都是在必需时才加载的。**这一点与许多传统语言都不同。动态加载的行为，在诸如C++这样的静态加载语言中是很难或者根本不可能复制的。

  在类加载阶段，类加载器首先检查这个类的Class对象是否已经被加载。如果尚未加载，默认的类加载器就会根据类的全限定名查找.class文件。在这个类的字节码被加载时，它们会接受验证，以确保其没有被破坏，并且不包含不良java代码。一旦某个类的Class对象被载入内存，我们就可以它来创建这个类的所有对象。



**那么如何获取这个Class对象呢？**

- Obj.getClass();
- Class.forName("类名");
- 具体类名.class。 //不会初始化
- xxxClassLoader.loadClass()。//不会初始化

**有了这个Class对象，如何获取类的对象**

可以通过Class对象来调用newInstance()方法，这种形式默认调用的是**无参构造**来返回一个对象。



### Class.forName()/getClass()/Class.class区别

https://www.cnblogs.com/Seachal/p/5371733.html

1.出现时机：**Class.forName()**是在运行时加载，**Class.class** **和getClass（）**是在编译时期加载

2.**Class.forName**是一个静态方法，**Class.class**是所有类的一个属性，**getClass()**是所有对象（Object类的对象）的成员方法



3.生成Class对象

**Class.class** 的形式会使 JVM 用类装载器将类装入内存（前提是类还没有装入内存），**不做类的初始化工作**，返回 Class 对象。

这样做不仅更简单，而且更安全，因为它在编译时就会受到检查(因此不需要置于try语句块中)。并且根除了对forName()方法的调用，所有也更高效。类字面量不仅可以应用于普通的类，也可以应用于接口、数组及基本数据类型。

注意：基本数据类型的Class对象和包装类的Class对象是不一样的：

```java
Class c1 = Integer.class;
Class c2 = int.class;
System.out.println(c1);
System.out.println(c2);
System.out.println(c1 == c2);
/* Output
class java.lang.Integer
int
false
*/
```

**Class.forName()**会调用类加载器去加载类并做**类的静态初始化**，返回 Class 对象。因为内部调用了一个native方法。

**getClass（）** 的形式会对**类进行静态初始化、非静态初始化，返回引用运行时真正所指的对象**（因为子对象的引用可能会赋给父对象的引用变量中）所属的类的 Class 对象。

使用new 创建一个对象实例的时候，就自动加载这个类到内存中，并进行初始化。所以执行d.getClass()的时候就直接从堆中返回该类Class引用

```java
package com.cry;
class Dog {
    static {
        System.out.println("Loading Dog");
    }
}
public class Test {
    public static void main(String[] args) {
        System.out.println("inside main");
        Dog d = new Dog();
        System.out.println("after creating Dog");
        Class c = d.getClass();
        System.out.println("finish main");
    }
}
/*　Output:
    inside main
    Loading Dog
    after creating Dog
    finish main
 */
```

下面的代码表示.class 和Class.forName()是否会引起初始化。

  **如果一个字段被static final修饰，我们称为”编译时常量“，就像Dog的s1字段那样，那么在调用这个字段的时候是不会对Dog类进行初始化的。**因为被static和final修饰的字段，在编译期就把结果放入了常量池中了。但是，如果只是将一个域设置为static 或final的，还不足以确保这种行为，就如调用Dog的s2字段后，会强制Dog进行类的初始化，因为s2字段不是一个编译时常量。

```java
package com.cry;
class Dog {
    static final String s1 = "Dog_s1";
    static  String s2 = "Dog_s2";
    static {
        System.out.println("Loading Dog");
    }
}
class Cat {
    static String s1 = "Cat_s1";
    static {
        System.out.println("Loading Cat");
    }
}
public class Test {
    public static void main(String[] args) throws ClassNotFoundException {
        System.out.println("----Star Dog----");
        Class dog = Dog.class;
        System.out.println("------");
        System.out.println(Dog.s1);
        System.out.println("------");
        System.out.println(Dog.s2);
        System.out.println("---start Cat---");
        Class cat = Class.forName("com.cry.Cat");
        System.out.println("-------");
        System.out.println(Cat.s1);
        System.out.println("finish main");
    }
}
/*　Output:
----Star Dog----
------
Dog_s1
------
Loading Dog
Dog_s2
---start Cat---
Loading Cat
-------
Cat_s1
finish main
 */
```

一旦一个类被加载进内存以后，返回的Class对象都是同一个java堆地址上的Class引用

```java
package com.cry;
class Cat {
    static {
        System.out.println("Loading Cat");
    }
}
public class Test {
    public static void main(String[] args) throws ClassNotFoundException {
        System.out.println("inside main");
        Class c1 = Cat.class;
        Class c2= Class.forName("com.cry.Cat");
        Class c3=new Cat().getClass();
        Class c4 =new Cat().getClass();
        System.out.println(c1==c2);
        System.out.println(c2==c3);
        System.out.println("finish main");
    }
}
/*　Output:
inside main
-------
Loading Cat
true
true
finish main
 */
```



 	**其实对于任意一个Class对象，都需要由它的类加载器和这个类本身一同确定其在Java虚拟机中的唯一性，也就是说，即使两个Class对象来源于同一个Class文件，只要加载它们的类加载器不同，那这两个Class对象就必定不相等。**这里的“相等”包括了代表类的Class对象的equals（）、isAssignableFrom（）、isInstance（）等方法的返回结果，也包括了使用instanceof关键字对对象所属关系的判定结果。所以在java虚拟机中使用**双亲委派模型**来组织类加载器之间的关系，来保证Class对象的唯一性。



**生成Class 对象的过程**

编写一个java类时，JVM会帮我们编译成Class 对象，存放在同名的.class文件里，在运行时，当需要生产这个类的对象，JVM会检查此类是否已经装载进内存。若没有，就会把.class文件装入内存。若已经装载，会根据class文件生成实例对象。

**静态编译和动态编译**

- 静态编译：在编译时确定类型，绑定对象
- 动态编译：在运行时确定类型，绑定对象。



## final、finally、finalize的区别

https://blog.csdn.net/cyl101816/article/details/67640843

**final：**前文已经有描述

**finally:**是在异常处理时提供finally块来执行任何清除操作。不管有没有异常被抛出、捕获，finally块都会被执行。try块中的内容是在无异常时执行到结束。catch块中的内容，是在try块内容发生catch所声明的异常时，跳转到catch块中执行。finally块则是无论异常是否发生，都会执行finally块的内容，所以在代码逻辑中有需要无论发生什么都必须执行的代码，就可以放在finally块中。

https://blog.csdn.net/qq_39135287/article/details/78455525

**finalize:**是方法名。java技术允许使用finalize（）方法在垃圾收集器将对象从内存中清除出去之前做必要的清理工作。这个方法是由垃圾收集器在确定这个对象没有被引用时，会自动调用这个方法。它是在object类中定义的，因此所有的类都继承了它。子类覆盖finalize（）方法以整理系统资源或者被执行其他清理工作。finalize（）方法是在垃圾收集器删除对象之前对这个对象调用的。 
		但是finalize（）调用时机并不确定，有时候资源已经耗尽，但是gc仍然没有触发，所以不能依赖finalize()来回收占用的资源。



## java创建对象的四种方式

- new 关键字创建一个对象

是最常规的创建方式。类会经过加载，初始化和实例化。

- 反射

1）Class.forName("类名")

​	使用Class.forName("类名")先获取这个类的Class 对象，然后调用Class内部的newInstance()方法，默认调用无参构造函数。**必须是public类型的，如果是private会创建失败。**

2）Constructor类的newInstance方法

​	和前一个方式很像，在java.lang.reflect.Constructor类里也有一个newInstance方法，可以通过这个方法调用有参和无参的构造函数。

```java
//先获取Person的Class对象，然后通过这个对象获取构造器。
Constructor<Person> constructor = Person.class.getConstructor();
//使用构造器的newInstance方法
Person person3 = constructor.newInstance();
```



- clone

  是Object里面的一个方法，protected类型，子类不可见行，所以必须重写这个clone()方法。要注意，需要实现clonable接口，否则会报java.lang.CloneNotSupportedException。

  clone（）出来的对象其实是**浅拷贝**：创建一个新对象，然后将当前对象的非静态字段复制到该对象，如果字段类型是值类型（基本类型）的，那么对该字段进行复制；如果字段是引用类型的，则只复制该字段的引用而不复制引用指向的对象。此时新对象里面的引用类型字段相当于是原始对象里面引用类型字段的一个副本，原始对象与新对象里面的引用字段指向的是同一个对象。

  **深拷贝：**和浅拷贝相对，把引用的对象也复制一遍。

- 反序列化

  在反序列化的时候jvm会创建一个单独的对象，但是不会调用任何构造函数。为了实现序列化/反序列化，需要实现Seralizable接口

### new 一个对象的过程

https://cloud.tencent.com/developer/article/1529790

https://www.cnblogs.com/daijiting/p/9960783.html

 虚拟机遇到一条new指令时，首先将去检查这个指令的参数是否能在常量池中定位到一个类的符号引用，并且检查这个符号引用代表的类是否已被加载，解析和初始化过，若没有，必须先执行相应的类加载过程。

​     类加载的过程在这篇文章中先不进行说明，简单地说下，类加载的过程就是将我们的java源代码编译后的class字节码文件加载进内存的过程，先说到这吧，后面会单独写一篇文章，大家一起交流交流。

​     在通过类加载检查通过后，接下来虚拟机将为新生对象分配内存。对象所需内存大小在类加载完成后可确定，为对象分配空间的任务相当于把一块确定大小的内存从Java堆中划分出来，一般存在两种方式，其一是指针碰撞，其二是空闲列表，具体选择哪种分配方式是根据Java堆是否规整来决定的。

Java堆的规整同时又取决于所采用的垃圾收集器是否带有压缩整理的功能所决定的，我们都知道垃圾收集器存在标记-清除，标记-整理等，因此Java堆是否规整就看你使用的是什么GC算法了。为了确保内存分配时的线程安全，通常使用两种解决方法：一种是对分配内存空间的动作进行同步处理--实际上虚拟机采用CAS配上失败重试的方式保证更新操作的原子性；

另外一种是把内存分配的动作线程划分在不同的空间之中，即每个线程在Java堆中预先分配一小块内存，称之为TLAB，是本地线程分配缓冲的简写形式，那个线程要分配内存，就在哪个线程的TLAB上分配，只有TLAB用完并分配新的TLAB时，才需要同步锁定操作。

​      内存分配完成后，虚拟机需要将分配到的内存空间都初始化为零值，如果使用TLAB，这一工作可以提前至TLAB分配时进行。这一步操作保证了对象的实例字段在java代码中可以不赋初始值就直接使用，程序能访问到这些字段的数据类型所对应的零值。

​      接下来的动作就是虚拟机要对对象进行必要的设置了，一般一个对象是属于某个类的实例中的一个，如何才能找到类的元数据信息，对象的哈希码就是hashCode了，对象的GC分代年龄等信息，这些信息是存在对象的对象头之中，当上面的工作完成了之后，从虚拟机的角度来看，一个新对象已经产生，但是从Java程序的角度来看，对象的创建才刚刚开始，一般来说，执行new执行之后会接着执行<init>方法，把对象按照程序设计人员的思维进行初始化，这样一个新对象才算完全产生出来。

​     在HotSpot虚拟机中，对象在内存中的存储布局可以分为三块区域：**对象头**，**实例数据**和**对齐填充**。

​     HotSpot虚拟机的**对象头**包括两部分信息，第一部分用于存储对象自身的运行时数据，如GC分代年龄，哈希码即hashCode，锁状态标识，线程持有的锁，偏向线程ID等信息，在这里我采用的都是文字描述，关于对象头信息，有一张图描述的很清楚，自行查阅吧，Mark Word被设计成一个非固定的数据结构，原因在于在极小的内存空间存储尽量多的信息，它会根据对象的状态复用自己的存储空间。

​       好了，我们继续吧，第二部分是类型指针，并不是所有的虚拟机都有，由于我们在说hotSpot，类型指针即对象指向它的类元数据的指针，虚拟机通过这个指针来确定对象是哪个类的实例（句柄和直接指针），此外，如果对象是一个Java数组，那么在对象头中还必须有一块用于记录数组长度的数据，因为虚拟机可以通过普通Java对象的元数据信息确定java对象的大小，但是从数组的元数组中却无法确定数组的大小，这块内容稍显晦涩难懂，大家有个印象就可以了，想深入了解的查阅对应的资料信息吧。

​       **实例数据**部分才是存储对象实例的有效信息，也是在程序代码中定义的各种类型的字段内容，无论是从父类继承下来的，还是在子类中定义的，都需要记录下来，这部分的存储顺序会受到虚拟机分配策略参数和字段在Java源码中定义的顺序的影响。HotSpot虚拟机默认的分配策略是相同宽度的字段总是被分配到一起，满足这个前提条件下，在父类中定义的变量会出现在子类之前。

​      对齐填充部分不是必然存在的，它仅仅起着占位符的作用，由于HotSpot虚拟机的自动内存管理机制要求对象的起始地址必须是8字节的整倍数，因此，当对象的实例数据部分没有对齐时，这个时候就需要对齐填充来补全了。

​      ok，这篇文章快要结束了，下面我们在说下一些内容，我们在程序中创建对象是为了使用对象，Java程序需要通过**栈上的引用来操作堆上的具体对象**，目前主流的访问方式有使用**句柄**和**直接指针**两种，如果使用句柄访问的话，那么Java堆中将会划分一块内存来作为句柄池，reference中存储的就是对象的句柄池地址，而句柄中包含了对象实例数据与类型数据各自的具体地址信息。如果使用直接指针访问，那么Java堆对象的布局中就必须考虑如何放置访问类型数据的相关信息了，而reference中存储的直接就是对象地址。

​      两种访问对象的方式其实各自有自己的优势，使用句柄来访问的最大好处就是reference中存储的是稳定的句柄地址，使用直接指针访问的方式优势就是速度更快，因为它节省了一次指针定位所带来的时间开销

## 序列化和反序列化

https://blog.csdn.net/qq_39751320/article/details/106328194

不会对静态变量进行序列化，因为只保存对象的状态，而静态变量时类的状态。

可以通过重写序列化和反序列化的方法，使只序列化 需要的那部分数据(ArrayList中的数组用transient修饰)



## Java socket

https://www.jianshu.com/p/cde27461c226

### 基本原理

![img](https://upload-images.jianshu.io/upload_images/206633-2d6f4a3abcd59745.png?imageMogr2/auto-orient/strip|imageView2/2/w/1064/format/webp)

socket是基于TCP/IP通信的一个抽象，对复杂的通信逻辑进行封装，提供简单的API实现网络连接。下面是一组socket通信图

![img](https://upload-images.jianshu.io/upload_images/206633-4b2d8622b6d9d48d.png?imageMogr2/auto-orient/strip|imageView2/2/w/696/format/webp)

### 最基本的实例

服务端

```java
public static void main(String[] args) {
try {

						// 初始化服务端socket并且绑定9999端口
            ServerSocket serverSocket  =new ServerSocket(9999);
            //等待客户端的连接
            Socket socket = serverSocket.accept();
            //获取输入流
            BufferedReader bufferedReader =new BufferedReader(new InputStreamReader(socket.getInputStream()));
            //读取一行数据
            String str = bufferedReader.readLine();
            //输出打印
            System.out.println(str);
        }catch (IOException e) {
e.printStackTrace();
        }
}
}
```

客户端

```java
public static void main(String[] args) {
try {
Socket socket =new Socket("127.0.0.1",9999);
            BufferedWriter bufferedWriter =new BufferedWriter(new OutputStreamWriter(socket.getOutputStream()));
            String str="你好，这是我的第一个socket";
            bufferedWriter.write(str);
  
  					//加了下面这两行代码，就可以让服务端正常接受数据了
  					//刷新输入流
            bufferedWriter.flush();
            //关闭socket的输出流
            socket.shutdownOutput();

        }catch (IOException e) {
e.printStackTrace();
        }
}
}
```

先启动服务端后会在accept（）处阻塞，等待客户端的连接。然后开启客户端，客户端的程序会马上关闭，但是服务器端会报错。原因就是服务器端在执行read方法时会阻塞，一直等客户端的数据发送完毕，但是上面客户端没有发送消息结束的标志符，所以服务端一直在等待。

通常大家会用以下方法进行进行结束：

socket.close() 或者调用socket.shutdownOutput();方法。调用这俩个方法，都会结束客户端socket。但是有本质的区别。socket.close() 将socket关闭连接，那边如果有服务端给客户端反馈信息，此时客户端是收不到的。而socket.shutdownOutput()是将输出流关闭，此时，如果服务端有信息返回，则客户端是可以正常接受的。

那么如果不使用上面两种方式，服务端如何判断客户端是否已经发送完数据呢？可以通过双方约定一个标志符

比如下面例子

服务端----增加了while循环判断是否接受完毕

```java
// 初始化服务端socket并且绑定9999端口
           ServerSocket serverSocket  =new ServerSocket(9999);
            //等待客户端的连接
            Socket socket = serverSocket.accept();
            //获取输入流,并且指定统一的编码格式
            BufferedReader bufferedReader =new BufferedReader(new InputStreamReader(socket.getInputStream(),"UTF-8"));
            //读取一行数据
            String str;
            //通过while循环不断读取信息，
            while ((str = bufferedReader.readLine())!=null){
//输出打印
                System.out.println(str);
            }

```

客户端---每次发送一行的数据并且加上换行符

```java
						//初始化一个socket
            Socket socket =new Socket("127.0.0.1",9999);
            //通过socket获取字符流
            BufferedWriter bufferedWriter =new BufferedWriter(new OutputStreamWriter(socket.getOutputStream()));
            //通过标准输入流获取字符流
            BufferedReader bufferedReader =new BufferedReader(new InputStreamReader(System.in,"UTF-8"));
          while (true){
String str = bufferedReader.readLine();
              bufferedWriter.write(str);
              bufferedWriter.write("\n");
              bufferedWriter.flush();
          }
```



那么如果有多个客户端向这个服务端发送数据，那么因为accept（）还有read（）都会阻塞，所以当服务端在接受一个客户端的数据时，会阻塞其他客户端的通信，这时我们可以在服务端中针对读取输入流部分功能，使用多线程。但是一旦连接的客户端数量达到一定数量，服务端就接受不了了。这时可以使用线程池，复用这些线程。

改良以后的服务端代码

```java
				// 初始化服务端socket并且绑定9999端口
        ServerSocket serverSocket =new ServerSocket(9999);
        //创建一个线程池
        ExecutorService executorService = Executors.newFixedThreadPool(100);
        while (true) {
//等待客户端的连接
            Socket socket = serverSocket.accept();
            Runnable runnable = () -> {
BufferedReader bufferedReader =null;
                try {
bufferedReader =new BufferedReader(new InputStreamReader(socket.getInputStream(), "UTF-8"));
                    //读取一行数据
                    String str;
                    //通过while循环不断读取信息，
                    while ((str = bufferedReader.readLine()) !=null) {
											//输出打印
                        System.out.println("客户端说：" + str);
                    }
}catch (IOException e) {
e.printStackTrace();
                }
};
            executorService.submit(runnable);
        }
}
```

在实际应用中，socket发送的数据不是一行一行的，而是采用 **数据长度+类型+数据的方式**

### socket指定长度发送数据

在实际应用中，网络的数据在TCP/IP协议下的socket都是采用数据流的方式进行发送，那么在发送过程中就要求我们将数据流转出字节进行发送，读取的过程中也是采用字节缓存的方式结束。那么问题就来了，在socket通信时候，我们大多数发送的数据都是不定长的，所有接受方也不知道此次数据发送有多长，因此无法精确地创建一个缓冲区（字节数组）用来接收，在不定长通讯中，通常使用的方式时每次默认读取8*1024长度的字节，若输入流中仍有数据，则再次读取，一直到输入流没有数据为止。但是如果发送数据过大时，发送方会对数据进行分包发送，这种情况下或导致接收方判断错误，误以为数据传输完成，因而接收不全。在这种情况下就会引出一些问题，诸如半包，粘包，分包等问题，为了后续一些例子中好理解，我在这里直接将半包，粘包，分包概念性东西在写一下（引用度娘）

**半包**

接受方没有接受到一个完整的包，只接受了部分。

原因：TCP为提高传输效率，将一个包分配的足够大，导致接受方并不能一次接受完。

影响：长连接和短连接中都会出现

**粘包**

发送方发送的多个包数据到接收方接收时粘成一个包，从接收缓冲区看，后一包数据的头紧接着前一包数据的尾。

分类：一种是粘在一起的包都是完整的数据包，另一种情况是粘在一起的包有不完整的包

出现粘包现象的原因是多方面的:

1)发送方粘包：由TCP协议本身造成的，TCP为提高传输效率，发送方往往要收集到足够多的数据后才发送一包数据。若连续几次发送的数据都很少，通常TCP会根据优化算法把这些数据合成一包后一次发送出去，这样接收方就收到了粘包数据。

2)接收方粘包：接收方用户进程不及时接收数据，从而导致粘包现象。这是因为接收方先把收到的数据放在系统接收缓冲区，用户进程从该缓冲区取数据，若下一包数据到达时前一包数据尚未被用户进程取走，则下一包数据放到系统接收缓冲区时就接到前一包数据之后，而用户进程根据预先设定的缓冲区大小从系统接收缓冲区取数据，这样就一次取到了多包数据。

**分包**

分包（1）：在出现粘包的时候，我们的接收方要进行分包处理；

分包（2）：一个数据包被分成了多次接收；

原因：1. IP分片传输导致的；2.传输过程中丢失部分包导致出现的半包；3.一个包可能被分成了两次传输，在取数据的时候，先取到了一部分（还可能与接收的缓冲区大小有关系）。

影响：粘包和分包在长连接中都会出现

那么如何解决半包和粘包的问题，就涉及一个一个数据发送如何标识结束的问题，通常有以下几种情况

固定长度：每次发送固定长度的数据；

特殊标示：以回车，换行作为特殊标示；获取到指定的标识时，说明包获取完整。

字节长度：包头+包长+包体的协议形式，当服务器端获取到指定的包长时才说明获取完整；

所以大部分情况下，双方使用socket通讯时都会约定一个定长头放在传输数据的最前端，用以标识数据体的长度，通常定长头有整型int，短整型short，字符串Strinng三种形式。



## 泛型相关

https://www.cnblogs.com/coprince/p/8603492.html

**为什么要使用泛型**

Java集合不会知道我们需要用它来保存什么类型的对象，所以他们把集合设计成能保存任何类型的对象，这样就具有很好的通用性。但这样做也带来两个问题：

- 集合对元素类型没有任何限制，这样可能引发一些问题：例如想创建一个只能保存Sting对象的集合，但程序也可以轻易地将int对象“丢”进去，所以可能引发异常。

- 由于把对象“丢进”集合时，集合丢失了对象的状态信息，集合只知道它盛装的是Object，因此取出集合元素后通常还需要进行强制类型转换。这种强制类型转换既会增加编程的复杂度、也可能引发ClassCastException。

#### 什么是泛型

```
泛型，就是将类型由原来的具体的类型参数化，类似于方法中的变量参数，此时类型也定义成参数形式，然后在使用/调用时传入具体的类型

泛型的本质是为了参数化类型（在不创建新的类型的情况下，通过泛型指定的不同类型来控制形参具体限制的类型）。也就是说在泛型使用过程中，

操作的数据类型被指定为一个参数，这种参数类型可以用在类、接口和方法中，分别被称为泛型类、泛型接口、泛型方法。
```

```java
List arrayList = new ArrayList();
arrayList.add("aaaa");
arrayList.add(100);

for(int i = 0; i< arrayList.size();i++){
    String item = (String)arrayList.get(i);
    Log.d("泛型测试","item = " + item);
}
```

上面的例子会运行的时候会报错，但是编译的时候没问题，因为ArrayList()可以存放任何类型，但是这样不利于写代码，最好在编译的时候就会报错，这样能尽快的修改代码。--------**使用泛型**

可以对第一行进行修改,此时编译器在编译阶段就会报错了。<String>是类型实参，传入到List中对应的形参

```java
List<String> arrayList = new ArrayList<String>();
...
//arrayList.add(100); 在编译阶段，编译器就会报错
```

#### 特性

泛型只在编译阶段有效。

```java
List<String> stringArrayList = new ArrayList<String>();
List<Integer> integerArrayList = new ArrayList<Integer>();

Class classStringArrayList = stringArrayList.getClass();
Class classIntegerArrayList = integerArrayList.getClass();

if(classStringArrayList.equals(classIntegerArrayList)){
    Log.d("泛型测试","类型相同");
}


结果：
泛型测试: 类型相同
```

在编译之后程序会采取去泛型化的措施，也就是说Java的泛型只在编译阶段有效。在编译阶段检验了泛型以后，就会将泛型的相关信息擦除。在运行的时候可以看成是相同的类型，也就是说编译过后的class文件是不包含任何泛型信息的。

**对此总结成一句话：泛型类型在逻辑上看以看成是多个不同的类型，实际上都是相同的基本类型**



#### 泛型的使用方法

泛型有三种使用方法：泛型类、泛型接口、泛型方法

**1.泛型类**

那些容器类都是，如List、Set、Map

```java
//此处T可以随便写为任意标识，常见的如T、E、K、V等形式的参数常用于表示泛型
//在实例化泛型类时，必须指定T的具体类型
public class Generic<T>{ 
    //key这个成员变量的类型为T,T的类型由外部指定  
    private T key;

    public Generic(T key) { //泛型构造方法形参key的类型也为T，T的类型由外部指定
        this.key = key;
    }

    public T getKey(){ //泛型方法getKey的返回值类型为T，T的类型由外部指定
        return key;
    }
}

//泛型的类型参数只能是类类型（包括自定义类），不能是简单类型（基本类型如int、double）
//传入的实参类型需与泛型的类型参数类型相同，即为Integer.
Generic<Integer> genericInteger = new Generic<Integer>(123456);

//传入的实参类型需与泛型的类型参数类型相同，即为String.
Generic<String> genericString = new Generic<String>("key_vlaue");
Log.d("泛型测试","key is " + genericInteger.getKey());
Log.d("泛型测试","key is " + genericString.getKey());
```

定义一个泛型类不一定要传入**泛型类型实参**，如果传入了泛型实参，此时会根据传入的泛型实参做限制，如果不传入的话，在泛型类中使用泛型的方法活成员变量可以为任何类型

**2.泛型接口**

泛型接口与泛型类的定义及使用基本相同

当实现泛型接口的类，未传入泛型实参时：

```java
/**
 * 未传入泛型实参时，与泛型类的定义相同，在声明类的时候，需将泛型的声明也一起加到类中
 * 即：class FruitGenerator<T> implements Generator<T>{
 * 如果不声明泛型，如：class FruitGenerator implements Generator<T>，编译器会报错："Unknown class"
 */
class FruitGenerator<T> implements Generator<T>{
    @Override
    public T next() {
        return null;
    }
}
```

当实现泛型接口的类，传入泛型实参时：

```java
/**
 * 传入泛型实参时：
 * 定义一个生产器实现这个接口,虽然我们只创建了一个泛型接口Generator<T>
 * 但是我们可以为T传入无数个实参，形成无数种类型的Generator接口。
 * 在实现类实现泛型接口时，如已将泛型类型传入实参类型，则所有使用泛型的地方都要替换成传入的实参类型
 * 即：Generator<T>，public T next();中的的T都要替换成传入的String类型。
 */
public class FruitGenerator implements Generator<String> {

    private String[] fruits = new String[]{"Apple", "Banana", "Pear"};

    @Override
    public String next() {
        Random rand = new Random();
        return fruits[rand.nextInt(3)];
    }
}
```

**3.泛型通配符**

如果一个方法传入的是泛型实参，那么可能创建实例是这个实参的子类，也不能调用这个方法。这时可以讲实参改成“？”

```java
List<Integer> ex_int= new ArrayList<Integer>();    
List<Number> ex_num = ex_int; //非法的  
```

虽然Integer是Number的子类，但是List<Integer>不是List<Number>的子类。如果第二行是正确的，那么ex_num中添加double类型的对象以后，在从List取出来时就出现了问题，不知道该转型为Integer还是Double。这时候可以使用通配符

```java
public static void main(String[] args) {  
    FX<Number> ex_num = new FX<Number>(100);  
    FX<Integer> ex_int = new FX<Integer>(200);  
    getData(ex_num);  
    getData(ex_int);//编译错误  
}  
  
public static void getData(FX<Number> temp) { //此行若把Number换为“？”编译通过  
    //do something...  
}  
      
public static class FX<T> {  
    private T ob;   
    public FX(T ob) {  
        this.ob = ob;  
    }  
}  
```



**4.泛型方法**

```java
/**
 * 泛型方法的基本介绍
 * @param tClass 传入的泛型实参
 * @return T 返回值为T类型
 * 说明：
 *     1）public 与 返回值中间<T>非常重要，可以理解为声明此方法为泛型方法。
 *     2）只有声明了<T>的方法才是泛型方法，泛型类中的使用了泛型的成员方法并不是泛型方法。
 *     3）<T>表明该方法将使用泛型类型T，此时才可以在方法中使用泛型类型T。
 *     4）与泛型类的定义一样，此处T可以随便写为任意标识，常见的如T、E、K、V等形式的参数常用于表示泛型。
 */
public <T> T genericMethod(Class<T> tClass)throws InstantiationException ,
  IllegalAccessException{
        T instance = tClass.newInstance();
        return instance;
}

		//这不是一个泛型方法，这就是一个普通的方法，只是使用了Generic<Number>这个泛型类做形参而已。
    public void showKeyValue1(Generic<Number> obj){
        Log.d("泛型测试","key value is " + obj.getKey());
    }

    //这也不是一个泛型方法，这也是一个普通的方法，只不过使用了泛型通配符?
    //同时这也印证了泛型通配符章节所描述的，?是一种类型实参，可以看做为Number等所有类的父类
    public void showKeyValue2(Generic<?> obj){
        Log.d("泛型测试","key value is " + obj.getKey());
    }
```

静态方法有一种需要注意，那就是在类中的静态方法使用泛型：静态方法无法访问类上定义的泛型；如果静态方法操作的引用数据类型不确定的时候，必须要将泛型定义在方法上。

即：如果静态方法要使用泛型的话，必须将静态方法也定义成泛型方法 。

```java
public class StaticGenerator<T> {
    ....
    ....
    /**
     * 如果在类中定义使用泛型的静态方法，需要添加额外的泛型声明（将这个方法定义成泛型方法）
     * 即使静态方法要使用泛型类中已经声明过的泛型也不可以。
     * 如：public static void show(T t){..},此时编译器会提示错误信息：
          "StaticGenerator cannot be refrenced from static context"
     */
    public static <T> void show(T t){

    }
}
```

**5.泛型的上下边界**

在使用泛型的时候，我们还可以为传入的泛型类型实参进行上下边界的限制，如：类型实参只准传入某种类型的父类或某种类型的子类。

为泛型添加上边界，即传入的类型实参必须是指定类型的子类型

```java
public void showKeyValue1(Generic<? extends Number> obj){
    Log.d("泛型测试","key value is " + obj.getKey());
}
```

1,`<? extends Parent>` 指定了泛型类型的上届
2,`<? super Child>` 指定了泛型类型的下届

泛型方法的例子：

```java
//在泛型方法中添加上下边界限制的时候，必须在权限声明与返回值之间的<T>上添加上下边界，即在泛型声明的时候添加
//public <T> T showKeyName(Generic<T extends Number> container)，编译器会报错："Unexpected bound"
public <T extends Number> T showKeyName(Generic<T> container){
    System.out.println("container key :" + container.getKey());
    T test = container.getKey();
    return test;
}
```



**6.泛型中的约束和局限性**

- 不能实例化泛型类
- 静态变量或方法不能引用泛型类型变量，但静态泛型方法是可以的
- 基本数据类型不能作为泛型类型
- 不能使用instanceof关键字货==判断泛型类的类型
- 泛型类不能继承Exception或者Throwable
- 泛型数组可以申明但是无法实例化

```java
List<String>[] lsa = new List<String>[10]; // Not really allowed.    
Object o = lsa;    
Object[] oa = (Object[]) o;    
List<Integer> li = new ArrayList<Integer>();    
li.add(new Integer(3));    
oa[1] = li; // Unsound, but passes run time store check    
String s = lsa[1].get(0); // Run-time error: ClassCastException.    
```

这种情况下，由于JVM泛型的擦除机制，在运行时JVM是不知道泛型信息的，所以可以给oa[1]赋上一个ArrayList<Integer>而不会出现异常，但是在取出数据的时候却要做一次类型转换，所以就会出现ClassCastException，如果可以进行泛型数组的声明，上面说的这种情况在编译期将不会出现任何的警告和错误，只有在运行时才会出错。

下面采用通配符的方式是被允许的：

```java
List<?>[] lsa = new List<?>[10]; // OK, array of unbounded wildcard type.    
Object o = lsa;    
Object[] oa = (Object[]) o;    
List<Integer> li = new ArrayList<Integer>();    
li.add(new Integer(3));    
oa[1] = li; // Correct.    
Integer i = (Integer) lsa[1].get(0); // OK  
```



#### 泛型的类型擦除

https://www.cnblogs.com/wuqinglong/p/9456193.html

Java的泛型是伪泛型，这是因为Java在编译期间，所有的泛型信息都会被擦掉，正确理解泛型概念的首要前提是理解类型擦除。Java的泛型基本上都是在编译器这个层次上实现的，在生成的字节码中是不包含泛型中的类型信息的，使用泛型的时候加上类型参数，在编译器编译的时候会去掉，这个过程成为类型擦除。

```java
public class Test {

    public static void main(String[] args) throws Exception {

        ArrayList<Integer> list = new ArrayList<Integer>();

        list.add(1);  //这样调用 add 方法只能存储整形，因为泛型类型的实例为 Integer

        list.getClass().getMethod("add", Object.class).invoke(list, "asd");

        for (int i = 0; i < list.size(); i++) {
            System.out.println(list.get(i));
        }
    }

}
```

上面的例子没有错误，通过反射调用add方法时可以存储字符串，说明Integer泛型在编译之后被擦除掉了。只保留原始类型。

**原始类型**：擦去泛型信息，最后在字节码中的类型变量的真正类型，无论何时定义一个泛型，都会用限定类型（无限定类型用Object）替换

```java
public class Pair<T extends Comparable> {}
```

原始类型就是Comparable

#### 类型擦除引起的问题

**1.先检查，在编译**

类型变量会在编译的时候擦除掉，那为什么以下代码还会报错,不是都是Object类型了吗？

**Java编译器是通过先检查代码中的泛型类型，在进行类型擦除，在进行编译。**

```java
public static  void main(String[] args) {  

    ArrayList<String> list = new ArrayList<String>();  
    list.add("123");  
    list.add(123);//编译错误  
}
```



**相关面试题**

https://cloud.tencent.com/developer/article/1033693





## 一致性哈希

https://cloud.tencent.com/developer/article/1491613

https://blog.csdn.net/kefengwang/article/details/81628977



**虚拟节点**

节点数越少，越容易出现节点在哈希环上的分布不均匀，导致各节点映射的对象数量严重不均衡(数据倾斜)；相反，节点数越多越密集，数据在哈希环上的分布就越均匀。
但实际部署的物理节点有限，我们可以用有限的物理节点，虚拟出足够多的虚拟节点(Virtual Node)，最终达到数据在哈希环上均匀分布的效果：
如下图，实际只部署了2个节点 Node A/B，
每个节点都复制成3倍，结果看上去是部署了6个节点。
可以想象，当复制倍数为 2^32 时，就达到绝对的均匀，通常可取复制倍数为32或更高。
虚拟节点哈希值的计算方法调整为：对“节点的IP(或机器名)+虚拟节点的序号(1~N)”作哈希。

手撕 一致性哈希（带虚拟节点）

```java
/**
 * @author: https://kefeng.wang
 * @date: 2018-08-10 11:08
 **/
public class ConsistentHashing {
    // 物理节点
    private Set<String> physicalNodes = new TreeSet<String>() {
        {
            add("192.168.1.101");
            add("192.168.1.102");
            add("192.168.1.103");
            add("192.168.1.104");
        }
    };

    //虚拟节点
    private final int VIRTUAL_COPIES = 1048576; // 物理节点至虚拟节点的复制倍数
    private TreeMap<Long, String> virtualNodes = new TreeMap<>(); // 哈希值 => 物理节点

    // 32位的 Fowler-Noll-Vo 哈希算法
    // https://en.wikipedia.org/wiki/Fowler–Noll–Vo_hash_function
    private static Long FNVHash(String key) {
        final int p = 16777619;
        Long hash = 2166136261L;
        for (int idx = 0, num = key.length(); idx < num; ++idx) {
            hash = (hash ^ key.charAt(idx)) * p;
        }
        hash += hash << 13;
        hash ^= hash >> 7;
        hash += hash << 3;
        hash ^= hash >> 17;
        hash += hash << 5;

        if (hash < 0) {
            hash = Math.abs(hash);
        }
        return hash;
    }

    // 根据物理节点，构建虚拟节点映射表
    public ConsistentHashing() {
        for (String nodeIp : physicalNodes) {
            addPhysicalNode(nodeIp);
        }
    }

    // 添加物理节点
    public void addPhysicalNode(String nodeIp) {
        for (int idx = 0; idx < VIRTUAL_COPIES; ++idx) {
            long hash = FNVHash(nodeIp + "#" + idx);
            virtualNodes.put(hash, nodeIp);
        }
    }

    // 删除物理节点
    public void removePhysicalNode(String nodeIp) {
        for (int idx = 0; idx < VIRTUAL_COPIES; ++idx) {
            long hash = FNVHash(nodeIp + "#" + idx);
            virtualNodes.remove(hash);
        }
    }

    // 查找对象映射的节点
    public String getObjectNode(String object) {
        long hash = FNVHash(object);
        SortedMap<Long, String> tailMap = virtualNodes.tailMap(hash); // 所有大于 hash 的节点
        Long key = tailMap.isEmpty() ? virtualNodes.firstKey() : tailMap.firstKey();
        return virtualNodes.get(key);
    }

    // 统计对象与节点的映射关系
    public void dumpObjectNodeMap(String label, int objectMin, int objectMax) {
        // 统计
        Map<String, Integer> objectNodeMap = new TreeMap<>(); // IP => COUNT
        for (int object = objectMin; object <= objectMax; ++object) {
            String nodeIp = getObjectNode(Integer.toString(object));
            Integer count = objectNodeMap.get(nodeIp);
            objectNodeMap.put(nodeIp, (count == null ? 0 : count + 1));
        }

        // 打印
        double totalCount = objectMax - objectMin + 1;
        System.out.println("======== " + label + " ========");
        for (Map.Entry<String, Integer> entry : objectNodeMap.entrySet()) {
            long percent = (int) (100 * entry.getValue() / totalCount);
            System.out.println("IP=" + entry.getKey() + ": RATE=" + percent + "%");
        }
    }

    public static void main(String[] args) {
        ConsistentHashing ch = new ConsistentHashing();

        // 初始情况
        ch.dumpObjectNodeMap("初始情况", 0, 65536);

        // 删除物理节点
        ch.removePhysicalNode("192.168.1.103");
        ch.dumpObjectNodeMap("删除物理节点", 0, 65536);

        // 添加物理节点
        ch.addPhysicalNode("192.168.1.108");
        ch.dumpObjectNodeMap("添加物理节点", 0, 65536);
    }
}
```

## String的最大长度

String内部使用一个字符数组来维护字符序列的

```java
private final Byte value[];
```

所以String的最大长度，取决于value数组的最大长度，可以看一下String类中的length()源码:

```java
public int length() {
    return value.length;
}
```

可以发现，返回类型是int，所以最大长度理论上是2^31-1。但是实际申请这么长的数组是会产生OOM错误（内存溢出），因为系统无法分配这么大的连续内存，一个byte类型占1字节，2147483647 个 byte类型就是 2147483647 字节，这接近于 2GB 大小，想要申请这么一大块连续的内存空间，失败也就不足为奇了。

## byte、 int、char、long、float、double各占多少字节数？

| 类型    | 字符数    |
| ------- | --------- |
| byte    | 1字节     |
| char    | 2字节     |
| short   | 2字节     |
| int     | 4字节     |
| float   | 4字节     |
| long    | 8字节     |
| double  | 8字节     |
| boolean | 至少1字节 |

int的取值范围 -2^31~2^31-1 (-2147483648 ~2147483647)，第一位是符号位所以这里只取31位。

## String，StringBuilder和StringBuffer的区别

https://blog.csdn.net/qushaming/article/details/82971901

速度StringBuilder>StringBuffer>String（如果仅仅是字符串字面量的拼接则String最快如String a="abc"+"bcd"+"efg"）

线程安全 StringBuffer，String

线程不安全 StringBuilder

StringBuffer线程安全是因为它大部分方法都是由synchronized关键字修饰。

## equals()/hashCode()

#### equals()方法

Object类中的equals()方法

```java
public boolean equals(Object obj) {  
    return (this == obj);  
}  
```

表示两个对象的指向的地址（引用）是否相同。但是String、Math、Integer、Double都重写了equals()方法，实现方式都类似，但是都是判断里面的内容是否相同（不是地址判断）。

- 检查是否为同一个对象的引用，如果是直接返回 true；
- 检查是否是同一个类型，如果不是，直接返回 false；
- 将 Object 对象进行转型；
- 判断每个关键域是否相等。

**equals与==的比较**

1. 对于基本类型(int,double,float等)==进行的是值的判断，没有equals()方法。
2. 对于引用类型，==判断两个变量是否引用同一个变量（地址判断），equals()判断引用的对象内容是否相同

相关性质如下：

- **自反性**（reflexive）。对于任意不为`null`的引用值x，`x.equals(x)`一定是`true`。
- **对称性**（symmetric）。对于任意不为`null`的引用值`x`和`y`，当且仅当`x.equals(y)`是`true`时，`y.equals(x)`也是`true`。
- **传递性**（transitive）。对于任意不为`null`的引用值`x`、`y`和`z`，如果`x.equals(y)`是`true`，同时`y.equals(z)`是`true`，那么`x.equals(z)`一定是`true`。
- **一致性**（consistent）。对于任意不为`null`的引用值`x`和`y`，如果用于equals比较的对象信息没有被修改的话，多次调用时`x.equals(y)`要么一致地返回`true`要么一致地返回`false`。
- 对于任意不为`null`的引用值`x`，`x.equals(null)`返回`false`。



#### hashCode()方法

Object中的hashCode（）方法直接返回对象的地址。

hashCode()返回的是哈希值，用equals()判断两个对象是否等价。**等价的对象hash值一定相等，但是hash值相同的两个对象，不一定等价，因为hash值的计算可能存在随机性.**

所以当equals方法被重写时，hashCode也要被重写，保证等价的两个对象的哈希值也相等。

**那么为什么要引入hashCode呢？**

因为Java的集合类中有Set，里面存储的都是无序的，不重复的元素。如何判断元素是否重复呢，用equals()当然可以，但是如果这个集合很大，已经存了10000个元素，这时第100001个元素进来，要和前面每个元素进行equals判断，效率太低了。这时候可以使用hashCode，就是依据特定的算法指定到一个地址上。这样如果一个元素进来，先计算哈希值，判断该地址上有没有元素，如果没有的话，表示没有重复的元素，直接插入；如果有元素，在进行equals判断。

**相同的对象为什么必须要有相同的hashCode**

如果两个对象相同，但是hashCode不一样的话，那判断的时候会因为哈希值不同，认为没有重复的元素，就插入进去了，显然这是不允许的。

**两个对象hashCode相同，但是并不相同**

这就是存在哈希冲突的情况。

## 深拷贝和浅拷贝

clone()是Object的一个**protected方法**，这就表示，如果子类不去重写这个clone方法的话，其他类是不能调用这个类的clone()。原因请看protected访问权限。

```java
public class CloneExample {
    private int a;
    private int b;

    @Override
    public CloneExample clone() throws CloneNotSupportedException {
        return (CloneExample)super.clone();
    }
}

```

```java
CloneExample e1 = new CloneExample();
try {
    CloneExample e2 = e1.clone();
} catch (CloneNotSupportedException e) {
    e.printStackTrace();
}

```

因为CloneExample类没有实现Cloneable接口，所以会报CloneNotSupportedException异常。

**注意：clone并不是Cloneable接口的方法，是object的protected方法。Cloneable接口只是规定，如果一个类没有实现这个接口调用clone()方法，会抛出CloneNotSupportedException异常**

浅拷贝：object的clone方法拷贝的是同一个引用，堆中并没有多出一个对象

深拷贝：自己重写clone()方法的时候不调用父类super.clone()，自己新创建一个对象。

最好使用拷贝工厂





## Java Switch

- 能用switch判断的类型：byte、short、int、char还有枚举类型，在JDK7以后新增了String 类型
- 如果case语句没有写break，编译器不会报错，但会执行**之后所有case里的语句，不再进行判断**

- default 并不是必须的，可以不写

```java
/*
     * case语句中少写了break，编译不会报错
     *     但是会一直执行之后所有case条件下的语句，并不再进行判断，直到default语句
     *    
     */
    private static void breakTest() {
        char ch = 'A';
        switch (ch) {
        case 'B':
            System.out.println("case one");

        case 'A':
            System.out.println("case two");

        case 'C':
            System.out.println("case three");
        default:
            break;
        }
    }
输出如下：
--------------------------
    case two
    case three
```

