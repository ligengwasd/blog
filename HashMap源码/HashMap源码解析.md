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

# 3 - HashMap.hash()方法

```java
	static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

从上面的代码可以看到key的hash值的计算方法。key的hash值高16位不变，低16位与高16位异或作为key的最终hash值。（h >>> 16，表示无符号右移16位，高位补0，任何数跟0异或都是其本身，因此key的hash值高16位不变。） 

![](https://github.com/ligengwasd/blog/blob/master/HashMap%E6%BA%90%E7%A0%81/images/20160408155045341.jpg?raw=true)

为什么要这么干呢？  这个与HashMap中table下标的计算有关。

```java
n = table.length;
index = （n-1） & hash;
```

因为，table的长度都是2的幂，因此index仅与hash值的低n位有关（此n非table.leng，而是2的幂指数），hash值的高位都被与操作置为0了。  假设table.length=2^4=16。 

![](https://github.com/ligengwasd/blog/blob/master/HashMap%E6%BA%90%E7%A0%81/images/20160408155102734.jpg?raw=true)

由上图可以看到，只有hash值的低4位参与了运算。  

所以这个方法的作用是让key的原始hash值的高位和低位都参与运算。

#  4 - 链表节点Node

```java
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;//哈希值
        final K key;//key
        V value;//value
        Node<K,V> next;//链表后置节点

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        public final K getKey()        { return key; }
        public final V getValue()      { return value; }
        public final String toString() { return key + "=" + value; }

        //每一个节点的hash值，是将key的hashCode 和 value的hashCode 亦或得到的。
        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }
        //设置新的value 同时返回旧value
        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }

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
    }
```

**这是一个单链表~。**

**每一个节点的hash值，是将key的hashCode 和 value的hashCode 亦或得到的。**

# 5 - 扩容函数resize()

**初始化或加倍哈希桶大小。如果是当前哈希桶是null,分配符合当前阈值的初始容量目标。  否则，因为我们扩容成以前的两倍。  在扩容时，要注意区分以前在哈希桶相同index的节点，现在是在以前的index里，还是index+oldlength 里**

```java
    final Node<K,V>[] resize() {
        //oldTab 为当前表的哈希桶
        Node<K,V>[] oldTab = table;
        //当前哈希桶的容量 length
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        //当前的阈值
        int oldThr = threshold;
        //初始化新的容量和阈值为0
        int newCap, newThr = 0;
        //如果当前容量大于0
        if (oldCap > 0) {
            //如果当前容量已经到达上限
            if (oldCap >= MAXIMUM_CAPACITY) {
                //则设置阈值是2的31次方-1
                threshold = Integer.MAX_VALUE;
                //同时返回当前的哈希桶，不再扩容
                return oldTab;
            }//否则新的容量为旧的容量的两倍。 
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)//如果旧的容量大于等于默认初始容量16
                //那么新的阈值也等于旧的阈值的两倍
                newThr = oldThr << 1; // double threshold
        }//如果当前表是空的，但是有阈值。代表是初始化时指定了容量、阈值的情况
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;//那么新表的容量就等于旧的阈值
        else {//如果当前表是空的，而且也没有阈值。代表是初始化时没有任何容量/阈值参数的情况               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;//此时新表的容量为默认的容量 16
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);//新的阈值为默认容量16 * 默认加载因子0.75f = 12
        }
        if (newThr == 0) {//如果新的阈值是0，对应的是  当前表是空的，但是有阈值的情况
            float ft = (float)newCap * loadFactor;//根据新表容量 和 加载因子 求出新的阈值
            //进行越界修复
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        //更新阈值 
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
        //根据新的容量 构建新的哈希桶
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        //更新哈希桶引用
        table = newTab;
        //如果以前的哈希桶中有元素
        //下面开始将当前哈希桶中的所有节点转移到新的哈希桶中
        if (oldTab != null) {
            //遍历老的哈希桶
            for (int j = 0; j < oldCap; ++j) {
                //取出当前的节点 e
                Node<K,V> e;
                //如果当前桶中有元素,则将链表赋值给e
                if ((e = oldTab[j]) != null) {
                    //将原哈希桶置空以便GC
                    oldTab[j] = null;
                    //如果当前链表中就一个元素，（没有发生哈希碰撞）
                    if (e.next == null)
                        //直接将这个元素放置在新的哈希桶里。
                        //注意这里取下标 是用 哈希值 与 桶的长度-1 。 由于桶的长度是2的n次方，这么做其实是等于 一个模运算。但是效率更高
                        newTab[e.hash & (newCap - 1)] = e;
                        //如果发生过哈希碰撞 ,而且是节点数超过8个，转化成了红黑树（暂且不谈 避免过于复杂， 后续专门研究一下红黑树）
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    //如果发生过哈希碰撞，节点数小于8个。则要根据链表上每个节点的哈希值，依次放入新哈希桶对应下标位置。
                    else { // preserve order
                        //因为扩容是容量翻倍，所以原链表上的每个节点，现在可能存放在原来的下标，即low位， 或者扩容后的下标，即high位。 high位=  low位+原哈希桶容量
                        //低位链表的头结点、尾节点
                        Node<K,V> loHead = null, loTail = null;
                        //高位链表的头节点、尾节点
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;//临时节点 存放e的下一个节点
                        do {
                            next = e.next;
                            //这里又是一个利用位运算 代替常规运算的高效点： 利用哈希值 与 旧的容量，可以得到哈希值去模后，是大于等于oldCap还是小于oldCap，等于0代表小于oldCap，应该存放在低位，否则存放在高位
                            if ((e.hash & oldCap) == 0) {
                                //给头尾节点指针赋值
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }//高位也是相同的逻辑
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }//循环直到链表结束
                        } while ((e = next) != null);
                        //将低位链表存放在原index处，
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        //将高位链表存放在新index处
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```

# 6 - put()函数

往哈希表里插入一个节点的`putVal`函数,如果参数`onlyIfAbsent`是true，那么不会覆盖相同key的值value。如果`evict`是false。那么表示是在初始化时调用的

```java
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        //tab存放 当前的哈希桶， p用作临时链表节点  
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        //如果当前哈希表是空的，代表是初始化
        if ((tab = table) == null || (n = tab.length) == 0)
            //那么直接去扩容哈希表，并且将扩容后的哈希桶长度赋值给n
            n = (tab = resize()).length;
        //如果当前index的节点是空的，表示没有发生哈希碰撞。 直接构建一个新节点Node，挂载在index处即可。
        //这里再啰嗦一下，index 是利用 哈希值 & 哈希桶的长度-1，替代模运算
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {//否则 发生了哈希冲突。
            //e
            Node<K,V> e; K k;
            //如果哈希值相等，key也相等，则是覆盖value操作
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;//将当前节点引用赋值给e
            else if (p instanceof TreeNode)//红黑树暂且不谈
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {//不是覆盖操作，则插入一个普通链表节点
                //遍历链表
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {//遍历到尾部，追加新节点到尾部
                        p.next = newNode(hash, key, value, null);
                        //如果追加节点后，链表数量》=8，则转化为红黑树
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    //如果找到了要覆盖的节点
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            //如果e不是null，说明有需要覆盖的节点，
            if (e != null) { // existing mapping for key
                //则覆盖节点值，并返回原oldValue
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                //这是一个空实现的函数，用作LinkedHashMap重写使用。
                afterNodeAccess(e);
                return oldValue;
            }
        }
        //如果执行到了这里，说明插入了一个新的节点，所以会修改modCount，以及返回null。

        //修改modCount
        ++modCount;
        //更新size，并判断是否需要扩容。
        if (++size > threshold)
            resize();
        //这是一个空实现的函数，用作LinkedHashMap重写使用。
        afterNodeInsertion(evict);
        return null;
    }
```

小结： 

* 运算尽量都用位运算代替，更高效。 
* 对于扩容导致需要新建数组存放更多元素时，除了要将老数组中的元素迁移过来，也记得将老数组中的引用置null，以便GC 
* 取下标 是用 哈希值 与运算 （桶的长度-1） i = (n - 1) & hash。 由于桶的长度是2的n次方，这么做其实是等于 一个模运算。但是效率更高 
* 扩容时，如果发生过哈希碰撞，节点数小于8个。则要根据链表上每个节点的哈希值，依次放入新哈希桶对应下标位置。 
* 因为扩容是容量翻倍，所以原链表上的每个节点，现在可能存放在原来的下标，即low位， 或者扩容后的下标，即high位。 high位= low位+原哈希桶容量 
* 利用哈希值 与运算 旧的容量 ，if ((e.hash & oldCap) == 0),可以得到哈希值去模后，是大于等于oldCap还是小于oldCap，等于0代表小于oldCap，应该存放在低位，否则存放在高位。这里又是一个利用位运算 代替常规运算的高效点 
* 如果追加节点后，链表数量》=8，则转化为红黑树 
* 插入节点操作时，有一些空实现的函数，用作LinkedHashMap重写使用。
---------------------
