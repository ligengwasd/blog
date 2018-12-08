# 1 - 包位置

```java
org.apache.ibatis.reflection
```

# 2 - Reflector & ReflectorFactory

Reflector中缓存了反射操作需要使用的类的元信息(字段、方法等)

## 2.1 属性

```java
private static final String[] EMPTY_STRING_ARRAY = new String[0];
// 对应的Class类型
private Class<?> type;
// getter方法对应的属性
private String[] readablePropertyNames = EMPTY_STRING_ARRAY;
// setter方法对应的属性
private String[] writeablePropertyNames = EMPTY_STRING_ARRAY;
// key是属性名称，value是对应的Invoker对象
private Map<String, Invoker> setMethods = new HashMap<String, Invoker>();
// 同上
private Map<String, Invoker> getMethods = new HashMap<String, Invoker>();
// 属性对应的setter方法的参数类型
private Map<String, Class<?>> setTypes = new HashMap<String, Class<?>>();
// 属性对应的getter方法的返回值类型
private Map<String, Class<?>> getTypes = new HashMap<String, Class<?>>();
// 默认的构造方法
private Constructor<?> defaultConstructor;
// 所有属性名称集合
private Map<String, String> caseInsensitivePropertyMap = new HashMap<String, String>();
```

## 2.2 构造器

对象初始化的时候根据Class对象解析获取到以上数据，过程较为繁琐，不一一解析。

```java
public Reflector(Class<?> clazz) {
  type = clazz;
  addDefaultConstructor(clazz);
  addGetMethods(clazz);
  addSetMethods(clazz);
  addFields(clazz);
  readablePropertyNames = getMethods.keySet().toArray(new String[getMethods.keySet().size()]);
  writeablePropertyNames = setMethods.keySet().toArray(new String[setMethods.keySet().size()]);
  for (String propName : readablePropertyNames) {
    caseInsensitivePropertyMap.put(propName.toUpperCase(Locale.ENGLISH), propName);
  }
  for (String propName : writeablePropertyNames) {
    caseInsensitivePropertyMap.put(propName.toUpperCase(Locale.ENGLISH), propName);
  }

```

## 2.3 Invoker类。

addSetMethods、addGetMethods、addGetFields、addSetFields方法在向上述集合添加元素时，会将相应的Method对象以及字段对应的Field对象统一封装成Invoker对象。

<img width="374" height="101" src="https://raw.githubusercontent.com/ligengwasd/blog/master/MyBatis%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB/images/3.14.56.png"/>

## 2.4 ReflectorFactory工厂类

生成并缓存Reflector对象。