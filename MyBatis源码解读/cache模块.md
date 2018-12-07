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

# 4 - decorators包为缓存动态添加扩展功能









