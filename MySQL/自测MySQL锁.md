# 1.准备数据

**测试基于MySQL8**

```java
CREATE TABLE `task_queue` (
  `Id` int(11) NOT NULL AUTO_INCREMENT,
  `taskId` int(11) DEFAULT NULL,
  PRIMARY KEY (`Id`),
  KEY `taskId` (`taskId`)
) ENGINE=InnoDB AUTO_INCREMENT=51 DEFAULT CHARSET=utf8;
INSERT INTO `task_queue` VALUES ('1', '2'), ('3', '21'), ('40', '41'), ('50', '50');
```

id     taskId

1	2
3	21
40	41
50	50

# 2.主键

## 2.1找不到那条记录

执行

```sql
BEGIN;
SELECT * FROM task_queue WHERE id = 30 FOR UPDATE;
```

查看data_locks

```
INNODB	140551101959344:1064:140550992913048	3249	5093	18	lock-test	task_queue				140550992913048	TABLE	IX	GRANTED	
INNODB	140551101959344:3:4:5:140550992910104	3249	5093	18	lock-test	task_queue			PRIMARY	140550992910104	RECORD	X,GAP	GRANTED	40
```

## 2.1找到那条记录

执行

```sql
BEGIN;
SELECT * FROM task_queue WHERE id = 40 FOR UPDATE;
```

查看data_locks

```
INNODB	140551101959344:1064:140550992913048	3273	5093	42	lock-test	task_queue				140550992913048	TABLE	IX	GRANTED	
INNODB	140551101959344:3:4:5:140550992910104	3273	5093	42	lock-test	task_queue			PRIMARY	140550992910104	RECORD	X,REC_NOT_GAP	GRANTED	40
```



# 3.普通索引

## 3.1找不到那条记录

执行

```sql
BEGIN;
SELECT * FROM task_queue WHERE taskId = 30 FOR UPDATE;
```

查看data_locks

```
INNODB	140551101959344:1064:140550992913048	3269	5093	23	lock-test	task_queue				140550992913048	TABLE	IX	GRANTED	
INNODB	140551101959344:3:5:4:140550992910104	3269	5093	23	lock-test	task_queue			taskId	140550992910104	RECORD	X,GAP	GRANTED	41, 40
```

获取两把锁：

X,GAP：next key锁

## 3.2找到那条记录

执行

```sql
BEGIN;
SELECT * FROM task_queue WHERE taskId = 41 FOR UPDATE;
```

查看data_locks

```
INNODB	140551101959344:1064:140550992913048	3271	5093	33	lock-test	task_queue				140550992913048	TABLE	IX	GRANTED	
INNODB	140551101959344:3:5:4:140550992910104	3271	5093	33	lock-test	task_queue			taskId	140550992910104	RECORD	X	GRANTED	41, 40
INNODB	140551101959344:3:4:5:140550992910448	3271	5093	33	lock-test	task_queue			PRIMARY	140550992910448	RECORD	X,REC_NOT_GAP	GRANTED	40
INNODB	140551101959344:3:5:5:140550992910792	3271	5093	33	lock-test	task_queue			taskId	140550992910792	RECORD	X,GAP	GRANTED	50, 50

```

获取三把锁：

IX ：意向表锁

X ：next key锁，锁住记录本身和记录之前的间隙。  

X,REC_NOT_GAP ：行锁

X,GAP ： GAP锁

# 4.唯一索引

先修改taskId为唯一索引

## 4.1找不到那条记录

执行

```SQL
BEGIN;
SELECT * FROM task_queue WHERE taskId = 30 FOR UPDATE;
```

查看data_locks

```
INNODB	140551101958432:1064:140550992906904	3322	5179	14	lock-test	task_queue				140550992906904	TABLE	IX	GRANTED	
INNODB	140551101958432:3:5:4:140550992903864	3322	5179	14	lock-test	task_queue			taskl	140550992903864	RECORD	X,GAP	GRANTED	41
```

## 4.2找到那条记录

执行

```sql
BEGIN;
SELECT * FROM task_queue WHERE taskId = 41 FOR UPDATE;
```

查看data_locks

```
INNODB	140551101958432:1064:140550992906904	3323	5179	19	lock-test	task_queue				140550992906904	TABLE	IX	GRANTED	
INNODB	140551101958432:3:5:4:140550992903864	3323	5179	19	lock-test	task_queue			taskl	140550992903864	RECORD	X,REC_NOT_GAP	GRANTED	41
INNODB	140551101958432:3:4:5:140550992904208	3323	5179	19	lock-test	task_queue			PRIMARY	140550992904208	RECORD	X,REC_NOT_GAP	GRANTED	40
```

# 总结

| 索引     | 记录是否存在 | 加锁情况                                                     |
| -------- | ------------ | ------------------------------------------------------------ |
| 主键     | 否           | IX<br />X,GAP                                                |
|          | 是           | IX<br />X,REC_NOT_GAP                                        |
| 普通索引 | 否           | IX<br />X,GAP                                                |
|          | 是           | IX<br />X：next key锁，锁住记录本身和记录之前的间隙。<br />X,REC_NOT_GAP ：行锁<br />X,GAP ： GAP锁 |
| 唯一索引 | 否           | IX<br />X,GAP                                                |
|          | 是           | IX<br />X,REC_NOT_GAP：唯一索引<br />X,REC_NOT_GAP：主键     |



