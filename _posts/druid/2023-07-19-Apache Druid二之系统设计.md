---
title: Apache Druid二之系统设计
layout: post
published: true
author: 陈家辉
tags:
  - Druid
  - 中间件
  - 数据库
---



Druid有着云友好且易于操作的分布式架构。支持独立配置和扩展服务，在集群操作方面拥有最大的灵活性。这种设计增强了系统容错性：一个组件宕机不会立刻影响到其它组件。

# Druid架构

下图展示了组成Druid架构的服务，这些服务怎样被组合部署到服务器上以及查询请求和数据流在架构中如何流动。

![img](https://druid.apache.org/docs/latest/assets/druid-architecture.png)

## Druid 进程

Druid有以下几种服务：

* Coordinator 负责集群的数据可用性
* Overlord 控制数据摄取工作负载的分配
* Broker 处理来自外部客户端的查询请求
* Router 是可选的；用于路由请求到Broker，Coordinator或Overlord
* Historical 存储可查询的数据
* MiddleManager 负责抓取数据

这些服务可以再Web console中查看：

![Druid services](https://druid.apache.org/docs/latest/assets/services-overview.png)

## Druid 服务

Druid进程可以以任意你喜欢的方式部署，但是为了便于部署，建议将它们组织成三种服务类型: Master、Query和Data。

* Master：运行Coordinator和Overlord，用于管理数据可用性和数据摄取
* Query：运行Broker和可选的Router，处理客户端查询请求
* Data：运行Historical和MiddleManager，执行摄取工作负载和存储所有可查询数据

## 外部依赖

Druid有三个外部依赖。

### Deep storage

Druid使用Deep storage存储摄取的数据。Deep storage是一个可以被Druid服务访问的共享文件存储。在集群部署中，它一般是一个分布式对象存储，例如S3或者HDFS，或者是一个网络挂载文件系统。如果是单机部署，可以使用本地磁盘。

Druid仅将deep storage作为数据备份以及一种在Druid进程之间的后台传输数据的方式。Druid存储的数据文件叫做*segments*。Historical将segments缓存在本地磁盘上，并且从内存以及本地磁盘进行数据查询。这意味着Druid服务在处理查询请求的时候不需要访问Deep storage，这有助于提供更低的查询时延。

Deep storage是druid弹性、容错设计的重要组成部分。即使每个数据服务器丢失并重新配置，druid也会从Deep storage拉取数据。

### 元数据存储

存储多种共享的系统元数据，例如segment使用信息，任务信息等。在集群部署中，通常使用PostgreSQL或者MySQL。单体服务部署一般使用Derby数据库。

### Zookeeper

用于内部服务发现，协调以及选举。

## 存储设计

### Datasources 和 segments

Druid数据存储在*datasource*，类似于传统关系型数据库中的tables。每个datasource通过时间进行分区，你也可以选择被其他属性进行分区。每个时间分区称为一个chunk，在一个chunk内，数据又被划分为一个或多个*segment*。每个Segment是一个文件，一般包含几百万行数据。

![img](https://druid.apache.org/docs/latest/assets/druid-timeline.png)

一个datasource可能有几个甚至成百上千，上百万的*segment*，每个segment都是由MiddleManager创建为可变的以及未提交的。数据一旦被加入未提交的segment，就可以被查询。segment通过产生一个紧凑的，具有索引的文件用于加速后续的查询：

* 转化为列式格式

* 通过bitmap进行索引

* 压缩

  * 将字符串列通过字典转化为id最小存储
  * 将bitmap索引进行压缩
  * 对所有列进行自适应压缩

  segment将会周期性地被提交发布到deep storage，成为不可变数据，同时从MiddleManager转移到Historical。关于这个segment的entry将会被写入metadata store。这个entry里包含了该segment的元数据，包括schema，大小，deep storage的存储位置。这些信息告诉协调者在集群中，什么数据是可用的。

### 索引与切换

建立一个新的segment的过程叫做索引，将segment发布并且开始服务于Historical叫做切换

索引视角：

1. 索引任务开始运行并构建新segment。它必须在开始构建segment之前确定segment的标识符。对于一个正在append的任务(比如 Kafka 任务，或者append模式下的索引任务) ，这是通过调用 Overlord 上的“allocate” API 来完成的，以便潜在地将一个新的分区添加到一组现有的segment中。对于正在覆盖的任务(比如 Hadoop 任务，或者不处于追加模式的索引任务) ，可以通过锁定一个间隔并创建一个新的版本号和一组新的segment来完成。
2. 如果索引任务是一个实时任务(如kafka task) ，那么segment在这一点上是立即可查询的。数据可用，但是未发布。
3. 当索引任务完成了segment的数据读取后，它将segment推入deep storage，然后通过将记录写入meta storage来发布它。
4. 如果索引任务是一个实时任务，那么为了确保数据持续可用于查询，它将等待一个历史进程来加载segment。如果索引任务不是实时任务，则立即退出。

Coordinator/Historical 视角：

1. Coordinator定期(默认情况下，每1分钟)轮询新发布的段的metadata store。
2. 当协调器发现一个已发布和使用但不可用的segment时，它会选择一个 Historical进程来加载这个segment。
3. Historical加载该segment并开始为其提供服务。
4. 此时，如果索引任务正在等待切换，它将退出。

### Segment标识符

segment由四部分组成：

* Datasource名称
* 时间segment（根据这个segment处于的时间chunk，这对应于摄取时指定的`segmentGranularity`）
* 版本号（通常是一个 ISO8601时间戳，与segment集合首次启动时间相对应）。
* 分区号(一个整数，在Datasource名称 + 时间间隔 + 版本中唯一; 不一定是连续的)。

例如，这是数据源`clarity-cloud0`中一个segment的标识符—— Cloud0，时间chunk`2018-05-21T16:00:00.000Z/2018-05-21T17:00:00.000Z`，版本`2018-05-21T15:56:09.909Z`，以及分区号1:

> clarity-cloud0_2018-05-21T16:00:00.000Z_2018-05-21T17:00:00.000Z_2018-05-21T15:56:09.909Z_1

分区号为0是会被省略

> clarity-cloud0_2018-05-21T16:00:00.000Z_2018-05-21T17:00:00.000Z_2018-05-21T15:56:09.909Z

### Segment版本号

版本号提供了一种多版本并发控制(MVCC)形式，以支持批处理模式覆盖。如果您只是添加数据，那么每个时间块只有一个版本。但是当您覆盖数据时，Druid 将无缝地从查询旧版本切换到查询新的更新版本。具体来说，使用相同的数据源、相同的时间间隔、但版本号更高的数据创建一组新的segment。这向德鲁伊系统的其他部分发出了一个信号，即旧版本应该从集群中删除，新版本应该替换旧版本。

这种切换似乎是即时发生在用户身上的，因为 Druid 通过首先加载新数据(但不允许查询)来处理这种情况，一旦新数据全部加载，切换所有新查询以使用这些新segment。几分钟后，它把旧的部分丢掉。

### Segment生命周期

每个Segment都有一个生命周期，涉及以下三个主要领域:

1. 元数据存储: 一旦构造完一个segment，元数据存储就会存储segment元数据(一个小的 JSON 有效负载通常不超过几 KB)。将segment的记录插入到元数据存储中的行为称为发布。这些元数据记录有一个名为 used 的布尔标志，用于控制segment是否应该是可查询的。实时任务创建的segment将在发布之前可用，因为它们只在segment完成时发布，不会接受任何额外的数据行。

2. deep storage: segment数据文件一旦构建完成，就被推送到深度存储。这在将元数据发布到元数据存储区之前立即发生。

3. 可用于查询: segment在某些druid数据服务上可用于查询，如实时任务或Historical。

您可以使用 Druid SQL `sys.lines` 表来检查当前活动segment的状态，它包含以下几种状态：

* `is_published`: 如果segment元数据已经发布到元数据存储并，则为 True。
* `is_available`: 如果segment当前可用于查询，则为 True，无论是在实时任务上还是在Historical进程上。
* `is_realtime`: 如果segment仅在实时任务上可用，则为 True。对于使用实时摄入的数据源，这通常会从 true 开始，然后在segment发布和传递时变为 false。
* `is_ overshadowed`: 如果segment被发布(`used`设置为 True)且被其他一些已发布segment完全覆盖，则为 True。一般来说，这是一个暂时的状态，处于这种状态的segment很快就会将它们的`used`标志自动设置为 false。

## 可用性以及一致性

Druid 在摄入和查询之间有一个架构上的分离，如上面的索引和切换中所描述的。这意味着在理解 Druid 的可用性和一致性属性时，我们必须分别研究每个部分。

在摄取方面，Druid 的主要摄取方法都是基于拉的，并提供事务保证。这意味着您可以保证使用这些方法的摄取将以全有或全无的方式发布:

* 有监督的“流”摄取方式，如kafka和Kinesis。使用这些方式，Druid 在同一事务中将流偏移量与segment元数据一起提交到其元数据存储中。注意，如果摄取任务失败，那么可以回滚对尚未发布的数据的摄取。在这种情况下，部分摄取的数据被丢弃，Druid 将从最后提交的流偏移量集恢复摄取。这确保了精确一次的发布行为。
* 基于hadoop的批量摄取。每次任务发布所有的segment元数据在一次事务里。
* 本机批量摄取。在并行模式下，可观测任务在所有的子任务结束后在一次事务中发布所有的sement元数据。在简单(单任务)模式下，单个任务在完成后在单个事务中发布所有段元数据。

此外，一些摄取方式提供幂等性保证。这说明重复地执行相同摄取任务不会导致数据重复摄取。

* 有监督的“可寻找流”摄取方法，如kafka和 Kinesis 是幂等的，因为流偏移量和段元数据存储在一起，并在锁步更新。
* 基于 Hadoop 的批量摄取是等幂的，除非您的输入源之一与您正在摄取的druid数据源相同。在这种情况下，两次运行同一个任务是非等幂的，因为您是在添加现有数据，而不是覆盖它。
* 本机批量摄取是幂等的，除非 `appendToExisting`为 true，或者您的某个输入源与您正在摄取的druid数据源相同。在这两种情况中的任何一种情况下，两次运行同一个任务都是非幂等的，因为您是在添加现有数据，而不是覆盖它。

在查询方面，Druid Broker 负责确保在给定的查询中包含一组一致的segment。它根据当前可用的内容选择适当的segment版本集，以便在查询启动时使用。这是由原子替换支持的，这个特性可以确保从用户的角度来看，查询可以立即从旧版本的数据切换到新的数据集，而不会造成一致性或性能影响。(参见上面的段版本控制。)这用于基于 Hadoop 的批处理摄取、当`appendToExisting`为 false 时的本机批处理摄取和压缩。

### 查询过程

查询请求会分布在Druid集群中，通过Broker管理。查询首先会到达Broker，Broker标识那些segments可能与此次查询有关。这些segment通过时间进行剪枝，也可以通过你的datasource指定的数据分区指标剪枝。Broker会指出哪些Historical和MiddlManager拥有这些segment并且分发重写的子查询到对应的进程。这些进程(Historical/MiddlManager)进程执行每个子查询然后返回结果给Broker。Broker会对这些分散的数据做聚合得到最终的结果进行相应。

时间和属性剪枝是Druid减少每个查询必须扫描的数据量的重要方法，但这不是唯一的方法。对于粒度级别比 Broker 可以用于修剪的过滤器低，每个segment内的索引结构允许 Historical 在查看任何数据行之前找出哪些(如果有的话)行匹配过滤器集。一旦Historical知道哪些行匹配特定的查询，它只访问该查询所需的特定行和列。

综上所述，Druid使用三种不同的技术来优化查询性能：

* 减少需要访问的segment的数量
* 在每个segment中，使用索引确定哪些行需要访问
* 在每个segment中，只读与此次查询相关特定的行和列

