# 1 - 包路径

```
org.apache.ibatis.executor
```

# 2 - Executor接口

```java
public interface Executor {

  ResultHandler NO_RESULT_HANDLER = null;
  // 执行update、insert、delete三种类型的SQL语句
  int update(MappedStatement ms, Object parameter) throws SQLException;
  // 执行select语句，返回值为对象列表或者游标对象
  <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey cacheKey, BoundSql boundSql) throws SQLException;

  <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException;

  <E> Cursor<E> queryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds) throws SQLException;
  // 批量执行SQL语句
  List<BatchResult> flushStatements() throws SQLException;
  // 提交事务
  void commit(boolean required) throws SQLException;
  // 回滚事务
  void rollback(boolean required) throws SQLException;
  // 创建CacheKey
  CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql);
  // 查找焕春
  boolean isCached(MappedStatement ms, CacheKey key);
  // 清空一级缓存
  void clearLocalCache();
  // 延迟加载一级缓存中的数据，deferLoad后面讲。
  void deferLoad(MappedStatement ms, MetaObject resultObject, String property, CacheKey key, Class<?> targetType);
  // 获取事务对象
  Transaction getTransaction();
  // 关闭事务
  void close(boolean forceRollback);
  // 检测Executor是否已关闭
  boolean isClosed();

  void setExecutorWrapper(Executor executor);

}
```

Executor的子类：

<img width="565" height="176" src="https://raw.githubusercontent.com/ligengwasd/blog/master/读书笔记-MyBatis技术内幕/images/2.27.54.png"/>

`CachingExecutor`：功能是为Executor添加二级缓存。

# 3 - BaseExecutor

## 3.1 属性

```java
protected Transaction transaction;
protected Executor wrapper;
// 延迟加载队列
protected ConcurrentLinkedQueue<DeferredLoad> deferredLoads;
// 一级缓存，用于缓存该Executor对象查询结果集映射得到的结果对象
protected PerpetualCache localCache;
// 一级缓存，用于缓存输出类型的参数。
protected PerpetualCache localOutputParameterCache;
protected Configuration configuration;
```

## 3.2 一级缓存简介

会话级别的缓存，在MyBatis中每创建一个`SQLSession`对象，就表示开启一次数据库会话。生命周期与SQLSession（Executor）相同。

Executor.update、close方法会让缓存失效。

## 3.3 一级缓存的管理

### 3.3.1 创建缓存键的方法

**可以看到`CacheKey`由MappedStatement的id、对应的offset和limit、SQL语句（“包含？占位符”）、用户传入的实参、Environment的id五部分组成。**

```java
public CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql) {
  if (closed) {
    throw new ExecutorException("Executor was closed.");
  }
  CacheKey cacheKey = new CacheKey();
  cacheKey.update(ms.getId());// 将MappedStatement的id添加到CacheKey中
  cacheKey.update(rowBounds.getOffset());// 将offset添加到CacheKey中
  cacheKey.update(rowBounds.getLimit());// 将limit添加到CacheKey中
  cacheKey.update(boundSql.getSql());// 将SQL语句添加到CacheKey中
  List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
  TypeHandlerRegistry typeHandlerRegistry = ms.getConfiguration().getTypeHandlerRegistry();
  // mimic DefaultParameterHandler logic
  // 获取用户传入的实参，并添加到CacheKey中
  for (ParameterMapping parameterMapping : parameterMappings) {
    if (parameterMapping.getMode() != ParameterMode.OUT) {// 过滤输出类型的参数
      Object value;
      String propertyName = parameterMapping.getProperty();
      if (boundSql.hasAdditionalParameter(propertyName)) {
        value = boundSql.getAdditionalParameter(propertyName);
      } else if (parameterObject == null) {
        value = null;
      } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
        value = parameterObject;
      } else {
        MetaObject metaObject = configuration.newMetaObject(parameterObject);
        value = metaObject.getValue(propertyName);
      }
      cacheKey.update(value);
    }
  }
  if (configuration.getEnvironment() != null) {
    // issue #176
    // 如果Environment不为空，添加到CacheKey中
    cacheKey.update(configuration.getEnvironment().getId());
  }
  return cacheKey;
}
```

### 3.3.2 

```java
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
   BoundSql boundSql = ms.getBoundSql(parameter);
   CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
   return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
}
```





## 3.4 事务相关操作

