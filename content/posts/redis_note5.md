---
title: "Redis核心技术与实战笔记5-切片集群"
date: 2020-11-08T10:03:22+08:00
draft: false
author: "fan"
subtitle: "redis"
tags: ["Redis"]
categories: ["Redis"]
toc:
  enable: true
  auto: true
---

## 为什么会有切片集群这个概念？

假如要用 Redis 保存 5000 万个键值对，每个键值对大约是 512B， 为了能快速部署并对外提供服务，我们采用云主机来运行 Redis 实例，那么，该如何选择 云主机的内存容量呢?

粗略地计算了一下，这些键值对所占的内存空间大约是 25GB(5000 万 *512B)。所 以，那我们选择选择一台 32GB 内存的云主机来部署 Redis。 这样
 32GB 的内存能保存所有数据，而且还留有 7GB，可以保证系统的正常运行。同时，我还 采用 RDB 对数据做持久化，以确保 Redis 实例故障后，还能从 RDB 恢复数据。

 但是实际这样使用的时候就会发现：Redis 的响应有时会非常慢。并且查看查看 Redis 的 `latest_fork_usec` 指标值 也会发现这个指标值会比较高

 就像之前博客中整理说的，其实这里是因为使用RDB做持久化导致的，在使用 RDB 进行持久化时，Redis 会 fork 子进程来完 成，fork 操作的用时和 Redis 的数据量是正相关的，而 fork 在执行时会阻塞主线程。数 据量越大，fork 操作造成的主线程阻塞的时间越长。所以，在使用 RDB 对 25GB 的数据 进行持久化时，数据量较大，后台运行的子进程在 fork 创建时阻塞了主线程，于是就导致 Redis 响应变慢了。


 这个时候就需要切片集群了，也叫分片集群。就是指启动多个 Redis 实例组成一个集群，然后按照一定的规 则，把收到的数据划分成多份，每一份用一个实例来保存。

 这个时候我们再看上面的问题，其实可以把25GB 的数据平均分成 5 份(当然，也可以不做均分)，使用 5 个实例来保存，每个实 例只需要保存 5GB 数据。

 [![BIHkcD.png](https://s1.ax1x.com/2020/11/08/BIHkcD.png)](https://imgchr.com/i/BIHkcD)

这样实例为5GB数据生成RDB，fork子进程的时候一般就不会刚给主线程带来较长时间的阻塞。

## 如何保存更多数据

对于保存大量数据的时候，主要就是两种方案：纵向扩展和横向扩展

- 纵向扩展:升级单个 Redis 实例的资源配置，包括增加内存容量、增加磁盘容量、使用 更高配置的 CPU。
- 横向扩展:横向增加当前 Redis 实例的个数，如原来使用 1 个 8GB 内存、 50GB 磁盘的实例，现在使用三个相同配置的实例。

## 数据切片和实例的对应分布关系
在切片集群中，数据需要分布在不同实例上，数据和实例之间是如何对应的。
从 3.0 开始，官方提供了一个 名为 Redis Cluster 的方案，用于实现切片集群。

Redis Cluster 方案采用哈希槽，来处理数据和实例之间的映射关系。在 Redis Cluster 方案中，一个切片集群共有 16384 个哈希槽，这些哈希槽类似于数据分区，每个键值对都会根据它的 key，被映射到一个哈 希槽中。

具体的映射过程分为两大步:首先根据键值对的 key，按照`CRC16` 算法计算一个 16 bit 的值;然后，再用这个 16bit 值对 16384 取模，得到 0~16383 范围内的模数，每个模数
代表一个相应编号的哈希槽。

哈希槽又是如何被映射到具体的 Redis 实例上?
在部署 Redis Cluster 方案时，可以使用 cluster create 命令创建集群，此时，Redis 会自动把这些槽平均分布在集群实例上。
我们也可以使用 cluster meet 命令手动建立实例间的连接，形成集群，再使用 cluster addslots 命令，指定每个实例上的哈希槽个数。

集群中不同 Redis 实例的内存大小配置不一，如果把哈希槽均分在各个实 例上，在保存相同数量的键值对时，和内存大的实例相比，内存小的实例就会有更大的容 量压力。遇到这种情况时，你可以根据不同实例的资源配置情况，使用 cluster addslots 命令手动分配哈希槽。


**注意：在手动分配哈希槽时，需要把 16384 个槽都分配完，否则 Redis 集群无法正常工作。**


## 客户端如何定位数据?

一般来说，客户端和集群实例建立连接后，实例就会把哈希槽的分配信息发给客户端。但 是，在集群刚刚创建的时候，每个实例只知道自己被分配了哪些哈希槽，是不知道其他实 例拥有的哈希槽信息的。

客户端为什么可以在访问任何一个实例时，都能获得所有的哈希槽信息呢?这是因 为，Redis 实例会把自己的哈希槽信息发给和它相连接的其它实例，来完成哈希槽分配信 息的扩散。当实例之间相互连接后，每个实例就有所有哈希槽的映射关系了。


客户端收到哈希槽信息后，会把哈希槽信息缓存在本地。当客户端请求键值对时，会先计 算键所对应的哈希槽，然后就可以给相应的实例发送请求了。

但是，在集群中，实例和哈希槽的对应关系并不是一成不变的，最常见的变化有两个:

- 在集群中，实例有新增或删除，Redis 需要重新分配哈希槽; 
- 为了负载均衡，Redis 需要把哈希槽在所有实例上重新分布一遍。

此时，实例之间还可以通过相互传递消息，获得最新的哈希槽分配信息，但是，客户端是 无法主动感知这些变化的。这就会导致，它缓存的分配信息和最新的分配信息就不一致 了，那该怎么办呢?

Redis Cluster 方案提供了一种重定向机制，所谓的“重定向”，就是指，客户端给一个实 例发送数据读写操作时，这个实例上并没有相应的数据，客户端要再给一个新实例发送操 作命令。

那客户端又是怎么知道重定向时的新实例的访问地址呢?当客户端把一个键值对的操作请 求发给一个实例时，如果这个实例上并没有这个键值对映射的哈希槽，那么，这个实例就 会给客户端返回下面的 MOVED 命令响应结果，这个结果中就包含了新实例的访问地址。
```
1 GET hello:key
2 (error) MOVED 13320 172.16.19.5:6379
```
其中，MOVED 命令表示，客户端请求的键值对所在的哈希槽 13320，实际是在 172.16.19.5 这个实例上。通过返回的 MOVED 命令，就相当于把哈希槽所在的新实例的 信息告诉给客户端了。这样一来，客户端就可以直接和 172.16.19.5 连接，并发送操作请 求了。

实际中需要注意：如果实例数据较多的时候，数据从一个实例迁移到另外一个实例的时间内可能稍微有点长，假如在这种迁移部分完成的情况下，客户端请求会收到一个ASK报错信息。

```
1 GET hello:key
2 (error) ASK 13320 172.16.19.5:6379
```

这个结果中的 ASK 命令就表示，客户端请求的键值对所在的哈希槽 13320，在 172.16.19.5 这个实例上，但是这个哈希槽正在迁移。此时，客户端需要先给 172.16.19.5 这个实例发送一个 ASKING 命令。这个命令的意思是，让这个实例允许执行客户端接下来 发送的命令。然后，客户端再向这个实例发送 GET 命令，以读取数据。

和 MOVED 命令不同，ASK 命令并不会更新客户端缓存的哈希槽分配信息。这也就是说， ASK 命令的作用只是让客户端能给新实例发送一次请求，而不像 MOVED 命令那样，会更 改本地缓存，让后续所有命令都发往新实例。


## 面试题

Redis Cluster 方案通过哈希槽的方式把键值对分配到不同 的实例上，这个过程需要对键值对的 key 做 CRC 计算，然后再和哈希槽做映射，这样做有 什么好处吗?如果用一个表直接把键值对和实例的对应关系记录下来(例如键值对 1 在实 例 2 上，键值对 2 在实例 1 上)，这样就不用计算 key 和哈希槽的对应关系了，只用查表 就行了，Redis 为什么不这么做呢?

1、整个集群存储key的数量是无法预估的，key的数量非常多时，直接记录每个key对应的实例映射关系，这个映射表会非常庞大，这个映射表无论是存储在服务端还是客户端都占用了非常大的内存空间。

2、Redis Cluster采用无中心化的模式（无proxy，客户端与服务端直连），客户端在某个节点访问一个key，如果这个key不在这个节点上，这个节点需要有纠正客户端路由到正确节点的能力（MOVED响应），这就需要节点之间互相交换路由表，每个节点拥有整个集群完整的路由关系。如果存储的都是key与实例的对应关系，节点之间交换信息也会变得非常庞大，消耗过多的网络资源，而且就算交换完成，相当于每个节点都需要额外存储其他节点的路由表，内存占用过大造成资源浪费。

3、当集群在扩容、缩容、数据均衡时，节点之间会发生数据迁移，迁移时需要修改每个key的映射关系，维护成本高。

4、而在中间增加一层哈希槽，可以把数据和节点解耦，key通过Hash计算，只需要关心映射到了哪个哈希槽，然后再通过哈希槽和节点的映射表找到节点，相当于消耗了很少的CPU资源，不但让数据分布更均匀，还可以让这个映射表变得很小，利于客户端和服务端保存，节点之间交换信息时也变得轻量。

5、当集群在扩容、缩容、数据均衡时，节点之间的操作例如数据迁移，都以哈希槽为基本单位进行操作，简化了节点扩容、缩容的难度，便于集群的维护和管理。