## 内容摘要

- #### 分布式系统存储特点

- #### Redis概述及使用场景

- #### Redis重要特性

- #### Redis安装

- #### 配置、启动、操作、关闭Redis

- #### Redis持久化

### 分布式系统存储特点

- 一致性
- 可用性
- 分区容错性（基本要求）

#### CAP理论：基于上述特点衍生而来

- C：Consistency,多个数据节点上的数据保持一致
- A：Availability,用户发出请求后的有限时间范围内返回结果
- P：network partition，网络发生分区后，服务是否依然可用
- CAP：一个分布式系统不可能同时满足C/A/P三个特性，最多同时满足其中两者；对于分布式系统满足分区容错性是必须的；

#### BASE理论，基于CAP衍生而来

- BA：Basically Available,基本可用
- Soft State, 软状态/柔性事务，即状态可在一个时间窗口内是不同步的
- E：Eventually consistency，最终一致性
- 分布式一致性协议：paxos，raft, zab(用于zookeeper)等等

##### NewSQL: 分布式的SQL系统

##### 分布式存储系统：分布式是指每个节点只存放整个数据中的一部分数据

#### 数据存储分布式的方式

##### 方式一：

按范围分布：一共500W数据，共5个节点，每个节点按ID号次序存放100W数据；并无实际意义

##### 方式二：

列表法: 事先定义好那些列表对应数据存放在哪些节点上；

弊端：如查找id号1~10之间的数据，需要广播各节点查找，并需重新排序

#### 方式三：

hash法：更具有随机性；

弊端：类似于方式二

### Redis概述

伴随业务增长和产品的不断完善，急速增长的数据给传统的关系型数据库存储、响应、查询等等诸多方面带来很大压力，已无法满足业务需求；需要另外一种数据存储工作模式来提高数据读写、查询等诸多方面的响应能力；NoSQL数据库为此应运而生；而NoSQL家族中的内存数据库，将经常访问的数据转移至内存中操作；众所周知，内存的数据处理速度高出磁盘好几个量级，这样，就有效提高了数据库数据处理的快速响应能力，从而缓解了对数据库的访问压力，进而改善用户体验。

Redis(Remote Dictionary Server)是一个开源的、基于key-value键值对的NoSQL数据库；与很多键值对数据库不同，Redis中的值可由string、hash、list、set、zset(有序集合)、Bitmaps(位图)、HyperLogLog、GEO(地理信息定位)等多种数据结构和算法组成，由此，它可以满足诸多应用场景；

### Redis适用场景

- ##### Redis做缓存服务器

  合理使用缓存可加快数据访问速度，同时有效降低后端数据源服务器压力；Redis提供了键值过期时间设置和灵活控制最大内存即内存溢出后的淘汰策略；

- ##### Redis做排行榜系统

  例如按热度排名的排行榜、按发布时间的排行榜等等；通过Redis内置的list和zset数据结构可方便构建各种排行榜系统；

- ##### 社交网络

  赞/踩、粉丝、共同好友/喜好、推送、下拉刷新等社交网站必备功能；由于社交网络访问量大，传统关系型数据库不适合保存这种类型数据库，Redis提供的数据结构可比较容易实现这些功能。

#### Redis不擅长做什么

- 数据规模角度

  Redis数据存放在内存中，若每天有几亿个用户行为数据，使用Redis存储，经济成本过高；

- 数据冷热角度

  热数据通常指需频繁访问操作的数据，反之为冷数据；对视频网站来讲，视频基本信息需要经常操作，而用户观看记录不一定是经常要访问的数据；因此视频信息属于热数据，用户观看记录属于冷数据；若将冷数据存放在Redis中，是对内存的一种浪费；但一些热数据存放在Redis中可以加速读写，也可减轻后端存储负载，达到事半功倍效果。

### Redis重要特性

- 速度快

  Redis执行命令的速度非常快，官方给出数字是读写性能可达到10万/秒，当然实际应用情况还要取决于机器性能

  速度快的原因

  - redis所有数据存放于内存中
  - redis是用C语言实现
  - redis使用单线程架构，预防了多线程可能产生的资源竞争问题
  - redis作者对redis源代码的精打细磨

- 基于键值对的数据结构服务器

  类似于Java中的map，Python中的dict等字典数据结构组织数据的方式叫做基于键值的方式；跟很多键值数据库不同，**Redis中的值不仅可以是字符串，也可以是具体数据结构**；这样不仅便于在诸多应用场景的开发，同时也能提高开发效率；基于诸多的数据结构，可以开发出各种逗儿比的应用；

- 丰富的功能

  除了5种关键数据结构，还提供诸多额外功能

  - 键过期功能，可用来实现缓存；
  - 发布订阅功能，可用来实现消息系统；
  - 支持Lua脚本功能，可利用Lua创造新的redis命令；
  - 简单事务功能，一定程度上保证事务特性；
  - 流水线pipeline功能，这样客户端能将一批命令一次性传到redis，减少了网络开销；

- 简单稳定

  源代码仅仅从早期的2万行增加到5万行左右，相对于很多NoSQL数据库代码量少很多，易于'吃透'它；

- 客户端语言多

  提供简单TCP通信协议，很多编程语言可以方便接入到Redis，支持如Java、PHP、Python、C、C++、Nodejs等众多客户端语言；

- 持久化

  一旦发生断电或机器故障或服务重启等等，存放在内存中的数据可能会丢失；因此Redis提供了两种持久化方式：RDB和AOF，即可以用两种策略将内存的数据保存到硬盘中，这样就保证了数据的可持久性；

- 主从复制

  Redis提供了复制功能，实现将多个相同数据的Redis副本，复制功能是分布式Redis的基础；

- 高可用和分布式

  Redis从2.8版本正式提供了高可用实现Redis Sentinel, 能够保证Redis节点的故障发现和自动转移；从3.0版本正式提供了分布式实现Redis Cluster，它是Redis真正的分布式实现，提供了高可用、读写和容量的扩展性；

### 基于Linux环境安装Redis

由于Redis的更新速度相对较快，而系统自带的诸如yum等管理工具不一定能更新到最新版本，同时redis的安装本身不是很复杂，所以一般推荐源码方式安装；下面是redis的安装流程：

```shell
#Redis单实例部署
#安装Redis前，需安装Redis的依赖程序tcl，否则在Redis执行make test时容易报错
yum -y install tcl tcl-devel
wget http://download.redis.io/releases/redis-3.2.6.tar.gz
tar zxvf redis-3.2.6.tar.gz -C /usr/local
cd /usr/local
#不把redis目录固定在指定版本上，利于redis未来版本升级，是一种安装软件的好习惯
ln -s redis-3.2.6 redis
cd redis
make 
make test
#将redis相关运行文件放到/usr/local/bin下，这样可在任何目录下执行redis命令
make install 
#比较规范的配置
mkdir /data/redis/member-6379/{log,conf,pid,data}
cp /usr/local/redis/redis.conf /data/redis/member-6379/conf
#以redis用户启动redis
useradd -s /bin/false -M redis
chown -R /data/redis
#启动redis
sudo -u redis `which redis-server` /data/redis/member-6379/conf/redis.conf
```

### 配置、启动、操作、关闭Redis

#### 启动redis

有三种方法启动redis：默认配置、运行配置、配置文件启动

1）默认配置

命令行运行`redis-server`,会打印输出一些日志信息：

- 当前的redis版本
- Redis的默认端口6379
- Redis建议使用配置文件启动
- 直接启动无法自定义配置，故这种方式不会应用于生产环境

2）运行启动

redis-server加上要修改的配置名和值(可以是多对),没有设置的配置将使用默认配置：

```shell
redis-server --configKey1 configValue1 --configKey2 configValue2
#例如，用6380作为端口启动redis
redis-server --port 6380
```

​	缺点：若需要修改的配置较多或希望将配置保存到文件中，不建议这种方式

3）配置文件启动

将配置写到指定文件里，只需执行如下命令即可启动redis:

```shell
redis-server /data/redis/member-6379/conf/redis.conf
```

#### 停止Redis服务

Redis提供了shutdown命令来停止Redis服务，例如要停掉127.0.0.1上的6379端口上的Redis服务，可执行：

```shell
redis-cli shutdown
#也可以这样停止
kill $REDIS_PID
```

停止redis服务时，相关的日志输出：

```shell
#客户端发出的shutdown命令
User requested shutdown... 
#保存RDB持久化文件
* Saving the final RDB snapshot before exiting.
#将RDB文件保存在磁盘上
* DB saved on dis
#关闭
Redis is now  ready to exit, bye bye... 
```

这里有几点需要注意：

1】Redis关闭的过程：断开于客户端的连接、持久化文件的生成，是一种相对优雅的关闭方式；

2】除了可以用`shutdown`命令关闭redis服务以外，还可以通过`kill`进程号的方式关闭掉redis，但是不要粗暴地使用`kill -9`强制杀死Redis服务，不但不会做持久化操作，还会造成缓冲区等资源不能被优雅关闭，极端情况会造成AOF和复制丢失数据的情况；

3】`shutdown`还有一个参数，代表是否在关闭Redis前，生成持久化文件：

`redis-cli shutdown nosave|save`

### 配置Redis

Redis有60多个配置，这里只给出一些重要配置

#### Redis的基础配置

| 配置名    | 配置说明                                |
| :-------- | --------------------------------------- |
| port      | 端口                                    |
| logfile   | 日志文件                                |
| dir       | Redis工作目录(存放持久化文件和日志文件) |
| daemonize | 是否以守护进程方式启动Redis             |



dir				redis工作目录（存放持久化文件和日志文件）

daemonize		是否以守护进程方式启动Redis

### Redis API的理解和使用

Redis提供了5种数据结构，理解每种数据结构特点对Redis维护很重要；同时掌握Redis的单线程命令处理机制，会使数据结构和命令的选择事半功倍；

#### Redis全局命令

Redis有5中数据结构，它们是键值对中的值，对于键来说有一些通用命令：

1）查看所有键

``keys *``

插入3对字符串类型键值对

```shell
127.0.0.1:6382> set hello world
OK
127.0.0.1:6382> set java jedis
OK
127.0.0.1:6382> set python redis-py
OK
#这里出于安全考虑，在redis配置文件中定义了keys命令别名
127.0.0.1:6382> keys *
(error) ERR unknown command 'keys'
127.0.0.1:6382> redis_keys *
1) "java"
2) "python"
3) "hello"
```

2）键总数

`dbsize`

插入一个列表类型的键值对（值由多个元素组成）：

```shell
127.0.0.1:6382> rpush mylist a b c d e f g
(integer) 7
127.0.0.1:6382> dbsize
(integer) 4
```

**dbsize命令在计算键总数时不会遍历所有键，而是直接获取Redis内置的键总数变量，所以dbsize命令的时间复杂度是O(1)；而keys命令会遍历所有键，所以它的时间复杂度是O(n)，当Redis保存了大量键时，线上环境禁止使用。**

3）检查键是否存在

`exists key`  #若键不存在返回0，存在则返回1；

4）删除键

del  key  [key ...]

del是一个普通命令，无论值是什么数据结构类型，del命令都可以将其删除；

若删除一个不存在的键，就会返回0；同时del支持删除多个键；

5）键过期

`expire  key  seconds`

Redis支持对键添加过期时间，当超过过期时间后，会自动删除键，例如为键hello设置了10秒过期时间：

```shell
127.0.0.1:6382> set hello world
OK
127.0.0.1:6382> expire hello 10
(integer) 1
```

ttl命令会返回键的剩余过期时间，它有三种返回值：

- 大于等于0的整数：键剩余的过期时间；
- -1：键没设置过期时间；
- -2：键不存在

6）键的数据结构类型

`type key`

例如键hello是字符串类型，返回结果是string。键mylist是列表类型，返回结果为list;

```shell
127.0.0.1:6382> set a b
OK
127.0.0.1:6382> type a
string
```

注：其它如字符串类命令、hash哈希类命令等等数据结构相关命令，请自行详细查阅文档手册；

### Redis持久化

Redis支持RDB和AOF两种持久化机制，持久化功能有效避免因进程退出造成的数据丢失问题，当下次重启时利用之前持久化的文件即可实现数据恢复；

#### RDB

RDB持久化时把当前进程数据生成快照保存到硬盘上的过程，触发RDB持久化的过程分为手动触发和自动触发

##### 触发机制

手动触发分别对应save和bgsave命令：

- save命令：阻塞当前redis服务器，直到RDB过程完成为止，对内存较大的实例会造成长时间阻塞，线上环境不建议使用。运行save命令对应的redis日志如下：

  `* DB saved on disk`

- bgsave命令：Redis进程执行fork操作创建子进程，RDB持久化过程由子进程负责，完成后自动结束。阻塞只发生在fork阶段，一般时间很短，运行bgsave命令对应的日志如下：

  `* Background saving started by pid 3133`

  `* DB saved on disk`

  `* RDB: 0 MB of memory used by copy-on-write`

  `Background saving terminated with success`

- bgsave命令是针对save阻塞问题做的优化。因此Redis内部所有涉及RDB的操作都采用bgsave方式，而save命令已经废弃

除了执行命令手动触发外，Redis内部还存在自动触发RDB的持久化机制，例如以下场景：

1. 使用save相关配置，如"save m n ". 表示m秒内数据集存在n次修改时，自动触发bgsave;
2. 若从节点执行全量复制操作，主节点自动执行bgsave生成RDB文件并发送给从节点；
3. 执行debug reload命令重新加载Redis时，也会自动触发save操作;
4. **默认情况下，执行shutdown命令时，若没有开启AOF持久化功能则自动执行bgsave;**

##### RDB文件的处理

- 保存

  RDB文件保存在`dir`配置指定的目录下，文件名通过dbfilename配置指定。可通过执行`config set dir {newDir}`和`config set dbfilename {new filename}` 运行期动态执行，当下次运行时RDB文件会保存到新目录

- 压缩

  Redis默认采用LZF算法对RDB文件做压缩处理，压缩后的文件远小于内存大小，默认开启，可通过参数`config set rdbcompression {yes|no}`动态修改；

##### RDB的优缺点

- RDB是一个紧凑压缩的二进制文件，代表Redis在某个时间点上的数据快照。适用于备份、全量复制等场景。如每6小时执行bgsave备份，并把RDB文件拷贝到远程机器或文件系统中(如HDFS)，用于灾难恢复
- Redis加载RDB恢复数据远远快于AOF的方式
- RDB方式数据没办法做到实时持久化/秒级持久化
- RDB文件使用特定二进制格式保存，存在老版本Redis服务无法兼容新版RDB格式问题
- 针对RDB不适合实时持久化问题，Redis提供了AOF持久化方式来解决

#### AOF

AOF（append only file）持久化：以独立日志的方式记录每次写命令，重启时再重新执行AOF文件中的命令达到恢复数据目的。**AOF的主要作用是解决了数据持久化的实时性**，目前已是Redis持久化的主流方式。

##### 使用AOF

开启AOF功能需要设置：`appendonly yes`，默认不开启；AOF文件名通过`appendfilename`配置设置，默认文件名是`appendonly.aof` 。保存路径同RDB持久化方式一致，通过dir配置指定。