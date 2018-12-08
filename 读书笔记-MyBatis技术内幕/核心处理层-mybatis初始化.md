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

<img width="465" height="622" src="https://raw.githubusercontent.com/ligengwasd/blog/master/读书笔记-MyBatis技术内幕/images/11.29.15.png"/>





# 2 - 