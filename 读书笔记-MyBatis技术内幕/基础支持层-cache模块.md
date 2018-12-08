​	



# 1 - 源码所在包

缓存模块源码所在包：
```java
org.apache.ibatis.cache
```

<img width="465" height="622" src="https://raw.githubusercontent.com/ligengwasd/blog/master/读书笔记-MyBatis技术内幕/images/5.29.14.png"/>

# 2 - 代码结构（装饰器模式）

![](https://raw.githubusercontent.com/ligengwasd/blog/master/读书笔记-MyBatis技术内幕/images/11.28.15.png)

对应：

Component：Cache接口

ConcreteComponet：PerpetualCache类

ConcreteDecorator：decorators包

# 3 - PerpetualCache类提供基本的缓存功能

底层用HashMap作为缓存的数据数据仓库。很简单。

```java
public class PerpetualCache implements Cache {
    private String id;
    // 缓存数据仓库
    private Map<Object, Object> cache = new HashMap();

    public PerpetualCache(String id) {
        this.id = id;
    }

    public String getId() {
        return this.id;
    }

    public int getSize() {
        return this.cache.size();
    }

    public void putObject(Object key, Object value) {
        this.cache.put(key, value);
    }

    public Object getObject(Object key) {
        return this.cache.get(key);
    }

    public Object removeObject(Object key) {
        return this.cache.remove(key);
    }

    public void clear() {
        this.cache.clear();
    }
	// 留给子类覆盖
    public ReadWriteLock getReadWriteLock() {
        return null;
    }
	// 重写equals方法
    public boolean equals(Object o) {
        if (this.getId() == null) {
            throw new CacheException("Cache instances require an ID.");
        } else if (this == o) {
            return true;
        } else if (!(o instanceof Cache)) {
            return false;
        } else {
            Cache otherCache = (Cache)o;
            return this.getId().equals(otherCache.getId());
        }
    }
	// 重写hashCode方法
    public int hashCode() {
        if (this.getId() == null) {
            throw new CacheException("Cache instances require an ID.");
        } else {
            return this.getId().hashCode();
        }
    }
}
```

# 4 - decorators包为缓存动态添加扩展功能

## 4.1 BlockingCache类

**功能**：保证只有一个线程到数据库中查找指定key对应的数据

简单注释一下逻辑：

```java
    // 阻塞时长
    private long timeout;
    // 被装饰的底层Cache对象
    private final Cache delegate;
    // 每个key对应的锁
    private final ConcurrentHashMap<Object, ReentrantLock> locks;

    public Object getObject(Object key) {
        this.acquireLock(key);
        Object value = this.delegate.getObject(key);
		// 命中缓存，释放锁。调用方必须自己加锁。
        if (value != null) {
            this.releaseLock(key);
        }

        return value;
    }
```

## 4.2 FifoCache类

**功能**：限定缓存数量上限，如果超出上限，清除最老的缓存。

```java
    private final Cache delegate;
	// 记录key进入缓存的顺序，底层用LinkedList
    private Deque<Object> keyList;
	// 缓存数量上限，超过上限清理缓存。
    private int size;
    public FifoCache(Cache delegate) {
        this.delegate = delegate;
        this.keyList = new LinkedList();
        this.size = 1024;
    }
    public FifoCache(Cache delegate) {
        this.delegate = delegate;
        this.keyList = new LinkedList();
        this.size = 1024;
    }
	// 加入缓存之前，先清理一个最老的缓存。
    public void putObject(Object key, Object value) {
        this.cycleKeyList(key);
        this.delegate.putObject(key, value);
    }
    private void cycleKeyList(Object key) {
        this.keyList.addLast(key);
        if (this.keyList.size() > this.size) {
            Object oldestKey = this.keyList.removeFirst();
            this.delegate.removeObject(oldestKey);
        }
    }

```

## 4.3 LruCache类

**功能**：按照近期最少使用法（**Least Recently Used, LRU**）进行缓存清理。需要清理缓存时，会清除最近使用最少的缓存项。

```java
    private final Cache delegate;
	// LinkedHashMap，有序map，记录key最近的使用情况
    private Map<Object, Object> keyMap;
	// 最少被使用的缓存项
    private Object eldestKey;
    
	public LruCache(Cache delegate) {
        this.delegate = delegate;
        this.setSize(1024);
    }

    public void setSize(final int size) {
        // LinkedHashMap初始化的三个参数，true表示该map记录的顺序是access order，也就是说调用LinkedHashMap.get()方法会改变其记录顺序
        this.keyMap = new LinkedHashMap<Object, Object>(size, 0.75F, true) {
            private static final long serialVersionUID = 4267176411845948333L;
			// 调用LinkedHashMap.put()方法时，会调用该方法。
            protected boolean removeEldestEntry(Entry<Object, Object> eldest) {
                boolean tooBig = this.size() > size;
                if (tooBig) {
                    // 达到缓存上限，更新eldestKey，后面会删除该项
                    LruCache.this.eldestKey = eldest.getKey();
                }

                return tooBig;
            }
        };
    }
	
    public Object getObject(Object key) {
        this.keyMap.get(key);//修改LinkedHashMap中记录顺序
        return this.delegate.getObject(key);
    }
    public void putObject(Object key, Object value) {
        this.delegate.putObject(key, value);//添加缓存项
        this.cycleKeyList(key);
    }
    private void cycleKeyList(Object key) {
        this.keyMap.put(key, key);
        if (this.eldestKey != null) {// 不为空表示，已达到缓存上限
            this.delegate.removeObject(this.eldestKey);// 删除最久未使用的缓存
            this.eldestKey = null;
        }
    }
```

## 4.4 SoftCache & WeakCache

**软引用**：内存不足是清除

**弱引用**：下一次GC清除

**虚引用**：必须指定引用队列，调用get方法始终返回null

**引用队列：**创建SoftReference和WeakReference时可关联一个队列。当SoftReference所关联的对象被回收之后，会将SoftReference对象本身加入该队列。

```java
	// 在SoftCache中，最近使用的一部分缓存项不会被GC回收，这就是通过将其value添加到hardLinksToAvoidGarbageCollection形成强引用。用LinkedList实现。
	private final Deque<Object> hardLinksToAvoidGarbageCollection;
	// 引用队列，记录被回收的对象的<软引用>
	private final ReferenceQueue<Object> queueOfGarbageCollectedEntries;
    private final Cache delegate;
	// 强链接的个数
    private int numberOfHardLinks;

    public SoftCache(Cache delegate) {
        this.delegate = delegate;
        this.numberOfHardLinks = 256;
        this.hardLinksToAvoidGarbageCollection = new LinkedList();
        this.queueOfGarbageCollectedEntries = new ReferenceQueue();
    }
    private static class SoftEntry extends SoftReference<Object> {
        private final Object key;

        SoftEntry(Object key, Object value, ReferenceQueue<Object> garbageCollectionQueue) {
            super(value, garbageCollectionQueue);// value是软引用
            this.key = key;// key是强引用
        }
    }
    public void putObject(Object key, Object value) {
        this.removeGarbageCollectedItems();
        this.delegate.putObject(key, new SoftCache.SoftEntry(key, value, 			this.queueOfGarbageCollectedEntries));
    }
    private void removeGarbageCollectedItems() {
        SoftCache.SoftEntry sv;
        while((sv = (SoftCache.SoftEntry)this.queueOfGarbageCollectedEntries.poll()) != null) {
			// 被GC回收的value对象对应的缓存项清除
            this.delegate.removeObject(sv.key); 
        }

    }	
    public Object getObject(Object key) {
        Object result = null;
        SoftReference<Object> softReference = (SoftReference)this.delegate.getObject(key);
        if (softReference != null) {// 检测缓存中是否有对应的缓存项
            result = softReference.get();
            if (result == null) {// 已经被GC回收
                this.delegate.removeObject(key);// 删除缓存
            } else {
                Deque var4 = this.hardLinksToAvoidGarbageCollection;
                synchronized(this.hardLinksToAvoidGarbageCollection) {
                    // 缓存项的value放在强引用队列里
                    this.hardLinksToAvoidGarbageCollection.addFirst(result);
                    if (this.hardLinksToAvoidGarbageCollection.size() > this.numberOfHardLinks) {
                        // 强引用满了，删除最老的
                        this.hardLinksToAvoidGarbageCollection.removeLast();
                    }
                }
            }
        }

        return result;
    }

    public Object removeObject(Object key) {
        this.removeGarbageCollectedItems();
        return this.delegate.removeObject(key);
    }
```

## 4.5 ScheduledCache类

**功能**：周期性清除所有缓存项

## 4.6 LoggingCache类

**功能**：添加日志

## 4.7 SynchronizedCache类

## 4.8 SerializedCache类

**功能**：缓存序列化

