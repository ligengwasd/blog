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
  // 查找缓存
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

采用模板方法，如update()方法最终调用的doUpdate()方法，而doUpdate交给子类实现。继承BaseExecutor的子类只要实现doUpdate()、doQuery()、doQueryCursor()、doFlushStatements()四个方法，其它功能在BaseExecutor中实现。

**BaseExecutor两大功能：**

- **缓存管理（包括延迟加载）**
- **事务管理**



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
// 用来记录嵌套查询的层数，和DefaultResultHandler类有关。
protected int queryStack;
```

## 3.2 一级缓存简介

会话级别的缓存，在MyBatis中每创建一个`SQLSession`对象，就表示开启一次数据库会话。生命周期与SQLSession（Executor）相同。

Executor.update、close方法会让缓存失效。

## 3.3 一级缓存的管理

### 3.3.1 createCacheKey的方法

**创建缓存键。可以看到`CacheKey`由MappedStatement的id、对应的offset和limit、SQL语句（“包含？占位符”）、用户传入的实参、Environment的id五部分组成。**

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

### 3.3.2 query方法

```java
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
   BoundSql boundSql = ms.getBoundSql(parameter);
   CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
   return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
}
```

```java
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
  ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
  if (closed) {
    throw new ExecutorException("Executor was closed.");
  }
  if (queryStack == 0 && ms.isFlushCacheRequired()) {
    clearLocalCache();
  }
  List<E> list;
  try {
    queryStack++;// 查询层数
    // 查询一级缓存
    list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
    if (list != null) {
      // 针对存储过程调用的处理，其功能是：在一级缓存命中时，获取缓存中保存的输出类型参数。
      // 并设置到用户传入的实参(parameter)对象中，代码就不贴出来啦，读者可以查看源码实习。
      handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
    } else {
      // 其中会调用doQuery()方法完成数据库查询，并得到映射后的结果对象，doQuery()方法是一个
      // 抽象方法，也就是上述四个基本方法之一，由BaseExecutor的子类具体实现。
      list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
    }
  } finally {
    queryStack--;// 当前查询完成，查询层数减少
  }
  if (queryStack == 0) {
    for (DeferredLoad deferredLoad : deferredLoads) {
      // 在最外层的查询结束时，所有嵌套查询也已经完成，相关缓存也已经完全加载，
      // 所以在这里可以触发DeferredLoad记载一级缓存中记录的嵌套查询的结果对象。
      deferredLoad.load();
    }
    // issue #601
    // 加载完成后清空deferredLoads集合。
    deferredLoads.clear();
    if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
      // issue #482
      // 根据LocalCacheScope配置决定是否清空一级缓存，
      // LocalCacheScope配置是影响一级缓存中结果对象存活时长的第二个方面
      clearLocalCache();
    }
  }
  return list;
}
```

一级缓存的两个功能：

- 缓存查询结果
- 如果一级缓存中记录的嵌套查询结果对象并未加载，可以通过DeferredLoad实现类似延迟加载的功能。

### 3.3.3 deferLoad方法

```java
public void deferLoad(MappedStatement ms, MetaObject resultObject, String property, CacheKey key, Class<?> targetType) {
  if (closed) {
    throw new ExecutorException("Executor was closed.");
  }
  // 创建DeferredLoad类
  DeferredLoad deferredLoad = new DeferredLoad(resultObject, property, key, localCache, configuration, targetType);
  if (deferredLoad.canLoad()) {
    // 一级缓存中已经记录了指定查询的结果对象，直接从缓存中加载对象，并设置到外层对象中
    deferredLoad.load();
  } else {
    // 将DeferredLoad对象添加到deferredLoads队列中，待整个外层查询结束后，再加载该对象。
    deferredLoads.add(new DeferredLoad(resultObject, property, key, localCache, configuration, targetType));
  }
}
```

### 3.3.4 queryFromDatabase方法

```java
private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
  List<E> list;
  // 占位符放入缓存
  localCache.putObject(key, EXECUTION_PLACEHOLDER);
  try {
    // 查库
    list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
  } finally {
    // 删除占位符
    localCache.removeObject(key);
  }
  // 缓存真正的结果对象
  localCache.putObject(key, list);
  if (ms.getStatementType() == StatementType.CALLABLE) {
    // 如果是存储过程，缓存输入类型的参数。
    localOutputParameterCache.putObject(key, parameter);
  }
  return list;
}
```

### 3.3.5 DeferredLoad内部类

#### 3.3.5.1 属性

```java
private final MetaObject resultObject;// 外层对象对应的MetaObject对象
private final String property;// 延迟加载的属性名称
private final Class<?> targetType;
private final CacheKey key;// 延迟加载的结果对象在一级缓存中相应的CacheKey对象
// 一级缓存，与外层BaseExecutor.localCache字段指向统一个PerpetualCache对象
private final PerpetualCache localCache;
private final ObjectFactory objectFactory;
private final ResultExtractor resultExtractor;// 负责结果对象的类型转换
```

#### 3.3.5.2 canLoad方法

```java
// 判断对象是否加载到了缓存中。
public boolean canLoad() {
  return localCache.getObject(key) != null && localCache.getObject(key) != EXECUTION_PLACEHOLDER;
}
```

#### 3.3.5.3 load方法

```java
public void load() {
  @SuppressWarnings( "unchecked" )
  // we suppose we get back a List
  // 从缓存中查询指定的结果对象
  List<Object> list = (List<Object>) localCache.getObject(key);
  // 将缓存的结果对象转换成指定类型
  Object value = resultExtractor.extractObjectFromList(list, targetType);
  // 设置到外层对象的对应属性
  resultObject.setValue(property, value);
}
```

### 3.3.6 update方法

执行SQL(包括update、delete、insert)前清除缓存。

## 3.4 事务相关操作

在`BatchExecutor`实现中，可以缓存多条SQL语句，等待合适的时机将缓存的多条SQL一起执行。`Executor.flushStatements()`方法主要针对批处理多条SQL语句的，它会调用`doFlushStatements()`这个基本方法处理Executor中缓存的多条SQL语句。在BatchExecutor.commit、rollback方法中都会首先调用flushStatements方法，然后再执行相关事务操作。

# 4 - SimpleExecutor

继承自BaseExecutor类，实现doUpdate()、doQuery()、doQueryCursor()、doFlushStatements()四个方法即可。

看下doUpdate()方法：

```java
public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
  Statement stmt = null;
  try {
    Configuration configuration = ms.getConfiguration();
    StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
    stmt = prepareStatement(handler, ms.getStatementLog());
    return handler.<E>query(stmt, resultHandler);
  } finally {
    closeStatement(stmt);
  }
}
```

prepareStatement方法的实现

```java
private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
  Statement stmt;
  Connection connection = getConnection(statementLog);
  stmt = handler.prepare(connection, transaction.getTimeout());// 创建Statement对象
  handler.parameterize(stmt);// 处理占位符
  return stmt;
}
```

# 5 - ReuseExecutor

# 6 - BatchExecutor

# 7 - CachingExecutor