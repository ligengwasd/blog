# 1 - 源码路径

```
org.apache.ibatis.executor.keygen
```

# 2 - KeyGenerator

```java
public interface KeyGenerator {
  // 在执行insert之前，设置属性order="BEFORE"
  void processBefore(Executor executor, MappedStatement ms, Statement stmt, Object parameter);
  // 在执行insert之后，设置属性order="AFTER"
  void processAfter(Executor executor, MappedStatement ms, Statement stmt, Object parameter);

}
```

Mybatis提供了三个KeyGenerator接口的时候

<img width="819" height="195" src="https://raw.githubusercontent.com/ligengwasd/blog/master/读书笔记-MyBatis技术内幕/images/9.08.51.png"/>

# 3 - Jdbc3KeyGenerator

Jdbc3KeyGenerator用户取回数据库生成的自增id，它对应于mybatis-config.xml配置文件中的useGeneratedKeys全局配置，以及映射配置文件中的SQL节点(<insert&gt;节点)的useGeneratedKeys属性。

一般情况下对于单行插入操作，入参是一个JavaBean或者Map对象，则该对象对应一次插入操作的内容；对于多行插入，入参可以是对象或者map对象的数组或集合，集合中每一个元素都对应一次插入操作。

```java
public class Jdbc3KeyGenerator implements KeyGenerator {

  @Override
  public void processBefore(Executor executor, MappedStatement ms, Statement stmt, Object parameter) {
    // do nothing
  }

  @Override
  public void processAfter(Executor executor, MappedStatement ms, Statement stmt, Object parameter) {
    // 将用户传入的实参parameter封装成集合类型，然后传入processBatch()方法中处理
    processBatch(ms, stmt, getParameters(parameter));
  }

  public void processBatch(MappedStatement ms, Statement stmt, Collection<Object> parameters) {
    ResultSet rs = null;
    try {
      // 获取数据库自动生成的主键，如果没有生成主键，则返回结果集为空
      rs = stmt.getGeneratedKeys();
      final Configuration configuration = ms.getConfiguration();
      final TypeHandlerRegistry typeHandlerRegistry = configuration.getTypeHandlerRegistry();
      // 获得keyProperties属性指定的属性名称，它表示主键对应的属性名称
      final String[] keyProperties = ms.getKeyProperties();
      // 获取ResultSet的元数据信息
      final ResultSetMetaData rsmd = rs.getMetaData();
      TypeHandler<?>[] typeHandlers = null;
      // 检测数据库生成的主键的列数和keyProperties属性指定的列数是否匹配
      if (keyProperties != null && rsmd.getColumnCount() >= keyProperties.length) {
        for (Object parameter : parameters) {
          // there should be one row for each statement (also one for each parameter)
          if (!rs.next()) {// parameters中有多少个元素，就插入了多少行，对应生成多少个主键。
            break;
          }
          // 为用户传入的实参创建相应的MetaObject对象
          final MetaObject metaParam = configuration.newMetaObject(parameter);
          if (typeHandlers == null) {
            // 获取对应的TypeHandler对象
            typeHandlers = getTypeHandlers(typeHandlerRegistry, metaParam, keyProperties, rsmd);
          }
          // 将生成的主键设置到用户传入的参数的对应位置。
          populateKeys(rs, metaParam, keyProperties, typeHandlers);
        }
      }
    } catch (Exception e) {
      throw new ExecutorException("Error getting generated key or setting result to parameter object. Cause: " + e, e);
    } finally {
      if (rs != null) {
        try {
          rs.close();
        } catch (Exception e) {
          // ignore
        }
      }
    }
  }

  private Collection<Object> getParameters(Object parameter) {
    Collection<Object> parameters = null;
    if (parameter instanceof Collection) {
	  // 参数为Collection对象
      parameters = (Collection) parameter;
    } else if (parameter instanceof Map) {
      // 参数为map类型，则获取其中指定的key
      Map parameterMap = (Map) parameter;
      if (parameterMap.containsKey("collection")) {
        parameters = (Collection) parameterMap.get("collection");
      } else if (parameterMap.containsKey("list")) {
        parameters = (List) parameterMap.get("list");
      } else if (parameterMap.containsKey("array")) {
        parameters = Arrays.asList((Object[]) parameterMap.get("array"));
      }
    }
    if (parameters == null) {
      // 参数为普通对象或不包括上述key的map集合，则创建ArrayList
      parameters = new ArrayList<Object>();
      parameters.add(parameter);
    }
    return parameters;
  }

  private TypeHandler<?>[] getTypeHandlers(TypeHandlerRegistry typeHandlerRegistry, MetaObject metaParam, String[] keyProperties, ResultSetMetaData rsmd) throws SQLException {
    TypeHandler<?>[] typeHandlers = new TypeHandler<?>[keyProperties.length];
    for (int i = 0; i < keyProperties.length; i++) {
      if (metaParam.hasSetter(keyProperties[i])) {
        Class<?> keyPropertyType = metaParam.getSetterType(keyProperties[i]);
        TypeHandler<?> th = typeHandlerRegistry.getTypeHandler(keyPropertyType, JdbcType.forCode(rsmd.getColumnType(i + 1)));
        typeHandlers[i] = th;
      }
    }
    return typeHandlers;
  }

  private void populateKeys(ResultSet rs, MetaObject metaParam, String[] keyProperties, TypeHandler<?>[] typeHandlers) throws SQLException {
    for (int i = 0; i < keyProperties.length; i++) {
      String property = keyProperties[i];
      if (!metaParam.hasSetter(property)) {
        throw new ExecutorException("No setter found for the keyProperty '" + property + "' in " + metaParam.getOriginalObject().getClass().getName() + ".");
      }
      TypeHandler<?> th = typeHandlers[i];
      if (th != null) {
        Object value = th.getResult(rs, i + 1);
        metaParam.setValue(property, value);
      }
    }
  }

}

```



举个例子：

```xml
<insert id="test" useGeneratedKeys="true" keyProperty="id">
    insert into t_user(username, pwd) values
    <foreach item="item" colloction="list" separator=",">
        (#{item.username}, #{item.pwd})
    </foreach>
</insert>
```

下图描述了示例的执行流程：

<img width="819" height="195" src="https://raw.githubusercontent.com/ligengwasd/blog/master/读书笔记-MyBatis技术内幕/images/9.28.24.png"/>





