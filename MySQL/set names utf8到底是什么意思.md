# set names utf8到底是什么意思？

先说MySQL的字符集问题。Windows下可通过修改my.ini内的

```
1. #CLIENT SECTION
2. [mysql]
3. default-character-set=utf8
4. # SERVER SECTION
5. [mysqld]
6. default-character-set=utf8
```

这两个字段来更改数据库的默认字符集。第一个是客户端默认的字符集，第二个是服务器端默认的字符集。假设我们把两个都设为utf8，然后在MySQL Command Line Client里面输入 `show variebles like "character_set_%"`; ，可看到如下字符：

```
1. character_set_client latin1
2. character_set_connection latin1
3. character_set_database utf8
4. character_set_results latin1
5. character_set_server utf8
6. character_set_system utf8
```

其中的utf8随着我们上面的设置而改动。此时，要是我们通过采用UTF-8的PHP程序从数据库里读取数据，很有可能是一串”?????” 或者是其他乱码。网上查了半天，解决办法倒是简单，在连接数据库之后，读取数据之前，先执行一项查询  SET NAMES UTF8 ，即在php里为 

 `mysql_query("SET NAMES UTF8");` 

即可显示正常（只要数据库里信息的字符正常）。为什么会这样？这句查询“SET NAMES UTF8”到底是什么作用？

到mysql命令行输入 `SET NAMES UTF8`; ，然后执行 show variebles like "character_set_%"; ，发现原来为 latin1 的那些变量 character_set_client 、 character_set_connection 、 character_set_results 的值全部变为 utf8 了，原来是这3个变量在捣蛋。查阅手册，上面那句等于：

```
1. SET character_set_client = utf8;
2. SET character_set_results = utf8;
3. SET character_set_connection = utf8;
```

看看这3个变量的作用：

信息输入路径：client→connection→server；

信息输出路径：server→connection→results。

换句话说，每个路径要经过3次改变字符集编码。以出现乱码的输出为例，server里utf8的数据，传入connection转为latin1，传入results转为latin1，utf-8页面又把results转过来。如果两种字符集不兼容，比如latin1和utf8，转化过程就为不可逆的，破坏性的。所以就转不回来了。

但这里要声明一点， SET NAMES UTF8 作用只是临时的，MySQL重启后就恢复默认了。

接下来就说到MySQL在服务器上的配置问题了。岂不是我们每次对数据库读写都得加上 SET NAMES UTF8 ，以保证数据传输的编码一致？能不能通过配置MySQL来达到那三个变量默认就为我们要想的字符集？手册上没说，我在网上也没找到答案。所以，从服务器配置的角度而言，是没办法省略掉那行代码的。

总结：为了让你的网页能在更多的服务器上正常地显示，还是加上 SET NAMES UTF8 吧，即使你现在没有加上这句也能正常访问。