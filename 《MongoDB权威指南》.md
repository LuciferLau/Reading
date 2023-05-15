# Chapter1：Intro
- 易于使用  
使用 document 替代了 row，是 document-oriented 的数据库，没有固定的模式，k-v 类型和大小也不定。
- 易于拓展  
垂直/水平？纵向/横向？Scale up/Scale out √ MongoDB 使用横向扩展，面向文档的模型使其数据分割变得容易。  
- 功能丰富  
支持索引、聚合、特殊集合如会话，日志；文件存储。但不支持 连接(join) 和 多行事务。  
- 性能卓越  
document 可以动态填充，也可预分配空间换性能，对索引进行 cache，提高查询效率。  

# Chapter2：基础
- 文档 document  
形如 { "Foo" : "hello" } 的结构，每个文档都有个特殊的键 "\_id" ，键不能含有 '\0' 空字符或 '.' '$' 这些字符都有特殊含义，K-V 都区分大小写。
- 集合 collection  
集合类似表，用于存储collection。  
- 动态模式  
注意数据设计，提高查询效率。  
- 命名  
集合名不能是 "" 空字符串，也不能是 '\0' 空字符，不能以 system. 开头，不能有 '$' 字符(系统创建的集合会使用该字符)。  
- 数据库 db  
命名限制：不能为 "" 空字符串，不得含有 '/' '\' '.' '"' '\*' '<' '>' ':' '|' '?' '$' '\0' ' ' 这些字符，只能使用ASCII中的字母和数字；  
区分大小写，最多为64字节。  
- 启动DB
运行 mongod 命令，默认监听 27017 端口，还会起一个比主端口号大 1000 的端口，这里为 28017 端口作为HTTP服务器，可获取DB的管理信息。  
- MongoDB Shell
运行 mongo 命令，就可以进入 shell，支持 JS 的标准库/函数，在这里可以进行 DB 的增删查改。  
- 数据类型  
  + null：用于表示空值
  + 布尔值：true / false
  + 数值：NumberInt(4 byte) / NumberLong(8 byte)
  + 字符串：UTF-8字符串
  + 日期：Date，参照JS日期类
  + 正则表达式：查询时可用于作为条件，JS正则格式
  + 数组：数据列表
  + 内嵌文档：套娃
  + 对象ID：唯一标识，12字节组成
  + 二进制数据：非UTF-8字符存储方式
  + 代码：JS代码  
 
Object ID 是 "\_id" 的默认类型：timestamp | name | pid | counter
0-3 4字节用文档创建时间戳转化，  
4-6 3字节由机器名转化，  
7-8 2字节由当前进程PID转化，  
9-11 3字节由自增的计数器转化；  
前面 4+3+2 9字节保证了同一秒内不同机器不同进程的ID唯一，后面3字节则允许该进程同一秒内有 256<sup>3</sup> 个不同的ID。  

# Chapter3：CRUD
OP | SAMPLE
-- | :--
插入文档 | db.foo.insert({"abc":123})
批量插入文档 | db.foo.batchInsert([{"abc":123}, {"cba":321}])
删除集合中文档 | db.foo.remove()
删除集合 | db.foo.drop()
更新文档 | db.foo.update()，可用 $set/$unset 修改内嵌文档，用 $inc 增加/减少数值，用 $push 插入值，用 $each 和 $slice 控制操作数量，用 $sort 排序数据，用 $addToSet 插入唯一值，用 $pop 删除数组对应位置值，用 $pull 删除数组匹配值 ...
插入更新文档 | db.foo.upsert()  

# Chapter4：FIND
> 操作符的使用是查询的关键，基础命令都是 db.foo.find()  

操作符 | 作用
-- | :--
$lt $lte | < <=
$gt $gte | > >=
$in $nin | 匹配一个 key 的多个 val
$or $nor | 匹配多个 key 的多个 val
$not | 结果取反
$and | 结果并集
$exists | 判断null值
$all | 通过元素匹配数组
$size | 根据数组长度查询
$elemMatch | 多条件查询，如可以同时使用lt+gt
$where | 不推荐使用，限制少 安全差 效率低，不操作BSON，只操作JS对象

游标：find的返回值，可用于遍历集合，hasNext() 和 next() 配合 foreach 使用  
limit(n)：限制返回查询结果数为n个  
skip(n)：忽略前n个匹配结果  
sort({"key":1/-1})：按key升/降序排序  
数据库命令：runCommand()，可用 db.listCommand() 查看  

# Chapter5：索引
> 根本目的：减少查询次数，提高查询速度，可用 db.foo.find().explain() 查看查询过程做的事；  
> 每个集合限制 64 个索引，通常在一个特定集合上不应该有 2 个或以上的索引；  
> 对 1 个以上的key同时建立索引，那就是复合索引。  、

- explain 各项指标说明  
tag | desc
-- | :--
cursor | 可以是 BtreeCursor / BasicCursor 前者说明适用了索引
isMultiKey | 是否使用多键索引
n | 返回 document 数量
nscannedObjects | 查询磁盘上实际 document 数量
nscanned | 查询 document 数量，如果使用了索引就是检查过索引的数量
scanAndOrder | 是否在内存中对结果集进行排序
indexOnly | 是否只使用了索引完成查询
nYields | 挂起次数，有写入请求时会暂停让写请求先执行
millis | 本次查询消耗毫秒时
indexBounds | 索引的遍历范围

- 何时不应该使用索引
原则：结果集占数据集的比例越大，索引的速度就越慢，最坏情况下会是全表扫描的 2 倍查询次数（1次索引 + 1次查文档）  

# Chapter6：特殊的索引和集合
（略）

# Chapter7：聚合
> MongoDB提供了聚合框架，MapReduce，聚合命令如count、distinct、group这些聚合工具  
- 聚合框架  
  + 算数表达式  
  $add：加  
  $subtract：减  
  $multiply：乘  
  $divide：除  
  $mod：模  
  + 字符串表达式  
  $substr：子串
  $concat：连接
  $toLower：小写
  $toUpper：大写
  + 逻辑表达式  
  $cmp：真>0，等=0，假<0
  $strcasecmp：比较字符串，区分大小写
  $eq：$ne：$gt：$gte：$lt：$lte：$and：$or：$not：略，前面提及  
  $cond：cond？true：false；类似Cpp的条件表达式
  $ifNull：expr？expr2；如果是null，返回expr2，否则返回expr
```
以书中例子介绍聚合框架及其操作符：  
db.aggregate({$project:{"author":1}},
  {$group:{"_id":"$author", "count":{"$sum":1}}},
  {$sort:{"count":-1}},
  {$limit:5})  
  
$project：从 document 中提取字段，甚至可以重命名字段，并且不会影响原数据，这里把author提取出来  
$match：用于筛选条件，可以使用所有常规的查询操作符  
$group：根据 document 中的字段分组，并且可以对其进行表达式运算：$sum求和，$avg求平均，$max/min取最值，$first/last取分值首尾值，$addToSet/push为数组尝试添加值  
$unwind：将数组中的每个值拆分为单独的 document  
$sort：对单个或多个字段进行排序，这里对count进行排序  
$limit：限制结果集的数量，这里限制前5个结果  
$skip：丢弃结果集的n个结果  
```

- MapReduce  
（略），简单来说是将大任务可以通过 map 映射，洗牌分组，然后 reduce 化简，重复洗牌直至剩下一个值。  
用于大问题拆分到多个小问题由不同机器处理，利用了分布式的思想，但游戏服一般不需要这种重量级操作。  

- 聚合命令  
count()：返回 document 数量
distinct：获取给定 key 的不同 val  

# Chapter8：应用程序设计
