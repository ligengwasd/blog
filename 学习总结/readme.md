### 2018-11-30

看源码是一件费时费力还不讨好的事儿，但是看多了受益无穷。

面对海量源码从而看呢，总结出两点心得。

第一种，找入口，顺藤摸瓜比如MyBatis，从创建SessionFactory开始，获取mapper，到执行SQL。一层一层往下看。

```java
public class MyBatisTest {
    private SqlSessionFactory sqlSessionFactory;

    @Before
    public void prepare() throws IOException {
//        String resource = "mybatis-config.xml";
        InputStream inputStream = new FileInputStream("/Users/ligeng/Documents/source/study-test/spring-boot-demo/src/test/java/com/ydb/mybatistest/mybatis-config.xml");
//                Resources.getResourceAsStream(resource);
        this.sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
        inputStream.close();
    }

    @Test
    public void testMyBatis() {
        SqlSession session = sqlSessionFactory.openSession();
        try {
            SysDataMapper sysDataMapper = session.getMapper(SysDataMapper.class);
            SysData sysData = sysDataMapper.findById(169452);
            System.out.println(1);
        } finally {
            session.commit();
            session.close();
        }
    }
}
```

第二种，宏观的看。

![](https://raw.githubusercontent.com/ligengwasd/blog/master/%E5%AD%A6%E4%B9%A0%E6%80%BB%E7%BB%93/images/4.05.58.png)

包逐个看，弄清楚每个包的功能和责任。



