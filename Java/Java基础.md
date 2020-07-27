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



## final/static关键字

### final

**1.数据**

声明数据为常量，可以是编译时常量，也可以是在运行时被初始化后不能被改变的常量。

- 对于基本类型，final 使数值不变；
- 对于引用类型，final 使引用不变，也就不能引用其它对象，但是被引用的对象本身是可以修改的。

```java
final int x = 1;
// x = 2;  // cannot assign value to final variable 'x'
final A y = new A();
y.a = 1;
```

**2.方法**

声明方法不能被子类重写。

private 方法隐式地被指定为 final，如果在子类中定义的方法和基类中的一个 private 方法签名相同，此时子类的方法不是重写基类方法，而是在子类中定义了一个新的方法。

**3.类**

声明类不允许被继承。



### static

**1.静态变量**

- 静态变量：又称为类变量，也就是说这个变量属于类的，类所有的实例都共享静态变量，可以直接通过类名来访问它。静态变量在内存中只存在一份。
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

静态语句块在类初始化时运行一次。

**4.静态内部类**

非静态内部类依赖于外部类的实例，也就是说需要先创建外部类实例，才能用这个实例去创建非静态内部类。而静态内部类不需要。

静态内部类不能访问外部类的非静态的变量和方法。



**5.初始化顺序**

静态变量和静态语句块优先于实例变量和普通语句块，静态变量和静态语句块的初始化顺序取决于它们在代码中的顺序。

最后才是构造函数的初始化

在继承的情况下，初始化顺序为：

- 父类（静态变量、静态语句块）
- 子类（静态变量、静态语句块）
- 父类（实例变量、普通语句块）
- 父类（构造函数）
- 子类（实例变量、普通语句块）
- 子类（构造函数）



## 继承

### 访问权限

Java 中有三个访问权限修饰符：private、protected 以及 public，如果不加访问修饰符，表示包级可见。

可以对类或类中的成员（字段和方法）加上访问修饰符。

- 类可见表示其它类可以用这个类创建实例对象。
- 成员可见表示其它类可以用这个类的实例对象访问到该成员；

如果子类的方法重写了父类的方法，那么子类中该方法的访问级别不允许低于父类的访问级别。这是为了确保可以使用父类实例的地方都可以使用子类实例去代替，也就是确保满足里氏替换原则。

字段决不能是公有的，因为这么做的话就失去了对这个字段修改行为的控制，客户端可以对其随意修改。例如下面的例子中，AccessExample 拥有 id 公有字段，如果在某个时刻，我们想要使用 int 存储 id 字段，那么就需要修改所有的客户端代码。

### 抽象类和接口

**抽象类**

抽象类和抽象方法都使用 abstract 关键字进行声明。如果一个类中包含抽象方法，那么这个类必须声明为抽象类。

- 抽象类和普通类最大的区别是，抽象类不能被实例化，只能被继承。但是抽象类可以有构造方法，供子类创建对象，初始化父类使用。

- 子类如果实现抽象类，就必须实现父类所有的抽象方法
- 

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

- **性能开销** ：反射涉及了动态类型的解析，所以 JVM 无法对这些代码进行优化。因此，反射操作的效率要比那些非反射操作低得多。我们应该避免在经常被执行的代码或对性能要求很高的程序中使用反射。
- **安全限制** ：使用反射技术要求程序必须在一个没有安全限制的环境中运行。如果一个程序必须在有安全限制的环境中运行，如 Applet，那么这就是个问题了。
- **内部暴露** ：由于反射允许代码执行一些在正常情况下不被允许的操作（比如访问私有的属性和方法），所以使用反射可能会导致意料之外的副作用，这可能导致代码功能失调并破坏可移植性。反射代码破坏了抽象性，因此当平台发生改变的时候，代码的行为就有可能也随着变化。

### Class类详解

https://blog.csdn.net/dufufd/article/details/80537638

 在java世界里，一切皆对象。从某种意义上来说，java有两种对象：实例对象和Class对象。每个类的运行时的**类型信息**就是用Class对象表示的。它包含了与类有关的信息。其实我们的实例对象就通过Class对象来创建的。Java使用Class对象执行其RTTI（运行时类型识别，Run-Time Type Identification），多态是基于RTTI实现的。

  每一个类都有一个Class对象，每当编译一个新类就产生一个Class对象，基本类型 (boolean, byte, char, short, int, long, float, and double)有Class对象，数组有Class对象，就连关键字void也有Class对象（void.class）。Class对象对应着java.lang.Class类，如果说类是对象抽象和集合的话，那么Class类就是对类的抽象和集合。

  Class类没有公共的构造方法，Class对象是在类加载的时候由**Java虚拟机**以及通过调用类加载器中的 defineClass 方法自动构造的，因此不能显式地声明一个Class对象。一个类被加载到内存并供我们使用需要经历如下三个阶段：

1. **加载**，这是由类加载器（ClassLoader）执行的。通过一个类的全限定名来获取其定义的二进制字节流（Class字节码），将这个字节流所代表的静态存储结构转化为方法区的运行时数据接口，根据字节码在java堆中生成一个代表这个类的java.lang.Class对象。
2. **链接**。在链接阶段将验证Class文件中的字节流包含的信息是否符合当前虚拟机的要求，为静态域分配存储空间并设置类变量的初始值（默认的零值），并且如果必需的话，将常量池中的符号引用转化为直接引用。
3. **初始化**。到了此阶段，才真正开始执行类中定义的java程序代码。用于执行该类的静态初始器和静态初始块，如果该类有父类的话，则优先对其父类进行初始化。

  
  **所有的类都是在对其第一次使用时，动态加载到JVM中的（懒加载）。**当程序创建第一个对类的静态成员的引用时，就会加载这个类。使用new创建类对象的时候也会被当作对类的静态成员的引用。**因此java程序程序在它开始运行之前并非被完全加载，其各个类都是在必需时才加载的。**这一点与许多传统语言都不同。动态加载使能的行为，在诸如C++这样的静态加载语言中是很难或者根本不可能复制的。

  在类加载阶段，类加载器首先检查这个类的Class对象是否已经被加载。如果尚未加载，默认的类加载器就会根据类的全限定名查找.class文件。在这个类的字节码被加载时，它们会接受验证，以确保其没有被破坏，并且不包含不良java代码。一旦某个类的Class对象被载入内存，我们就可以它来创建这个类的所有对象。



**那么如何获取这个Class对象呢？**

- Obj.getClass();
- Class.forName("类名");
- 具体类名.class

**有了这个Class对象，如何获取类的对象**

可以通过Class对象来调用newInstance()方法，这种形式默认调用的事无餐构造来返回一个对象。



### Class.forName()/getClass()/Class.class区别

https://www.cnblogs.com/Seachal/p/5371733.html

1.出现时机：**Class.forName()**是在运行时加载，**Class.class** **和getClass（）**是在编译时期加载

2.**Class.forName**是一个静态方法，**Class.class**是所有类的一个属性，**getClass()**是所有对象（Object类的对象）的成员方法



3.生成Class对象

**Class.class** 的形式会使 JVM 将使用类装载器将类装入内存（前提是类还没有装入内存），不做类的初始化工作，返回 Class 对象。

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

**Class.forName()**会调用类加载器去加载类并做类的静态初始化，返回 Class 对象。

**getClass（）** 的形式会对类进行静态初始化、非静态初始化，返回引用运行时真正所指的对象（因为子对象的引用可能会赋给父对象的引用变量中）所属的类的 Class 对象。

使用new 创建一个对象实例的时候，就自动加载这个类到内存中，并进行初始化。，所以执行d.getClass()的时候就直接从堆中返回该类Class引用

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



## final、finally、finalize的区别

https://blog.csdn.net/cyl101816/article/details/67640843

**final：**前文已经有描述

**finally:**是在异常处理时提供finally块来执行任何清除操作。不管有没有异常被抛出、捕获，finally块都会被执行。try块中的内容是在无异常时执行到结束。catch块中的内容，是在try块内容发生catch所声明的异常时，跳转到catch块中执行。finally块则是无论异常是否发生，都会执行finally块的内容，所以在代码逻辑中有需要无论发生什么都必须执行的代码，就可以放在finally块中。

**finalize:**是方法名。java技术允许使用finalize（）方法在垃圾收集器将对象从内存中清除出去之前做必要的清理工作。这个方法是由垃圾收集器在确定这个对象没有被引用时，会自动调用这个方法。它是在object类中定义的，因此所有的类都继承了它。子类覆盖finalize（）方法以整理系统资源或者被执行其他清理工作。finalize（）方法是在垃圾收集器删除对象之前对这个对象调用的。 
				但是finalize（）调用时机并不确定，有时候资源已经耗尽，但是gc仍然没有触发，所以不能依赖finalize()来回收占用的资源。

