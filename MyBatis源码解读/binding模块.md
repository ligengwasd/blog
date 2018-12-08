# 1 - 包路径

```
org.apache.ibatis.binding;
```

# 2 - 功能描述

将mybatis的mapper接口中的方法和sql语句绑定。

核心类：

- MapperRegistry：注册器
- MapperProxyFactory：注册器中存储的是MapperProxyFactory对象。
- MapperProxy：根据mapper接口生成的代理对象。由MapperProxyFactory生成。
- MapperMethod：对应mapper接口中的方法。

# 3 - MapperRegistry&MapperProxyFactory

```java
// Configuration对象，Mybatis全局唯一配置对象。
private final Configuration config;
// 记录了mapper接口与对应的MapperProxyFactory之间的关系
private final Map<Class<?>, MapperProxyFactory<?>> knownMappers = new HashMap<Class<?>, MapperProxyFactory<?>>();
```

