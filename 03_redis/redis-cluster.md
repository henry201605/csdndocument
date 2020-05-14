Redis Cluster 集群搭建



## 1、引言

在上一篇，进行了Redis Sentinel的搭建，Sentinel有其自身的优点，也有一些不足之处，如下：

1、主从切换的过程中会丢失数据，因为只有一个 master。 

2、只能单点写，没有解决水平扩容的问题。 



## 2、Redis Cluster介绍

### 2.1概念

Redis 集群是一个提供在**多个Redis间节点间共享数据**的程序集。

Redis集群并不支持处理多个keys的命令,因为这需要在不同的节点间移动数据,从而达不到像Redis那样的性能,在高负载的情况下可能会导致不可预料的错误.

Redis 集群通过分区来提供**一定程度的可用性**,在实际环境中当某个节点宕机或者不可达的情况下继续处理命令. Redis 集群的优势:

- 自动分割数据到不同的节点上。
- 整个集群的部分节点失败或者不可达的情况下能够继续处理命令。

### 2.2Redis 集群的数据分片

Redis 既没有用哈希取模，也没有用一致性哈希，而是用虚拟槽来实现的。 

Redis 创建了 16384 个槽（slot），每个节点负责一定区间的 slot。比如 Node1 负责 0-5460，Node2 负责5461-10922，Node3 负责 10923-16383。



![image-20200411111612626](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200411111612626.png)



Redis 的每个 master 节点维护一个 16384 位（2048bytes=2KB）的位序列，比如： 序列的第 0 位是 1，就代表第一个 slot 是它负责；序列的第 1 位是 0，代表第二个 slot 不归它负责。 

对象分布到 Redis 节点上时，对 key 用 CRC16 算法计算再%16384，得到一个 slot的值，数据落到负责这个 slot 的 Redis 节点上。 

这种结构很容易添加或者删除节点. 比如如果我想新添加个节点Node4, 我需要从节点 Node1, Node2, Node3中得部分槽到Node4上. 如果我想移除节点Node1,需要将Node1中的槽移到Node2和Node3节点上,然后将没有任何槽的Node1节点从集群中移除即可. 由于从一个节点将哈希槽移动到另一个节点并不会停止服务,所以无论添加删除或者改变某个节点的哈希槽的数量都不会造成集群不可用的状态.

## 3、架构

参考`官网`：https://redis.io/topics/cluster-tutorial/

Redis Cluster 可以看成是由多个 Redis 实例组成的数据集合。客户端不需要关注数 

据的子集到底存储在哪个节点，只需要关注这个集合整体。 

以 3 主 3 从为例，节点之间两两交互，共享数据分片、节点状态等信息；

![image-20200411105816311](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200411105816311.png)



## 4、Redis Cluster搭建

本文为了节省机器，直接把6个Redis实例安装在同一台机器上（3主3从），只是使用不同的端口号。

### 4.1 环境

1、操作系统：Centos7.7

2、服务器配置如下：

| 角色   |     ip      | 端口 |
| ------ | :---------: | :--- |
| master | 172.27.0.13 | 6373 |
| master | 172.27.0.13 | 6374 |
| master | 172.27.0.13 | 6375 |
| slave  | 172.27.0.13 | 6376 |
| slave  | 172.27.0.13 | 6377 |
| slave  | 172.27.0.13 | 6378 |

### 4.2 redis安装

具体单机版的redis安装步骤可以参考https://blog.csdn.net/qq_33996921/article/details/105131596

下面简单列一下本次redis的安装步骤：

```shell
`下载`
cd /usr/local/soft/
wget http://download.redis.io/releases/redis-5.0.8.tar.gz

`解压缩`
tar -zvxf redis-5.0.8.tar.gz

`编译安装`
cd redis-5.0.5
make
cd src
make install
```

### 4.3 创建多个实例

以上为安装过程，下面开始进行redis的配置。

Redis 单机多实例部署方法十分简单，只要复制多个 redis 配置文件即可。需要注意每个实例的端口不能冲突。

```shell
[root@m redis-5.0.8]# cd /usr/local/soft/redis-5.0.8/
[root@m redis-5.0.8]# mkdir redis-cluster
[root@m redis-5.0.8]# cd redis-cluster/
[root@m redis-cluster]# mkdir -p ./{6373,6374,6375,6376,6377,6378}
[root@m redis-cluster]# ls
6373  6374  6375  6376  6377  6378
```

### 4.4 配置文件

* 将`redis.conf`复制到6378文件夹下：

```shell
`复制配置文件`
[root@m redis-cluster]# cp /usr/local/soft/redis-5.0.8/redis.conf /usr/local/soft/redis-5.0.8/redis-cluster/6378
`修改配置文件`
[root@m redis-cluster]# cd 6378
[root@m 6378]# vim redis.conf
```

配置文件信息如下：

```shell
port 6378
daemonize yes
protected-mode no
dir /usr/local/soft/redis-5.0.8/redis-cluster/6378/
cluster-enabled yes
cluster-config-file nodes-6378.conf
cluster-node-timeout 5000
appendonly yes
pidfile /var/run/redis_6378.pid
```

* 将配置文件复制到其它5个文件夹下：

```shell
[root@m 6378]# cd /usr/local/soft/redis-5.0.8/redis-cluster/6378
[root@m 6378]# cp redis.conf ../6377
[root@m 6378]# cp redis.conf ../6376
[root@m 6378]# cp redis.conf ../6375
[root@m 6378]# cp redis.conf ../6374
[root@m 6378]# cp redis.conf ../6373
```

* 批量替换内容

```shell
cd /usr/local/soft/redis-5.0.8/redis-cluster
sed -i 's/6378/6377/g' 6377/redis.conf
sed -i 's/6378/6376/g' 6376/redis.conf
sed -i 's/6378/6375/g' 6375/redis.conf
sed -i 's/6378/6374/g' 6374/redis.conf
sed -i 's/6378/6373/g' 6373/redis.conf
```

### 4.5 Redis启动

启动6个redis实例。

```
/usr/local/soft/redis-5.0.8/src/redis-server /usr/local/soft/redis-5.0.8/redis-cluster/6378/redis.conf

/usr/local/soft/redis-5.0.8/src/redis-server /usr/local/soft/redis-5.0.8/redis-cluster/6377/redis.conf

/usr/local/soft/redis-5.0.8/src/redis-server /usr/local/soft/redis-5.0.8/redis-cluster/6376/redis.conf

/usr/local/soft/redis-5.0.8/src/redis-server /usr/local/soft/redis-5.0.8/redis-cluster/6375/redis.conf

/usr/local/soft/redis-5.0.8/src/redis-server /usr/local/soft/redis-5.0.8/redis-cluster/6374/redis.conf

/usr/local/soft/redis-5.0.8/src/redis-server /usr/local/soft/redis-5.0.8/redis-cluster/6373/redis.conf
```

查看进程：

```shell
[root@m redis-cluster]# ps -ef|grep redis
root     17873     1  0 12:45 ?        00:00:00 /usr/local/soft/redis-5.0.8/src/redis-server *:6378 [cluster]
root     17875     1  0 12:45 ?        00:00:00 /usr/local/soft/redis-5.0.8/src/redis-server *:6377 [cluster]
root     17877     1  0 12:45 ?        00:00:00 /usr/local/soft/redis-5.0.8/src/redis-server *:6376 [cluster]
root     17885     1  0 12:45 ?        00:00:00 /usr/local/soft/redis-5.0.8/src/redis-server *:6375 [cluster]
root     17887     1  0 12:45 ?        00:00:00 /usr/local/soft/redis-5.0.8/src/redis-server *:6374 [cluster]
root     18313     1  0 12:45 ?        00:00:00 /usr/local/soft/redis-5.0.8/src/redis-server *:6373 [cluster]
root     26028 28580  0 12:48 pts/1    00:00:00 grep --color=auto redis

```

### 4.6启动集群

```shell
[root@m redis-cluster]# cd /usr/local/soft/redis-5.0.8/src/
[root@m src]# redis-cli --cluster create 172.27.0.13:6373 172.27.0.13:6374 172.27.0.13:6375 172.27.0.13:6376 172.27.0.13:6377 172.27.0.13:6378 --cluster-replicas 1
`-------------------集群搭建信息--------------------------`
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 172.27.0.13:6377 to 172.27.0.13:6373
Adding replica 172.27.0.13:6378 to 172.27.0.13:6374
Adding replica 172.27.0.13:6376 to 172.27.0.13:6375
>>> Trying to optimize slaves allocation for anti-affinity
[WARNING] Some slaves are in the same host as their master
M: ae0b3359ec6f52558ce572144770dd38e36851f4 172.27.0.13:6373
   slots:[0-5460] (5461 slots) master
M: a22f06edc5ae87f16703f4bcaf6e9de30d944c63 172.27.0.13:6374
   slots:[5461-10922] (5462 slots) master
M: bec4ddd8d2ead3b2c08d0e1c3e6dc710ad123aa5 172.27.0.13:6375
   slots:[10923-16383] (5461 slots) master
S: 24ce5079e076fedae92ed77c8920287f0ca060cb 172.27.0.13:6376
   replicates ae0b3359ec6f52558ce572144770dd38e36851f4
S: 062719135b227803a1e91f18606d58bd455e5ae9 172.27.0.13:6377
   replicates a22f06edc5ae87f16703f4bcaf6e9de30d944c63
S: efeb5e7c5e3cd428341b3de4b6b40171aadda3e8 172.27.0.13:6378
   replicates bec4ddd8d2ead3b2c08d0e1c3e6dc710ad123aa5
Can I set the above configuration? (type 'yes' to accept): yes

```

系统会给出一个集群的预分配方案，没有问题，直接yes，继续操作：

```shell
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
...
>>> Performing Cluster Check (using node 172.27.0.13:6373)
M: ae0b3359ec6f52558ce572144770dd38e36851f4 172.27.0.13:6373
   `slots:[0-5460] (5461 slots) master`
   1 additional replica(s)
S: efeb5e7c5e3cd428341b3de4b6b40171aadda3e8 132.232.115.96:6378
   slots: (0 slots) slave
   replicates bec4ddd8d2ead3b2c08d0e1c3e6dc710ad123aa5
M: a22f06edc5ae87f16703f4bcaf6e9de30d944c63 132.232.115.96:6374
   `slots:[5461-10922] (5462 slots) master`
   1 additional replica(s)
S: 062719135b227803a1e91f18606d58bd455e5ae9 132.232.115.96:6377
   slots: (0 slots) slave
   replicates a22f06edc5ae87f16703f4bcaf6e9de30d944c63
M: bec4ddd8d2ead3b2c08d0e1c3e6dc710ad123aa5 132.232.115.96:6375
   `slots:[10923-16383] (5461 slots) master`
   1 additional replica(s)
S: 24ce5079e076fedae92ed77c8920287f0ca060cb 132.232.115.96:6376
   slots: (0 slots) slave
   replicates ae0b3359ec6f52558ce572144770dd38e36851f4
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

由以上信息可以看出，3主3从的Redis集群就搭建好了。并且还可以看出master的slot分布

| 角色   | 端口 | 位置        | 数量 |
| ------ | :--- | ----------- | ---- |
| master | 6373 | 0-5460      | 5461 |
| master | 6374 | 5461-10922  | 5462 |
| master | 6375 | 10923-16383 | 5461 |

`注`：

重置集群的方式是在每个节点上个执行`cluster reset`，然后重新创建集群。



### 4.7 查看集群信息



```shell
[root@m src]# /usr/local/soft/redis-5.0.8/src/redis-cli -h 172.27.0.13 -p 6373
`查看集群信息`
172.27.0.13:6373> cluster info
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
`cluster_known_nodes:6`
`cluster_size:3`
cluster_current_epoch:6
cluster_my_epoch:1
cluster_stats_messages_ping_sent:4314
cluster_stats_messages_pong_sent:4234
cluster_stats_messages_sent:8548
cluster_stats_messages_ping_received:4229
cluster_stats_messages_pong_received:4314
cluster_stats_messages_meet_received:5
cluster_stats_messages_received:8548

```



查看集群节点：

```shell
172.27.0.13:6373> cluster nodes
efeb5e7c5e3cd428341b3de4b6b40171aadda3e8 132.232.115.96:6378@16378 slave bec4ddd8d2ead3b2c08d0e1c3e6dc710ad123aa5 0 1586583465323 6 connected
ae0b3359ec6f52558ce572144770dd38e36851f4 172.27.0.13:6373@16373 myself,master - 0 1586583465000 1 connected 0-5460
a22f06edc5ae87f16703f4bcaf6e9de30d944c63 132.232.115.96:6374@16374 master - 0 1586583465527 2 connected 5461-10922
062719135b227803a1e91f18606d58bd455e5ae9 132.232.115.96:6377@16377 slave a22f06edc5ae87f16703f4bcaf6e9de30d944c63 0 1586583465835 5 connected
bec4ddd8d2ead3b2c08d0e1c3e6dc710ad123aa5 132.232.115.96:6375@16375 master - 0 1586583466337 3 connected 10923-16383
24ce5079e076fedae92ed77c8920287f0ca060cb 132.232.115.96:6376@16376 slave ae0b3359ec6f52558ce572144770dd38e36851f4 0 1586583466537 4 connected

```

## 5、集群操作

### 5.1 客户端重定向

进入客户端进行操作：

```shell
[root@m ~]# /usr/local/soft/redis-5.0.8/src/redis-cli -p 6373
127.0.0.1:6373> set henry 123
(error) MOVED 16265 132.232.115.96:6375
```

服务端返回 MOVED，也就是根据 key 计算出来的 slot 不归 6373端口管理，而是 归 6375端口管理，服务端返回 MOVED 告诉客户端去 6375端口操作。 这个时候有两种操作方式：

1、更换端口，用 redis-cli –p 6375操作，才会返回 OK。

```shell
[root@m ~]#  /usr/local/soft/redis-5.0.8/src/redis-cli -p 6375
127.0.0.1:6375> set henry 123
OK
```

2、或者用./redis-cli -c -p port 的命令（c 代表 cluster）。这样客户端需要连接两次。

```shell
[root@m ~]# /usr/local/soft/redis-5.0.8/src/redis-cli -c -p 6373
127.0.0.1:6373> set henry 123
-> Redirected to slot [16265] located at 132.232.115.96:6375
OK
```

`注`:

Jedis 等客户端会在本地维护一份 slot——node 的映射关系，大部分时候不需要重 定向，所以叫做 smart jedis（需要客户端支持）。

### 5.2 数据迁移

​    因为 key 和 slot 的关系是永远不会变的，当新增了节点的时候，需要把原有的 slot 分配给新的节点负责，并且把相关的数据迁移过来。

### 5.3 增加一个master节点

1）新增一个端口号为6372的redis节点。

```shell
`启动redis`
[root@m redis-cluster]# /usr/local/soft/redis-5.0.8/src/redis-server /usr/local/soft/redis-5.0.8/redis-cluster/6372/redis.conf

[root@m redis-cluster]# ps -ef|grep redis
root     15185     1  0 14:23 ?        00:00:00 /usr/local/soft/redis-5.0.8/src/redis-`server *:6372 [cluster]`
root     15439  7584  0 14:23 pts/3    00:00:00 grep --color=auto redis
root     17873     1  0 12:45 ?        00:00:08 /usr/local/soft/redis-5.0.8/src/redis-server *:6378 [cluster]
root     17875     1  0 12:45 ?        00:00:08 /usr/local/soft/redis-5.0.8/src/redis-server *:6377 [cluster]
root     17877     1  0 12:45 ?        00:00:08 /usr/local/soft/redis-5.0.8/src/redis-server *:6376 [cluster]
root     17885     1  0 12:45 ?        00:00:07 /usr/local/soft/redis-5.0.8/src/redis-server *:6375 [cluster]
root     17887     1  0 12:45 ?        00:00:07 /usr/local/soft/redis-5.0.8/src/redis-server *:6374 [cluster]
root     18313     1  0 12:45 ?        00:00:07 /usr/local/soft/redis-5.0.8/src/redis-server *:6373 [cluster]
```

2）向集群中添加

在redis-5中redis-trib.rb的功能被集成到了redis-cli中，大大简化了redis的集群部署，加快了进群部署的速度，也方便后期维护与扩容。

```shell
redis-cli --cluster add-node 172.27.0.13:6372 172.27.0.13:6373
```

`语法`，一个新节点IP：端口 空格 一个旧节点IP：端口；

`注意`：

> 1）不能多个新节点一次性添加；
>
> 2)新节点后是旧节点；
>
> 3）旧节点可以是任意一个已经存在的节点的IP和端口；

添加节点：

```shell
[root@m log]# /usr/local/soft/redis-5.0.8/src/redis-cli --cluster add-node 172.27.0.13:6372 172.27.0.13:6373
>>> Adding node 172.27.0.13:6372 to cluster 172.27.0.13:6373
>>> Performing Cluster Check (using node 172.27.0.13:6373)
M: ae0b3359ec6f52558ce572144770dd38e36851f4 172.27.0.13:6373
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: efeb5e7c5e3cd428341b3de4b6b40171aadda3e8 132.232.115.96:6378
   slots: (0 slots) slave
   replicates bec4ddd8d2ead3b2c08d0e1c3e6dc710ad123aa5
M: a22f06edc5ae87f16703f4bcaf6e9de30d944c63 132.232.115.96:6374
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: 24ce5079e076fedae92ed77c8920287f0ca060cb 132.232.115.96:6376
   slots: (0 slots) slave
   replicates ae0b3359ec6f52558ce572144770dd38e36851f4
S: 062719135b227803a1e91f18606d58bd455e5ae9 132.232.115.96:6377
   slots: (0 slots) slave
   replicates a22f06edc5ae87f16703f4bcaf6e9de30d944c63
M: bec4ddd8d2ead3b2c08d0e1c3e6dc710ad123aa5 132.232.115.96:6375
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
>>> Send CLUSTER MEET to node 172.27.0.13:6372 to make it join the cluster.
[OK] New node added correctly.

```

3）查看集群的节点

```shell

172.27.0.13:6373> cluster nodes
`-----新增的6372----`
`ede53d3ca156bfe99493209aaf99e51ea9b2a8e9 132.232.115.96:6372@16372 master - 0 1586591227000 0 connected`
efeb5e7c5e3cd428341b3de4b6b40171aadda3e8 132.232.115.96:6378@16378 slave bec4ddd8d2ead3b2c08d0e1c3e6dc710ad123aa5 0 1586591226538 6 connected
a22f06edc5ae87f16703f4bcaf6e9de30d944c63 132.232.115.96:6374@16374 master - 0 1586591226029 2 connected 5461-10922
24ce5079e076fedae92ed77c8920287f0ca060cb 132.232.115.96:6376@16376 slave ae0b3359ec6f52558ce572144770dd38e36851f4 0 1586591227549 4 connected
ae0b3359ec6f52558ce572144770dd38e36851f4 172.27.0.13:6373@16373 myself,master - 0 1586591225000 1 connected 0-5460
062719135b227803a1e91f18606d58bd455e5ae9 132.232.115.96:6377@16377 slave a22f06edc5ae87f16703f4bcaf6e9de30d944c63 0 1586591227549 5 connected
bec4ddd8d2ead3b2c08d0e1c3e6dc710ad123aa5 132.232.115.96:6375@16375 master - 0 1586591226000 3 connected 10923-16383
```

redis日志：

```shell
27207:M 11 Apr 2020 15:44:49.032 # IP address for this node updated to 172.27.0.13
27207:M 11 Apr 2020 15:44:49.032 # Address updated for node ae0b3359ec6f52558ce572144770dd38e36851f4, now 132.232.115.96:6373
27207:M 11 Apr 2020 15:44:54.022 # Cluster state changed: ok
27207:M 11 Apr 2020 16:05:07.496 # New configEpoch set to 7
27207:M 11 Apr 2020 16:05:07.496 # configEpoch updated after importing slot 5461
```



4）分配hash槽

4.1) 建立连接：

新增的节点没有哈希槽，不能分布数据，在原来的任意一个节点上执行如下操作：

```shell
redis-cli --cluster reshard 132.232.115.96:6373
```

```shell
[root@m 6372]# redis-cli --cluster reshard 132.232.115.96:6373
>>> Performing Cluster Check (using node 132.232.115.96:6373)
M: ae0b3359ec6f52558ce572144770dd38e36851f4 132.232.115.96:6373
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
M: ede53d3ca156bfe99493209aaf99e51ea9b2a8e9 132.232.115.96:6372
   slots: (0 slots) master
S: efeb5e7c5e3cd428341b3de4b6b40171aadda3e8 132.232.115.96:6378
   slots: (0 slots) slave
   replicates bec4ddd8d2ead3b2c08d0e1c3e6dc710ad123aa5
M: a22f06edc5ae87f16703f4bcaf6e9de30d944c63 132.232.115.96:6374
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: 24ce5079e076fedae92ed77c8920287f0ca060cb 132.232.115.96:6376
   slots: (0 slots) slave
   replicates ae0b3359ec6f52558ce572144770dd38e36851f4
S: 062719135b227803a1e91f18606d58bd455e5ae9 132.232.115.96:6377
   slots: (0 slots) slave
   replicates a22f06edc5ae87f16703f4bcaf6e9de30d944c63
M: bec4ddd8d2ead3b2c08d0e1c3e6dc710ad123aa5 132.232.115.96:6375
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
How many slots do you want to move (from 1 to 16384)? 
```

4.2）确定hash槽数量

输入需要分配的hash槽的数量（比如 1000），和hash槽的来源节点（可以输入 all 或 

者 id）

```shell
`1、输入需要分配的hash槽数--1000`
How many slots do you want to move (from 1 to 16384)? 1000
```

4.3) 接收槽的结点id

```shell
What is the receiving node ID? ede53d3ca156bfe99493209aaf99e51ea9b2a8e9
Please enter all the source node IDs.
  Type 'all' to use all the nodes as source nodes for the hash slots.
  Type 'done' once you entered all the source nodes IDs.
```

4.4)输入源节点的id

```shell
Source node #1: all

Ready to move 1000 slots.
  Source nodes:
    M: ae0b3359ec6f52558ce572144770dd38e36851f4 132.232.115.96:6373
       slots:[0-5460] (5461 slots) master
       1 additional replica(s)
    M: a22f06edc5ae87f16703f4bcaf6e9de30d944c63 132.232.115.96:6374
       slots:[5461-10922] (5462 slots) master
       1 additional replica(s)
    M: bec4ddd8d2ead3b2c08d0e1c3e6dc710ad123aa5 132.232.115.96:6375
       slots:[10923-16383] (5461 slots) master
       1 additional replica(s)
  Destination node:
    M: ede53d3ca156bfe99493209aaf99e51ea9b2a8e9 132.232.115.96:6372
       slots: (0 slots) master
  Resharding plan:
    Moving slot 5461 from a22f06edc5ae87f16703f4bcaf6e9de30d944c63
    Moving slot 5462 from a22f06edc5ae87f16703f4bcaf6e9de30d944c63
    Moving slot 5463 from a22f06edc5ae87f16703f4bcaf6e9de30d944c63
       ..............................
   Moving slot 11251 from bec4ddd8d2ead3b2c08d0e1c3e6dc710ad123aa5
    Moving slot 11252 from bec4ddd8d2ead3b2c08d0e1c3e6dc710ad123aa5
    Moving slot 11253 from bec4ddd8d2ead3b2c08d0e1c3e6dc710ad123aa5
    Moving slot 11254 from bec4ddd8d2ead3b2c08d0e1c3e6dc710ad123aa5
    Moving slot 11255 from bec4ddd8d2ead3b2c08d0e1c3e6dc710ad123aa5
Do you want to proceed with the proposed reshard plan (yes/no)? 
```

输入源结点`id`，槽将从源结点中拿，分配后的槽在源结点中就不存在了；
输入`all`表示从所有源结点中获取槽；
输入`done`取消分配；

4.5）移动槽到目标结点`id`

```shell
Do you want to proceed with the proposed reshard plan (yes/no)? yes
Moving slot 5461 from 132.232.115.96:6374 to 132.232.115.96:6372: 
Moving slot 5462 from 132.232.115.96:6374 to 132.232.115.96:6372: 
Moving slot 5463 from 132.232.115.96:6374 to 132.232.115.96:6372: 
Moving slot 5464 from 132.232.115.96:6374 to 132.232.115.96:6372: 
Moving slot 5465 from 132.232.115.96:6374 to 132.232.115.96:6372: 
.....................................
```

4.6）查看集群nodes分布

```shell
[root@m ~]# /usr/local/soft/redis-5.0.8/src/redis-cli -h 172.27.0.13 -p 6373
172.27.0.13:6373> cluster nodes
`ede53d3ca156bfe99493209aaf99e51ea9b2a8e9 132.232.115.96:6372@16372 master - 0 1586592942000 7 connected 0-332 5461-5794 10923-11255`
efeb5e7c5e3cd428341b3de4b6b40171aadda3e8 132.232.115.96:6378@16378 slave bec4ddd8d2ead3b2c08d0e1c3e6dc710ad123aa5 0 1586592943538 6 connected
a22f06edc5ae87f16703f4bcaf6e9de30d944c63 132.232.115.96:6374@16374 master - 0 1586592942823 2 connected 5795-10922
24ce5079e076fedae92ed77c8920287f0ca060cb 132.232.115.96:6376@16376 slave ae0b3359ec6f52558ce572144770dd38e36851f4 0 1586592942000 4 connected
ae0b3359ec6f52558ce572144770dd38e36851f4 172.27.0.13:6373@16373 myself,master - 0 1586592942000 1 connected 333-5460
062719135b227803a1e91f18606d58bd455e5ae9 132.232.115.96:6377@16377 slave a22f06edc5ae87f16703f4bcaf6e9de30d944c63 0 1586592943538 5 connected
bec4ddd8d2ead3b2c08d0e1c3e6dc710ad123aa5 132.232.115.96:6375@16375 master - 0 1586592943000 3 connected 11256-16383

```

从信息中可以看到新增加的节点6372已经获取到了新槽，槽的分布为0-332、5461-5794、10923-11255；

至此，master节点增加完成；

### 5.4 增加一个slave节点

1)启动6371的实例：

```shell
`启动redis`
[root@m src]# /usr/local/soft/redis-5.0.8/src/redis-server /usr/local/soft/redis-5.0.8/redis-cluster/6371/redis.conf
[root@m src]# ps -ef|grep redis
root     19226     1  0 23:09 ?        00:00:00 /usr/local/soft/redis-5.0.8/src/redis-`server *:6371` [cluster]
root     19251 14782  0 23:09 pts/2    00:00:00 grep --color=auto redis
root     21447     1  0 14:54 ?        00:00:44 /usr/local/soft/redis-5.0.8/src/redis-server *:6378 [cluster]
root     21452     1  0 14:54 ?        00:00:44 /usr/local/soft/redis-5.0.8/src/redis-server *:6377 [cluster]
root     21454     1  0 14:54 ?        00:00:44 /usr/local/soft/redis-5.0.8/src/redis-server *:6376 [cluster]
root     21462     1  0 14:54 ?        00:00:39 /usr/local/soft/redis-5.0.8/src/redis-server *:6375 [cluster]
root     21468     1  0 14:54 ?        00:00:39 /usr/local/soft/redis-5.0.8/src/redis-server *:6374 [cluster]
root     21482     1  0 14:54 ?        00:00:39 /usr/local/soft/redis-5.0.8/src/redis-server *:6373 [cluster]
root     27207     1  0 14:56 ?        00:00:37 /usr/local/soft/redis-5.0.8/src/redis-server *:6372 [cluster]

```

2)添加slave节点

添加语法：

```shell
add-node 
new_host:new_port existing_host:existing_port  #添加节点，把新节点加入到指定的集群，默认添加主节点
                 --cluster-slave               #新节点作为从节点，默认随机一个主节点
                 --cluster-master-id <arg>     #给新节点指定主节点
```

为6372master节点增加一个slave节点：

```shell
./redis-cli --cluster add-node 132.232.115.96:6371 132.232.115.96:7001 --cluster-slave --cluster-master-id ede53d3ca156bfe99493209aaf99e51ea9b2a8e9
```

```shell
[root@m src]# cd /usr/local/soft/redis-5.0.8/src/
[root@m src]# ./redis-cli --cluster add-node 132.232.115.96:6371 132.232.115.96:6378 --cluster-slave --cluster-master-id ede53d3ca156bfe99493209aaf99e51ea9b2a8e9
>>> Adding node 132.232.115.96:6371 to cluster 132.232.115.96:6378
>>> Performing Cluster Check (using node 132.232.115.96:6378)
S: efeb5e7c5e3cd428341b3de4b6b40171aadda3e8 132.232.115.96:6378
   slots: (0 slots) slave
   replicates bec4ddd8d2ead3b2c08d0e1c3e6dc710ad123aa5
M: bec4ddd8d2ead3b2c08d0e1c3e6dc710ad123aa5 132.232.115.96:6375
   slots:[11256-16383] (5128 slots) master
   1 additional replica(s)
M: ae0b3359ec6f52558ce572144770dd38e36851f4 132.232.115.96:6373
   slots:[333-5460] (5128 slots) master
   1 additional replica(s)
S: 062719135b227803a1e91f18606d58bd455e5ae9 132.232.115.96:6377
   slots: (0 slots) slave
   replicates a22f06edc5ae87f16703f4bcaf6e9de30d944c63
M: a22f06edc5ae87f16703f4bcaf6e9de30d944c63 132.232.115.96:6374
   slots:[5795-10922] (5128 slots) master
   1 additional replica(s)
S: 24ce5079e076fedae92ed77c8920287f0ca060cb 132.232.115.96:6376
   slots: (0 slots) slave
   replicates ae0b3359ec6f52558ce572144770dd38e36851f4
M: ede53d3ca156bfe99493209aaf99e51ea9b2a8e9 132.232.115.96:6372
   slots:[0-332],[5461-5794],[10923-11255] (1000 slots) master
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
>>> Send CLUSTER MEET to node 132.232.115.96:6371 to make it join the cluster.
Waiting for the cluster to join
...
>>> Configure node as replica of 132.232.115.96:6372.
[OK] New node added correctly.
```

3)查看新节点的redis日志

```shell
27207:M 11 Apr 2020 23:12:29.755 * Replica 132.232.115.96:6371 asks for synchronization
27207:M 11 Apr 2020 23:12:29.755 * Starting BGSAVE for SYNC with target: disk
27207:M 11 Apr 2020 23:12:29.755 * Background saving started by pid 25129
25129:C 11 Apr 2020 23:12:29.769 * DB saved on disk
25129:C 11 Apr 2020 23:12:29.770 * RDB: 0 MB of memory used by copy-on-write
27207:M 11 Apr 2020 23:12:29.788 * Background saving terminated with success
27207:M 11 Apr 2020 23:12:29.788 * Synchronization with replica 132.232.115.96:6371 succeeded

```

日志显示已经加入到集群中.

4)查看集群中的nodes;

```shell
[root@m ~]# /usr/local/soft/redis-5.0.8/src/redis-cli -h 172.27.0.13 -p 6372
172.27.0.13:6372> cluster nodes
ede53d3ca156bfe99493209aaf99e51ea9b2a8e9 172.27.0.13:6372@16372 myself,master - 0 1586618184000 7 connected 0-332 5461-5794 10923-11255
efeb5e7c5e3cd428341b3de4b6b40171aadda3e8 132.232.115.96:6378@16378 slave bec4ddd8d2ead3b2c08d0e1c3e6dc710ad123aa5 0 1586618184000 3 connected
062719135b227803a1e91f18606d58bd455e5ae9 132.232.115.96:6377@16377 slave a22f06edc5ae87f16703f4bcaf6e9de30d944c63 0 1586618183847 2 connected
`新增节点6371`
`cc2baec3cf2b4567838bf39865327503e1e06dfb 132.232.115.96:6371@16371 slave ede53d3ca156bfe99493209aaf99e51ea9b2a8e9 0 1586618184354 7 connected`
bec4ddd8d2ead3b2c08d0e1c3e6dc710ad123aa5 132.232.115.96:6375@16375 master - 0 1586618184000 3 connected 11256-16383
a22f06edc5ae87f16703f4bcaf6e9de30d944c63 132.232.115.96:6374@16374 master - 0 1586618184554 2 connected 5795-10922
ae0b3359ec6f52558ce572144770dd38e36851f4 132.232.115.96:6373@16373 master - 0 1586618185060 1 connected 333-5460
24ce5079e076fedae92ed77c8920287f0ca060cb 132.232.115.96:6376@16376 slave ae0b3359ec6f52558ce572144770dd38e36851f4 0 1586618185363 1 connected
```

### 5.5  删除节点

```shell
redis-cli --cluster del-node 132.232.115.96:6371 cc2baec3cf2b4567838bf39865327503e1e06dfb
redis-cli --cluster del-node 132.232.115.96:6372 ede53d3ca156bfe99493209aaf99e51ea9b2a8e9

```

`说明`：指定IP、端口和node_id 来删除一个节点，从节点可以直接删除，主节点不能直接删除，删除之后，该节点会被shutdown。

1)删除slave节点

```shell
[root@m src]# redis-cli --cluster del-node 132.232.115.96:6371 cc2baec3cf2b4567838bf39865327503e1e06dfb
>>> Removing node cc2baec3cf2b4567838bf39865327503e1e06dfb from cluster 132.232.115.96:6371
>>> Sending CLUSTER FORGET messages to the cluster...
>>> SHUTDOWN the node.

```

2）删除master节点

```shell
[root@m src]# redis-cli --cluster del-node 132.232.115.96:6372 ede53d3ca156bfe99493209aaf99e51ea9b2a8e9
>>> Removing node ede53d3ca156bfe99493209aaf99e51ea9b2a8e9 from cluster 132.232.115.96:6372
[ERR] Node 132.232.115.96:6372 is not empty! Reshard data away and try again.

```

`注`：

* 当被删除掉的节点重新起来之后不能自动加入集群，但其和主的复制还是正常的，也可以通过该节点看到集群信息（通过其他正常节点已经看不到该被del-node节点的信息）。

* 如果想要再次加入集群，则需要先在该节点执行cluster reset，再用add-node进行添加，进行增量同步复制。

查看此时集群情况：

```shell
172.27.0.13:6372> cluster nodes
ede53d3ca156bfe99493209aaf99e51ea9b2a8e9 172.27.0.13:6372@16372 myself,master - 0 1586619235000 7 connected 0-332 5461-5794 10923-11255
efeb5e7c5e3cd428341b3de4b6b40171aadda3e8 132.232.115.96:6378@16378 slave bec4ddd8d2ead3b2c08d0e1c3e6dc710ad123aa5 0 1586619237109 3 connected
062719135b227803a1e91f18606d58bd455e5ae9 132.232.115.96:6377@16377 slave a22f06edc5ae87f16703f4bcaf6e9de30d944c63 0 1586619237000 2 connected
bec4ddd8d2ead3b2c08d0e1c3e6dc710ad123aa5 132.232.115.96:6375@16375 master - 0 1586619237110 3 connected 11256-16383
a22f06edc5ae87f16703f4bcaf6e9de30d944c63 132.232.115.96:6374@16374 master - 0 1586619237109 2 connected 5795-10922
ae0b3359ec6f52558ce572144770dd38e36851f4 132.232.115.96:6373@16373 master - 0 1586619237000 1 connected 333-5460
24ce5079e076fedae92ed77c8920287f0ca060cb 132.232.115.96:6376@16376 slave ae0b3359ec6f52558ce572144770dd38e36851f4 0 1586619236000 1 connected

```

可以看到6371的节点已被删除。

### 5.6 集群检查

1)任意连接一个集群节点，进行集群状态检查:

```shell
redis-cli --cluster check 132.232.115.96:6372 --cluster-search-multiple-owners
```

```shell
[root@m src]# redis-cli --cluster check 132.232.115.96:6372 --cluster-search-multiple-owners
132.232.115.96:6372 (ede53d3c...) -> 0 keys | 1000 slots | 0 slaves.
132.232.115.96:6375 (bec4ddd8...) -> 1 keys | 5128 slots | 1 slaves.
132.232.115.96:6374 (a22f06ed...) -> 0 keys | 5128 slots | 1 slaves.
132.232.115.96:6373 (ae0b3359...) -> 0 keys | 5128 slots | 1 slaves.
[OK] 1 keys in 4 masters.
0.00 keys per slot on average.
>>> Performing Cluster Check (using node 132.232.115.96:6372)
M: ede53d3ca156bfe99493209aaf99e51ea9b2a8e9 132.232.115.96:6372
   slots:[0-332],[5461-5794],[10923-11255] (1000 slots) master
S: efeb5e7c5e3cd428341b3de4b6b40171aadda3e8 132.232.115.96:6378
   slots: (0 slots) slave
   replicates bec4ddd8d2ead3b2c08d0e1c3e6dc710ad123aa5
S: 062719135b227803a1e91f18606d58bd455e5ae9 132.232.115.96:6377
   slots: (0 slots) slave
   replicates a22f06edc5ae87f16703f4bcaf6e9de30d944c63
M: bec4ddd8d2ead3b2c08d0e1c3e6dc710ad123aa5 132.232.115.96:6375
   slots:[11256-16383] (5128 slots) master
   1 additional replica(s)
M: a22f06edc5ae87f16703f4bcaf6e9de30d944c63 132.232.115.96:6374
   slots:[5795-10922] (5128 slots) master
   1 additional replica(s)
M: ae0b3359ec6f52558ce572144770dd38e36851f4 132.232.115.96:6373
   slots:[333-5460] (5128 slots) master
   1 additional replica(s)
S: 24ce5079e076fedae92ed77c8920287f0ca060cb 132.232.115.96:6376
   slots: (0 slots) slave
   replicates ae0b3359ec6f52558ce572144770dd38e36851f4
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
>>> Check for multiple slot owners...
```

2)查看集群信息:

检查key、slots、从节点个数的分配情况

```shell
[root@m src]# redis-cli --cluster info 132.232.115.96:6373
132.232.115.96:6373 (ae0b3359...) -> 0 keys | 5128 slots | 1 slaves.
132.232.115.96:6372 (ede53d3c...) -> 0 keys | 1000 slots | 0 slaves.
132.232.115.96:6374 (a22f06ed...) -> 0 keys | 5128 slots | 1 slaves.
132.232.115.96:6375 (bec4ddd8...) -> 1 keys | 5128 slots | 1 slaves.
[OK] 1 keys in 4 masters.
0.00 keys per slot on average.

```

## 6、高可用和主从切换原理

当 slave 发现自己的 master 变为 FAIL 状态时，便尝试进行 Failover，以期成为新的master。由于挂掉的master可能会有多个slave，从而存在多个slave竞争成为master节点的过程， 其过程如下： 

1）slave 发现自己的 master 变为 FAIL ；

2）将自己记录的集群 currentEpoch 加 1，并广播 FAILOVER_AUTH_REQUEST 信息 ；

3）其他节点收到该信息，只有 master 响应，判断请求者的合法性，并发送FAILOVER_AUTH_ACK，对每一个 epoch 只发送一次 ack ；

4）尝试 failover 的 slave 收集 FAILOVER_AUTH_ACK ；

5）超过半数后变成新 Master；

## 结语：

至此，Redis Cluster集群搭建完成！

