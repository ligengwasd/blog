#mongo shell的使用

database -> table 表 -> row 行

database -> collection 集合 -> document  文档

**MongoDB Shell 是MongoDB**自带的JavaScript **Shell**，随**MongoDB**一同发布，它**是**MonoDB客户端工具，可以在**Shell**中使用命令与**MongoDB**实例交互，对数据库的管理操作（CURD、集群配置、状态查看等）都可以通过**MongoDB Shell**来完成。 

###1、数据库管理

创建数据库的语法，如果数据库不存在，则创建数据库，否则切换到指定数据库。

```
use DATABASE_NAME
```
查看所有数据库
```
show dbs
```

可以看到，我们刚创建的数据库 runoob 并不在数据库的列表中， 要显示它，我们需要向 runoob 数据库插入一些数据。

```
db.COLLECTION_NAME.insert({"name":"enmonster"})
```

###2、分配权限

用root用户登录。我的root用户是'admin'，登录命令

```
use admin;//建议所有用户都建在admin库，所以登录前必须切换到admin库去验证。
db.auth("admin","admin")
```

项目代码一般需要的是读写权限。

```
db.createUser({
	user:"user",
	pwd:"pwd",
	roles: [ { role: "readWrite", db: "DATABASE_NAME" } ]
})
```

角色列表：

```
read：允许用户读取指定数据库
readWrite：允许用户读写指定数据库
dbAdmin：允许用户在指定数据库中执行管理函数，如索引创建、删除，查看统计或访问system.profile
userAdmin：允许用户向system.users集合写入，可以找指定数据库里创建、删除和管理用户
clusterAdmin：只在admin数据库中可用，赋予用户所有分片和复制集相关函数的管理权限。
readAnyDatabase：只在admin数据库中可用，赋予用户所有数据库的读权限
readWriteAnyDatabase：只在admin数据库中可用，赋予用户所有数据库的读写权限
userAdminAnyDatabase：只在admin数据库中可用，赋予用户所有数据库的userAdmin权限
dbAdminAnyDatabase：只在admin数据库中可用，赋予用户所有数据库的dbAdmin权限。
root：只在admin数据库中可用。超级账号，超级权限
```
查看当前库所有用户

```
use DATABASE_NAME;//切换到要查看的库。
show users;
```
###3、增删改查语法

####3.1 新增

```
// 语法：
db.collection.insert(
   <document or array of documents>,
   {
     writeConcern: <document>,//写关注。可中断的阻塞式操作
     ordered: <boolean>,//true 批量插入出错不执行后续；false 出错执行后续；(有唯一索引的时候批量插入容错)
   }
)
// insert
db.products.insert(
    { item: "envelopes", qty : 100, type: "Clasp" },
    { writeConcern: { w: "majority", wtimeout: 5000 } }
)
// insert one
db.inventory.insertOne(
   { item: "canvas", qty: 100, tags: ["cotton"], size: { h: 28, w: 35.5, uom: "cm" } }
)
// insert many
db.inventory.insertMany([
   { item: "journal", qty: 25, tags: ["blank", "red"], size: { h: 14, w: 21, uom: "cm" } },
   { item: "mat", qty: 85, tags: ["gray"], size: { h: 27.9, w: 35.5, uom: "cm" } },
   { item: "mousepad", qty: 25, tags: ["gel", "blue"], size: { h: 19, w: 22.85, uom: "cm" } }
])
```
####3.2修改
```
// 语法
db.collection.update(
   <query>,
   <update>,
   {
     upsert: <boolean>,// true:找不到创建一个文档；false：不创建.默认 false
     multi: <boolean>,// true：更新多个，false 只更新最先找到的文档。默认 false
     writeConcern: <document>,// 写关注
     collation: <document>,// version 3.4
     arrayFilters: [ <filterdocument1>, ... ]// version 3.6
   }
)
// 示例
db.inventory.insertMany( [
   { item: "canvas", qty: 100, size: { h: 28, w: 35.5, uom: "cm" }, status: "A" },
   { item: "journal", qty: 25, size: { h: 14, w: 21, uom: "cm" }, status: "A" },
   { item: "mat", qty: 85, size: { h: 27.9, w: 35.5, uom: "cm" }, status: "A" },
   { item: "mousepad", qty: 25, size: { h: 19, w: 22.85, uom: "cm" }, status: "P" },
   { item: "notebook", qty: 50, size: { h: 8.5, w: 11, uom: "in" }, status: "P" },
   { item: "paper", qty: 100, size: { h: 8.5, w: 11, uom: "in" }, status: "D" },
   { item: "planner", qty: 75, size: { h: 22.85, w: 30, uom: "cm" }, status: "D" },
   { item: "postcard", qty: 45, size: { h: 10, w: 15.25, uom: "cm" }, status: "A" },
   { item: "sketchbook", qty: 80, size: { h: 14, w: 21, uom: "cm" }, status: "A" },
   { item: "sketch pad", qty: 95, size: { h: 22.85, w: 30.5, uom: "cm" }, status: "A" }
] );
db.inventory.updateOne(
   { item: "paper" },
   {
     $set: { "size.uom": "cm", status: "P" },
     $currentDate: { lastModified: true }
   }
)
```
####3.3删除
```
// 语法
db.collection.remove(
   <query>,
   {
     justOne: <boolean>,// 是否删除多个。默认false， 切记
     writeConcern: <document>,
     collation: <document>
   }
)
```
####3.4查找
#####3.4.1 检索对象
```
// 插入示例数据
db.inventory.insertMany( [
   { item: "journal", qty: 25, size: { h: 14, w: 21, uom: "cm" }, status: "A" },
   { item: "notebook", qty: 50, size: { h: 8.5, w: 11, uom: "in" }, status: "A" },
   { item: "paper", qty: 100, size: { h: 8.5, w: 11, uom: "in" }, status: "D" },
   { item: "planner", qty: 75, size: { h: 22.85, w: 30, uom: "cm" }, status: "D" },
   { item: "postcard", qty: 45, size: { h: 10, w: 15.25, uom: "cm" }, status: "A" }
]);
```

1. 比对整个对象

   ```
   db.inventory.find(  { size: { w: 21, h: 14, uom: "cm" } }  )
   ```

2. "."连接符

   ```
   db.inventory.find( { "size.h": { $lt: 15 } } )
   ```

3. 默认"and"连接符

   ```
   db.inventory.find( { "size.h": { $lt: 15 }, "size.uom": "in", status: "D" } )
   ```

#####3.4.2 检索数组

```
db.inventory.insertMany([
   { item: "journal", qty: 25, tags: ["blank", "red"], dim_cm: [ 14, 21 ] },
   { item: "notebook", qty: 50, tags: ["red", "blank"], dim_cm: [ 14, 21 ] },
   { item: "paper", qty: 100, tags: ["red", "blank", "plain"], dim_cm: [ 14, 21 ] },
   { item: "planner", qty: 75, tags: ["blank", "red"], dim_cm: [ 22.85, 30 ] },
   { item: "postcard", qty: 45, tags: ["blue"], dim_cm: [ 10, 15.25 ] }
]);
```

1. 数组比对数组
   ```
   // 严格匹配
   db.inventory.find( { tags: ["red", "blank"] } // 元素必须完全相同，顺序必须一致才能匹配
   // 包含匹配
   db.inventory.find( { tags: { $all: ["red", "blank"] } } )// 包含这两项即可匹配
   ```

2. 至少有一个元素匹配

   ```
   db.inventory.find( { tags: "red" } )// 至少一个元素等于“red”
   db.inventory.find( { dim_cm: { $gt: 25 } } )
   db.inventory.find( { dim_cm: { $gt: 15, $lt: 20 } } )// 至少一个元素大于15并且至少一个元素小于20
   db.inventory.find( { dim_cm: { $elemMatch: { $gt: 22, $lt: 30 } } } )//至少一个元素同时大于22小于30
   ```

3. 指定数组下标检索

   ```
   db.inventory.find( { "dim_cm.1": { $gt: 25 } } )
   ```

4. 检索数组长度

   ```
   db.inventory.find( { "tags": { $size: 3 } } )
   ```

#####3.4.3 检索对象数组

```
db.inventory.insertMany( [
   { item: "journal", instock: [ { warehouse: "A", qty: 5 }, { warehouse: "C", qty: 15 } ] },
   { item: "notebook", instock: [ { warehouse: "C", qty: 5 } ] },
   { item: "paper", instock: [ { warehouse: "A", qty: 60 }, { warehouse: "B", qty: 15 } ] },
   { item: "planner", instock: [ { warehouse: "A", qty: 40 }, { warehouse: "B", qty: 5 } ] },
   { item: "postcard", instock: [ { warehouse: "B", qty: 15 }, { warehouse: "C", qty: 35 } ] }
]);
```

​	

#####3.4.3 返回指定属性

插入示例数据

```
db.inventory.insertMany( [
  { item: "journal", status: "A", size: { h: 14, w: 21, uom: "cm" }, instock: [ { warehouse: "A", qty: 5 } ] },
  { item: "notebook", status: "A",  size: { h: 8.5, w: 11, uom: "in" }, instock: [ { warehouse: "C", qty: 5 } ] },
  { item: "paper", status: "D", size: { h: 8.5, w: 11, uom: "in" }, instock: [ { warehouse: "A", qty: 60 } ] },
  { item: "planner", status: "D", size: { h: 22.85, w: 30, uom: "cm" }, instock: [ { warehouse: "A", qty: 40 } ] },
  { item: "postcard", status: "A", size: { h: 10, w: 15.25, uom: "cm" }, instock: [ { warehouse: "B", qty: 15 }, { warehouse: "C", qty: 35 } ] }
]);
```

e.g.：

```
db.inventory.find( { status: "A" }, { item: 1, status: 1 } )//返回指定字段，默认返回“_id”字段。
db.inventory.find( { status: "A" }, { status: 0, instock: 0 } )// 返回所有字段，指定不返回一些字段。
db.inventory.find( { status: "A" }, { item: 1, status: 1, "instock.qty": 1 } )// 指定内联数组里的对象里的字段
db.inventory.find( { status: "A" }, { item: 1, status: 1, instock: { $slice: -1 } } )//指定返回数组的最后一项。
```

#####3.4.4 null和属性是否存在

```
db.inventory.insertMany([
   { _id: 1, item: null },
   { _id: 2 }
])
```

```
db.inventory.find( { item: null } )// 匹配null
db.inventory.find( { item : { $type: 10 } } )// 通过类型检查匹配null
db.inventory.find( { item : { $exists: false } } )// 有咩有这个属性
```

#####3.4.5 迭代器

```
var myCursor = db.users.find( { type: 2 } );

while (myCursor.hasNext()) {
   print(tojson(myCursor.next()));
}

// foreach方法
var myCursor =  db.users.find( { type: 2 } );

myCursor.forEach(printjson);
```

数据迁移的时候比较有用。

###4、聚合管道Aggregation Pipeline

管道：在Unix和Linux中一般用于将当前命令的输出结果作为下一个命令的参数。

MongoDB的聚合管道将MongoDB文档在一个管道处理完毕后将结果传递给下一个管道处理。管道操作是可以重复的。

![Diagram of the annotated aggregation pipeline operation. The aggregation pipeline has two stages: ``$match`` and ``$group``. â Enlarged](https://docs.mongodb.com/manual/_images/aggregation-pipeline.bakedsvg.svg)

e.g. 实现mysql的group by查询。

```
// 查询所有正常订单收退款总笔数和总金额。
db.daily_match.aggregate([
    {$project:{matchRes:1,billDt:1,tradeType:1,oOffAmt:1,orderId:1,shopId:1}},
    {$match:{"matchRes":"CONSISTENCY","billDt": 20180401,"shopId": 165560}},
//     {$limit:100000},
//     {$count:"count"},
    {$group:{
        _id:"$tradeType", count:{$sum:1}, sumAmount:{$sum:"$oOffAmt"},
//         avgAmount:{$avg:"$oOffAmt"}, min:{$min:"$oOffAmt"}, max:{$max:"$oOffAmt"},
        orderIdArray:{$push:"$orderId"}
    }},
    {$addFields:{"countAdd":{$add:["$count",100]}}},
    {$unwind:"$orderIdArray"}
//     {$out:"agg_res"}
])
```
常用 Aggregation Pipeline Stages

| Stage                                                        | Description                                                  |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| [`$addFields`](https://docs.mongodb.com/manual/reference/operator/aggregation/addFields/#pipe._S_addFields) | 向输入文档新增字段                                           |
| [`$count`](https://docs.mongodb.com/manual/reference/operator/aggregation/count/#pipe._S_count) | 计算结果集记录行数                                           |
| [`$group`](https://docs.mongodb.com/manual/reference/operator/aggregation/group/#pipe._S_group) | 分组                                                         |
| [`$indexStats`](https://docs.mongodb.com/manual/reference/operator/aggregation/indexStats/#pipe._S_indexStats) | Returns statistics regarding the use of each index for the collection. |
| [`$limit`](https://docs.mongodb.com/manual/reference/operator/aggregation/limit/#pipe._S_limit) | 截取结果集，限制返回就记录函数                               |
| [`$listSessions`](https://docs.mongodb.com/manual/reference/operator/aggregation/listSessions/#pipe._S_listSessions) | Lists all sessions that have been active long enough to propagate to the `system.sessions`collection. |
| [`$lookup`](https://docs.mongodb.com/manual/reference/operator/aggregation/lookup/#pipe._S_lookup) | Performs a left outer join to another collection in the *same* database to filter in documents from the “joined” collection for processing. |
| [`$match`](https://docs.mongodb.com/manual/reference/operator/aggregation/match/#pipe._S_match) | Filters the document stream to allow only matching documents to pass unmodified into the next pipeline stage. [`$match`](https://docs.mongodb.com/manual/reference/operator/aggregation/match/#pipe._S_match) uses standard MongoDB queries. For each input document, outputs either one document (a match) or zero documents (no match). |
| [`$out`](https://docs.mongodb.com/manual/reference/operator/aggregation/out/#pipe._S_out) | Writes the resulting documents of the aggregation pipeline to a collection. To use the [`$out`](https://docs.mongodb.com/manual/reference/operator/aggregation/out/#pipe._S_out) stage, it must be the last stage in the pipeline. |
| [`$project`](https://docs.mongodb.com/manual/reference/operator/aggregation/project/#pipe._S_project) | 对输入文档进行处理，返回新的文档                             |
| [`$redact`](https://docs.mongodb.com/manual/reference/operator/aggregation/redact/#pipe._S_redact) | Reshapes each document in the stream by restricting the content for each document based on information stored in the documents themselves. Incorporates the functionality of [`$project`](https://docs.mongodb.com/manual/reference/operator/aggregation/project/#pipe._S_project) and [`$match`](https://docs.mongodb.com/manual/reference/operator/aggregation/match/#pipe._S_match). Can be used to implement field level redaction. For each input document, outputs either one or zero documents. |
| [`$replaceRoot`](https://docs.mongodb.com/manual/reference/operator/aggregation/replaceRoot/#pipe._S_replaceRoot) | Replaces a document with the specified embedded document. The operation replaces all existing fields in the input document, including the `_id` field. Specify a document embedded in the input document to promote the embedded document to the top level. |
| [`$sample`](https://docs.mongodb.com/manual/reference/operator/aggregation/sample/#pipe._S_sample) | Randomly selects the specified number of documents from its input. |
| [`$skip`](https://docs.mongodb.com/manual/reference/operator/aggregation/skip/#pipe._S_skip) | Skips the first *n* documents where *n* is the specified skip number and passes the remaining documents unmodified to the pipeline. For each input document, outputs either zero documents (for the first *n* documents) or one document (if after the first *n* documents). |
| [`$sort`](https://docs.mongodb.com/manual/reference/operator/aggregation/sort/#pipe._S_sort) | Reorders the document stream by a specified sort key. Only the order changes; the documents remain unmodified. For each input document, outputs one document. |
| [`$sortByCount`](https://docs.mongodb.com/manual/reference/operator/aggregation/sortByCount/#pipe._S_sortByCount) | Groups incoming documents based on the value of a specified expression, then computes the count of documents in each distinct group. |
| [`$unwind`](https://docs.mongodb.com/manual/reference/operator/aggregation/unwind/#pipe._S_unwind) | 展开数组。<br />输入： ```[{ "_id" : 1, "item" : "ABC1", sizes: [ "S", "M", "L"] }]```<br />执行： ```db.inventory.aggregate( [ { $unwind : "$sizes" } ] ) ```<br />输出：```[{ "_id" : 1, "item" : "ABC1", "sizes" : "S" } { "_id" : 1, "item" : "ABC1", "sizes" : "M" } { "_id" : 1, "item" : "ABC1", "sizes" : "L" }]``` |

###5、查询计划

db.channel_daily_biz_bill.find({_id:"sss"}).explain()
db.collection.explain().count()



###6、索引的使用

```
db.channel_daily_biz_bill.createIndex({'billDt':1, 'matchRes':1});
db.daily_match.getIndexes()
db.daily_match.dropIndex()
```

###7、mongo的数据类型

Each BSON type has both integer and string identifiers as listed in the following table:

| Type                    | Number | Alias                 | Notes               |
| ----------------------- | ------ | --------------------- | ------------------- |
| Double                  | 1      | “double”              |                     |
| String                  | 2      | “string”              |                     |
| Object                  | 3      | “object”              |                     |
| Array                   | 4      | “array”               |                     |
| Binary data             | 5      | “binData”             |                     |
| Undefined               | 6      | “undefined”           | Deprecated.         |
| ObjectId                | 7      | “objectId”            |                     |
| Boolean                 | 8      | “bool”                |                     |
| Date                    | 9      | “date”                |                     |
| Null                    | 10     | “null”                |                     |
| Regular Expression      | 11     | “regex”               |                     |
| DBPointer               | 12     | “dbPointer”           | Deprecated.         |
| JavaScript              | 13     | “javascript”          |                     |
| Symbol                  | 14     | “symbol”              | Deprecated.         |
| JavaScript (with scope) | 15     | “javascriptWithScope” |                     |
| 32-bit integer          | 16     | “int”                 |                     |
| Timestamp               | 17     | “timestamp”           |                     |
| 64-bit integer          | 18     | “long”                |                     |
| Decimal128              | 19     | “decimal”             | New in version 3.4. |
| Min key                 | -1     | “minKey”              |                     |
| Max key                 | 127    | “maxKey”              |                     |

###8、运维监控


```
// 查看示例运行状况命令。
mongostat --host=127.0.0.1 --port=27017 -uadmin -padmin --authenticationDatabase=admin
mongotop --host=127.0.0.1 --port=27017 -uadmin -padmin --authenticationDatabase=admin
```
```
db.currentOp()// 查看正在执行的线程
db.killOp(opId)// 中止慢查
```
###9、js脚本
```
db.daily_match.find({
    "billDt":{$lte:20180531,$gte:20180501},
    "matchRes":{$in:["CONSISTENCY","ZERO_AMOUNT"]},
    "shopId":134477,
    "tradeType":"PAY",
    "oOffAmt":9900,
    "$where":function () {
        var datenew = this.oPayFeedBackTime
        datenew.setHours(datenew.getHours() + 8)
        var year=datenew.getFullYear();     
        var month=datenew.getMonth()+1;
        var date= datenew.getDate(); 
        var s = year + (month<10 ? "0"+month : month) + (date<10 ? "0"+date : date);
    
        var datenew2 = this.pCreDt
        if (!datenew2) {
            print(datenew2)
            return false
        }
        datenew2.setHours(datenew2.getHours() + 8)
        var year2=datenew2.getFullYear();     
        var month2=datenew2.getMonth()+1;
        var date2= datenew2.getDate(); 
        var s2= year2 + (month2<10 ? "0"+month2 : month2) + (date2<10 ? "0"+date2 : date2);
        return s2 - s >= 1
    }
})
    
```
###附录

#### 1、query语句操作符

1. 比较操作符

   | Name                                                         | Description                                                  |
   | ------------------------------------------------------------ | ------------------------------------------------------------ |
   | [`$eq`](https://docs.mongodb.com/manual/reference/operator/query/eq/#op._S_eq) | Matches values that are equal to a specified value.          |
   | [`$gt`](https://docs.mongodb.com/manual/reference/operator/query/gt/#op._S_gt) | Matches values that are greater than a specified value.      |
   | [`$gte`](https://docs.mongodb.com/manual/reference/operator/query/gte/#op._S_gte) | Matches values that are greater than or equal to a specified value. |
   | [`$in`](https://docs.mongodb.com/manual/reference/operator/query/in/#op._S_in) | Matches any of the values specified in an array.             |
   | [`$lt`](https://docs.mongodb.com/manual/reference/operator/query/lt/#op._S_lt) | Matches values that are less than a specified value.         |
   | [`$lte`](https://docs.mongodb.com/manual/reference/operator/query/lte/#op._S_lte) | Matches values that are less than or equal to a specified value. |
   | [`$ne`](https://docs.mongodb.com/manual/reference/operator/query/ne/#op._S_ne) | Matches all values that are not equal to a specified value.  |
   | [`$nin`](https://docs.mongodb.com/manual/reference/operator/query/nin/#op._S_nin) | Matches none of the values specified in an array.            |

2. 逻辑操作符

   | Name                                                         | Description                                                  |
   | ------------------------------------------------------------ | ------------------------------------------------------------ |
   | [`$and`](https://docs.mongodb.com/manual/reference/operator/query/and/#op._S_and) | Joins query clauses with a logical `AND` returns all documents that match the conditions of both clauses. |
   | [`$not`](https://docs.mongodb.com/manual/reference/operator/query/not/#op._S_not) | Inverts the effect of a query expression and returns documents that do *not* match the query expression. |
   | [`$nor`](https://docs.mongodb.com/manual/reference/operator/query/nor/#op._S_nor) | Joins query clauses with a logical `NOR` returns all documents that fail to match both clauses. |
   | [`$or`](https://docs.mongodb.com/manual/reference/operator/query/or/#op._S_or) | Joins query clauses with a logical `OR` returns all documents that match the conditions of either clause. |

3. 字段操作符

   | Name                                                         | Description                                            |
   | ------------------------------------------------------------ | ------------------------------------------------------ |
   | [`$exists`](https://docs.mongodb.com/manual/reference/operator/query/exists/#op._S_exists) | Matches documents that have the specified field.       |
   | [`$type`](https://docs.mongodb.com/manual/reference/operator/query/type/#op._S_type) | Selects documents if a field is of the specified type. |

4. 

   | Name                                                         | Description                                                  |
   | ------------------------------------------------------------ | ------------------------------------------------------------ |
   | [`$regex`](https://docs.mongodb.com/manual/reference/operator/query/regex/#op._S_regex) | Selects documents where values match a specified regular expression. |
   | [`$text`](https://docs.mongodb.com/manual/reference/operator/query/text/#op._S_text) | Performs text search.                                        |
   | [`$where`](https://docs.mongodb.com/manual/reference/operator/query/where/#op._S_where) | Matches documents that satisfy a JavaScript expression.      |

5. 数组操作符

   | Name                                                         | Description                                                  |
   | ------------------------------------------------------------ | ------------------------------------------------------------ |
   | [`$all`](https://docs.mongodb.com/manual/reference/operator/query/all/#op._S_all) | Matches arrays that contain all elements specified in the query. |
   | [`$elemMatch`](https://docs.mongodb.com/manual/reference/operator/query/elemMatch/#op._S_elemMatch) | Selects documents if element in the array field matches all the specified [`$elemMatch`](https://docs.mongodb.com/manual/reference/operator/query/elemMatch/#op._S_elemMatch)conditions. |
   | [`$size`](https://docs.mongodb.com/manual/reference/operator/query/size/#op._S_size) | Selects documents if the array field is a specified size.    |

#### 2、update语句操作符

1. 普通字段操作符

| Name                                                         | Description                                                  |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| [`$currentDate`](https://docs.mongodb.com/manual/reference/operator/update/currentDate/#up._S_currentDate) | Sets the value of a field to current date, either as a Date or a Timestamp. |
| [`$inc`](https://docs.mongodb.com/manual/reference/operator/update/inc/#up._S_inc) | Increments the value of the field by the specified amount.   |
| [`$min`](https://docs.mongodb.com/manual/reference/operator/update/min/#up._S_min) | Only updates the field if the specified value is less than the existing field value. |
| [`$max`](https://docs.mongodb.com/manual/reference/operator/update/max/#up._S_max) | Only updates the field if the specified value is greater than the existing field value. |
| [`$mul`](https://docs.mongodb.com/manual/reference/operator/update/mul/#up._S_mul) | Multiplies the value of the field by the specified amount.   |
| [`$rename`](https://docs.mongodb.com/manual/reference/operator/update/rename/#up._S_rename) | Renames a field.                                             |
| [`$set`](https://docs.mongodb.com/manual/reference/operator/update/set/#up._S_set) | Sets the value of a field in a document.                     |
| [`$setOnInsert`](https://docs.mongodb.com/manual/reference/operator/update/setOnInsert/#up._S_setOnInsert) | Sets the value of a field if an update results in an insert of a document. Has no effect on update operations that modify existing documents. |
| [`$unset`](https://docs.mongodb.com/manual/reference/operator/update/unset/#up._S_unset) | Removes the specified field from a document.                 |

2. 数组操作符

   | Name                                                         | Description                                                  |
   | ------------------------------------------------------------ | ------------------------------------------------------------ |
   | [`$`](https://docs.mongodb.com/manual/reference/operator/update/positional/#up._S_) | Acts as a placeholder to update the first element that matches the query condition. |
   | `$[]`                                                        | Acts as a placeholder to update all elements in an array for the documents that match the query condition. |
   | `$[<identifier>]`                                            | Acts as a placeholder to update all elements that match the `arrayFilters` condition for the documents that match the query condition. |
   | [`$addToSet`](https://docs.mongodb.com/manual/reference/operator/update/addToSet/#up._S_addToSet) | Adds elements to an array only if they do not already exist in the set. |
   | [`$pop`](https://docs.mongodb.com/manual/reference/operator/update/pop/#up._S_pop) | Removes the first or last item of an array.                  |
   | [`$pull`](https://docs.mongodb.com/manual/reference/operator/update/pull/#up._S_pull) | Removes all array elements that match a specified query.     |
   | [`$push`](https://docs.mongodb.com/manual/reference/operator/update/push/#up._S_push) | Adds an item to an array.                                    |
   | [`$pullAll`](https://docs.mongodb.com/manual/reference/operator/update/pullAll/#up._S_pullAll) | Removes all matching values from an array.                   |