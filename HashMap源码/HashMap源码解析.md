# 1 - HashMap的定义

话不多说，首先从HashMap的一些基础开始。我们先看一下HashMap的定义：

```java
public class HashMap<K,V> extends AbstractMap<K,V> implements Map<K,V>, Cloneable, Serializable
```

我们可以看出，HashMap继承了AbstractMap<K,V>抽象类，实现了Map<K,V>的方法。

#  2 - HashMap的属性

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

# 2 - 初始化

一般来说，我们初始化的时候会这样写： 

```java
Map<K,V> map = new HashMap<K,V>();
```

这个过程发生了什么呢？我们看看源码。

```java
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        // 初始化容量不能超过最大值
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);
}
```

 tableSizeFor方法

```java
	/**
     * Returns a power of two size for the given target capacity.
     * 返回比cap大的最小的2的幂次方
     */
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```

由此可以看到，当在实例化HashMap实例时，如果给定了initialCapacity，由于HashMap的capacity都是2的幂，因此这个方法用于找到大于等于initialCapacity的最小的2的幂（initialCapacity如果就是2的幂，则返回的还是这个数）。 
下面分析这个算法： 
首先，为什么要对cap做减1操作。int n = cap - 1; 
这是为了防止，cap已经是2的幂。如果cap已经是2的幂， 又没有执行这个减1操作，则执行完后面的几条无符号右移操作之后，返回的capacity将是这个cap的2倍。如果不懂，要看完后面的几个无符号右移之后再回来看看。 
下面看看这几个无符号右移操作： 
如果n这时为0了（经过了cap-1之后），则经过后面的几次无符号右移依然是0，最后返回的capacity是1（最后有个n+1的操作）。 
这里只讨论n不等于0的情况。 

**第一次右移**

```java
n |= n >>> 1;
```

由于n不等于0，则n的二进制表示中总会有一bit为1，这时考虑最高位的1。通过无符号右移1位，则将最高位的1右移了1位，再做或操作，使得n的二进制表示中与最高位的1紧邻的右边一位也为1，如000011xxxxxx。 

**第二次右移**

```java
n |= n >>> 2;
```

注意，这个n已经经过了n |= n >>> 1; 操作。假设此时n为000011xxxxxx ，则n无符号右移两位，会将最高位两个连续的1右移两位，然后再与原来的n做或操作，这样n的二进制表示的高位中会有4个连续的1。如00001111xxxxxx 。 
**第三次右移**

```java
n |= n >>> 4;
```

这次把已经有的高位中的连续的4个1，右移4位，再做或操作，这样n的二进制表示的高位中会有8个连续的1。如00001111 1111xxxxxx 。 
以此类推 
注意，容量最大也就是32bit的正数，因此最后n |= n >>> 16; ，最多也就32个1，但是这时已经大于了MAXIMUM_CAPACITY ，所以取值到MAXIMUM_CAPACITY 。 
**举一个例子说明下吧。** 

![](https://raw.githubusercontent.com/ligengwasd/blog/master/HashMap%E6%BA%90%E7%A0%81/images/20160408183651111.jpg)




这个算法着实牛逼啊！

注意，得到的这个capacity却被赋值给了threshold。

```java
this.threshold = tableSizeFor(initialCapacity);
```

开始以为这个是个Bug，感觉应该这么写：

```java
this.threshold = tableSizeFor(initialCapacity) * this.loadFactor;
```

这样才符合threshold的意思（当HashMap的size到达threshold这个阈值时会扩容）。 
但是，请注意，在构造方法中，并没有对table这个成员变量进行初始化，table的初始化被推迟到了put方法中，在put方法中会对threshold重新计算，put方法的具体实现请看这篇博文。

#  3 - put()方法

 

 