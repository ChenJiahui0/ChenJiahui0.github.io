---
title: Apache Druid番外篇之压缩
subtitle: LZ4|Roaring|Concise
layout: post
published: false
author: 陈家辉
tags:
  - Druid
  - 中间件
  - 数据库
  - 算法
  - 压缩
---

# 参考文献

[LZ4 explained](http://fastcompression.blogspot.com/2011/05/lz4-explained.html)

# LZ4

压缩块是Sequence的组合，下图为一个Sequence的组成部分。

![img](http://sd-1.archive-host.com/membres/images/182754578/LZ4_format.png)

**Token**：8bit，高4位记录literal长度，低4位match长度。作为解压时候memcpy的参数。

**Literal length+**：如果token高4位为15则添加Literal length+，表示范围（0-255），可以无限加下去。

**Literals**：没有重复、首次出现的字节流，即不可压缩的部分

**Offset**：表示向前偏移n位开始出现重复。

**Match**：重复项，可以压缩的部分

**Match length+**：如果token低4位为15则添加Match length+，表示范围为（0-255），可以无限加下去。

有一些特定的解析规则需要与参考解码器兼容:

1. 最后5个字节总是Literal
2. 最后一次匹配不能在最后12个字节内开始

因此，少于13字节的文件只能表示为文本，这些规则有利于提高速度，并确保缓冲区限制永远不会被越过。

# Roaring

# Concise
