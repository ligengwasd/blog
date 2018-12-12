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

```java
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

# 4 - ParameterHandler

在BoundSql中记录的SQL中可能包含"?"占位符，而每个占位符都对应了`BoundSql.parameterMappings`list集合中的一个元素，在该`ParameterMapping`对象中记录了对应的参数名称以及该参数的相关属性

在`ParameterHandler`接口只定义了一个`setParameters()`方法，改方法负责调用`PreparedStatement.set()*`方法为SQL语句绑定实参。`ParameterHandler`只有一个实现类`DefaultParameterHandler`。DefaultParameterHandler的核心字段如下

```java
private final TypeHandlerRegistry typeHandlerRegistry;
// 记录SQL节点相关的配置信息
private final MappedStatement mappedStatement;
// 用户传入的实参对象
private final Object parameterObject;
// 相应的PreparedStatment对象是boundSql中记录的sql创建的，boundSql中还记录了对应的参数相关的属性。
private BoundSql boundSql;
private Configuration configuration;
```

`DefaultParameterHandler.setParameters`方法会遍历`BoundSql.parameterMappings`结合中记录的`ParameterMapping`对象，并根据其中记录的参数名称查找相应实参，然后语言SQL语句绑定。setParameters代码：

```java
public void setParameters(PreparedStatement ps) {
  ……
  // 取出sql中的参数列表
  List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
  if (parameterMappings != null) {
    for (int i = 0; i < parameterMappings.size(); i++) {
      ParameterMapping parameterMapping = parameterMappings.get(i);
      // 过滤存储过程中的输出参数
      if (parameterMapping.getMode() != ParameterMode.OUT) {
        Object value;// 记录绑定的实参
        String propertyName = parameterMapping.getProperty();// 获取参数名称
        if (boundSql.hasAdditionalParameter(propertyName)) { // issue #448 ask first for additional params
          // 获取对应的实参值
          value = boundSql.getAdditionalParameter(propertyName);
        } else if (parameterObject == null) {// 实参为空
          value = null;
        } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
          // 实参可以直接通过TypeHandler转化成JbbcType
          value = parameterObject;
        } else {
          // 获取对象中相应的属性值或者查找Map对象中的值
          MetaObject metaObject = configuration.newMetaObject(parameterObject);
          value = metaObject.getValue(propertyName);
        }
        // 获取TypeHandler
        TypeHandler typeHandler = parameterMapping.getTypeHandler();
        JdbcType jdbcType = parameterMapping.getJdbcType();
        if (value == null && jdbcType == null) {
          jdbcType = configuration.getJdbcTypeForNull();
        }
        try {
          // 通过TypeHandler.setParameter方法调用PreparedStatement.set*()方法来为SQL语句绑定相应的实参
          typeHandler.setParameter(ps, i + 1, value, jdbcType);
        } catch (TypeException e) {
          throw new TypeException("Could not set parameters for mapping: " + parameterMapping + ". Cause: " + e, e);
        } catch (SQLException e) {
          throw new TypeException("Could not set parameters for mapping: " + parameterMapping + ". Cause: " + e, e);
        }
      }
    }
  }
}
```

为SQL语句绑定实参之后，就可以调用`Statement`相应的`excute()`方法，将SQL语句交给数据库执行。

# 5 - SimpleStatementHandler

`SimpleStatementHandler`继承了`BaseStatementHandler`抽象类。它底层使用`java.sql.Statement`对象完成数据库操作，所以SQL语句不能有占位符，相应的`SimpleStatementHandler.parameterize()`方法是空实现。

`SimpleStatementHandler.instantiateStatement()`方法直接通过jdbc connection创建statement对象，具体实现如下：

```java
protected Statement instantiateStatement(Connection connection) throws SQLException {
  if (mappedStatement.getResultSetType() != null) {
    return connection.createStatement(mappedStatement.getResultSetType().getValue(), ResultSet.CONCUR_READ_ONLY);
  } else {
    return connection.createStatement();
  }
}
```

`SimpleStatementHandler.update()`方法负责执行insert、update、delete等类型的SQL语句，并且会根据配置的KeyGenerator获取数据库生成的主键，具体实现如下：

```java
public int update(Statement statement) throws SQLException {
  String sql = boundSql.getSql();
  Object parameterObject = boundSql.getParameterObject();
  KeyGenerator keyGenerator = mappedStatement.getKeyGenerator();
  int rows;
  if (keyGenerator instanceof Jdbc3KeyGenerator) {
    statement.execute(sql, Statement.RETURN_GENERATED_KEYS);
    rows = statement.getUpdateCount();
    keyGenerator.processAfter(executor, mappedStatement, statement, parameterObject);
  } else if (keyGenerator instanceof SelectKeyGenerator) {
    statement.execute(sql);
    rows = statement.getUpdateCount();
    keyGenerator.processAfter(executor, mappedStatement, statement, parameterObject);
  } else {
    statement.execute(sql);
    rows = statement.getUpdateCount();
  }
  return rows;
}
```

# 6 - PreparedStatementHandler

```java
protected Statement instantiateStatement(Connection connection) throws SQLException {
  String sql = boundSql.getSql();
  if (mappedStatement.getKeyGenerator() instanceof Jdbc3KeyGenerator) {
    String[] keyColumnNames = mappedStatement.getKeyColumns();
    if (keyColumnNames == null) {
      // 返回数据库生成的主键
      return connection.prepareStatement(sql, PreparedStatement.RETURN_GENERATED_KEYS);
    } else {
      // 在insert语句完成之后，会将keyColumnNames指定的列返回
      return connection.prepareStatement(sql, keyColumnNames);
    }
  } else if (mappedStatement.getResultSetType() != null) {
    // 设置结果集是否可以滚动以及游标是否可以上下滚动，设置结果集是否可更新。
    return connection.prepareStatement(sql, mappedStatement.getResultSetType().getValue(), ResultSet.CONCUR_READ_ONLY);
  } else {
    // 创建普通的PreparedStament对象。
    return connection.prepareStatement(sql);
  }
}
```