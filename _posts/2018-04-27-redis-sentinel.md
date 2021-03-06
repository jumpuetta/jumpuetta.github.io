---
layout: post
title: Redis高可用-哨兵模式
date: 2018-04-27
categories: blog
tags: [redis]
subtitle: 
---
在同一主机上部署redis主从哨兵模式

## 一、Redis主从配置 ##

端口：7000~7003，一主三从：


7000(主)
```
port 7000
bind 127.0.0.1
daemonize yes
pidfile "./pid/redis_7000.pid"
appendonly yes
dir "/usr/local/redis"

```

7001(从)
```
port 7001
bind 127.0.0.1
daemonize yes
pidfile "./pid/redis_7001.pid"
appendonly yes
dir "/usr/local/redis"
slaveof 127.0.0.1 7000

```

7002(从)
```
port 7002
bind 127.0.0.1
daemonize yes
pidfile "./pid/redis_7002.pid"
appendonly yes
dir "/usr/local/redis"
slaveof 127.0.0.1 7000

```

7003(从)
```
port 7003
bind 127.0.0.1
daemonize yes
pidfile "./pid/redis_7003.pid"
appendonly yes
dir "/usr/local/redis"
slaveof 127.0.0.1 7000

```

## 二、sentinel集群配置 ##
三个sentinel节点：7004~7006
7004
```
port 7004
dir "/usr/local/redis"
sentinel monitor mymaster 127.0.0.1 7000 2
daemonize yes

```

7005
```
port 7005
dir "/usr/local/redis"
sentinel monitor mymaster 127.0.0.1 7000 2
daemonize yes

```

7006
```
port 7006
dir "/usr/local/redis"
sentinel monitor mymaster 127.0.0.1 7000 2
daemonize yes

```

## 三、启动、停止集群 ##
```
#redis主从启动
./bin/redis-server conf/7000.conf
./bin/redis-server conf/7001.conf
./bin/redis-server conf/7002.conf
./bin/redis-server conf/7003.conf
#sentinel哨兵集群启动
./bin/redis-sentinel conf/sentinel-7004.conf
./bin/redis-sentinel conf/sentinel-7005.conf
./bin/redis-sentinel conf/sentinel-7006.conf
#停止sentinel
./bin/redis-cli -p 7000 shutdown
./bin/redis-cli -p 7001 shutdown
./bin/redis-cli -p 7002 shutdown
./bin/redis-cli -p 7003 shutdown
./bin/redis-cli -p 7004 shutdown
./bin/redis-cli -p 7005 shutdown
./bin/redis-cli -p 7006 shutdown
```

## 四、测试集群 ##
主从测试
```
[root@centos redis]# ./bin/redis-cli -p 7000 -h 127.0.0.1
127.0.0.1:7000> set name jiangpeng
OK
127.0.0.1:7000> get name
"jiangpeng"
127.0.0.1:7000> info Replication 
# Replication
role:master
connected_slaves:3
slave0:ip=127.0.0.1,port=7003,state=online,offset=273683,lag=0
slave1:ip=127.0.0.1,port=7001,state=online,offset=273683,lag=0
slave2:ip=127.0.0.1,port=7002,state=online,offset=273419,lag=1
master_repl_offset:273683
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:10892
repl_backlog_histlen:262792


[root@centos redis]# ./bin/redis-cli -p 7001 -h 127.0.0.1
127.0.0.1:7001> get name
"jiangpeng"
127.0.0.1:7001> set age 25
(error) READONLY You can't write against a read only slave.
127.0.0.1:7001> 
#从节点只能读取不能写入
```

哨兵集群测试

查看所有被监控的主节点
```
[root@centos redis]# ./bin/redis-cli -p 7004 -h 127.0.0.1
127.0.0.1:7004> sentinel masters
1)  1) "name"
    2) "mymaster"
    3) "ip"
    4) "127.0.0.1"
    5) "port"
    6) "7000"
    7) "runid"
    8) "9fcf5b0c87cd67b4434efbc48ef8d7f9431c855e"
    9) "flags"
   10) "master"
 ......
```
查看主节点下的所有从节点
```
127.0.0.1:7004> sentinel slaves mymaster
1)  1) "name"
    2) "127.0.0.1:7001"
    3) "ip"
    4) "127.0.0.1"
    5) "port"
    6) "7001"
    7) "runid"
    8) "6eb94ff0da077fc907d15e2c965833d1625b7cbf"
    9) "flags"
   10) "slave"
......
2)  1) "name"
    2) "127.0.0.1:7002"
    3) "ip"
    4) "127.0.0.1"
    5) "port"
    6) "7002"
    7) "runid"
    8) "965efc82c2a87317635b84877db4b4633c10beab"
    9) "flags"
   10) "slave"
......
3)  1) "name"
    2) "127.0.0.1:7003"
    3) "ip"
    4) "127.0.0.1"
    5) "port"
    6) "7003"
    7) "runid"
    8) "e41f0d7fa6f2e42435acd5de94902e2b676669ef"
    9) "flags"
   10) "slave"
......

```

## 五、sentinel管理命令 ##
sentinel支持的合法命令如下：
```
PING sentinel回复PONG.

SENTINEL masters 显示被监控的所有master以及它们的状态.

SENTINEL master <master name> 显示指定master的信息和状态；

SENTINEL slaves <master name> 显示指定master的所有slave以及它们的状态；

SENTINEL get-master-addr-by-name <master name> 返回指定master的ip和端口，如果正在进行failover或者failover已经完成，将会显示被提升为master的slave的ip和端口。

SENTINEL reset <pattern> 重置名字匹配该正则表达式的所有的master的状态信息，清楚其之前的状态信息，以及slaves信息。

SENTINEL failover <master name> 强制sentinel执行failover，并且不需要得到其他sentinel的同意。但是failover后会将最新的配置发送给其他sentinel。
```

## 六、Sentinel和Redis身份验证 ##
当一个master配置为需要密码才能连接时，客户端和slave在连接时都需要提供密码。

master通过requirepass设置自身的密码，不提供密码无法连接到这个master。
slave通过masterauth来设置访问master时的密码。

但是当使用了sentinel时，由于一个master可能会变成一个slave，一个slave也可能会变成master，所以需要同时设置上述两个配置项。

## 七、Slave选举与优先级 ##
当一个sentinel准备好了要进行failover，并且收到了其他sentinel的授权，那么就需要选举出一个合适的slave来做为新的master。

**slave的选举主要会评估slave的以下几个方面：**

> 与master断开连接的次数   
> Slave的优先级   
> 数据复制的下标(用来评估slave当前拥有多少master的数据)   
> 进程ID   

如果一个slave与master失去联系超过10次，并且每次都超过了配置的最大失联时间(down-after-milliseconds option)，并且，如果sentinel在进行failover时发现slave失联，那么这个slave就会被sentinel认为不适合用来做新master的。

更严格的定义是，如果一个slave持续断开连接的时间超过

(down-after-milliseconds * 10) + milliseconds_since_master_is_in_SDOWN_state
就会被认为失去选举资格。
符合上述条件的slave才会被列入master候选人列表，并根据以下顺序来进行排序：

>1.sentinel首先会根据slaves的优先级来进行排序，优先级越小排名越靠前（？）。   
>2.如果优先级相同，则查看复制的下标，哪个从master接收的复制数据多，哪个就靠前。    
>3.如果优先级和下标都相同，就选择进程ID较小的那个。    

一个redis无论是master还是slave，都必须在配置中指定一个slave优先级。要注意到master也是有可能通过failover变成slave的。如果一个redis的slave优先级配置为0，那么它将永远不会被选为master。但是它依然会从master哪里复制数据。

补充一点：还可以向任意sentinel发生sentinel failover <masterName> 进行手动故障转移，这样就不需要经过上述主客观和选举的过程。

## 八、Sentinel支持集群 ##
很显然，只使用单个sentinel进程来监控redis集群是不可靠的，当sentinel进程宕掉后(sentinel本身也有单点问题，single-point-of-failure)整个集群系统将无法按照预期的方式运行。所以有必要将sentinel集群，这样有几个好处：

>即使有一些sentinel进程宕掉了，依然可以进行redis集群的主备切换；  
>如果只有一个sentinel进程，如果这个进程运行出错，或者是网络堵塞，那么将无法实现redis集群的主备切换（单点问题）;  
>如果有多个sentinel，redis的客户端可以随意地连接任意一个sentinel来获得关于redis集群中的信息。   

## 九、Sentinel的“仲裁会” ##

前面我们谈到，当一个master被sentinel集群监控时，需要为它指定一个参数，这个参数指定了当需要判决master为不可用，并且进行failover时，所需要的sentinel数量，本文中我们暂时称这个参数为票数

不过，当failover主备切换真正被触发后，failover并不会马上进行，还需要sentinel中的大多数sentinel授权后才可以进行failover。
当ODOWN时，failover被触发。failover一旦被触发，尝试去进行failover的sentinel会去获得“大多数”sentinel的授权（如果票数比大多数还要大的时候，则询问更多的sentinel)
这个区别看起来很微妙，但是很容易理解和使用。例如，集群中有5个sentinel，票数被设置为2，当2个sentinel认为一个master已经不可用了以后，将会触发failover，但是，进行failover的那个sentinel必须先获得至少3个sentinel的授权才可以实行failover。
如果票数被设置为5，要达到ODOWN状态，必须所有5个sentinel都主观认为master为不可用，要进行failover，那么得获得所有5个sentinel的授权。

sentinel is-master-down-by-addr这个命令有两个作用，一是确认下线判定，二是进行领导者选举。

**选举过程：**

>1.每个做主观下线的sentinel节点向其他sentinel节点发送上面那条命令，要求将它设置为领导者。    
>2.收到命令的sentinel节点如果还没有同意过其他的sentinel发送的命令（还未投过票），那么就会同意，否则拒绝。   
>3.如果该sentinel节点发现自己的票数已经过半且达到了quorum的值，就会成为领导者     
>4.如果这个过程出现多个sentinel成为领导者，则会等待一段时间重新选举。   


## 十、三个定时任务 ##
sentinel在内部有3个定时任务

**1.每10秒每个sentinel会对master和slave执行info命令**

这个任务达到两个目的：

1.发现slave节点

2.确认主从关系

**2.每2秒每个sentinel通过master节点的channel交换信息（pub/sub）**

master节点上有一个发布订阅的频道(__sentinel__:hello)。

sentinel节点通过__sentinel__:hello频道进行信息交换(对节点的"看法"和自身的信息)，达成共识。

**3.每1秒每个sentinel对其他sentinel和redis节点执行ping操作（相互监控）**

这个其实是一个心跳检测，是失败判定的依据。


## 十一、配置版本号 ##
为什么要先获得大多数sentinel的认可时才能真正去执行failover呢？

当一个sentinel被授权后，它将会获得宕掉的master的一份最新配置版本号，当failover执行结束以后，这个版本号将会被用于最新的配置。因为大多数sentinel都已经知道该版本号已经被要执行failover的sentinel拿走了，所以其他的sentinel都不能再去使用这个版本号。这意味着，每次failover都会附带有一个独一无二的版本号。我们将会看到这样做的重要性。

而且，sentinel集群都遵守一个规则：如果sentinel A推荐sentinel B去执行failover，A会等待一段时间后，自行再次去对同一个master执行failover，这个等待的时间是通过failover-timeout配置项去配置的。从这个规则可以看出，sentinel集群中的sentinel不会再同一时刻并发去failover同一个master，第一个进行failover的sentinel如果失败了，另外一个将会在一定时间内进行重新进行failover，以此类推。

redis sentinel保证了活跃性：如果大多数sentinel能够互相通信，最终将会有一个被授权去进行failover.
redis sentinel也保证了安全性：每个试图去failover同一个master的sentinel都会得到一个独一无二的版本号。

## 十二、配置传播 ##

一旦一个sentinel成功地对一个master进行了failover，它将会把关于master的最新配置通过广播形式通知其它sentinel，其它的sentinel则更新对应master的配置。

一个faiover要想被成功实行，sentinel必须能够向选为master的slave发送SLAVE OF NO ONE命令，然后能够通过INFO命令看到新master的配置信息。

当将一个slave选举为master并发送SLAVE OF NO ONE后，即使其它的slave还没针对新master重新配置自己，failover也被认为是成功了的，然后所有sentinels将会发布新的配置信息。

新配在集群中相互传播的方式，就是为什么我们需要当一个sentinel进行failover时必须被授权一个版本号的原因。

每个sentinel使用##发布/订阅##的方式持续地传播master的配置版本信息，配置传播的##发布/订阅##管道是：__sentinel__:hello。

因为每一个配置都有一个版本号，所以以版本号最大的那个为标准。

举个栗子：假设有一个名为mymaster的地址为192.168.1.50:6379。一开始，集群中所有的sentinel都知道这个地址，于是为mymaster的配置打上版本号1。一段时候后mymaster死了，有一个sentinel被授权用版本号2对其进行failover。如果failover成功了，假设地址改为了192.168.1.50:9000，此时配置的版本号为2，进行failover的sentinel会将新配置广播给其他的sentinel，由于其他sentinel维护的版本号为1，发现新配置的版本号为2时，版本号变大了，说明配置更新了，于是就会采用最新的版本号为2的配置。

这意味着sentinel集群保证了第二种活跃性：一个能够互相通信的sentinel集群最终会采用版本号最高且相同的配置。

## 十三、主观下线和客观下线 ##

sentinel对于不可用有两种不同的看法，一个叫主观不可用(SDOWN),另外一个叫客观不可用(ODOWN)。SDOWN是sentinel自己主观上检测到的关于master的状态，ODOWN需要一定数量的sentinel达成一致意见才能认为一个master客观上已经宕掉，各个sentinel之间通过命令SENTINEL is_master_down_by_addr来获得其它sentinel对master的检测结果。

从sentinel的角度来看，如果发送了PING心跳后，在一定时间内没有收到合法的回复，就达到了SDOWN的条件。这个时间在配置中通过is-master-down-after-milliseconds参数配置。

当sentinel发送PING后，以下回复之一都被认为是合法的：

>PING replied with +PONG.   
>PING replied with -LOADING error.    
>PING replied with -MASTERDOWN error.    
其它任何回复（或者根本没有回复）都是不合法的。

从SDOWN切换到ODOWN不需要任何一致性算法，只需要一个gossip协议：如果一个sentinel收到了足够多的sentinel发来消息告诉它某个master已经down掉了，SDOWN状态就会变成ODOWN状态。如果之后master可用了，这个状态就会相应地被清理掉。

正如之前已经解释过了，真正进行failover需要一个授权的过程，但是所有的failover都开始于一个ODOWN状态。

ODOWN状态只适用于master，对于不是master的redis节点sentinel之间不需要任何协商，slaves和sentinel不会有ODOWN状态。

在redis-sentinel的conf文件里有这么两个配置：

**1.sentinel monitor <masterName> <ip> <port> <quorum>**

四个参数含义：

masterName这个是对某个master+slave组合的一个区分标识（一套sentinel是可以监听多套master+slave这样的组合的）。

ip 和 port 就是master节点的 ip 和 端口号。

quorum这个参数是进行客观下线的一个依据，意思是至少有 quorum 个sentinel主观的认为这个master有故障，才会对这个master进行下线以及故障转移。因为有的时候，某个sentinel节点可能因为自身网络原因，导致无法连接master，而此时master并没有出现故障，所以这就需要多个sentinel都一致认为该master有问题，才可以进行下一步操作，这就保证了公平性和高可用。

**2.sentinel down-after-milliseconds <masterName> <timeout> **

这个配置其实就是进行主观下线的一个依据，masterName这个参数不用说了，timeout是一个毫秒值，表示：如果这台sentinel超过timeout这个时间都无法连通master包括slave（slave不需要客观下线，因为不需要故障转移）的话，就会主观认为该master已经下线（实际下线需要客观下线的判断通过才会下线）

那么，多个sentinel之间是如何达到共识的呢？

这就是依赖于前面说的第二个定时任务，某个sentinel先将master节点进行一个主观下线，然后会将这个判定通过sentinel is-master-down-by-addr这个命令问对应的节点是否也同样认为该addr的master节点要做客观下线。最后当达成这一共识的sentinel个数达到前面说的quorum设置的这个值时，就会对该master节点下线进行故障转移。quorum的值一般设置为sentinel个数的二分之一加1，例如3个sentinel就设置2


## 十四、Sentinel之间和Slaves之间的自动发现机制 ##

虽然sentinel集群中各个sentinel都互相连接彼此来检查对方的可用性以及互相发送消息。但是你不用在任何一个sentinel配置任何其它的sentinel的节点。因为sentinel利用了master的发布/订阅机制去自动发现其它也监控了统一master的sentinel节点。

通过向名为__sentinel__:hello的管道中发送消息来实现。

同样，你也不需要在sentinel中配置某个master的所有slave的地址，sentinel会通过询问master来得到这些slave的地址的。

每个sentinel通过向每个master和slave的发布/订阅频道__sentinel__:hello每秒发送一次消息，来宣布它的存在。
每个sentinel也订阅了每个master和slave的频道__sentinel__:hello的内容，来发现未知的sentinel，当检测到了新的sentinel，则将其加入到自身维护的master监控列表中。
每个sentinel发送的消息中也包含了其当前维护的最新的master配置。如果某个sentinel发现
自己的配置版本低于接收到的配置版本，则会用新的配置更新自己的master配置。

在为一个master添加一个新的sentinel前，sentinel总是检查是否已经有sentinel与新的sentinel的进程号或者是地址是一样的。如果是那样，这个sentinel将会被删除，而把新的sentinel添加上去。

## 十五、网络隔离时的一致性 ##

redis sentinel集群的配置的一致性模型为最终一致性，集群中每个sentinel最终都会采用最高版本的配置。然而，在实际的应用环境中，有三个不同的角色会与sentinel打交道：

Redis实例.
Sentinel实例.
客户端.
为了考察整个系统的行为我们必须同时考虑到这三个角色。

下面有个简单的例子，有三个主机，每个主机分别运行一个redis和一个sentinel:
```
             +-------------+
             | Sentinel 1  | <--- Client A
             | Redis 1 (M) |
             +-------------+
                     |
                     |
 +-------------+     |                     +------------+
 | Sentinel 2  |-----+-- / partition / ----| Sentinel 3 | <--- Client B
 | Redis 2 (S) |                           | Redis 3 (M)|
 +-------------+                           +------------+
```
在这个系统中，初始状态下redis3是master, redis1和redis2是slave。之后redis3所在的主机网络不可用了，sentinel1和sentinel2启动了failover并把redis1选举为master。

Sentinel集群的特性保证了sentinel1和sentinel2得到了关于master的最新配置。但是sentinel3依然持着的是就的配置，因为它与外界隔离了。

当网络恢复以后，我们知道sentinel3将会更新它的配置。但是，如果客户端所连接的master被网络隔离，会发生什么呢？

客户端将依然可以向redis3写数据，但是当网络恢复后，redis3就会变成redis的一个slave，那么，在网络隔离期间，客户端向redis3写的数据将会丢失。

也许你不会希望这个场景发生：

如果你把redis当做缓存来使用，那么你也许能容忍这部分数据的丢失。
但如果你把redis当做一个存储系统来使用，你也许就无法容忍这部分数据的丢失了。
因为redis采用的是异步复制，在这样的场景下，没有办法避免数据的丢失。然而，你可以通过以下配置来配置redis3和redis1，使得数据不会丢失。
```
min-slaves-to-write 1    
min-slaves-max-lag 10   
```
通过上面的配置，当一个redis是master时，如果它不能向至少一个slave写数据(上面的min-slaves-to-write指定了slave的数量)，它将会拒绝接受客户端的写请求。由于复制是异步的，master无法向slave写数据意味着slave要么断开连接了，要么不在指定时间内向master发送同步数据的请求了(上面的min-slaves-max-lag指定了这个时间)。

## 十六、Sentinel状态持久化 ##
snetinel的状态会被持久化地写入sentinel的配置文件中。每次当收到一个新的配置时，或者新创建一个配置时，配置会被持久化到硬盘中，并带上配置的版本戳。这意味着，可以安全的停止和重启sentinel进程。

## 十七、动态修改Sentinel配置 ## 
从redis2.8.4开始，sentinel提供了一组API用来添加，删除，修改master的配置。

需要注意的是，如果你通过API修改了一个sentinel的配置，sentinel不会把修改的配置告诉其他sentinel。你需要自己手动地对多个sentinel发送修改配置的命令。

以下是一些修改sentinel配置的命令：

>SENTINEL MONITOR <name> <ip> <port> <quorum> 这个命令告诉sentinel去监听一个新的master    
>SENTINEL REMOVE <name> 命令sentinel放弃对某个master的监听    
>SENTINEL SET <name> <option> <value> 这个命令很像Redis的CONFIG SET命令，用来改变指定master的配置。支持多个<option><value>。例如以下实例：    
>SENTINEL SET objects-cache-master down-after-milliseconds 1000    

只要是配置文件中存在的配置项，都可以用SENTINEL SET命令来设置。这个还可以用来设置master的属性，比如说quorum(票数)，而不需要先删除master，再重新添加master。例如：

>SENTINEL SET objects-cache-master quorum 5

## 十八、增加或删除Sentinel ## 
由于有sentinel自动发现机制，所以添加一个sentinel到你的集群中非常容易，你所需要做的只是监控到某个Master上，然后新添加的sentinel就能获得其他sentinel的信息以及masterd所有的slave。

如果你需要添加多个sentinel，建议你一个接着一个添加，这样可以预防网络隔离带来的问题。你可以每个30秒添加一个sentinel。最后你可以用SENTINEL MASTER mastername来检查一下是否所有的sentinel都已经监控到了master。

删除一个sentinel显得有点复杂：因为sentinel永远不会删除一个已经存在过的sentinel，即使它已经与组织失去联系很久了。
要想删除一个sentinel，应该遵循如下步骤：

> 停止所要删除的sentinel    
> 发送一个SENTINEL RESET * 命令给所有其它的sentinel实例，如果你想要重置指定master上面的sentinel，只需要把*号改为特定的名字，注意，需要一个接一个发，每次发送的间隔不低于30秒。    
> 检查一下所有的sentinels是否都有一致的当前sentinel数。使用SENTINEL MASTER mastername 来查询。    

## 十九、删除旧master或者不可达slave ## 
sentinel永远会记录好一个Master的slaves，即使slave已经与组织失联好久了。这是很有用的，因为sentinel集群必须有能力把一个恢复可用的slave进行重新配置。

并且，failover后，失效的master将会被标记为新master的一个slave，这样的话，当它变得可用时，就会从新master上复制数据。

然后，有时候你想要永久地删除掉一个slave(有可能它曾经是个master)，你只需要发送一个SENTINEL RESET master命令给所有的sentinels，它们将会更新列表里能够正确地复制master数据的slave。


## 二十、redis sentinel客户端实现原理 ##
在介绍 java api之前我们先看看redis sentinel客户端实现的一个原理（4个步骤）

**1.首先要获取所有sentinel节点的集合，获取一个可用节点，同时需要一个对应的masterName**   
**2.通过sentinel的一个api：get-master-addr-by-name-masterName(通过名字获取地址)，sentinel会返回master的真正地址和端口**    
**3.当客户端获取到master信息后，会通过role replication来进行一个验证是否是真正的master节点**     
**4.最后一步就是当sentinel感知到master的变化会通知客户端更换节点，其实内部是用的一个发布订阅模式（客户端订阅sentinel的某一个频道，当master发生变化，sentinel向这个频道publish一条消息，客户端就可以获取再对新的master进行一个连接）**    

客户端接入流程  
>1.需要一个sentinel地址的集合    
>2.需要masterName   
>3.不是代理模式（不是每次都需要去连接sentinel节点去获取master信息，这样效率很差，而是采用通知的形式）    


**使用Jedis访问sentinel**


```java
//初始化Sentinel连接池，注意：这里名字是JedisSentinelPool只是为了区分它是sentinel方式连接，其内部还是连接master
JedisSentinelPool sentinelPool = new JedisSentinelPool(masterName,SentinelSet,poolConfig,timeout);

Jedis jedis = null;
try{
  jedis = sentinelPool.getResource();
  //这里执行jedis command
}catch(Exception e){
  logger.error(e.getMessage(),e);
}finally{
  if(jedis != null)
     //归还连接
     jedis.close();
}
```


## 二十一、TILT 模式 ##
redis sentinel非常依赖系统时间，例如它会使用系统时间来判断一个PING回复用了多久的时间。
然而，假如系统时间被修改了，或者是系统十分繁忙，或者是进程堵塞了，sentinel可能会出现运行不正常的情况。
当系统的稳定性下降时，TILT模式是sentinel可以进入的一种的保护模式。当进入TILT模式时，sentinel会继续监控工作，但是它不会有任何其他动作，它也不会去回应is-master-down-by-addr这样的命令了，因为它在TILT模式下，检测失效节点的能力已经变得让人不可信任了。
如果系统恢复正常，持续30秒钟，sentinel就会退出TITL模式。
