---
layout: post
title: Redis集群
date: 2018-04-11
categories: blog
tags: [redis]
subtitle: 本文主要介绍redis在不同模式下的部署方式，并且对几种模式进行了一些简单的对比。
---

Redis 是一个开源（BSD许可）的，内存中的数据结构存储系统，它可以用作数据库、缓存和消息中间件。

本文主要介绍redis在不同模式下的部署方式，并且对几种模式进行了一些简单的对比。

下表列出了当前使用较多的redis部署方式:

![](/attach/20180411003.png)

通过上表比较可知：如果需要完整的分片、复制和高可用特性，在集群节点不多且在使用sentinel这种模式会带来性能瓶颈和资源消耗的情况下，可以选择使用 Redis集群；如果只需要一部分特性（比如只需要分片，但不需要复制和高可用），那么可以选择Redis Sentinel。



## 单实例模式 ##

单实例模式是指单台redis完成所有请求任务，因此复用和不具备容错性；同时在单台机器上如果只启用一个redis实例会造成资源浪费 。


## Redis集群 ##
Redis 集群是一个由多个节点组成的分布式服务器群，它具有复制、高可用和分片特性。Redis集群不需要sentinel哨兵也能完成节点移除和故障转移的功能。需要将每个节点设置成集群模式，这种集群模式没有中心节点，多个节点之间存在着网络通信的消耗。


多个节点按照分片来处理不同位置的槽，接受到不属于自己的槽操作的命令时会重新发送命令给正确的节点，这中间必然有一定的资源消耗。如同redis主从配置使用sentinel作为代理来处理请求一样。


下面三点是redis官方文档（官方文档Cluster Tutorial）中提到的redis最核心的目标：
▪  性能：这是Redis赖以生存的看家本领，增加集群功能后当然不能对性能产生太大影响，所以Redis采取了P2P而非Proxy方式、异步复制、客户端重定向等设计，而牺牲了部分的一致性、使用性。
▪  水平扩展：集群的最重要能力当然是扩展，文档中称可以线性扩展到1000节点。
▪  可用性：在Cluster推出之前，可用性要靠Sentinel保证。有了集群之后也自动具有了Sentinel的监控和自动Failover能力。


### 一、分布式 ###
Redis集群是一个由多个Redis服务器组成的分布式网络服务器群，集群中的各个服务器被称为节点（node），这些节点会相互连接并进行通信。分布式的Redis集群没有中心节点，所以用户不必担心某个节点会成为整个集群的性能瓶颈。



### 二、复制 ###
Redis 集群的每个节点都有两种角色可选，一个是主节点（master node），另一个是从节点（slavenode），其中主节点用于储存数据，而从节点则是某个主节点的复制品。当用户需要处理更多读请求的时候，可以添加从节点以扩展系统的读性能。因为Redis集群重用了单机Redis复制特性的代码，所以集群的复制行为和我们之前介绍的单机复制特性的行为是完全一样的。



###  三、节点故障检测和自动故障转移 ###
Redis 集群的主节点内置了类似Redis Sentinel的节点故障检测和自动故障转移功能，当集群中的某个主节点下线时，集群中的其他在线主节点会注意到这一点，并对已下线的主节点进行故障转移。集群进行故障转移的方法和Redis Sentinel进行故障转移的方法基本一样，不同的是，在集群里面，故障转移是由集群中其他在线的主节点负责进行的，所以集群不必另外使用Redis Sentinel 。


### 四、分片 ###
集群使用分片来扩展数据库的容量，并将命令请求的负载交给不同的节点来分担。
集群将整个数据库分为 16384 个槽（slot），所有键都属于这 16384 个槽的其中一个，计算键 key属于哪个槽的公式为 slot_number = crc16(key) % 16384 ，其中 crc16 为 16 位的循环冗余校验和函数。集群中的每个主节点都可以处理 0 个至 16384 个槽，当 16384 个槽都有某个节点在负责处理时，集群进入上线状态，并开始处理客户端发送的数据命令请求。

例如，我们有三个主节点7000、7001 和 7002，那么我们可以：
将槽0至5460指派给节点7000负责处理；
将槽 5461至 10922 指派给节点 7001 负责处理；
将槽 10923至 16383指派给节点 7002 负责处理；

这样就可以将16384个槽平均地指派给三个节点负责处理。


### 五、转向 ###
对于一个被指派了槽的主节点来说，这个主节点只会处理属于指派给自己的槽的命令请求。如果一个节点接收到了与自己处理的槽无关的命令请求，那么节点会向客户端返回一个转向错误（redirection error），告诉客户端，哪个节点负责处理这条命令，之后客户端需要根据错误中包含的地址和端口号重新向正确的节点发送命令请求。



### 六、Redis集群客户端 ###
因为集群功能比起单机功能要复杂得多，所以不同语言的 Redis 客户端通常需要为集群添加特别的支持，或者专门开发一个集群客户端。

目前主要的 Redis 集群客户端（或者说，支持集群功能的 Redis 客户端）有以下这些：
- redis-rb-cluster：antirez 使用 Ruby 编写的 Redis 集群客户端，集群客户端的官方实现。
- predis：Redis 的 PHP 客户端，支持集群功能。
- jedis：Redis 的 JAVA 客户端，支持集群功能。
- StackExchange.Redis：Redis 的 C# 客户端，支持集群功能。
-内置的 redis-cli ：在启动时给定 -c 参数即可进入集群模式，支持部分集群功能。



## Redis Sentinel集群 ##

Sentinel是一个管理redis实例的工具，它可以实现对redis的监控、通知、自动故障转移。sentinel不断地检测redis实例是否可以正常工作，通过API向其他程序报告redis的状态，如果redis master不能工作，则会自动启动故障转移进程，将其中的一个slave提升为master，其他的slave重新设置新的master服务器。


### Sentinel主要功能 ###
▪ 监控（Monitoring）：实时监控主服务器和从服务器运行状态。
▪ 提醒（Notification）：当被监控的某个Redis服务器出现问题时， Redis Sentinel可以向系统管理员发送通知，也可以通过API向其他程序发送通知。
▪ 自动故障转移（Automatic failover）：当一个主服务器不能正常工作时，Sentinel会开始一次自动故障迁移操作，它会将失效主服务器的其中一个从服务器升级为新的主服务器，并让失效主服务器的其他从服务器改为复制新的主服务器，当客户端试图连接失效的主服务器时集群也会向客户做出正确的应答。



### Redis Sentinel备份策略 ###
Redis提供两种相对有效的备份方法：RDB和AOF。


（一）RDB持久化设置
RDB是在某个时间点将内存中的所有数据的快照保存到磁盘上，在数据恢复时，可以恢复备份时间以前的所有数据，但无法恢复备份时间点后面的数据。

默认情况下Redis在磁盘上创建二进制格式的命名为dump.rdb的数据快照。可以通过配置文件配置每隔N秒且数据集上至少有M个变化时创建快照、是否对数据进行压缩、快照名称、存放快照的工作目录。


（二）AOF持久化设置
AOF是以协议文本的方式，将所有对数据库进行过写入的命令（及其参数）记录到 AOF 文件，以此达到记录数据库状态的目的。

优点是基本可以实现数据无丢失（缓存的数据有可能丢失），缺点是随着数据量的持续增加，AOF文件也会越来越大。

在保证数据安全的情况下，尽量避免因备份数据消耗过多的Redis资源，采用如下备份策略：

主实例：不采用任何备份机制。

Slave端：采用AOF（严格数据要求时可同时开启RDB），每天将AOF文件备份至备份服务器。

为了最大限度减少主实例的资源干扰，将备份相关全部迁移至Slave端完成。同时这样也有缺点，当主实例挂掉后，应用服务切换至Slave端，此时的Slave端的负载将会很大。

目前Redis不支持RDB和AOF参数动态修改，需要重启Redis生效，希望能在新的版本中实现更高效的修改方式。利用快照的持久化方式不是非常可靠，当运行Redis的计算机停止工作、意外掉电、意外杀掉了Redis进程那么最近写入Redis的数据将会丢。对于某些应用这或许不成问题，但对于持久化要求非常高的应用场景快照方式不是理想的选择。AOF文件是一个替代方案，用以最大限度的持久化数据。同样，可以通过配置文件来开闭AOF。

当主实例Redis服务崩溃（包含主机断电、进程消失等），Redis sentinel将Slave切换为读写状态，提供生产服务。通过故障诊断修复主实例，启动后会自动加入Sentinel并从Slave端完成数据同步，但不会切换。当主实例和Slave同时崩溃（如机房断电），启动服务器后，将备份服务器最新的AOF备份拷贝至主实例，启动主实例。一切完成后再启动Slave。


### 如何在Python 下使用Redis Sentinel？ ###
首先安装redis-py。
一个简单的测试代码如下，首先获得一个Sentinel对象，然后键入命令vim sentinel.py


执行后成功得到zhanyz这个值，键入python sentinel.py。
上面的master和slave都是标准的建立好连接的StrictRedis实例，slave则是sentinel查询到的第一个可用的slave。

如果正在连接的master不可用时，客户端会先抛出redis.exceptions.ConnectionError异常（此时还未开始failover），然后抛出redis.sentinel.MasterNotFoundError异常（failover进行中），在sentinel正常failover之后，实例正常。

### 如何在JAVA 下使用Redis Sentinel？ ###
使用Java操作Redis需要jedis-2.1.0.jar，如果需要使用Redis连接池的话，还需commons-pool-2.4.2.jar。JedisTemplate提供了一个template方法，负责对Jedis连接的获取与归还。
注意：
1)cluster环境下redis的slave不接受任何读写操作，比如sentinel模式下只有slave升级为主实例时才能进行读写操作。
2)client端不支持keys批量操作,不支持select dbNum操作，只有一个db:select 0，这个十分尴尬。 
3)JedisCluster 的info()等单机函数无法调用,返回(No way to dispatch this command to Redis Cluster)错误。.
4)JedisCluster 没有针对byte[]的API，需要自己扩展。
5) JedisTemplate中具体代码可以参见附件中JedisTemplate.java


版权所有 侵权必究

如需转载请联系

0571-28829811