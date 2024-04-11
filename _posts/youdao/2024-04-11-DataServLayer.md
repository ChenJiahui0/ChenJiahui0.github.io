---
title: 广告系统数据服务层
subtitle: 承载日请求十亿级服务的缓存系统
layout: post
author: 陈家辉
tags:
- 缓存
- 系统设计
---

# DataServLayer是什么

## 简介

数据服务层（Data Service Layer V2）是数据上行流程的中间件，实现从数据源到上层服务的数据流和控制流。

在广告系统中，数据上行流程是指广告数据（业务数据、索引数据等展示广告时所需要用到的数据）从业务数据库等数据源处处理、装配、过滤后，向展示服务群传递和更新的数据流程。

负责广告业务百万级别缓存数据的生产，查询，提供高效，可定制化，快速扩展的缓存服务系统。

# 架构设计

## 架构图

![image-20240411100326193](https://cdn.jsdelivr.net/gh/CJH876492153/picture@main/image-20240411100326193.png)

## 模块

1. 数据管理服务（DataManager）：负责所有数据生产相关的工作，数据生产方式由DataManager内部实现决定。当前实现的生产方式包括两种：定时全量生产和接受更新通知部分生产。一个DataManager可以对应多个域生产，也可以单独生产一个域。
2. 数据缓存服务（DataCache）：负责从DataManager同步数据，并将数据再同步到需要的客户端上，作用在于降低数据管理服务负载。一个DataCache将全量保存若干个域的数据，一个域可能保存在多个DataCache上。
3. 客户端（DataReadProxy）：负责从相应的缓存中同步数据，并让上层服务方便的使用。一个客户端对应一个数据域，若上层服务需要使用多个数据域，需要配置多个客户端。
4. 元数据中心(ZooKeeper)：一个类似文件系统的服务，用来记录DataManager和DataCache各自生产和储存域的情况。/dataserv为根目录，其下有producers，hosts和domains三个子目录分别表示数据管理服务，缓存服务和数据域的配置。producers的每一个子节点表示一个DataManager服务。而每一个DataManager服务节点下保存着一些数据域，表示该数据管理服务所管理的数据域，每个数据域下有若干节点，每个节点表示一个连接当前DataManager服务当前域的缓存。hosts的每一个子节点表示一个DataCache)服务。每一个DataCache下保存着一些数据域，表示该缓存服务所应该存储的数据域，每个数据域下有若干节点，每个节点表示一个连接当前DataCache服务当前域的客户端。数据管理服务和缓存服务的信息是由DataCenter的操作来注册到ZooKeeper上的。
5. 数据中心（DataCenter）：负责数据的管理和查看，通过数据中心管理每一个数据管理服务和缓存服务需要存储的数据域，并储存数据域信息。实时数据查看功能可以查看所有在DataCenter中注册的数据域的当前数据。
6. 通知者（Noticer）：用来实时通知数据更新，将消息发送到rabbitMQ服务器上，再转发给各个注册过的dataManager。这样上层服务就可以通过调用通知者客户端来通知DataManager数据已发生改变。
7. RabbitMQ Server：第三方消息队列。通知者将消息发送给RabbitMQ server，server再转发给所有注册了实时通知功能的dataManager。

## 数据流

数据服务层V2数据流向图:

数据源（db，file，...） -> 数据管理服务（DataManager） -> 数据缓存服务（DataCache） -> 数据服务层客户端（DataReadProxy） -> 上层服务，具体使用者（展示服务，统计服务等）

# 优化

## 数据压缩

提供Gzip压缩能力，支持缓存数据压缩，减少网络带宽压力。

## Copy on write

采用Copy on write机制，修改和查询操作可并发执行。

## 默克尔树

用于校验数据完整性，客户端读取数据后，对数据做hash操作，与服务端读取到的哈希值作比较，从而保证传输数据的完整性。同时这种结构有个优点，可以提取部分数据，做完整性校验，比如B3，B4这个子树。

常用于点对点网络，例如区块链。

<img src="https://cdn.jsdelivr.net/gh/CJH876492153/picture@main/image-20240411150907358.png" alt="image-20240411150907358" style="zoom:50%;" />

## 多级缓存

上层服务使用ReadProxy从DataCache中读取数据，将一个Domain作为一个基础单元，读取到本地进行本地缓存。Domain内部数据采用Merkle Tree进行存储，Merkle Tree叶子节点负责存储数据，并且叶子结点会存储数据过期时间，程序运行时，采用懒删除和GC线程的方式做GC。

## 通知机制

支持单条数据ms级更新，上层服务发现数据被修改，可以使用Noticer将数据发送到MQ中，DataManager会重新生产该数据，并且通过长连接快速同步缓存数据。