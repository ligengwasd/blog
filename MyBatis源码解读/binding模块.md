# 1 - 包路径

```
org.apache.ibatis.binding;
```

# 2 - 功能描述

将mybatis的mapper接口中的方法和sql语句绑定。

核心类：

- MapperRegistry：注册器
- MapperProxyFactory：注册器中存储的是MapperProxyFactory对象。
- MapperProxy：根据mapper接口生成的代理对象。由MapperProxyFactory生成。
- MapperMethod：对应mapper接口中的方法。

# 3 - MapperRegistry&MapperProxyFactory

## 3.1 属性

```java
// Configuration对象，Mybatis全局唯一配置对象。
private final Configuration config;
// 记录了mapper接口与对应的MapperProxyFactory之间的关系
private final Map<Class<?>, MapperProxyFactory<?>> knownMappers = new HashMap<Class<?>, MapperProxyFactory<?>>();
```

## 3.2 注册方法

mybatis初始化过程中会读取mapper xml文件和mapper接口中的注解信息，并调用addMapper方法填充knownMappers集合。

```java
public <T> void addMapper(Class<T> type) {
  if (type.isInterface()) {
    if (hasMapper(type)) {// 是否已加载过
      throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
    }
    boolean loadCompleted = false;
    try {
      knownMappers.put(type, new MapperProxyFactory<T>(type));
      //  MapperAnnotationBuilder是加载mapper方法的注解信息。
      MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
      parser.parse();
      loadCompleted = true;
    } finally {
      if (!loadCompleted) {
        knownMappers.remove(type);
      }
    }
  }
}
```

## 3.3 获取方法

当需要执行sql语句是会先调用MapperRegistry.getMapper方法获取实现了mapper接口的代理对象。

```java
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
  // 获取mapperProxyFactory
  final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
  if (mapperProxyFactory == null) {
    throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
  }
  try {
      // 获取代理对象
    return mapperProxyFactory.newInstance(sqlSession);
  } catch (Exception e) {
    throw new BindingException("Error getting mapper instance. Cause: " + e, e);
  }
}
```

## 3.4 创建代理对象

MapperProxyFactory负责创建代理对象。

```java
protected T newInstance(MapperProxy<T> mapperProxy) {
    // 调用jdk的动态代理类创建代理对象。
  return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
}

public T newInstance(SqlSession sqlSession) {
  final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
  return newInstance(mapperProxy);
}
```

# 4 - MapperProxy

MapperProxy实现了InvocationHandler接口。InvocationHandler接口是jdk动态代理的核心接口。

MapperProxy.invoke方法是代理对象执行的主要逻辑。

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
  try {
      // 目标方法继承自Object，直接调用目标方法
    if (Object.class.equals(method.getDeclaringClass())) {
      return method.invoke(this, args);
    } else if (isDefaultMethod(method)) {
        // 针对java7兼容的代码
      return invokeDefaultMethod(proxy, method, args);
    }
  } catch (Throwable t) {
    throw ExceptionUtil.unwrapThrowable(t);
  }
    // 缓存相关
  final MapperMethod mapperMethod = cachedMapperMethod(method);
    // 调用execute方法执行SQL语句
  return mapperMethod.execute(sqlSession, args);
}
```

# 5 - MapperMethod

```java
// 记录SQL语句名称和类型
private final SqlCommand command;
// 记录mapper接口中对应方法的相关信息
private final MethodSignature method;
```

## 5.1 SqlCommand内部类

属性

```java
// SQL语句名称
private final String name;
// SQL语句类型，SqlCommandType是枚举类，取值[UNKNOWN, INSERT, UPDATE, DELETE, SELECT, FLUSH]
private final SqlCommandType type;
```

初始化方法

```java
public SqlCommand(Configuration configuration, Class<?> mapperInterface, Method method) {
	// 名称 = 接口名 + 方法名
  String statementName = mapperInterface.getName() + "." + method.getName();
  MappedStatement ms = null;
  if (configuration.hasStatement(statementName)) {// 检测是否有该名称的SQL语句
	// MappedStatement对象中封装了SQL语句相关的信息，mybatis初始化时创建。
    ms = configuration.getMappedStatement(statementName);
  } else if (!mapperInterface.equals(method.getDeclaringClass())) { // issue #35
      // 知道方法可能是在父类中定义的
    String parentStatementName = method.getDeclaringClass().getName() + "." + method.getName();
    if (configuration.hasStatement(parentStatementName)) {
      ms = configuration.getMappedStatement(parentStatementName);
    }
  }
  if (ms == null) {
      // 处理@Flush注解
    if(method.getAnnotation(Flush.class) != null){
      name = null;
      type = SqlCommandType.FLUSH;
    } else {
      throw new BindingException("Invalid bound statement (not found): " + statementName);
    }
  } else {
      // 初始化name和type
    name = ms.getId();
    type = ms.getSqlCommandType();
    if (type == SqlCommandType.UNKNOWN) {
      throw new BindingException("Unknown execution method for: " + name);
    }
  }
}
```

## 5.2 ParamNameResolver

记录mapper的方法中的参数信息。

```java
// 该属性记录了参数位置和名称
SortedMap<Integer, String> names;
```

## 5.3 MethodSignature内部类

```java
// 是否返回多条记录
private final boolean returnsMany;
private final boolean returnsMap;
private final boolean returnsVoid;
private final boolean returnsCursor;
private final Class<?> returnType;
private final String mapKey;
private final Integer resultHandlerIndex;
private final Integer rowBoundsIndex;
// 记录方法参数信息。
// ParamNameResolver内置一个熟悉 SortedMap<Integer, String> names记录了参数位置和名称的对应关系
private final ParamNameResolver paramNameResolver;
```

