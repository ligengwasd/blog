​	



# 1 - 源码所在包

类型转换器源码所在包：
```java
org.apache.ibatis.cache
```

![](https://raw.githubusercontent.com/ligengwasd/blog/master/MyBatis%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB/images/5.29.14.png)



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

## 4.2 FifoCache类









