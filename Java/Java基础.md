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

1. **HashMap的工作原理**

   HashMap 底层是 hash 数组和单向链表实现，数组中的每个元素都是链表，由 Node 内部类（实现 Map.Entry接口）实现，HashMap 通过 put & get 方法存储和获取。

   存储对象时，将 K/V 键值传给 put() 方法：

   ①、调用 hash(K) 方法计算 K 的 hash 值，然后结合数组长度，计算得数组下标；

   ②、调整数组大小（当容器中的元素个数大于 capacity * loadfactor 时，容器会进行扩容resize 为 2n）；

   ③、
   i.如果 K 的 hash 值在 HashMap 中不存在，则执行插入，若存在，则发生碰撞；

   ii.如果 K 的 hash 值在 HashMap 中存在，且它们两者 equals 返回 true，则更新键值对；

   iii. 如果 K 的 hash 值在 HashMap 中存在，且它们两者 equals 返回 false，则插入链表的尾部（尾插法）或者红黑树中（树的添加方式）。（JDK 1.7 之前使用头插法、JDK 1.8 使用尾插法）（注意：当碰撞导致链表大于 TREEIFY_THRESHOLD = 8 时，就把链表转换成红黑树）

   获取对象时，将 K 传给 get() 方法：①、调用 hash(K) 方法（计算 K 的 hash 值）从而获取该键值所在链表的数组下标；②、顺序遍历链表，equals()方法查找相同 Node 链表中 K 值对应的 V 值。

   hashCode 是定位的，存储位置；equals是定性的，比较两者是否相等。

2. 