​	



# 1 - 源码所在包

缓存模块源码所在包：
```java
org.apache.ibatis.cache
```

<img width="465" height="622" src="https://raw.githubusercontent.com/ligengwasd/blog/master/MyBatis%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB/images/5.29.14.png"/>

# 2 - 代码结构（装饰器模式）

![](https://raw.githubusercontent.com/ligengwasd/blog/master/MyBatis%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB/images/11.28.15.png)

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

**功能**：限定缓存数量上限，超出上限，清除最老的缓存。

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







