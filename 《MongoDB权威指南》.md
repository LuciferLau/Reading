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
> 根本目的：减少查询次数，提高查询速度，可用 db.foo.find().explain() 查看查询过程做的事。  
