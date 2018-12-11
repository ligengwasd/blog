# 1 - StatementHandler

StatementHandler接口是MyBatis的核心接口之一，完成了MyBatis最核心的工作。也是Executor的基础。

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

StatementHandler接口的多种实现

<img width="1162" height="330" src="https://raw.githubusercontent.com/ligengwasd/blog/master/读书笔记-MyBatis技术内幕/images/11.29.51.png"/>



