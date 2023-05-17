# Chap 1. 为现代Web而生的数据库
- 介绍了 10gen 与 MongoDB 的诞生
- 介绍了 document 模型的基础含义  
  + 提到了和传统关系型数据库的区别，为什么要用join连接，把对象的属性放一起不是更方便吗  
  + 正规化 normalization 把对象数据拆分虽然好，但代价是查询数据时复杂度  
- 介绍了即时查询和数据集膨胀时使用的二级索引  
  + 在 MongoDB 中每个 collection 最多创建 64 个索引，支持能在 RDBMS 中找到的各种索引  
  + 二级索引用 B树 实现，类似大多数关系型数据库的索引数据结构  
- 介绍了副本集复制  
- 介绍了实现持久性的方式  
  + 默认的写操作都是非应答式的，称为 fire-and-forget（发射后就不管了）  
  + 对于价值较低的数据，这种方式是十分理想的，当然也可以采用应答式写保证高价值数据安全  
- 介绍了 副本集 和 分片的副本集  
- 介绍了 mongoshell 及其命令行工具如 dump restore import export sniff stat ...
- 介绍了 MongoDB 和其它主流关系/非关系型数据库对比  
- 介绍了适用场景
  + Web应用
  + 敏捷开发
  + 日志存储
  + 缓存？轻量的一般用Redis

# Chap 2. MongoDB JavaScript Shell
（略，权威指南基本说过，简读一遍）

# Chap 3. 使用MongoDB编写程序
> 因书中主要使用 Ruby驱动 作为示例，所以本章主要阐述思想  
- 驱动是如何工作的
  + 对象ID生成  
  每个 document 都有一个主键，驱动在向服务器发送文档前会检查是否有 \_id 字段，如果没有会生成一个  
  因为其中包含了机器ID和PID和计数器，故不需要担心客户端生成的唯一ID与服务器上的ID冲突  
  + BSON  
  本书编写时（2012）BSON规范包含了 19 种数据类型：
    - UTF-8 字符串
    - 32/64位整数
    - double
    - bool
    - timestamp
    - datetime
    - 针对模糊大字段的二进制 opaque blob
    - 符号类型 symbol type
    - ObjectId
    - .....
  + 网络传输  
  使用了基于TCP的自定义网络协议，因为对象ID可以在客户端生成，所以可以不等待写操作返回  
  当然驱动也会提供安全的应答式写操作，实际是调用 getLastError() 并且传入不同参数  

# Chap 4. 面向文档的数据
> 本章通过一个电商DB的设计引出了，一对多，多对多关系的文档应该怎么设计，但笔者对这个例子实在不感冒  
- 数据文件与空间分配  
```
-rw------- 1 root root    16384 May 16 15:55 collection-0-1334077603761531500.wt
-rw------- 1 root root    36864 May 16 15:55 collection-0--6879496898766315519.wt
-rw------- 1 root root 12201984 May 16 16:02 collection-0-7190526642408368948.wt
-rw------- 1 root root    36864 May 16 15:56 collection-2-1334077603761531500.wt
-rw------- 1 root root    36864 May 16 16:40 collection-4-1334077603761531500.wt
drwx------ 2 root root     4096 May 16 16:41 diagnostic.data
-rw------- 1 root root    16384 May 16 15:55 index-1-1334077603761531500.wt
-rw------- 1 root root    16384 May 16 10:45 index-1--6879496898766315519.wt
-rw------- 1 root root  9891840 May 16 16:02 index-1-7190526642408368948.wt
-rw------- 1 root root    36864 May 16 15:56 index-3-1334077603761531500.wt
-rw------- 1 root root    20480 May 16 16:40 index-5-1334077603761531500.wt
-rw------- 1 root root    20480 May 16 16:41 index-6-1334077603761531500.wt
drwx------ 2 root root     4096 May 16 16:01 journal
-rw------- 1 root root    36864 May 16 15:58 _mdb_catalog.wt
-rw------- 1 root root        4 May 16 15:55 mongod.lock
-rw------- 1 root root    36864 May 16 16:41 sizeStorer.wt
-rw------- 1 root root      114 May 16 10:28 storage.bson
-rw------- 1 root root       45 May 16 10:28 WiredTiger
-rw------- 1 root root     4096 May 16 15:55 WiredTigerLAS.wt
-rw------- 1 root root       21 May 16 10:28 WiredTiger.lock
-rw------- 1 root root     1112 May 16 16:41 WiredTiger.turtle
-rw------- 1 root root    69632 May 16 16:41 WiredTiger.wt
```
这是我本机一个简单demo的数据文件情况，版本是3.6，已经和书中例子基本不同了，那就写一下相同的部分吧  
mongod.lock：大小固定在16MB，大约能存储2w4个命名空间（实际上才4byte）  
每个数据文件大小是上一个的 2倍，最大是 2GB，这条规则基本还遵守着  
> ns 其实就是 namespace 命名空间的缩写  

- 集合和文档  
集合分为：标准集合，固定集合（环，最老的文档被最新的替代），系统集合（system）  
文档在发送给mongo服务器前都需要经过序列化，驱动一般会提供这个功能。并且一般言简意赅的键可以省下不少磁盘空间  

# Chap 5. 查询与聚合
> 开篇继续用电商DB举例解释查询，家人们谁懂啊 ...  
> 本章主要介绍 find / group 的使用，还有高级的 MapReduce，主要还是参考权威指南即可  

# Chap 6. 更新、原子操作与删除
- 原子文档处理  
db.foo.findAndModify()：这个命令结合了 find 和 update，原子的执行操作可以避免多线程带来的意料之外的问题  
- 并发性、原子性、隔离性  
语句中的 $atomic 操作符可以使 update remove 等操作变为原子的（不可暂停）  
> 书中太多特性过于老旧，尤其MongoDB迭代速度那么快，只能挑部分内容阐述  

# Chap 7. 索引与查询优化
- 创建索引  
  + 单键索引：db.foo.ensureIndex({key : 1})  
  + 多键索引：
  + 唯一性索引：db.foo.ensureIndex({key : 1},{unique : true})  
  对唯一性索引过的key插入新val，假设这个val已存在，插入就会失败  
  + 稀疏性索引：db.foo.ensureIndex({key : 1},{sparse : true})  
- 删除索引
db.runCommand({deleteIndexes : "colname", index : "key"})  
db.foo.dropIndex("indexname")  
- B tree  
在 MongoDB 的B树实现里，每个节点会有一些留空，新节点会分配 8192 字节，每个节点可以存放多少个索引键  
取决于索引键的平均大小，默认情况下，B树节点的内容被有意的维护在 60% 占比左右  

# Chap 8. 复制
> 本周主要介绍了 rs 命令集的使用，复制还是依靠固定集合oplog 和 不同节点的心跳检测  
> 副本集最多含有 12 个节点，其余细则和权威指南差不多，不赘述  
- 写关注
使用 getLastError({"w" : timeout})，指定写操作在指定 ms 内无法复制就返回错误，timeout可以是[2,500]  

# Chap 9. 分片
- 分片组件  
分片集群由 cluster、mongos router、config server 组成  
- 分片 cluster
每个分片都被部署成一个副本集，保存了集群整体数据的一部分，因为是副本集，所以有自己的复制机制和故障转移机制  
- mongos路由器  
mongos连接至分片集群，将所有读写请求转发到合适的分片上，mongos是轻量级和非持久化的  
- 配置服务器 config server  
配置服务器持久化了分片集群的元数据：包括了集群的配置，数据库和集合的位置，和变更记录  
实际上部署配置服务器时，必须运行 3 个配置服务器在不同的机器上以实现冗余（高可用）  
- 块 chunk 
表示一段连续范围的数据，默认的最大块是 64MB，或者说 10W 个文档，二者取min  
- 均衡器 balancer  
控制每个分片中块的数量相近，当集群中块最多的分片和最少的分片，差值 >8 时就会发起一次均衡处理  
- 分片的索引  
  + 每个分片应该维护自己的索引
  + 每个分片的分片集合都应该有相同的索引（查询性能稳定）
  + 分片集只允许在 \_id 字段 和 片键 上添加 唯一性索引（降低复杂度）
- 理想的片键  
  + 数据能够在各个分片上均匀分布
  + 利用程序的局部性原理
  + 有足够的颗粒度进行块拆分  
所以实际生产中常用复合分片键，userid 和 username 的组合就很不错

# Chap 10. 部署与管理
- CPU密集型 还是 I/O密集型  
MongoDB还是比较难达到CPU瓶颈的，除非请求充分发挥了局部性且足够频繁  
I/O密集型说明可能出现了慢查询的问题，需要定位问题所在，大多数性能问题的源头都是磁盘  
- 大端 还是 小段
服务器只能部署在小段机器，但客户端通过驱动，也可以在大端机器运行  
- 32位 还是 64位  
32位的寻址空间限制了性能发挥，尽量将服务器部署在64位机器  
- 文件系统和fd数量  
优先选择ext4这种fs，并且需要调大fd限额（默认1024）保证客户端连接正常  

# 附录 A. 安装
（略）
# 附录 B. 设计模式
- 树  
可以通过设置 depth（树深度） path（树的路径） 等字段实现  
- 工作队列  
使用 findAndModify 原子地处理队列内容  
- 动态属性  
通过设置固定的字符串键，对这些键建索引如 db.product.ensureIndex({"name" : 1, "val" : 1})  
插入数据时为 name 和 val 赋值其属性，用 elemMatch 查询，开销较大因为索引内有键名  
- 事务  
虽然没有ACID特性，但可以支持一致性读，单个文档的原子性，持久化更新，实现类似事务的效果  
- 反模式：错误示范  
  + 索引随意：按 Chap7 规范建立索引  
  + 类型杂乱：如一个 money 字段又用字符串又用number  
  + 桶集合：把不同的东西堆到一个集合里  
  + 大文档嵌套深：每个文档应该小于 100KB，除非是保存二进制数据，文档越小越容易CRUD  
  + 一个用户一个集合：命名空间容易超过默认的24000，超过后需要重新分DB，浪费空间  
  + 无法分片的集合：集合在设计时必须尽量设计成能够高效分片的 Chap 9   
# 附录 C. 二进制数据与GridFS
> 用于存储图片、音频、缩略图等任意大小的二进制文件  
# 附录 D. 其它语言的驱动
（略）  
# 附录 E. 空间索引
假设有一个数据库，里面有很多 document，所有 document 都有如下属性，
```
{
  location : { (±180)
    lon : 12, (longitude 经度)   
    lat : 34, (latitude 纬度)
}
```
我们可以建立一个坐标系，在二维平面上描绘出这些数据，  
针对某一个 document，我们对其画圆，找出其周边相邻范围 n 内的点，MongoDB 提供了 $near 操作符  
首先建立二维索引：db.col.ensureIndex({ location : '2d'})  
然后就可以对其进行查询，比如查询（12,34）附近的点  
db.col.find({'loc' : {$near : [12,34]}})  
返回结果默认限制在 100 条，可以用 limit 调整结果集大小  

当然更高级的，可以使用 geoNear 命令：  
db.runCommand({'geoNear' : 'col', near : [12,34], num : 2})  
命令返回 2 个结果，甚至会返回 dis 字段告诉你空间点之间的距离  
还有 within center box maxDistance 球面的 nearSphere centerSphere 等操作符的混合使用，这里暂不深究  
