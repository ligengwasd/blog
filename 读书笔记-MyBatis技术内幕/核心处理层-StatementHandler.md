# 1 - StatementHandler

`StatementHandler`接口是MyBatis的核心接口之一，完成了MyBatis最核心的工作。也是`Executor`的基础。

```java
public interface StatementHandler {
  // 从连接中获取一个statement
  Statement prepare(Connection connection, Integer transactionTimeout)
      throws SQLException;
  // 绑定statement执行是所需的实参
  void parameterize(Statement statement)
      throws SQLException;
  // 批量执行SQL语句
  void batch(Statement statement)
      throws SQLException;
  // 执行update/insert/delete语句
  int update(Statement statement)
      throws SQLException;
  // 执行select语句
  <E> List<E> query(Statement statement, ResultHandler resultHandler)
      throws SQLException;

  <E> Cursor<E> queryCursor(Statement statement)
      throws SQLException;

  BoundSql getBoundSql();
  // 获取其中封装的ParameterHandler，后面详述。
  ParameterHandler getParameterHandler();

}
```

`StatementHandler`接口的多种实现

<img width="1162" height="330" src="https://raw.githubusercontent.com/ligengwasd/blog/master/读书笔记-MyBatis技术内幕/images/11.29.51.png"/>

# 2 - RoutingStatementHandler

`RoutingStatementHandler`很简单就是一个路由的作用。

```
public RoutingStatementHandler(Executor executor, MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {

  switch (ms.getStatementType()) {
    case STATEMENT:
      delegate = new SimpleStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
      break;
    case PREPARED:
      delegate = new PreparedStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
      break;
    case CALLABLE:
      delegate = new CallableStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
      break;
    default:
      throw new ExecutorException("Unknown statement type: " + ms.getStatementType());
  }

}
```

# 3 - BaseStatementHandler

`BaseStatementHandler`是一个实现了`StatementHandler`接口的抽象类，它只提供了一些参数绑定相关的方法，并没有实现操作数据库的方法。核心字段如下：

```java
// 记录使用的ParamterHanlder对象，ParameterHandler的主要功能是为SQL语句绑定实参，
// 也就是使用传入的实参替换SQL语句的"?"占位符，后面详细介绍。
protected final ParameterHandler parameterHandler;
// 记录使用的ResultSetHandler对象，其主要功能是将执行结果映射成结果对象
protected final ResultSetHandler resultSetHandler;

protected final Executor executor;
protected final MappedStatement mappedStatement;
// 记录了用户设置的offset和limit，用于在结果集中定位映射的起始位置和结束位置
protected final RowBounds rowBounds;

protected BoundSql boundSql;
```

在`BaseStatementHandler`的构造方法中还会调用`KeyGenerator.processBefore`方法初始化SQL语句的主键，具体实现如下

```java
protected BaseStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
	……
    if (boundSql == null) { // issue #435, get the key before calculating the statement
      generateKeys(parameterObject);
      boundSql = mappedStatement.getBoundSql(parameterObject);
    }
    ……
  }
protected void generateKeys(Object parameter) {
  KeyGenerator keyGenerator = mappedStatement.getKeyGenerator();
  ErrorContext.instance().store();
  // 调用KeyGenerator.processBefore方法
  keyGenerator.processBefore(executor, mappedStatement, null, parameter);
  ErrorContext.instance().recall();
}
```

`BaseStatementHandler`实现了`StatementHandler`接口中的`prepare()`方法

```java
public Statement prepare(Connection connection, Integer transactionTimeout) throws SQLException {
  ……
    // 初始化java.sql.Statement对象
    statement = instantiateStatement(connection);
    // 设置超时时间
    setStatementTimeout(statement, transactionTimeout);
    // 设置fetchSize
    setFetchSize(statement);
  ……
}
```

`BaseStatementHandler`另一个重要的组件`ParameterHandler`

