---
title: Redis的持久化机制
categories:
  - 中间件
tags:
  - redis
abbrlink: e5adb42e
date: 2023-09-12 09:15:51
---

# Redis的持久化机制

### 引言

我们都知道，Redis 的数据存储在内存中, 一旦服务器宕机，内存中的数据将全部丢失。因此，对 Redis 来说，实现数据的持久化，避免从后端数据库中进行恢复，是至关重要的。本篇我们详细讲解下 Redis 的三种持久化机制，分别是 AOF（Append Only File） 日志和 RDB 快照 以及 混合持久化。

### AOF机制

AOF 日志是写后日志，也就是 Redis 先执行命令，然后将数据写入内存，最后才记录日志，重启时通过执行 AOF 文件中的 Redis 命令来恢复数据。如下图所示：

![](https://developer.qcloudimg.com/http-save/yehe-10418638/693b4cb76c16876336e4c5a3a06fed27.png)

类似MySql bin-log 的原理，AOF 能够解决数据持久化实时性问题，是目前 Redis 持久化机制中主流的方案。

### AOF持久化流程

AOF 持久化方案进行备份时，客户端所有请求的写命令都会被追加到 AOF 缓冲区中，缓冲区中的数据会根据 Redis 配置文件中配置的同步策略来同步到磁盘上的 AOF 文件中，追加保存每次写的操作到文件末尾。当 AOF 的文件达到重写策略配置的阈值时，Redis 会对 AOF 日志文件进行重写，给 AOF 日志文件瘦身。Redis 服务重启的时候，通过加载 AOF 日志文件来恢复数据。如下图所示：

![](https://developer.qcloudimg.com/http-save/yehe-10418638/bc15aa10e9c9d41ef356faa07b60cb82.png)

### Redis AOF 执行流程

AOF 为了避免额外的检查开销，并不会检查命令的正确性，如果先记录日志再执行命令，就有可能记录错误的命令，再通过 AOF 日志恢复数据的时候，就有可能出错，而且在执行完命令后记录日志也不会阻塞当前的写操作。但是 AOF 是存在一定的风险的，首先是如果刚执行一个命令，但是 AOF 文件中还没来得及保存就宕机了，那么这个命令和数据就会有丢失的风险，另外 AOF 虽然可以避免对当前命令的阻塞（因为是先写入再记录日志），但有可能会对下一次操作带来阻塞风险（可能存在写入磁盘较慢的情况）。这两种情况都在于 AOF 什么时候写入磁盘，针对这个问题 AOF 机制提供了三种同步策略（appendfsync 参数）。

### AOF 写入磁盘的同步策略

| 参数     | 同步策略                                                     |
| -------- | ------------------------------------------------------------ |
| Always   | 同步写入磁盘，只要有写入就会调用fsync函数；                  |
| Everysec | 每秒调用fsync函数一次，每个命令执行完，先把日志写入 AOF 文件的缓冲区，每隔一秒把缓冲区的内容写入磁盘 |
| No       | 不调用fsync，让操作系统决定何时同步磁盘。每个命令执行完，先将日志写入 AOF 文件的缓冲区，由操作系统决定何时把缓冲区的内容写入磁盘 |

### 三种同步策略的优缺点如下：

* Always：可靠性较高，数据基本不丢失，但是对性能影响较大
* Everysec：性能适中，及时宕机也只会丢失一秒的数据
* No：性能好，但发生当即情况时，

### AOF重写

我们上面说过，AOF 属于日志追加的形式来存储 Redis 的写指令，虽然有一定的写回策略，但毕竟 AOF 是通过文件的形式记录所有的写命令，但如果指令越来越多的时候，AOF 文件就会越来越大，可能会超出文件大小的限制。如果发生宕机，需要把 AOF 所有的命令重新执行，以用于故障恢复，数据过大的话这个恢复过程越漫长，也会影响 Redis 的使用。因此 Redis 提供 **重写机制**来解决这个问题。

AOF 重写的过程是通过主线程 fork 后台的 bgrewriteaof 子进程来实现的，可以避免阻塞主进导致性能下降，整个过程如下：

* AOF 每次重写，fork 过程会把主线程的内存拷贝一份 bgrewriteaof 子进程，里面包含了数据库的数据，拷贝的是父进程的页表，可以在不影响主进程的情况下逐一把拷贝的数据记入重写日志；

* 因为主线程没有阻塞，仍然可以处理新来的操作，如果这时候存在写操作，会先把操作先放入缓冲区，对于正在使用的日志，如果宕机了这个日志也是齐全的，可以用于恢复；对于正在更新的日志，也不会丢失新的操作，等到数据拷贝完成，就可以将缓冲区的数据写入到新的文件中，保证数据库的最新状态。

### RDB快照

RDB是一种快照存储持久化方式，具体就是将Redis某一时刻的内存数据保存到硬盘的文件当中，默认保存的文件名为dump.rdb，而在Redis服务器启动时，会重新加载dump.rdb文件的数据到内存当中恢复数据。

为了 RDB 数据恢复的可靠性，在进行快照的时候是全量快照，会将内存中所有的数据都记录到磁盘中，这就有可能会阻塞主线程的执行。Redis 提供了两个命令来生成 RDB 文件，分别是 **save** 和 **bgsave** ：

* **save**：执行 save 指令，阻塞 Redis 的其他操作，会导致 Redis 无法响应客户端请求，不建议使用。
* **bgsave**：执行 bgsave 指令，Redis 后台创建子进程，异步进行快照的保存操作，此时 Redis 仍然能响应客户端的请求。

### 自动间隔保存

Redis 可以设置间隔性保存，让它在“ N 秒内数据集至少有 M 个改动”这一条件被满足时，自动保存一次数据集。比如说， 以下设置会让 Redis 在满足“ 60 秒内有至少有 10 个键被改动”这一条件时，自动保存一次数据集：save 60 10

Redis 的默认配置如下，三个设置满足其一就会触发自动保存：

> save  60  10000
>
> save  900  10
>
> save  300  1

### RDB模式优点

* 相比AOF在恢复数据的时候需要一条条的执行操作命令，通过RDB文件恢复数据的效率更高；
* 同样规模的内存数据，RDB文件数据更加紧凑，磁盘空间占用更小；
* 适合全量备份内存数据场景；
* 可以根据不同的时间间隔保存RDB文件，在恢复数据的时候可以更加灵活地选择对应版本数据进行恢复

 ### RDB模式缺点

* 由于RDB数据保存存在一定的时间间隔，因此存在丢失缓存数据的风险；
* fork子进程进行RDB文件生成，由于是一次性生成一个内存快照文件，对于服务器磁盘IO以及Redis本身来说都属于重操作，可能会对服务器的磁盘IO造成压力。

### 混合使用AOF日志和RDB快照

Redis4.0 后大部分的使用场景都不会单独使用 RDB 或者 AOF 来做持久化机制，而是兼顾二者的优势混合使用。其原因是 RDB 虽然快，但是会丢失比较多的数据，不能保证数据完整性；AOF 虽然能尽可能保证数据完整性，但是性能确实是一个诟病，比如重放恢复数据。  

Redis从4.0版本开始引入 RDB-AOF 混合持久化模式，这种模式是基于 AOF 持久化模式构建而来的，混合持久化通过 **aof-use-rdb-preamble yes** 开启。这样的好处是 RDB 快照不需要很频繁的执行，可以避免频繁 fork 对主线程的影响，而且 AOF 日志也只记录两次快照期间的操作，不用记录所有操作，也不会出现文件过大的情况，避免了重写开销。

 ### 总结

本文主要分析了Redis AOF、RDB快照 以及混合持久化的内存数据持久化的机制原理，生产环境中推荐使用混合持久化，这种方式综合了RDB和AOF两种方式的优点。

