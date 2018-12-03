​	



# 1 - 源码所在包

类型转换器源码所在包：
```
org.apache.ibatis.type
```

# 2 - 类型转换逻辑

![]()

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