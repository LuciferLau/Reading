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
（略）

# Chapter9：创建副本集 replica set
> 为了提高数据库的可用性，将数据副本复制到多台MongoDB服务器上。由一个 primary 和多个 secondary 组成，
> 如果 primary 宕机了，会选一个 secondary 来成为新的 primary
（创建副本集接口的介绍/使用略过）

# Chapter10：副本集的组成⭐
- 同步
复制功能由 oplog 操作日志实现，日志中记录了 primary 中每一次写操作，是主节点local数据库中的一个固定集合（参考Chap6）；  
每个 secondary 中也维护着自己的 oplog，记录每次从 primary 中复制数据的操作，每个成员都可以作为同步源提供给其他节点读取；  
初始化同步：选择同步源，克隆数据到本地，并记录操作到oplog（如果重新克隆就重新记录oplog），建立索引，切换节点状态  
心跳：每 2s 向其它成员发送心跳请求确认成员状态

status | desc
-- | :--
STARTUP | 节点刚启动时的状态
STARTUP2 | 初始化同步时的状态，会创建用于复制/选举的线程
RECOVERING | 节点可能繁忙，可能是启动时检测，也可能与其它成员脱节，暂不能处理读请求
ARBITER | 仲裁者的状态（选举）
DOWN | 不可达，进程死亡/网络中断（他人视角）
UNKNOWN | 成员无法到达其它成员时（自己视角）
REMOVED | 成员被移除副本集时的状态
ROLLBACK | 数据回滚时的状态
FATAL | 发生了不可挽回的错误时，需要查日志定位问题

- 选举  
当一个成员无法到达主节点时，会申请选举为主节点，如果它获得大多数票，就会成为新的主节点  

- 回滚  
脑裂情况下，旧的主节点如果恢复的网络通信/重启，会比较新主节点的oplog，回滚到最后一个匹配的操作，然后才能进行数据同步  
PS：数据量 >300 MB 或 操作时间 >30 min 时，回滚操作会失败（节点落后太多的情况下）。  

# Chapter11：从应用程序连接副本集
> 核心：对一致性要求非常高的应用程序不应该从备份节点读取数据  

# Chapter12：管理(副本集)
（略，介绍了副本集的配置方法，注意避免复制循环）

# Chapter13：分片 cluster
> 分片指将数据拆分，将其分散存放在不同机器上的过程（scale out）  
mongos：维护着内容列表，指明了每个分片有什么数据内容，对客户端来说，连接到mongos就想连接到单机mongod一样无感  
sh命令：类似rs命令(副本集的命令)，有许多辅助函数用于管理分片  

# Chapter14：配置分片🌟
> 使用分片是为了：增加单机可用RAM/ROM；减轻单机负载；提高QPS  
- configServer  
实际上是一个普通的mongod服务器，用 --configsvr 选项启动，要先于其他进程启动（因为mongos从这里拉取配置）  
- mongos  
使用 --configdb 选项启动，后面接若干 configServer 的端口（通常每个应用程序服务器使用一个mongos）  
- 数据分片
 “块” 是MongoDB迁移数据的基本单元，文档被分组为块，每个块只存在于一个分片上，块之间不存在交集  
新分片集合起初只有一个块，范围是 $minKey ~ $maxKey 负正无穷，随着文档增多，块也会自动增加；  
> 在新的写请求到来时，mongos会维护块的计数器，在达到阈值后，会通知configSvr调整块大小（--nosplit 选项可以关闭此功能）   
> 这样会引出2个问题，  
> 1：mongos节点如果处于不稳定状态，重启后，计数器会重置导致，导致块不断膨胀无法分割  
> 2：mongod configSvr如果处于不可达（网络等问题），mongos无法更新元数据，不断收到写请求时，会循环尝试向configSvr发送拆分请求  
- 均衡器  
负责数据的迁移，周期性检查分片间是否存在不平衡，如果存在就进行块的迁移；  
虽然均衡器被看做单一实体，但有时mongos也会扮演均衡器的角色。  
> config.lock 集合中 \_id 为 “balancer” 的就是均衡器，state 0【非活跃】1【尝试加锁中】2【均衡中】  

# Chapter15：选择片键
> 片键：指用于拆分 collection 时使用的 document 里的 key
> 因为分片完成后修改它几乎是不可能的，正确地选择片键，才能体现出分片的优势，更好的实现数据分发  
- 散列片键：数据加载极快，但对范围查询不友好（因为做了hash）  
- 流水策略：用配置较好的机器处理数据插入，然后通过均衡器将数据转移到配置相对差的服务器  
- 多热点：通过设计复合片键，尽量将热点数据分布到不同的分片
> 除非特殊情况，否则都应该使用自动而非手动分片，尤其是均衡器还在开启的情况下  

# Chapter16：分片管理
👉 sh.status()  
块较少时打印每个块的位置，较多时打印块的概述，可以传参true以获取详情  
👉 mongos.config.shards  
记录集群内所有分片的信息  
👉 mongos.config.database  
记录集群内所有数据库的信息  
👉 mongos.config.collection  
记录集群内所有"分片"集合的信息  
👉 mongos.config.chunks  
记录集群内所有块的信息  
👉 mongos.config.changelog  
记录集群的操作（块的拆分/迁移）  
👉 mongos.config.tag  
记录分片标签(tag)的范围(min,max)  
👉 mongos.config.settings  
记录均衡器设置和块大小信息  

- 服务器相关  
  + 添加服务器：addShard
  + 修改分片的服务器：直连到分片所在服，修改其config.shards
  + 删除分片：不推荐，一般分片在未来都可能被使用，如果强行删除必须保证均衡器开启，数据正常排出
  + 修改配置服务器：
> 修改配置服务器 configSvr 是有风险的，通常需要停机，尽量通过修改 mongos 的配置来实现需求  
> 修改前需要关闭所有 mongos 进程，保证它们 --configdb 一致，关闭所有分片  

- 数据均衡
  + 修改块大小：默认为 64MB，通过修改 mongos 的 config.settings 中 chunksize 字段修改
  + 移动块：sh.moveChunk 可以移动块，sh.splitAt 可以拆分过大的块
  + 特大块：指超过了 chunksize 的块，无法被拆分（由于片键值的设置原因，如按日期做分片）

> 在执行几乎所有的数据库管理操作之前，都应关闭均衡器

# Chapter17：了解应用的动态
> 学会知道当前 MongoDB 在做什么，什么操作耗时较大  

db.currentOp()：获取当前正在执行的操作，及其详情  
db.killOp(opid)：终止opid对应的操作  
db.setProfilingLevel(lev)：根据分析等级，开关分析器，记录在 system.profile 集合中   
> 分析等级：【0】表示关闭，【1,100】表示记录耗时大于100ms的操作，【2】表示记录所有内容  

db.getProfilingLevel()：获取当前分析等级  
Object.bsonsize({\_id:ObjectId})：查询文档大小  
db.col.stats()：查询集合信息  
db.stats()：查询数据库信息

mongotop / mongostat：略，前类似top，后输出部分netstat和vmstat/sar的东西

# Chapter18：数据管理
（略）

# Chapter19：持久性
MongoDB 使用日记系统来实现它的“持久性”（因为不存在事务，所以持久性含义有所不同）  
每个写操作，都会产生一条日记 journal，记录了更改的磁盘地址和字节数；  
假设服务器宕机了，可以进行回放 replay，重新执行那些还未刷新 flush 到磁盘的写入操作。  

数据文件默认每 60s 刷新一次到磁盘，而MongoDB默认每隔 100ms 就会将写入数据的操作写入日记；  
即默认情况下，丢失的写入数据不会超过100ms，可通过 setParam 命令改变 journalCommitInterval 的值(2~500)    
重要写入可以用 getLastError("j":true) 来确保写入操作成功写入日志（最多等待30ms）  

> 持久性无法保证的情况：硬盘写入还未实际完成，却已经返回完成（因为硬件问题），此刻宕机数据丢失，副本集可以解决部分问题  
> db.col.validate()：可以用检测集合是否损坏，输出结果的 valid 字段true则为完好  

# Chapter20：启动和停止MongoDB
（略，擅用 mongod --help 即可）

# Chapter21：监控MongoDB
（略，暂未能使用到 MMS 这种服务）

# Chapter22：备份
- 文件系统快照 snapshot  
  系统支持快照，且mongod开启了journal
- 复制数据文件  
  将数据目录 /data/db 的文件全部复制到其他地方，要防止数据文件改变可用 db.fsynclock() 锁定数据库，并且将脏页刷新至磁盘  
- mongodump  
  备份和恢复速度较慢，处理副本集需加 --oplog 获取快照，恢复时需要 --oplogReplay 回放快照  
> 不可能对运行中的分片集群进行“完美的”备份  
> 在备份集群时，需先关闭 balancer 然后通过 mongos 运行 mongodump，恢复时也是对一个 mongos 运行 mongorestore  

- mongooplog(mon-goop-log)  
  增量备份，是最复杂的技术。假设有2台机器组成副本集，primary为A，secondary为B。  
  1、记录A的oplog最近一次optime；  
  2、对A进行一次完整备份，使用上述的任意方法，把这个备份的数据恢复到B；  
  3、定期添加A中的数据到B，在B机器上运行 mongooplog --from A --second (时间差值[now - optime] + 冗余的若干秒防止数据丢失)  
  
# Chapter23：部署MongoDB
（略）  
> 并没有钱自行部署mongo专用服务器，只能参考其中部分思想，  
> 但如 NUMA 和 大内存页 这种都是操作系统提供来优化其他程序运行效率的，  
> 关闭了虽然可能提高了 MongoDB 的表现，实际上可能得不偿失，具体情况具体分析。  

# 附录A：安装MongoDB
（略）  

# 附录A：深入MongoDB
- BSON —— Binary JSON  
  + 高效：描述数据最差情况比JSON效率略低，最好情况远超JSON  
  + 可遍历性：如给字符串数据加上长度字段，使其不依赖'\0'  
  + 高性能：编解码快速，用类C类型表示  
- 网络协议 wire protocol
一个轻量的 TCP/IP 协议，对 BSON 数据进行包装，20字节的首部包含了各种操作信息  
- 数据文件  
每个数据库对应若干文件，如 foo 库会有 foo.ns foo.0 foo.1 foo.2 ...  
每个数据文件大小是前者的 2倍 直至达到最大值 2GB  
这样做保证了稳定的性能表现，默认还会预分配数据文件，用 --noprealloc 选项可关闭  
- 命名空间和区段  
collection 中的 document 和 索引 都有自己的命名空间，命名空间的元信息存储于 .ns 文件中  
每个命名空间中的数据在磁盘上都会被分为几组数据文件，即区段  

- 内存映射存储引擎  
MongoDB 唯一(v2.0)支持的存储引擎就是 内存映射引擎，在服务器启动的时候，将所有数据文件映射到内存  
然后由 操作系统 负责将数据刷新到磁盘，以及管理内存中的数据页交换（应用层的内存管理较少？）  
> 实际上 3.2 版本开始已经在用 WiredTiger 作为其默认的存储引擎了  
> 并且在 4.2 版本将 MMAPv1（内存映射引擎） 删除了














