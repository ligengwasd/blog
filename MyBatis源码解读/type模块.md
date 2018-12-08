​	



# 1 - 源码所在包

类型转换器源码所在包：
```
org.apache.ibatis.type
```

# 2 - 类型转换逻辑

<img width="300" height="200" src="https://raw.githubusercontent.com/ligengwasd/blog/master/MyBatis%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB/images/11.36.42.png"/>

# 3 - TypeHandler类

```java
public interface TypeHandler<T> {
  // 在通过PrepareStatement为SQL语句绑定参数时，会将数据由Java类型转化为JdbcType类型
  void setParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType) throws SQLException;
  // 从ResultSet中获取数据是会调用此方法，会将数据由JdbcType类型转化为Java类型
  T getResult(ResultSet rs, String columnName) throws SQLException;

  T getResult(ResultSet rs, int columnIndex) throws SQLException;

  T getResult(CallableStatement cs, int columnIndex) throws SQLException;

}
```

# 4 - TypeHandlerRegistry类

## 4.1 属性：

```java
// 记录了JdbcType类型和TypeHandler的对应关系，其中JdbcType是一个枚举类型，他定义对应的JDBC类型。
// 该集合主要用于从结果集读取数据时，将数据从Jdbc类型转换成Java类型。
private final Map<JdbcType, TypeHandler<?>> JDBC_TYPE_HANDLER_MAP = new EnumMap(JdbcType.class);
// 记录了java类型向指定的JdbcType类型转换时，需要使用的TypeHandler对象。
// 例如：java中的String可能转换成数据库中的char、varchar等多种类型，所以存在一对多的关系。
private final Map<Type, Map<JdbcType, TypeHandler<?>>> TYPE_HANDLER_MAP = new ConcurrentHashMap();
// 记录了全部的TypeHandler类型以及该类型对应的TypeHandler对象。键是TypeHandler的class对象。
private final Map<Class<?>, TypeHandler<?>> ALL_TYPE_HANDLERS_MAP = new HashMap();
// 空TypeHandler集合的标识
private static final Map<JdbcType, TypeHandler<?>> NULL_TYPE_HANDLER_MAP = new HashMap<JdbcType, TypeHandler<?>>();
```

## 4.2 注册TypeHandler

TypeHandlerRegistry.register()方法实现了注册TypeHandler对象的功能，即向上面四个结合中添加TypeHandler对象。TypeHandlerRegistry中共有12个register方法，其重载关系如下图。

<img width="660" height="310" src="https://raw.githubusercontent.com/ligengwasd/blog/master/MyBatis%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB/images/12.40.15.png"/>

主要是1-6这6个重载方法实现注册功能，其余方法主要完成强制类型转换或初始化TypeHandler功能。

大部分方法最终都会调用4号方法完成注册，下面是4号方法的源码

```java
// java type + jdbc type + handler

public <T> void register(Class<T> type, JdbcType jdbcType, TypeHandler<? extends T> handler) {
  register((Type) type, jdbcType, handler);
}

private void register(Type javaType, JdbcType jdbcType, TypeHandler<?> handler) {
  if (javaType != null) {
    Map<JdbcType, TypeHandler<?>> map = TYPE_HANDLER_MAP.get(javaType);
    if (map == null) {
      map = new HashMap<JdbcType, TypeHandler<?>>();
      TYPE_HANDLER_MAP.put(javaType, map);
    }
    map.put(jdbcType, handler);
  }
  ALL_TYPE_HANDLERS_MAP.put(handler.getClass(), handler);
}
```

## 4.3 查找TypeHandler

