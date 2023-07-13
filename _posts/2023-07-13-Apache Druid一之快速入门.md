---
title: Apache Druid一之快速入门
layout: post
author: 陈家辉
tags:
  - Druid
  - 中间件
  - 数据库
---

# 简介

Apache Druid 是一个实时分析数据库，专为对大型数据集进行快速切片分析（“ [OLAP ”查询](http://en.wikipedia.org/wiki/Online_analytical_processing)）而设计。大多数情况下，Druid 为实时摄取、快速查询性能和高正常运行时间很重要的用例提供支持。

Druid 通常用作分析应用程序 GUI 或需要快速聚合的高并发 API 的数据库后端。Druid 最适合处理面向事件的数据。

Druid 的常见应用领域包括：

- 点击流分析，包括网络和移动分析
- 网络遥测分析，包括网络性能监控
- 服务器指标存储
- 供应链分析，包括制造指标
- 应用程序性能指标
- 数字营销/广告分析
- OLAP

## 关键特性

Druid 的核心架构结合了数据仓库、时间序列数据库和日志搜索系统的思想。Druid 的一些主要功能包括：

1. **列式存储格式。**Druid 使用面向列的存储。这意味着它仅加载特定查询所需的确切列。这极大地提高了仅检索几列的查询的速度。此外，为了支持快速扫描和聚合，Druid 根据每列的数据类型优化列存储。
2. **可扩展的分布式系统。**典型的 Druid 部署跨越数十到数百台服务器的集群。Druid 可以以每秒数百万条记录的速度摄取数据，同时保留数万亿条记录并保持从亚秒到几秒的查询延迟。
3. **大规模并行处理。**Druid 可以跨整个集群并行处理每个查询。
4. **实时或批量摄取。**Druid 可以实时或批量摄取数据。摄取的数据可立即用于查询。
5. **自愈、自平衡、操作方便。**作为操作员，您可以添加服务器以进行横向扩展或删除服务器以进行缩减。Druid 集群会在后台自动重新平衡，无需任何停机。如果 Druid 服务器出现故障，系统会自动路由数据绕过损坏的地方，直到可以更换服务器为止。Druid 被设计为连续运行，不会因任何原因出现计划停机。对于配置更改和软件更新来说也是如此。
6. **云原生、容错架构，不会丢失数据。**摄取后，Druid 会将您的数据副本安全地存储在[深度存储](https://druid.apache.org/docs/latest/design/architecture.html#deep-storage)中。深度存储通常是云存储、HDFS 或共享文件系统。即使在所有 Druid 服务器都发生故障的情况下，您也可以从深度存储中恢复数据。对于仅影响少数 Druid 服务器的有限故障，复制可确保在系统恢复期间仍然可以进行查询。
7. **用于快速过滤的索引。**Druid 使用[Roaring](https://roaringbitmap.org/)或 [CONCISE](https://arxiv.org/pdf/1004.0403)压缩位图索引来创建索引，以实现跨多个列的快速过滤和搜索。
8. **基于时间的分区。**Druid 首先按时间对数据进行分区。您可以选择根据其他字段实施附加分区。基于时间的查询仅访问与查询时间范围匹配的分区，从而显着提高性能。
9. **近似算法。**Druid 包括近似计数离散、近似排名以及近似直方图和分位数计算的算法。这些算法提供有限的内存使用，并且通常比精确计算快得多。对于准确性比速度更重要的情况，Druid 还提供精确的计数区分和精确排名。
10. **摄取时自动摘要。**Druid 有选择地支持在摄取时进行数据汇总。此汇总部分预聚合您的数据，可能会显着节省成本并提高性能。

## 适用场景

* 频繁插入，但是几乎不更新
* 主要是聚合查询，例如'group by'
* 目标查询延迟在100ms到几s之间
* 你的数据带有时间戳，druid根据时间戳对数据做出了性能优化和特殊设计
* 有许多表，但是每次查询只需要用到其中一张表。查询可能会触及多个较小的“lookup”表。
* 有大基数数据列，例如 URL、用户 ID，并且需要对它们进行快速计数和排序。
* 希望从 Kafka、 HDFS、平面文件或对象存储(如 AmazonS3)中加载数据。

## 不适用场景

* 根据主键低时延更新数据
* 您正在构建一个离线报告系统，其中查询延迟并不十分重要。
* 希望执行big join

# 实战操作

## 前置条件

机器至少有6GB内存，且满足以下条件：

- Linux, Mac OS X, or other Unix-like OS. (Windows is not supported)
- [Java 8u92+ or Java 11](https://druid.apache.org/docs/latest/operations/java.html)
- Python 3 (preferred) or Python 2
- Perl 5

## 要做什么

* 安装Druid
* 启动服务
* 使用SQL写入数据以及查询数据

## 安装并启动Druid

```shell
wget https://dlcdn.apache.org/druid/26.0.0/apache-druid-26.0.0-bin.tar.gz
tar -xzf apache-druid-26.0.0-bin.tar.gz
cd apache-druid-26.0.0

# 使用-m指定使用内存，否则Druid会使用80%的可用内存
./bin/start-druid -m 5g
```

![image-20230713164914785](https://cdn.jsdelivr.net/gh/CJH876492153/picture@main/image-20230713164914785.png)

## 访问web console

启动服务后，访问 http://localhost:8888/

在启动之后，如果访问出现错误，耐心等待一会儿，等Druid完全启动。

![image-20230713165123382](https://cdn.jsdelivr.net/gh/CJH876492153/picture@main/image-20230713165123382.png)

## 上传数据

1. 点击Load data - Local disk![image-20230713165758976](https://cdn.jsdelivr.net/gh/CJH876492153/picture@main/image-20230713165758976.png)

2. 选择Local disk，输入以下值

   - **Base directory**: `quickstart/tutorial/`
   - **File filter**: `wikiticker-2015-09-12-sampled.json.gz`

   ![image-20230713170210007](https://cdn.jsdelivr.net/gh/CJH876492153/picture@main/image-20230713170210007.png)

3. 点击Apply，看到已经读取到数据了

   ![image-20230713170550418](https://cdn.jsdelivr.net/gh/CJH876492153/picture@main/image-20230713170550418.png)

4. 点击右下角 Next：Parse data，进入parse页面后，可以做以下操作

   - 展开一行，查看相关数据
   - 自定义数据处理
   - Druid必须有一个timestamp列(内部存储列叫 `__time`). 如果你的数据不含有timestamp，druid会给一个默认值 `1970-01-01 00:00:00`。

   ![image-20230713170629473](https://cdn.jsdelivr.net/gh/CJH876492153/picture@main/image-20230713170629473.png)

5. Parse time

   ![image-20230713172156940](https://cdn.jsdelivr.net/gh/CJH876492153/picture@main/image-20230713172156940.png)

6. 一直next到最后一步，会生成一个Json，这个json将会发给Druid，生成一个创建dataSchema的task。点击submit。

   ![image-20230713172845190](https://cdn.jsdelivr.net/gh/CJH876492153/picture@main/image-20230713172845190.png)

7. 等待任务执行成功，在Query界面就可以看到对应的dataScheme了

   ![image-20230713173658353](https://cdn.jsdelivr.net/gh/CJH876492153/picture@main/image-20230713173658353.png)

## 查询数据

执行以下SQL，可以查询出数据

```sql
SELECT
  channel,
  COUNT(*)
FROM "wikiticker-2015-09-12-sampled"
GROUP BY channel
ORDER BY COUNT(*) DESC
```

![image-20230713173849969](https://cdn.jsdelivr.net/gh/CJH876492153/picture@main/image-20230713173849969.png)
