# 1、HashMap的定义

话不多说，首先从HashMap的一些基础开始。我们先看一下HashMap的定义：

```java
public class HashMap<K,V> extends AbstractMap<K,V> implements Map<K,V>, Cloneable, Serializable
```

我们可以看出，HashMap继承了AbstractMap<K,V>抽象类，实现了Map<K,V>的方法。

#  2、HashMap的属性

```java
//默认容量
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;
//最大容量
static final int MAXIMUM_CAPACITY = 1 << 30;
//默认加载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;
//链表转成红黑树的阈值
static final int TREEIFY_THRESHOLD = 8;
//红黑树转为链表的阈值
static final int UNTREEIFY_THRESHOLD = 6;
//存储方式由链表转成红黑树的容量的最小阈值
static final int MIN_TREEIFY_CAPACITY = 64;
//HashMap中存储的键值对的数量
transient int size;
//扩容阈值，当size>=threshold时，就会扩容
int threshold;
//HashMap的加载因子
final float loadFactor;
```

> 这里我们要知道<<运算符的意义，表示移位操作，每次向左移动一位(相对于二进制来说)，表示乘以2，此处1<<4表示00001中的1向左移动了4位，变成了10000，换算成十进制就是2^4=16，也就是HashMap的默认容量就是16。Java中还有一些位操作符，比如类似的>>(右移)，还有>>>(无符号右移)等，也是需要我们掌握的。这些位操作符的计算速度很快，我们在平时的工作中可以使用它们来提升我们系统的性能。

这里我们需要加载因子(load_factor)，加载因子默认为0.75，当HashMap中存储的元素的数量大于(容量×加载因子)，也就是默认大于16*0.75=12时，HashMap会进行扩容的操作。





 

 

 

 

 