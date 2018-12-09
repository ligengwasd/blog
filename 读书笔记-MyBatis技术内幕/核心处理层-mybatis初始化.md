# 1 - BaseBuilder类

请先阅读《例子》

Mybatis初始化入口是SqlSessionFactoryBuilder.build()方法

```java
public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
  try {
    // 读取配置文件
    XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
	// 解析配置文件得到Configuration对象，创建DefaultSqlSessionFactory对象
    return build(parser.parse());
  } catch (Exception e) {
    throw ExceptionFactory.wrapException("Error building SqlSession.", e);
  } finally {
    ErrorContext.instance().reset();
    try {
      inputStream.close();
    } catch (IOException e) {
      // Intentionally ignore. Prefer previous error.
    }
  }
}
```
SqlSessionFactoryBuilder.build()会创建XMLConfigBuilder对象来解析mybatis-config.xml配置文件，而XMLConfigBuilder继承自BaseBuilder抽象类，BaseBuilder子类如图

<img width="663" height="157" src="https://raw.githubusercontent.com/ligengwasd/blog/master/读书笔记-MyBatis技术内幕/images/11.29.15.png"/>

```java
// BaseBuilder只有三个属性
protected final Configuration configuration;
protected final TypeAliasRegistry typeAliasRegistry;
protected final TypeHandlerRegistry typeHandlerRegistry;
// 构造器
public BaseBuilder(Configuration configuration) {
    this.configuration = configuration;
    this.typeAliasRegistry = this.configuration.getTypeAliasRegistry();
    this.typeHandlerRegistry = this.configuration.getTypeHandlerRegistry();
  }
```

BaseBuilder.resolveTypeHandler()方法：依赖TypeHandlerRegistry类解析TypeHandler。

BaseBuilder.resolveAlias()方法：依赖TypeAliasRegistry类解析别名。

以及其他resolve方法。都是用于从xml文件解析相应的对象。

# 2 - XMLConfigBuilder类

## 2.1 属性：

```java
// mybatis-config.xml是否被解析过
private boolean parsed;
// xml解析器
private XPathParser parser;
// 标识<environment>配置的名称，默认读取<environment>标签的default属性
private String environment;
// ReflectorFactory负责创建Reflector对象。
private ReflectorFactory localReflectorFactory = new DefaultReflectorFactory();
```

## 2.2 解析器入口

XMLConfigBuilder.parse()方法是解析mybatis-config.xml文件的入口，通过调用XMLConfigBuilder.parseConfiguration方法实现整个解析过程：

```java
private void parseConfiguration(XNode root) {
  try {
    //issue #117 read properties first
    // 解析<properties>节点
    propertiesElement(root.evalNode("properties"));
    // 解析<settings>节点
    Properties settings = settingsAsProperties(root.evalNode("settings"));
	loadCustomVfs(settings);
    // 解析<typeAliases>节点
    typeAliasesElement(root.evalNode("typeAliases"));
	// 解析<plugins>节点
	pluginElement(root.evalNode("plugins"));
	// 解析<objectFactory>节点
	objectFactoryElement(root.evalNode("objectFactory"));
	// 解析<objectWrapperFactory>节点
	objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
	// 解析<reflectorFactory>节点
	reflectorFactoryElement(root.evalNode("reflectorFactory"));
    settingsElement(settings);
    // read it after objectFactory and objectWrapperFactory issue #631
    // 解析<environments>节点
    environmentsElement(root.evalNode("environments"));
	// 解析<databaseIdProvider>节点
	databaseIdProviderElement(root.evalNode("databaseIdProvider"));
	// 解析<typeHandlers>节点
	typeHandlerElement(root.evalNode("typeHandlers"));
    // 解析<mappers>节点
    mapperElement(root.evalNode("mappers"));
  } catch (Exception e) {
    throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
  }
}
```

## 2.3 解析typeAliases节点

```java
private void typeAliasesElement(XNode parent) {
  if (parent != null) {
    for (XNode child : parent.getChildren()) {// 获取全部子节点
      if ("package".equals(child.getName())) {// 处理<package>节点
        String typeAliasPackage = child.getStringAttribute("name");//获取包名
        // 扫描包中所有类，并解析类中的@Alias注解，完成别名注册。
        configuration.getTypeAliasRegistry().registerAliases(typeAliasPackage);
      } else {// 处理<TypeAlias>节点
        String alias = child.getStringAttribute("alias");
        String type = child.getStringAttribute("type");
        try {
          Class<?> clazz = Resources.classForName(type);
          if (alias == null) {
            typeAliasRegistry.registerAlias(clazz);// 扫描@Alias注解，完成注册
          } else {
            typeAliasRegistry.registerAlias(alias, clazz);// 注册别名
          }
        } catch (ClassNotFoundException e) {
          throw new BuilderException("Error registering typeAlias for '" + alias + "'. Cause: " + e, e);
        }
      }
    }
  }
}
```

## 2.4 解析mappers节点

mybatis初始化除了加载mybatis-config.xml配置文件，还会加载全部的映射配置文件。`<mappers>`节点会告诉Mybatis去那些位置查找映射文件以及使用了配置注解标识的接口。

```java
private void mapperElement(XNode parent) throws Exception {
  if (parent != null) {
    for (XNode child : parent.getChildren()) {// 获取<mappers>所有子节点
      if ("package".equals(child.getName())) {// <package>子节点
        String mapperPackage = child.getStringAttribute("name");
		// 扫描包，向MapperRegistry注册mapper接口
        configuration.addMappers(mapperPackage);
      } else {
        // resource、url、mapperClass只能有一个属性不为空
        String resource = child.getStringAttribute("resource");
        String url = child.getStringAttribute("url");
        String mapperClass = child.getStringAttribute("class");
        // 如果指定了resource或url，创建XMLMapperBuilder对象。
        // 并通过该对象解析指定的mapper配置文件
        if (resource != null && url == null && mapperClass == null) {
          ErrorContext.instance().resource(resource);
          // 创建XMLMapperBuilder对象，解析映射配置文件
          InputStream inputStream = Resources.getResourceAsStream(resource);
          XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
          mapperParser.parse();
        } else if (resource == null && url != null && mapperClass == null) {
          ErrorContext.instance().resource(url);
          // 创建XMLMapperBuilder对象，解析映射配置文件
          InputStream inputStream = Resources.getUrlAsStream(url);
          XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());
          mapperParser.parse();
        } else if (resource == null && url == null && mapperClass != null) {
          // 指定class属性，直接向MapperRegistry注册该接口。
          Class<?> mapperInterface = Resources.classForName(mapperClass);
          configuration.addMapper(mapperInterface);
        } else {
          throw new BuilderException("A mapper element may only specify a url, resource or class, but not more than one.");
        }
      }
    }
  }
}
```

# 3 - XMLMapperBuilder类

**功能：解析mapper文件**

## 3.1 解析器入口

```java
public void parse() {
  if (!configuration.isResourceLoaded(resource)) {
    configurationElement(parser.evalNode("/mapper"));// 处理<mapper>节点
    configuration.addLoadedResource(resource);// 解析过的映射文件，放入Configuration的loadedResources(是一个HashSet)属性中保存。
    bindMapperForNamespace();// 注册mapper接口
  }

  parsePendingResultMaps();// 处理configurationElement()方法中解析失败的<resultMap>节点
  parsePendingChacheRefs();// 处理configurationElement()方法中解析失败的<cache-ref>节点
  parsePendingStatements();// 处理configurationElement()方法中解析失败的SQL语句节点
}
private void configurationElement(XNode context) {
    try {
      String namespace = context.getStringAttribute("namespace");
      if (namespace == null || namespace.equals("")) {
        throw new BuilderException("Mapper's namespace cannot be empty");
      }
      // 记录命名空间
      builderAssistant.setCurrentNamespace(namespace);
      // 解析<cache-ref>节点
      cacheRefElement(context.evalNode("cache-ref"));
      // 解析<cache>节点
      cacheElement(context.evalNode("cache"));
      // 解析<parameterMap>节点
      parameterMapElement(context.evalNodes("/mapper/parameterMap"));
      // 解析<resultMap>节点
      resultMapElements(context.evalNodes("/mapper/resultMap"));
      // 解析<sql>节点
      sqlElement(context.evalNodes("/mapper/sql"));
      // 解析<select|insert|update|delete>节点
      buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing Mapper XML. Cause: " + e, e);
    }
  }
```

## 3.2 解析resultMap节点

&lt;resultMap>节点下除了&lt;discriminator>子节点的其它子节点，都会被解析成对应的ResultMapping对象。

ResultMapping类核心字段

```java
private Configuration configuration;
private String property;
private String column;
private Class<?> javaType;
private JdbcType jdbcType;
private TypeHandler<?> typeHandler;
// 对应resultMap属性
private String nestedResultMapId;
// 对应select属性，引用另一个<select>节点
private String nestedQueryId;
private Set<String> notNullColumns;
private String columnPrefix;
private List<ResultFlag> flags;
private List<ResultMapping> composites;
private String resultSet;
private String foreignColumn;
private boolean lazy;// 对应fetchType属性
```

待续。。

## 3.3 XMLStatementBuilder类

**功能：解析mapper文件中<select|insert|update|delete>节点**

Mybatis使用SqlSource接口表示映射文件或注解中定义的SQL语句，但它表示的是不能被数据库执行的原始SQL，里面包含动态SQL语句相关的节点或者占位符需要解析的元素。

```java
public interface SqlSource {
  // 根据原始SQL和参数，返回数据库可执行的SQL
  BoundSql getBoundSql(Object parameterObject);

}
```

Mybatis使用MappedStatement表示映射配置文件中定义的SQL节点，重要属性如下：

```java
// 节点id属性，包括命名空间
private String resource;
private SqlSource sqlSource;// 对应一条SQL
// SQL类型
private SqlCommandType sqlCommandType;
```

解析SQL的入口函数是 `XMLStatementBuilder.parseStatementNode()` 方法

# 4 - 绑定mapper接口

在`XMLMapperBuilder.bindMapperForNamespace()`方法中完成了配置文件与对应的mapper接口的绑定。

```java
private void bindMapperForNamespace() {
  // 获取命名空间
  String namespace = builderAssistant.getCurrentNamespace();
  if (namespace != null) {
    Class<?> boundType = null;
    try {
      // 解析命名空间对应的类型
      boundType = Resources.classForName(namespace);
    } catch (ClassNotFoundException e) {
      //ignore, bound type is not required
    }
    if (boundType != null) {
      if (!configuration.hasMapper(boundType)) {// 是否已加载了boundType接口
        // Spring may not know the real resource name so we set a flag
        // to prevent loading again this resource from the mapper interface
        // look at MapperAnnotationBuilder#loadXmlResource
        // 解析过的mapper文件进入集合记录下来
        configuration.addLoadedResource("namespace:" + namespace);
        // 注册mapper接口
        configuration.addMapper(boundType);
      }
    }
  }
}
```

