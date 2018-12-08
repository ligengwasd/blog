# 1 - BaseBuilder类

请先阅读《例子》

Mybatis初始化入口是SqlSessionFactoryBuilder.build()方法

```java
public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
  try {
    // 读取配置文件
    XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
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

