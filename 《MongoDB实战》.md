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
每个数据文件大小是上一个的 2倍，最大是 2GB，这个基本还遵守着。  
> ns 其实就是 namespace 命名空间的缩写  

