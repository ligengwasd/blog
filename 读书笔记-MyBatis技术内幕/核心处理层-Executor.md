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

# 3 - 

