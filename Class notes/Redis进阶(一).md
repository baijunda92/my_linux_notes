## 内容摘要

- #### Redis持久化实践及灾难恢复模拟

- #### redis主从技术应用

- #### 基于Redis Sentinel实现Redis集群

### Redis持久化实践及灾难恢复模拟

#### 一、redis持久化知识点回顾

目前Redis持久化的方式有两种：RDB和AOF

首先，应该明确持久化的数据有什么用，答案是用于重启后的数据恢复；

Redis是一个内存数据库，无论是RDB还是AOF，都只是保证数据恢复的措施；

所以，**Redis在利用RDB和AOF进行恢复的时候，都会读取RDB和AOF文件，重新加载到内存中**；

RDB就是Snapshot快照存储，是默认的持久化方式；可理解为半持久化方式，即按照一定的策略周期性的将数据保存到磁盘；对应产生的数据文件为dump.rdb，通过配置文件中的`save`参数来定义快照的周期，通用的快照设置：

```shell
save 900 1  #当有一条Keys数据被改变时，900秒刷新到Disk一次
save 300 10 #当有10条Keys数据被改变时，300秒刷新到Disk一次
save 60 10000 #当有10000条Keys数据被改变时，60秒刷新到Disk一次
```

Redis的RDB文件不会坏掉，因为其**写操作是在一个新进程中进行的**。

当生成一个新的RDB文件时，Redis生成的子进程会先将数据写到一个临时文件中，然后通过原子性rename系统调用将临时文件重命名为RDB文件；

这样，在任何时候出现故障，Redis的RDB文件都总是可用的；同时，Redis的RDB文件也是Redis主从同步内部实现的一环；

第一次Slave向Master同步的实现是：

**Slave向Master发出同步请求，Master先dump出rdb文件，然后将rdb文件全量传输给Slave，然后Master把缓存的命令转发给Slave，初次同步完成**；

第二次及以后的同步实现：

**Master将变量的快照直接实时依次发送给各个Slave；但不管什么原因导致Slave和Master断开重连都会重复以上两个步骤的过程；**Redis的主从复制就是建立在内存快照的持久化基础上的，只要有Slave就一定会有内存快照发生**。

可以明显看到，RDB方式一旦数据库出现问题，则RDB文件中保存的数据并不是全新的。从上次RDB文件生成到Redis停机这段时间的数据全部丢掉了。

AOF(Append-Only file)比RDB方式有更好的持久化性。

由于在使用AOF持久化时，Redis将每一个收到的写命令都通过Write函数追加到文件中，类似MySQL的binlog。当Redis重启时会通过重新执行文件中保存的写命令来在内存中重建整个数据库的内容。

对应的设置参数为：

```shell
vim /data/redis/member/conf/redis-6379.conf
appendonly yes #启用AOF持久化方式
appendfilename appendonly.aof #AOF文件的默认名称，可自定义
#appendfsync always #每次收到写命令就立即强制写入磁盘，是最有保证的完全持久化，但速度也是最慢的，一般不推荐使用；
appendfsync everysec #每秒钟强制写入磁盘一次，在性能和持久化方面做了很好的折中，是受推荐的方式；
#appendfsync no #完全依赖OS的写入，一般为30秒左右一次，性能最好但是持久化最没有保证，不被推荐；
```

AOF的完全持久化方式同时也带来了另一个问题，持久化文件会变得越来越大；比如我们调用INCR test命令100次，文件中就必须保存全部的100条命令，但其实99条都是多余的。 因为要恢复数据库的状态其实文件中保存一条SET test 100就够了。 

为了压缩AOF的持久化文件，Redis提供了bgrewriteaof命令。 收到此命令后Redis将使用与快照类似的方式将内存中的数据以命令的方式保存到临时文件中，最后替换原来的文件，以此来实现控制AOF文件的增长。 由于是模拟快照的过程，因此在重写AOF文件时并没有读取旧的AOF文件，而是将整个内存中的数据库内容用命令的方式重写了一个新的AOF文件。

 对应的设置参数为：

```shell 
vim /data/redis/member/conf/redis-6379.conf
#在日志重写时，不进行命令追加操作，而是将其放在缓冲区里，避免与命令的追加造成DISK IO上的冲突
no-appendfsync-on-rewrite yes 
#当前AOF文件大小是上次日志重写得到AOF文件大小的二倍时，自动启动新的日志重写过程
auto-aof-rewrite-percentage 100
#当前AOF文件启动新的日志重写过程的最小值，避免刚刚启动Redis时由于文件尺寸较小导致频繁的重写
auto-aof-rewrite-min-size 64mb
```

到底选择什么呢？下面是来自官方的建议： 通常，如果你要想提供很高的数据保障性，那么建议你同时使用两种持久化方式。 如果你可以接受灾难带来的几分钟的数据丢失，那么你可以仅使用RDB。 很多用户仅使用了AOF，但是我们建议，既然RDB可以时不时的给数据做个完整的快照，并且提供更快的重启，所以最好还是也使用RDB。 因此，我们希望可以在未来（长远计划）统一AOF和RDB成一种持久化模式。 

在数据恢复方面： 

RDB的启动时间会更短，原因有两个： 一是RDB文件中每一条数据只有一条记录，不会像AOF日志那样可能有一条数据的多次操作记录。所以每条数据只需要写一次就行了。 另一个原因是RDB文件的存储格式和Redis数据在内存中的编码格式是一致的，不需要再进行数据编码工作，所以在CPU消耗上要远小于AOF日志的加载。 

#### 二、灾难恢复模拟

如果数据要做持久化又想保证稳定性，则建议留空一半的物理内存。因为在进行快照的时候，fork出来进行dump操作的子进程会占用与父进程一样的内存，真正的copy-on-write，对性能的影响和内存的耗用都是比较大的。 目前，通常的设计思路是利用**Replication机制来弥补aof、snapshot性能上的不足，达到了数据可持久化**。 **即Master上Snapshot和AOF都不做，来保证Master的读写性能，而Slave上则同时开启Snapshot和AOF来进行持久化，保证数据的安全性**。

做模拟实验的两台虚拟机：

Master：172.20.109.251:6379

Slave:  172.20.128.29: 6380

首先，修改Master上的如下配置：

```shell
vim /data/redis/member/conf/redis-6379.conf
#save 900 1 #禁用Snapshot
#save 300 10 
#save 60 10000
appendonly no #禁用AOF

###安全参数###
requirepass  123456789
masterauth 123456789
rename-command KEYS REDIS_KEYS
rename-command FLUSHDB REDIS_FLUSHDB
rename-command FLUSHALL REDIS_FLUSHALL
```

接着，修改Slave上的如下配置：

```shell
vim /data/redis/member/conf/redis-6380.conf
###基本参数###
port 6380
bind 0.0.0.0
dir /data/redis/member/data
slaveof 172.20.109.251 6379
###RDB持久化参数###
save 900 1 #启用Snapshot
save 300 10 
save 60 10000
dbfilename "dump-6380.rdb"

###AOF持久化参数####
appendonly yes #启用AOF
appendfilename appendonly-6380.aof #AOF文件的名称
#appendfsync always
appendfsync everysec #每秒钟强制写入磁盘一次
#appendfsync no 

#在日志重写时，不进行命令追加操作
no-appendfsync-on-rewrite yes
#自动启动新的日志重写过程
auto-aof-rewrite-percentage 100
#启动新的日志重写过程的最小值
auto-aof-rewrite-min-size 64mb 
####安全参数###
#requirepass  123456789
masterauth 123456789
rename-command KEYS REDIS_KEYS
rename-command FLUSHDB REDIS_FLUSHDB
rename-command FLUSHALL REDIS_FLUSHALL
```

分别启动Master与Slave

```shell
sudo -u redis `which redis-server` /data/redis/member/conf/redis-6379.conf
sudo -u redis `which redis-server` /data/redis/member/conf/redis-6380.conf
```

启动完成后在Master中确认未启动Snapshot参数

```shell
redis-cli -h 172.20.109.251 -a 123456789
redis 172.20.109.251:6379> config get save
1)"save"
2)""
```

然后通过以下脚本在Master中生成25万条数据：

```shell 
vim redis-cli-generate.temp.sh
#!/bin/bash
#
REDIS_CLI="redis-cli -h 172.20.109.251 -a 123456789 -n 1 SET"
ID=1
 
while(($ID<10001))
do
     INSTANCE_NAME="i-2-$ID-VM"
     UUID=`cat /proc/sys/kernel/random/uuid`
     PRIVATE_IP_ADDRESS=10.`echo "$RANDOM % 255 + 1" | bc`.`echo "$RANDOM % 255 + 1" | bc`.`echo "$RANDOM" % 255 + 1 | bc`
     CREATED=`date "+%Y-%m-%d %H:%M:%S"`
 
     $REDIS_CLI vm_instance:$ID:instance_name "$INSTANCE_NAME"
     $REDIS_CLI vm_instance:$ID:uuid "$UUID"
     $REDIS_CLI vm_instance:$ID:private_ip_address "$PRIVATE_IP_ADDRESS"
     $REDIS_CLI vm_instance:$ID:created "$CREATED"
 
     $REDIS_CLI vm_instance:$INSTANCE_NAME:id "$ID"
 
     ID=$(($ID+1))
done
```

之后，Master端再次确认生成的键值对数据：

```shell
redis-cli -h 172.20.109.251 -a 123456789
172.20.109.251:6379> select 1 #选择1号数据库，因为脚本中是向1号库导入的数据
172.20.109.251:6379[1]> dbsize #数据量过大，导入部分测试即可	
(integer) 42099
172.20.109.251:6379[1]> redis_keys *
...
42083) "vm_instance:4230:created"
42084) "vm_instance:4764:instance_name"
42085) "vm_instance:6042:private_ip_address"
42086) "vm_instance:3937:instance_name"
42087) "vm_instance:23:created"
42088) "vm_instance:i-2-1408-VM:id"
...
```

在数据的生成过程中，可以很清楚的看到Master上仅在第一次做Slave同步时创建了dump.rdb文件，之后就通过增量传输命令的方式给Slave了。 

dump.rdb文件没有再增大。 而Slave上则可以看到dump.rdb文件和AOF文件在不断的增大，并且AOF文件的增长速度明显大于dump.rdb文件。 

```shell 
[root@router ~]# cd /data/redis/member/data/
[root@router data]# ll -h
-rw-r--r-- 1 redis redis 2.9M Jul 26 09:11 appendonly-6380.aof
-rw-r--r-- 1 redis redis 1.9M Jul 26 09:12 dump-6380.rdb
```

等待数据插入完成以后，首先确认当前的数据量:

```shell
172.20.109.251:6379[1]> info
# Server
redis_version:3.2.3
redis_git_sha1:00000000
redis_git_dirty:0
redis_build_id:cddaed2fff092221
redis_mode:standalone
os:Linux 3.10.0-693.el7.x86_64 x86_64
arch_bits:64
multiplexing_api:epoll
gcc_version:4.8.5
process_id:1521
run_id:09aa131702950481b5ed336c034056bec5ad7858
tcp_port:6379
uptime_in_seconds:368
uptime_in_days:0
hz:10
lru_clock:5869995
executable:/usr/local/bin/redis-server
config_file:/data/redis/member/conf/redis-6379.conf

# Clients
connected_clients:1
client_longest_output_list:0
client_biggest_input_buf:0
blocked_clients:0

# Memory
used_memory:6017576
used_memory_human:5.74M
used_memory_rss:16240640
used_memory_rss_human:15.49M
used_memory_peak:6604856
used_memory_peak_human:6.30M
total_system_memory:1023713280
total_system_memory_human:976.29M
used_memory_lua:37888
used_memory_lua_human:37.00K
maxmemory:8000000000
maxmemory_human:7.45G
maxmemory_policy:volatile-lru
mem_fragmentation_ratio:2.70
mem_allocator:jemalloc-4.0.3

# Persistence
loading:0
rdb_changes_since_last_save:0
rdb_bgsave_in_progress:0
rdb_last_save_time:1532596291
rdb_last_bgsave_status:ok
rdb_last_bgsave_time_sec:0
rdb_current_bgsave_time_sec:-1
aof_enabled:1
aof_rewrite_in_progress:0
aof_rewrite_scheduled:0
aof_last_rewrite_time_sec:-1
aof_current_rewrite_time_sec:-1
aof_last_bgrewrite_status:ok
aof_last_write_status:ok
aof_current_size:3158480
aof_base_size:3158480
aof_pending_rewrite:0
aof_buffer_length:0
aof_rewrite_buffer_length:0
aof_pending_bio_fsync:0
aof_delayed_fsync:0

# Stats
total_connections_received:3
total_commands_processed:361
instantaneous_ops_per_sec:1
total_net_input_bytes:12804
total_net_output_bytes:19246895
instantaneous_input_kbps:0.04
instantaneous_output_kbps:0.00
rejected_connections:0
sync_full:1
sync_partial_ok:0
sync_partial_err:0
expired_keys:0
evicted_keys:0
keyspace_hits:0
keyspace_misses:0
pubsub_channels:0
pubsub_patterns:0
latest_fork_usec:21393
migrate_cached_sockets:0

# Replication
role:master
connected_slaves:1
slave0:ip=172.20.128.29,port=6380,state=online,offset=491,lag=1
master_repl_offset:491
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:2
repl_backlog_histlen:490

# CPU
used_cpu_sys:0.67
used_cpu_user:0.27
used_cpu_sys_children:0.09
used_cpu_user_children:0.02

# Cluster
cluster_enabled:0

# Keyspace
db1:keys=42099,expires=0,avg_ttl=0
```

当前的数据量为42099条Key, 占用内存5.74M；然后直接Kill掉Master的Redis进程，模拟灾难：

```shell
sudo killall -9 redis-server
```

到Slave中查看状态：

```shell
[root@router conf]# redis-cli -h 172.20.128.29 -p 6380
172.20.128.29:6380> clear
172.20.128.29:6380> info
# Server
redis_version:3.2.3
redis_git_sha1:00000000
redis_git_dirty:0
redis_build_id:eb9c5b33f4d02b6a
redis_mode:standalone
os:Linux 3.10.0-693.el7.x86_64 x86_64
arch_bits:64
multiplexing_api:epoll
gcc_version:4.8.5
process_id:1942
run_id:49b3230e90716b709333b59aa877adf921b2db69
tcp_port:6380
uptime_in_seconds:688
uptime_in_days:0
hz:10
lru_clock:5870323
executable:/usr/local/bin/redis-server
config_file:/data/redis/member/conf/redis-6380.conf

# Clients
connected_clients:1
client_longest_output_list:0
client_biggest_input_buf:0
blocked_clients:0

# Memory
used_memory:4874912
used_memory_human:4.65M
used_memory_rss:12083200
used_memory_rss_human:11.52M
used_memory_peak:4895328
used_memory_peak_human:4.67M
total_system_memory:1023713280
total_system_memory_human:976.29M
used_memory_lua:37888
used_memory_lua_human:37.00K
maxmemory:8000000000
maxmemory_human:7.45G
maxmemory_policy:volatile-lru
mem_fragmentation_ratio:2.48
mem_allocator:jemalloc-4.0.3

# Persistence
loading:0
rdb_changes_since_last_save:0
rdb_bgsave_in_progress:0
rdb_last_save_time:1532596352
rdb_last_bgsave_status:ok
rdb_last_bgsave_time_sec:0
rdb_current_bgsave_time_sec:-1
aof_enabled:1
aof_rewrite_in_progress:0
aof_rewrite_scheduled:0
aof_last_rewrite_time_sec:0
aof_current_rewrite_time_sec:-1
aof_last_bgrewrite_status:ok
aof_last_write_status:ok
aof_current_size:2961966
aof_base_size:2961966
aof_pending_rewrite:0
aof_buffer_length:0
aof_rewrite_buffer_length:0
aof_pending_bio_fsync:0
aof_delayed_fsync:0

# Stats
total_connections_received:1
total_commands_processed:65
instantaneous_ops_per_sec:0
total_net_input_bytes:1945247
total_net_output_bytes:5921619
instantaneous_input_kbps:0.00
instantaneous_output_kbps:0.00
rejected_connections:0
sync_full:0
sync_partial_ok:0
sync_partial_err:0
expired_keys:0
evicted_keys:0
keyspace_hits:0
keyspace_misses:0
pubsub_channels:0
pubsub_patterns:0
latest_fork_usec:1334
migrate_cached_sockets:0

# Replication
role:slave
master_host:172.20.109.251
master_port:6379
master_link_status:down
master_last_io_seconds_ago:-1
master_sync_in_progress:0
slave_repl_offset:897
master_link_down_since_seconds:33
slave_priority:100
slave_read_only:1
connected_slaves:0
master_repl_offset:0
repl_backlog_active:0
repl_backlog_size:1048576
repl_backlog_first_byte_offset:0
repl_backlog_histlen:0

# CPU
used_cpu_sys:1.09
used_cpu_user:0.57
used_cpu_sys_children:0.03
used_cpu_user_children:0.11

# Cluster
cluster_enabled:0

# Keyspace
db1:keys=42099,expires=0,avg_ttl=0
```

可以看到master_link_status的状态已经是down了，Master已经不可访问了；而此时，Slave依然运行良好，并且保留有AOF与RDB文件。

下面通过Slave上保存好的AOF与RDB文件来恢复Master上的数据。

首先，将Slave上的同步状态取消，避免主库在未完成数据恢复前就重启，进而直接覆盖掉从库上的数据：

```shell
172.20.128.29:6380> slaveof no one
OK
```

在Slave上复制数据文件：

```shell
[root@router data]# tar cvf /root/redis_data.tar appendonly-6380.aof dump-6380.rdb 
appendonly-6380.aof
dump-6380.rdb
[root@router ~]# scp redis_data.tar root@172.20.109.251:
root@172.20.109.251's password: 
redis_data.tar    
```

将redis_data.tar上传到Master上，尝试恢复数据：

可以看到Master目录下有一个初始化Slave的数据文件，很小，将其删除：

```shell
[root@client1 ~]# cd /data/redis/member/data/
[root@client1 data]# ll 
total 8
-rw-r--r-- 1 redis redis  76 Jul 26 09:42 dump.rdb
[root@client1 data]# rm -f dump.rdb 
```

然后解压缩数据文件：

```shell
[root@client1 data]# tar xvf /root/redis_data.tar -C .
appendonly-6380.aof
dump-6380.rdb
#文件名修改成与Master的Redis配置文件中对应文件名相一致，用于启动加载
[root@client1 data]# mv appendonly-6380.aof appendonly-6379.aof 
[root@client1 data]# mv dump-6380.rdb dump-6379.rdb 
#之后Master配置文件中重新启用RDB和AOF持久化配置
####RDB持久化参数###
 save 900 1
 save 300 10
 save 60 10000
 stop-writes-on-bgsave-error yes
 rdbcompression yes
 rdbchecksum yes
 dbfilename "dump-6379.rdb"
 
 ####AOF持久化参数###
 no-appendfsync-on-rewrite yes
 appendonly yes
 appendfilename "appendonly-6379.aof"
 appendfsync no
 auto-aof-rewrite-min-size 512mb
 auto-aof-rewrite-percentage 100
 aof-load-truncated yes
 aof-rewrite-incremental-fsync yes
```

再次进行查看，发现数据已正常恢复；

此时，可以在Slave端重新恢复Slave的同步设置了

```shell 
172.20.128.29:6380> slaveof 172.20.109.251 6379
OK
172.20.128.29:6380> info
# Replication
role:slave
master_host:172.20.109.251
master_port:6379
master_link_status:up
#发现master_link_status up ,同步状态恢复正常
```

#### 总结

在此次恢复的过程中，我们同时复制了AOF与RDB文件，那么到底是哪一个文件完成了数据的恢复呢？ 

实际上，当Redis服务器挂掉时，重启时将按照以下优先级恢复数据到内存： 

1. 如果只配置AOF,重启时加载AOF文件恢复数据 
2. 若同时配置了RDB和AOF，启动时只加载AOF文件恢复数据
3. 若只配置RDB，启动是将加载dump文件恢复数据

也就是说，AOF的优先级要高于RDB，因为AOF本身对数据的完整性保障要高于RDB；

在此次的实验中，通过Slave上启用了AOF与RDB来保障了数据并恢复了Master;

但是在目前一般的线上环境中，由于数据都设置有过期时间，采用AOF的方式会不太实用，过于频繁的写操作会使AOF文件增长到异常的庞大，大大超过了实际的数据量，这也会导致在进行数据恢复时耗用大量的时间；

**因此，可以在Slave上仅开启Snapshot来进行本地化，同时可以考虑将save中的频率调高一些或者调用一个计划任务来进行定期bgsave的快照存储，来尽可能的保障本地化数据的完整性。**

在这样的架构下，若仅仅是Master挂掉，Slave完整，数据恢复可达到100%；

若Master与Slave同时挂掉的话，数据的恢复也可以达到一个可接受的程度。



另附：redis启动时有段报错日志， 请自行根据日志报错信息添加或修改相关系统内核参数

30014:M 25 Jul 20:28:40.399 # WARNING: The TCP backlog setting of 65535 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 2048.
30014:M 25 Jul 20:28:40.399 # Server started, Redis version 3.2.3
30014:M 25 Jul 20:28:40.399 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
30014:M 25 Jul 20:28:40.399 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
30014:M 25 Jul 20:28:40.399 * The server is now ready to accept connections on port 6379
INSTANCE_NAME="i-2-$ID-VM"^C