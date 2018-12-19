# 1 - 包路径

```
org.apache.ibatis.executor.resultset
org.apache.ibatis.executor.result
```

# 2 - ResultSetHandler

在StatementHandler接口执行完select语句之后，会将查询到的结果集交给ResultSetHandler完成映射处理。ResultSetHandler除了负责映射select查询到的结果集，还会处理存储过程执行后的参数。

```java
public interface ResultSetHandler {
  // 处理结果集，生成相应的结果对象集合
  <E> List<E> handleResultSets(Statement stmt) throws SQLException;
  // 处理结果集，返回相应的游标对象
  <E> Cursor<E> handleCursorResultSets(Statement stmt) throws SQLException;
  // 处理存储过程的输出参数
  void handleOutputParameters(CallableStatement cs) throws SQLException;

}
```

DefaultResultSetHandler是ResultSetHandler的唯一实现：

## 2.1 核心字段：

```java
private final Executor executor;
private final Configuration configuration;
private final MappedStatement mappedStatement;
private final RowBounds rowBounds;

private final ParameterHandler parameterHandler;
private final ResultHandler<?> resultHandler;
private final BoundSql boundSql;
private final TypeHandlerRegistry typeHandlerRegistry;
private final ObjectFactory objectFactory;
private final ReflectorFactory reflectorFactory;

// nested resultmaps
private final Map<CacheKey, Object> nestedResultObjects = new HashMap<CacheKey, Object>();
private final Map<String, Object> ancestorObjects = new HashMap<String, Object>();
private Object previousRowValue;

// multiple resultsets
private final Map<String, ResultMapping> nextResultMaps = new HashMap<String, ResultMapping>();
private final Map<CacheKey, List<PendingRelation>> pendingRelations = new HashMap<CacheKey, List<PendingRelation>>();

// Cached Automappings
private final Map<String, List<UnMappedColumnAutoMapping>> autoMappingsCache = new HashMap<String, List<UnMappedColumnAutoMapping>>();

// temporary marking flag that indicate using constructor mapping (use field to reduce memory usage)
private boolean useConstructorMappings;

private final PrimitiveTypes primitiveTypes;
```

待续……