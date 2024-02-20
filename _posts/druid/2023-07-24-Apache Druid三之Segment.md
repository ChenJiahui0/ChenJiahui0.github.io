---
title: Apache Druid三之Segment
layout: post
published: true
author: 陈家辉
tags:
  - Druid
  - 中间件
  - 数据库
---

Druid将数据和索引通过时间分块存储在segment文件里。Druid会为每个拥有数据的时间间隔建立segment，如果在一个时间间隔内通过多个不同的摄取任务摄取数据，在一个时间间隔内会产生多个segment。Druid会尝试将一个分区内的多个segment压缩为一个单一的segment，以此提高性能。

时间间隔可以用`granularitySpec`的`segmentGranularity`参数进行设置。

```json
"granularitySpec": {
  "segmentGranularity": "day",
  "queryGranularity": "none",
  "intervals": [
    "2013-08-31/2013-09-01"
  ],
  "rollup": true
}
```

为了让druid在繁重的查询负载下运行良好，segment文件大小推荐在300-700 MB 范围内。如果segment文件大于这个范围，那么可以考虑更改segment时间间隔的粒度，或者对数据进行分区，或者在 `PartitionsSpec` 中调整 `targetRowsPerSegment`。此参数推荐设置为500万。具体可以看[Batch ingestion](https://druid.apache.org/docs/latest/ingestion/hadoop.html#partitionsspec)。

# Semgent文件结构

Segment文件是柱状的：每一列数据以单独的数据结构存储。通过分开存储，Druid通过扫描需要的特定列以减少查询延迟。有三种基础的列类型，分别是：timestamp，dimensions和metrics。

![Druid column types](https://druid.apache.org/docs/latest/assets/druid-column-types.png)

Timestamp和metrics列是用 LZ4压缩的整数或浮点值数组。一旦查询确定哪些行需要选取，它会解压他们，提取相关的行，并应用所需的聚合操作符。如果查询不需要列，druid会跳过该列的数据。

Dimension列支持filter和group-by操作，所以每个dimension需要以下三个数据结构：

* Dictionary：把值映射为整形id（一般字符串），可以根据List和bitmap的值进行压缩
* List：列的值，使用dictionary编码。可以根据List做GroupBy和TopN查询。
* Bitmap：对列中的每个不同值使用一个Bitmap，以指示哪些行包含该值。位图允许快速过滤操作，因为它们方便快速应用 AND 和 OR 运算符。也叫做倒排索引。

```python
1: Dictionary
   {
    "Justin Bieber": 0,
    "Ke$ha":         1
   }

2: List of column data
   [0,
   0,
   1,
   1]

3: Bitmaps
   value="Justin Bieber": [1,1,0,0]
   value="Ke$ha":         [0,0,1,1]
```

Dictionary和List根据数据大小线性增长，Bitmap与数据大小和列数量都有关系。每个单独的列值有一个位图。具有相同值的列共享相同的位图。

对于列数据表中的每一行，只有一个位图具有非零数据。这意味着高基数列具有极其稀疏的位图，因此具有高度可压缩的位图。Druid使用特别适合位图的压缩算法来利用这一点，比如[Roaring bitmap compression](https://github.com/RoaringBitmap/RoaringBitmap)。

# 空值处理

默认情况下，Druid String类型列使用`''`或`null`。Numeric和metric列不能用`null`，转而使用null去表示`0`。但是，Druid 提供了 SQL 兼容的 null 处理模式，您可以通过 `Druid.generic.useDefaultValueForNull` 在系统级启用该模式。当设置为 false 时，允许 Druid 在摄取时创建segment，其中发生以下情况:

* String列可以区分`''`和`null`
* Numeric列可以表示`null`而不是0

在 SQL 兼容的空处理模式下，字符串维度列不包含任何其他列结构。相反，它们为空值保留了一个额外的Dictionary字segment。数值列存储在带有附加位图的segment中，其中设置的位表示空值行。

除了略微增加的segment大小之外，SQL 兼容的 null 处理可能会在查询时造成性能损失，因为需要检查 null 位图。这种性能开销只发生在实际包含空值的列上。

# 不同schema的Segment

相同数据源的segment可能有不同的schema。如果字符串列(维度)只存在于一个segment中，而不存在于另一个segment中，则涉及两个segment的查询仍然可以工作。在默认模式下，对没有维度的segment的查询表现得好像维度只包含空白值。在 SQL 兼容模式下，对没有维度的segment的查询表现得好像维度只包含空值。类似地，如果一个segment具有数字列(metric) ，而另一个segment没有，则对不具有metric的segment的查询通常按预期操作。缺少metric的聚合操作就好像这个metric不存在一样。

# 列格式

每个列存储为两部分:

* Jackson 序列化的 `ColumnDescriptor`。
* 列的二进制数据。

`ColumnDescriptor` 是内部 Druid `ColumnDescriptor` 类的 Jackson 序列化实例。它允许使用 Jackson 的多态反序列化来添加新的、有趣的序列化方法，从而使代码受到的影响最小。它由关于列的一些元数据(例如: type，是否是多值的)和一系列序列化/反序列化逻辑组成，这些逻辑可以反序列化二进制文件的其余部分。

# 多值列

多值列存储长下面这样

```python
1: Dictionary
   {
    "Justin Bieber": 0,
    "Ke$ha":         1
   }

2: List of column data
   [0,
   [0,1],  <--Row value in a multi-value column can contain an array of values
   1,
   1]

3: Bitmaps
   value="Justin Bieber": [1,1,0,0]
   value="Ke$ha":         [0,1,1,1]
                            ^
                            |
                            |
   Multi-value column contains multiple non-zero entries
```

# 压缩

在默认情况下，Druid 使用 LZ4来压缩string, long, float, and double列。druid使用 Roaring 压缩字符串列和数值空值的位图。

druid还支持Concise位图压缩。对于字符串列位图，对于高基数列，使用Roaring和Concise之间的差异最为明显。在这种情况下，Roaring 在匹配多个值的过滤器上要快得多，但是在某些情况下，由于 Roaring 格式的开销(但是在匹配多个值的情况下仍然要慢一些) ，Concise 可能占用更少的内存。可以在segment级别配置压缩，而不是为列配置压缩。
