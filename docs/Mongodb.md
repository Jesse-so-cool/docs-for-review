# Mongodb

## 架构

- Master-Slaver： 是一种主从副本的模式，目前已经不推荐使用。
- Replica Set ：模式取代了 Master-Slaver 模式，是一种互为主从的关系。Replica Set 将数据复制多份保存，不同服务器保存同一份数据，在出现故障时自动切换，实现故障转移，在实际生产中非常实用。
- Sharding ：模式适合处理大量数据，它将数据分开存储，不同服务器保存不同的数据，所有服务器数据的总和即为整个数据集。

### 副本集

![img](https://pic4.zhimg.com/v2-67e34a1f8c071021e23d0f21b47ff587_b.webp)



主节点负责数据的**写入和更新**，并在更新数据的同时，将操作信息写入名为 oplog 的日志文件当中。主节点还负责指定其他节点为从节点，并设置从节点数据的可读性，从而让从节点来分担集群读取数据的压力。

用户还可以使用副本集来扩展读性能，客户端有能力发送读写操作给不同的服务器，也可以在不同的数据中心获取不同的副本来扩展分布式应用的能力。

- 主从切换

副本集中有一个额外的仲裁节点（不需要使用专用的硬件设备），负责在主节点发生故障时，参与选举新节点作为主节点。

副本集中的各节点会通过心跳信息来检测各自的健康状况，当主节点出现故障时，多个从节点会触发一次新的选举操作，并选举其中一个作为新的主节点。为了保证选举票数不同，副本集的节点数保持为奇数。



### 分片

> 副本集可以解决主节点发生故障导致数据丢失或不可用的问题，但遇到需要存储海量数据的情况时，副本集机制就束手无策了。







构建一个 MongoDB 的分片集群，需要三个重要的组件，分别是分片服务器（Shard Server）、配置服务器（Config Server）和路由服务器（Route Server）。

- Shard Server
- 每个 Shard Server 都是一个 mongod 数据库实例，用于存储实际的数据块。整个数据库集合分成多个块存储在不同的 Shard Server 中。
- 在实际生产中，一个 Shard Server 可由几台机器组成一个副本集来承担，防止因主节点单点故障导致整个系统崩溃。
- Config Server
- 这是独立的一个 mongod 进程，保存集群和分片的元数据，在集群启动最开始时建立，保存各个分片包含数据的信息。
- Route Server
- 这是独立的一个 mongos 进程，Route Server 在集群中可作为路由使用，客户端由此接入，让整个集群看起来像是一个单一的数据库，提供客户端应用程序和分片集群之间的接口。
- Route Server 本身不保存数据，启动时从 Config Server 加载集群信息到缓存中，并将客户端的请求路由给每个 Shard Server，在各 Shard Server 返回结果后进行聚合并返回客户端。



#### 如何分片

MongoDB数据库中数据的分片是以集合为基本单位的，集合中的数据通过片键被分成多部分。

对集合进行分片时，您需要选择一个**片键** , 片键是每条记录都必须包含的，且建立了索引的单个字段或复合字段，MongoDB数据库按照片键将数据划分到不同的数据块中，并将数据块均衡地分布到所有分片中。

片键分片模式有两种：

- 使用基于范围的分片方式

- 基于哈希的分片方式。

#### 分片原理

使用 MongoDB sharding 后，数据会以 chunk 为单位（默认64MB）根据 `shardKey` 分散到后端1或多个 shard 上。

每个 database 会有一个 `primary shard`，在数据库创建时分配

- database 下启用分片（即调用 `shardCollection` 命令）的集合，刚开始会生成一个[minKey, maxKey] 的 chunk，该 chunk 初始会存储在 `primary shard` 上，然后随着数据的写入，不断的发生 chunk 分裂及迁移，整个过程如下图所示。
- database 下没有启用分片的集合，其所有数据都会存储到 `primary shard`



##### chunk

MongoDB sharding 主要将shardkey分割成块。每个块都由连续范围的key组成。如下图



![Diagram of the shard key value space segmented into smaller ranges or chunks.](E:\坚果云\Typora\img\sharding-range-based.bakedsvg.svg)

一个chunk最小是由一个key的value组成 例如{25：25}（当单个key太多 即使超过chunk也不会触发分裂）

##### chunk size

chunk-size默认64M

调整默认值的影响

- 小块会导致更均匀的数据分布，而代价是更频繁的迁移。这会在查询路由（mongos）层产生开销。
- 大块导致更少的迁移。从网络角度和查询路由层的内部开销来看，这都更有效。但是，这些效率是以潜在的数据分布不均为代价的。
- 块大小影响每个块要迁移的最大文档数。
- 分片现有集合时，块大小会影响最大集合大小。分片后，块大小不限制集合大小

##### chunk分裂

mongoDB 的自动 chunk 分裂只会发生在 mongos 写入数据时，当写入的数据超过一定量时，就会触发 chunk 的分裂

chunkSize 为默认64MB是，分裂阈值如下

| 集合 chunk 数量 | 分裂阈值 |
| :-------------: | :------: |
|        1        |  1024B   |
|     [1, 3)      |  0.5MB   |
|     [3, 10)     |   16MB   |
|    [10, 20)     |   32MB   |
|    [20, max)    |   64MB   |

写入数据时，当 chunk 上写入的数据量，超过分裂阈值时，就会触发 chunk 的分裂，chunk 分裂后，当出现各个 shard 上 chunk 分布不均衡时，就会触发 chunk 迁移。

##### 何时触发 chunk 迁移？

默认情况下，MongoDB 会开启 balancer，在各个 shard 间迁移 chunk 来让各个 shard 间负载均衡。用户也可以手动的调用 `moveChunk` 命令在 shard 之间迁移数据。

Balancer 在工作时，会根据`shard tag`、`集合的 chunk 数量`、`shard 间 chunk 数量差值` 来决定是否需要迁移。

![Diagram of a collection distributed across three shards. For this collection, the difference in the number of chunks between the shards reaches the *migration thresholds* (E:\坚果云\Typora\img\sharding-migrating.bakedsvg.svg) and triggers migration.](https://docs.mongodb.com/manual/images/sharding-migrating.bakedsvg.svg)

> 在某些情况下，块可以超出指定的块大小，但不能进行分割。最常见的情况是当一个块表示一个shard键值时。由于区块无法分割，它会继续超出区块大小，成为一个巨型区块。当这些巨型块继续增长时，它们可能成为性能瓶颈，特别是当碎片键值出现频率很高时。
>
> 从mongodb5.0开始，您可以通过更改文档的shard键来重新硬化集合。
>
> 从MongoDB 4.4开始，MongoDB提供了refineCollectionShardKey命令。优化集合的shard密钥可以实现更细粒度的数据分发，并且可以解决现有密钥基数不足导致大数据块的情况。



## 操作

### GridFS

GridFS是MongoDB的一种存储机制，用来存储大型二进制文件。

 GridFS会自动平衡已有的复制或者为MongoDB设置的自动分片，所以对文件存储做故障转移或者横向扩展会更容易。

**优点**：例如，在GridFS文件系统中，如果在同一个目录下存储大量的文件，没有任何问题。 

在GridFS中，文件存储的集中度会比较高，因为MongoDB是以2 GB为单位来分配数据文件的。

GridFS也有一些**缺点**

- GridFS的性能比较低：从MongoDB中访问文件，不如直接从文件系统中访问文件速度快

- 如果要修改GridFS上的文档，只能先将已有文档删除，然后再将整个文档重新保存

- MongoDB将文件作为多个文档进行存储，所以它无法在同一时间对文件中的所有块加锁

  通常来说，如果你有一些不常改变但是经常需要连续访问的大文件，那么使用GridFS再合适不过了。

### 聚合

使用聚合框架可以对集合中的文档进行变换和组合。基本上，可以用多个构件创建一个管道（pipeline），用于对一连串的文档进行处理。这些构件包括筛选（filtering）、投射（projecting）、分组（grouping）、排序（sorting）、限制（limiting）和跳过（skipping）。

例如，有一个保存着杂志文章的集合，你可能希望找出发表文章最多的那个作者。假设每篇文章被保存为MongoDB中的一个文档，可以按照如下步骤创建管道。

（1）将每个文章文档中的作者投射出来。（2）将作者按照名字排序，统计每个名字出现的次数。（3）将作者按照名字出现次数降序排列。（4）将返回结果限制为前5个。

![image-20210721154459130](E:\坚果云\Typora\img\image-20210721154459130.png)



## 审计



MongoDB **2.6企业版**增加了对审计的支持。你可以配置MongoDB实例，对于感兴趣的MongoDB操作，像用户登录、DDL修改、复制集配置修改等，生成审计事件。这能让你使用已经存在企业审计工具来获取和处理需要的事件。更详细的信息参考MongoDB可被审计的事件列表。

https://docs.mongodb.com/manual/core/auditing/

中文：https://docs.jinmu.info/MongoDB-Manual-zh/docs/10-Security/Auditing.html

修改输出:https://www.cnblogs.com/myzony/p/10265395.html



