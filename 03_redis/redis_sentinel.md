



# Redis的Sentinel机制搭建

参照`官网`：https://redis.io/topics/sentinel

Redis Sentinel 是一个分布式系统， 你可以在一个架构中运行多个 Sentinel 进程（progress）， 这些进程使用流言协议（gossip protocols)来接收关于主服务器是否下线的信息， 并使用投票协议（agreement protocols）来决定是否执行自动故障迁移， 以及选择哪个从服务器作为新的主服务器。

Redis 的 Sentinel 系统用于管理多个 Redis 服务器（instance）， 该系统执行以下三个任务：

- **监控（Monitoring**）： Sentinel 会不断地检查你的主服务器和从服务器是否运作正常。
- **提醒（Notification）**： 当被监控的某个 Redis 服务器出现问题时， Sentinel 可以通过 API 向管理员或者其他应用程序发送通知。
- **自动故障迁移（Automatic failover）**： 当一个主服务器不能正常工作时， Sentinel 会开始一次自动故障迁移操作， 它会将失效主服务器的其中一个从服务器升级为新的主服务器， 并让失效主服务器的其他从服务器改为复制新的主服务器； 当客户端试图连接失效的主服务器时， 集群也会向客户端返回新主服务器的地址， 使得集群可以使用新主服务器代替失效服务器。



为了保证 Sentinel 的高可用，Sentinel 也需要做集群部署，集群中至少需要三个Sentinel 实例（推荐奇数个）。

## 1、环境：

1、操作系统：Centos7.7

2、服务器配置如下：（本文采用了在单台机子上部署多个实例来模拟多机部署）

| 主机     |     ip      | 角色和端口 |
| -------- | :---------: | :--------- |
| master   | 172.27.0.13 | 6380       |
| slave1   | 172.27.0.13 | 6381       |
| slave2   | 172.27.0.13 | 6382       |
| Sentinel | 172.27.0.13 | 26380      |
| Sentinel | 172.27.0.13 | 26381      |
| Sentinel | 172.27.0.13 | 26382      |



## 2、redis安装

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

## 3、Redis单机多实例

以上为安装过程，下面开始进行redis的配置。

Redis 单机多实例部署方法十分简单，只要复制多个 redis 配置文件即可。需要注意每个实例的端口不能冲突。

```shell
[root@m soft]# cd /usr/local/soft

[root@m soft]# mkdir master_6380
[root@m soft]# mkdir slave1_6381
[root@m soft]# mkdir slave2_6382
[root@m soft]# ls
master_6380  redis-5.0.8  redis-5.0.8.tar.gz  slave1_6381  slave2_6382
[root@m soft]# mkdir sentinel_16380
[root@m soft]# mkdir sentinel_16381
[root@m soft]# mkdir sentinel_16382
```

### 3.1 Redis配置文件

1、复制Redis的配置文件`redis.conf`到master_6380目录中，

```shell
[root@m soft]# cp /usr/local/soft/redis-5.0.8/redis.conf /usr/local/soft/master_6380/
```

2、修改配置文件`redis.conf`文件:

* 实现后台启动 

```shell
daemonize no 改为 daemonize yes
```

* 需要将`bind 127.0.0.1`改成 `bind 0.0.0.0` 或注释，否则只能在本机访问；

* 其他配置修改：

```shell
#指定新的端口号
port 6380

#外部网络连接redis服务(no：可以连接)
protected-mode no

#指定新的PID文件路径
pidfile /var/run/redis_6380.pid

#本地数据库存放目录
dir /usr/local/soft/master_6380

#指定新的日志文件路径
logfile "/usr/local/soft/log/redis_6380.log"

 #指定新的转储文件路径
dbfilename dump_6380.rdb
```



3、把master_6380下的redis.conf分别复制到slave1_6381和 slave2_6382两个目录。

```shell
[root@m master_6380]# cp ./redis.conf ../slave1_6381/
[root@m master_6380]# cp ./redis.conf ../slave2_6382/
```



4、批量替换内容

```shell
[root@m master_6380]# cd /usr/local/soft
[root@m soft]# sed -i 's/6380/6381/g' slave1_6381/redis.conf
[root@m soft]# sed -i 's/6380/6382/g' slave2_6382/redis.conf
```

5、修改本地数据库存放目录

* slave1_6381下的`redis.conf`:

```shell
dir /usr/local/soft/slave1_6381
```

* slave2_6382下的`redis.conf`:

```shell
dir /usr/local/soft/slave2_6382
```

`重点`：

在slave的redis.conf配置文件中加入：

```shell
slaveof 172.27.0.13 6380
```



### 3.2 启动redis

```shell
[root@m soft]# /usr/local/soft/redis-5.0.8/src/redis-server /usr/local/soft/master_6380/redis.conf 
[root@m soft]# /usr/local/soft/redis-5.0.8/src/redis-server /usr/local/soft/slave1_6381/redis.conf 
[root@m soft]# /usr/local/soft/redis-5.0.8/src/redis-server /usr/local/soft/slave2_6382/redis.conf 

```

查看启动的进程

```shell
[root@m ~]# ps -ef|grep redis
root      9169     1  0 12:04 ?        00:00:01 /usr/local/soft/redis-5.0.8/src/redis-server *:6380
root     14179     1  0 12:06 ?        00:00:01 /usr/local/soft/redis-5.0.8/src/redis-server *:6381
root     19641     1  0 12:08 ?        00:00:01 /usr/local/soft/redis-5.0.8/src/redis-server *:6382
root     31704 31485  0 12:28 pts/9    00:00:00 grep --color=auto redis

```

### 3.3 建立master-slave关系

1、查看`master`主机

```shell
[root@m ~]# /usr/local/soft/redis-5.0.8/src/redis-cli -h 172.27.0.13 -p 6380
172.27.0.13:6380> info replication
# Replication
role:master
`connected_slaves:2`
slave0:ip=132.232.115.96,port=6381,state=online,offset=429,lag=0
slave1:ip=132.232.115.96,port=6382,state=online,offset=429,lag=1
master_replid:b985e172b1a1edc76f88b04aad2e06e86d4b424c
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:429
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:429
```

从上面的信息可以看出，master主机下有两台slave。

2、查看`salve`

```shell
[root@m ~]#  /usr/local/soft/redis-5.0.8/src/redis-cli -h 172.27.0.13 -p 6382
172.27.0.13:6382> info replication
# Replication
role:slave
master_host:132.232.115.96
master_port:6380
master_link_status:up
master_last_io_seconds_ago:9
master_sync_in_progress:0
slave_repl_offset:317
slave_priority:100
slave_read_only:1
connected_slaves:0
master_replid:b985e172b1a1edc76f88b04aad2e06e86d4b424c
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:317
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:234
repl_backlog_histlen:84
```

从上面的信息可以看出，6382端口下的Redis的主机端口号为6380。

## 4、Sentinel

### 4.1 Sentinel配置文件

在`sentinel_16380`文件下创建`sentinel.conf`文件：

```shell
[root@m sentinel_16380]# cd /usr/local/soft/sentinel_16380
[root@m sentinel_16380]# vim sentinel.conf
```



* 配置`sentinel.conf`文件

```shell
port   16380

dir "/usr/local/soft/sentinel_16380"
# sentinel启动后日志文件
logfile "/usr/local/soft/sentinel_16380/sentinel_16380.log"

# 启动sentinel是否是后台应用程序，默认是no，修改成yes后台启动
daemonize yes

# 格式：sentinel <option_name> <master_name> <option_value>；#该行的意思是：监控的master的名字叫做mymaster（自定义）,地址为127.0.0.1:6378 ，行尾最后的一个1代表在sentinel集群中，多少个sentinel认为masters死了，才能真正认为该master不可用了，该值需要小于当前到sentinel个数(基数个)，不然启动sentinel一直提示  +sentinel-address-switch 信息。
sentinel monitor mymaster 172.27.0.13 6380  2

# sentinel会向master发送心跳PING来确认master是否存活，如果master在“一定时间范围”内不回应PONG 或者是回复了一个错误消息，那么这个sentinel会主观地(单方面地)认为这个master已经不可用了(subjectively down, 也简称为SDOWN)。而这个down-after-milliseconds就是用来指定这个“一定时间范围”的，单位是毫秒，默认30秒。
sentinel down-after-milliseconds mymaster  15000

# failover过期时间，当failover开始后，在此时间内仍然没有触发任何failover操作，当前sentinel将会认为此次failoer失败。默认180秒，即3分钟。
sentinel failover-timeout mymaster  120000

# 在发生failover主备切换时，这个选项指定了最多可以有多少个slave同时对新的master进行同步，这个数字越小，完成failover所需的时间就越长，但是如果这个数字越大，就意味着越多的slave因为replication而不可用。可以通过将这个值设为 1 来保证每次只有一个slave处于不能处理命令请求的状态。
sentinel parallel-syncs mymaster  1
```

* 不带注释版：

```shell
port   16380

dir "/usr/local/soft/sentinel_16380"

logfile "/usr/local/soft/sentinel_16380/sentinel_16380.log"

daemonize yes

sentinel monitor mymaster 172.27.0.13 6380  2

sentinel down-after-milliseconds mymaster  15000

sentinel failover-timeout mymaster  120000

sentinel parallel-syncs mymaster  1
```



* 把sentinel_16380下的·分别复制到sentinel_16381和 sentinel_16382两个目录。

```shell
[root@m sentinel_16380]# cp /usr/local/soft/sentinel_16380/sentinel.conf /usr/local/soft/sentinel_16381/
[root@m sentinel_16380]# cp /usr/local/soft/sentinel_16380/sentinel.conf /usr/local/soft/sentinel_16382
```

* 批量替换内容

```shell
[root@m master_6380]# cd /usr/local/soft
[root@m soft]# sed -i 's/16380/16381/g' sentinel_16381/sentinel.conf
[root@m soft]# sed -i 's/16380/16382/g' sentinel_16382/sentinel.conf
```

###  4.2 启动sentinel

```shell
[root@m soft]# /usr/local/soft/redis-5.0.8/src/redis-sentinel /usr/local/soft/sentinel_16380/sentinel.conf 
[root@m soft]# /usr/local/soft/redis-5.0.8/src/redis-sentinel /usr/local/soft/sentinel_16381/sentinel.conf 
[root@m soft]# /usr/local/soft/redis-5.0.8/src/redis-sentinel /usr/local/soft/sentinel_16382/sentinel.conf 
```

查看启动进程：

```shell
[root@m ~]# ps -ef|grep redis
root      5006     1  0 12:16 ?        00:00:01 /usr/local/soft/redis-5.0.8/src/redis-sentinel *:16380 [sentinel]
root      9169     1  0 12:04 ?        00:00:01 /usr/local/soft/redis-5.0.8/src/redis-server *:6380
root     14179     1  0 12:06 ?        00:00:01 /usr/local/soft/redis-5.0.8/src/redis-server *:6381
root     19641     1  0 12:08 ?        00:00:01 /usr/local/soft/redis-5.0.8/src/redis-server *:6382
root     26835     1  0 12:26 ?        00:00:00 /usr/local/soft/redis-5.0.8/src/redis-sentinel *:16381 [sentinel]
root     28779     1  0 12:26 ?        00:00:00 /usr/local/soft/redis-5.0.8/src/redis-sentinel *:16382 [sentinel]
root     31704 31485  0 12:28 pts/9    00:00:00 grep --color=auto redis

```

再次查看`sentinel.conf`配置文件（此处为16380的配置文件，其他的类似）

```shell
port 16380

dir "/usr/local/soft/sentinel_16380"
logfile "/usr/local/soft/sentinel_16380/sentinel_16380.log"

daemonize yes

sentinel myid 13ada554be134d6ce80315d87e59275bc7aba142

sentinel deny-scripts-reconfig yes

sentinel monitor mymaster 132.232.115.96 6380 2

sentinel down-after-milliseconds mymaster 15000
# Generated by CONFIG REWRITE
protected-mode no
sentinel failover-timeout mymaster 120000
sentinel config-epoch mymaster 0
sentinel leader-epoch mymaster 0
#从节点的信息
sentinel known-replica mymaster 132.232.115.96 6381
sentinel known-replica mymaster 132.232.115.96 6382
#其他sentinel的信息
sentinel known-sentinel mymaster 172.27.0.13 16382 5144e1f35b2063021bbf3574e8f2008aa28d7af3
sentinel known-sentinel mymaster 172.27.0.13 16381 631a777dd5fa2c2ce00af5dc81070687b980d17f
sentinel current-epoch 0

```

经过上文发现配置文件已经跟原来不一样了，增添了一些内容。加入了redis的slave节点的信息，以及其他sentinel的信息。

### 4.3 查看sentinel的信息



```shell
[root@m ~]# /usr/local/soft/redis-5.0.8/src/redis-cli -h 132.232.115.96 -p 16380
132.232.115.96:16380> info sentinel
# Sentinel
sentinel_masters:1
sentinel_tilt:0
sentinel_running_scripts:0
sentinel_scripts_queue_length:0
sentinel_simulate_failure_flags:0
master0:name=mymaster,status=ok,address=132.232.115.96:6382,slaves=2,sentinels=3
```

可以看出该sentinel包含1个master、2个slave、3个sentinel。

下面先来看一下master的信息：

```shell
[root@m ~]# /usr/local/soft/redis-5.0.8/src/redis-cli -h 132.232.115.96 -p 16380
132.232.115.96:16380> sentinel masters
1)  1) "name"
    2) "mymaster"
    3) "ip"
    4) "132.232.115.96"
    5) "port"
    6) "6380"
    7) "runid"
    8) "59040849935cbbb944cae708249ce066781ea518"
    9) "flags"
   10) "master"
   。。。。。。。。。。。。。。。。。。。。。。。。。。。
```

再来看一下salve的信息：

```shell
[root@m ~]# /usr/local/soft/redis-5.0.8/src/redis-cli -h 132.232.115.96 -p 16380
132.232.115.96:16380> sentinel slaves  masters
(error) ERR No such master with that name
132.232.115.96:16380> sentinel slaves mymaster
1)  1) "name"
    2) "132.232.115.96:6381"
    3) "ip"
    4) "132.232.115.96"
    5) "port"
    6) "6381"
    7) "runid"
    8) "aeb68bd611a9f3420a5976a3678376ed20746de1"
    9) "flags"
   10) "slave"
...................................
2)  1) "name"
    2) "132.232.115.96:6382"
    3) "ip"
    4) "132.232.115.96"
    5) "port"
    6) "6382"
    7) "runid"
    8) "54db1272beaa679ab3d2fa4ed8d4e7572b28dcf3"
    9) "flags"
   10) "slave"
 ......................................
```

## 5、故障转移

下面开始模拟master宕机，使用sentinel进行故障转移的案例。

### 5.1 master宕机

1、查看进程

```shell
[root@m ~]# ps -ef|grep redis
root      5006     1  0 12:16 ?        00:00:01 /usr/local/soft/redis-5.0.8/src/redis-sentinel *:16380 [sentinel]
root      9169     1  0 12:04 ?        00:00:01 /usr/local/soft/redis-5.0.8/src/redis-server *:6380
root     14179     1  0 12:06 ?        00:00:01 /usr/local/soft/redis-5.0.8/src/redis-server *:6381
root     19641     1  0 12:08 ?        00:00:01 /usr/local/soft/redis-5.0.8/src/redis-server *:6382
root     26835     1  0 12:26 ?        00:00:00 /usr/local/soft/redis-5.0.8/src/redis-sentinel *:16381 [sentinel]
root     28779     1  0 12:26 ?        00:00:00 /usr/local/soft/redis-5.0.8/src/redis-sentinel *:16382 [sentinel]
root     31704 31485  0 12:28 pts/9    00:00:00 grep --color=auto redis

```

2、将master端口为6380的进程kill掉：

```shell
[root@m ~]# kill -9 9169
```

然后再查看一下进程，发现6380的进程已经被kill掉，表明master已宕机。

3、查看sentinel的日志

* 先来查看16380端口的sentinel的日志：

  ```shell
  5006:X 10 Apr 2020 12:31:00.605 # +sdown master mymaster 132.232.115.96 6380
  5006:X 10 Apr 2020 12:31:00.777 # +new-epoch 1
  5006:X 10 Apr 2020 12:31:00.786 # +vote-for-leader 5144e1f35b2063021bbf3574e8f2008aa28d7af3 1
  5006:X 10 Apr 2020 12:31:01.534 # +config-update-from sentinel 5144e1f35b2063021bbf3574e8f2008aa28d7af3 172.27.0.13 16382 @ mymaster 132.232.115.96 6380
  5006:X 10 Apr 2020 12:31:01.534 # +switch-master mymaster 132.232.115.96 6380 132.232.115.96 6382
  5006:X 10 Apr 2020 12:31:01.534 * +slave slave 132.232.115.96:6381 132.232.115.96 6381 @ mymaster 132.232.115.96 6382
  5006:X 10 Apr 2020 12:31:01.534 * +slave slave 132.232.115.96:6380 132.232.115.96 6380 @ mymaster 132.232.115.96 6382
  5006:X 10 Apr 2020 12:31:16.570 # +sdown slave 132.232.115.96:6380 132.232.115.96 6380 @ mymaster 132.232.115.96 6382
  ```



* 查看16381端口的sentinel的日志：

```shell
26835:X 10 Apr 2020 12:31:00.658 # +sdown master mymaster 132.232.115.96 6380
26835:X 10 Apr 2020 12:31:00.748 # +odown master mymaster 132.232.115.96 6380 #quorum 3/2
26835:X 10 Apr 2020 12:31:00.748 # +new-epoch 1
26835:X 10 Apr 2020 12:31:00.748 # +try-failover master mymaster 132.232.115.96 6380
26835:X 10 Apr 2020 12:31:00.761 # +vote-for-leader 631a777dd5fa2c2ce00af5dc81070687b980d17f 1
26835:X 10 Apr 2020 12:31:00.768 # 5144e1f35b2063021bbf3574e8f2008aa28d7af3 voted for 5144e1f35b2063021bbf3574e8f2008aa28d7af3 1
26835:X 10 Apr 2020 12:31:00.790 # 13ada554be134d6ce80315d87e59275bc7aba142 voted for 5144e1f35b2063021bbf3574e8f2008aa28d7af3 1
26835:X 10 Apr 2020 12:31:01.532 # +config-update-from sentinel 5144e1f35b2063021bbf3574e8f2008aa28d7af3 172.27.0.13 16382 @ mymaster 132.232.115.96 6380
26835:X 10 Apr 2020 12:31:01.532 # +switch-master mymaster 132.232.115.96 6380 132.232.115.96 6382
26835:X 10 Apr 2020 12:31:01.532 * +slave slave 132.232.115.96:6381 132.232.115.96 6381 @ mymaster 132.232.115.96 6382
26835:X 10 Apr 2020 12:31:01.533 * +slave slave 132.232.115.96:6380 132.232.115.96 6380 @ mymaster 132.232.115.96 6382
26835:X 10 Apr 2020 12:31:16.599 # +sdown slave 132.232.115.96:6380 132.232.115.96 6380 @ mymaster 132.232.115.96 6382

```

* 查看16382端口的sentinel的日志：

```shell
`发现 master 检测失败，主观认为该节点挂掉，进入 sdown 状态。`
28779:X 10 Apr 2020 12:31:00.654 # +sdown master mymaster 132.232.115.96 6380
`有两个 sentinel 节点认为 master 6380 挂掉，达到配置的 quorum 值 2，因此认为 master 已经客观挂掉，进入 odown 状态。`
28779:X 10 Apr 2020 12:31:00.745 # +odown master mymaster 132.232.115.96 6380 #quorum 2/2
28779:X 10 Apr 2020 12:31:00.745 # +new-epoch 1
`开始恢复故障`
28779:X 10 Apr 2020 12:31:00.745 # +try-failover master mymaster 132.232.115.96 6380
`准备选举一个 sentinel leader 来开始 failover。`
28779:X 10 Apr 2020 12:31:00.761 # +vote-for-leader 5144e1f35b2063021bbf3574e8f2008aa28d7af3 1
28779:X 10 Apr 2020 12:31:00.766 # 631a777dd5fa2c2ce00af5dc81070687b980d17f voted for 631a777dd5fa2c2ce00af5dc81070687b980d17f 1
28779:X 10 Apr 2020 12:31:00.789 # 13ada554be134d6ce80315d87e59275bc7aba142 voted for 5144e1f35b2063021bbf3574e8f2008aa28d7af3 1
`选举节点后替换3680`
28779:X 10 Apr 2020 12:31:00.833 # +elected-leader master mymaster 132.232.115.96 6380
`开始为故障的master选举`
28779:X 10 Apr 2020 12:31:00.833 # +failover-state-select-slave master mymaster 132.232.115.96 6380
`节点选举结果，选中132.232.115.96:6382来替换master`
28779:X 10 Apr 2020 12:31:00.913 # +selected-slave slave 132.232.115.96:6382 132.232.115.96 6382 @ mymaster 132.232.115.96 6380
`确认节点选举结果为6382`
28779:X 10 Apr 2020 12:31:00.913 * +failover-state-send-slaveof-noone slave 132.232.115.96:6382 
`选中的节点6382正在升级为master`
132.232.115.96 6382 @ mymaster 132.232.115.96 6380
28779:X 10 Apr 2020 12:31:01.013 * +failover-state-wait-promotion slave 132.232.115.96:6382 132.232.115.96 6382 @ mymaster 132.232.115.96 6380
`选中的节点已成功升级为master`
28779:X 10 Apr 2020 12:31:01.469 # +promoted-slave slave 132.232.115.96:6382 132.232.115.96 6382 @ mymaster 132.232.115.96 6380
`切换故障master的状态`
28779:X 10 Apr 2020 12:31:01.470 # +failover-state-reconf-slaves master mymaster 132.232.115.96 6380
`其他节点同步故障master信息`
28779:X 10 Apr 2020 12:31:01.530 * +slave-reconf-sent slave 132.232.115.96:6381 132.232.115.96 6381 @ mymaster 132.232.115.96 6380
28779:X 10 Apr 2020 12:31:01.889 # -odown master mymaster 132.232.115.96 6380
28779:X 10 Apr 2020 12:31:02.469 * +slave-reconf-inprog slave 132.232.115.96:6381 132.232.115.96 6381 @ mymaster 132.232.115.96 6380
`其他节点完成故障master的同步`
28779:X 10 Apr 2020 12:31:02.469 * +slave-reconf-done slave 132.232.115.96:6381 132.232.115.96 6381 @ mymaster 132.232.115.96 6380
`故障恢复完成`
28779:X 10 Apr 2020 12:31:02.529 # +failover-end master mymaster 132.232.115.96 6380
`master从132.232.115.96:6380  变为 132.232.115.96:6382`
28779:X 10 Apr 2020 12:31:02.529 # +switch-master mymaster 132.232.115.96 6380 132.232.115.96 6382
`为其他节点指定新的master`
28779:X 10 Apr 2020 12:31:02.529 * +slave slave 132.232.115.96:6381 132.232.115.96 6381 @ mymaster 132.232.115.96 6382
`故障master指定新的master`
28779:X 10 Apr 2020 12:31:02.530 * +slave slave 132.232.115.96:6380 132.232.115.96 6380 @ mymaster 132.232.115.96 6382
`132.232.115.96:6380宕机，待恢复`
28779:X 10 Apr 2020 12:31:17.561 # +sdown slave 132.232.115.96:6380 132.232.115.96 6380 @ mymaster 132.232.115.96 6382

```



经过上面3个sentinel的日志分析可以，可以看出16382是作为最后sentinel选举出来的sentinel进行了master的制定，它制定了6382作为新的master。通过16382的日志可以相信看出整个sentinel故障转移的机制。简单总结一下：

1)、发现master宕机了，进行主观下线；

2)、获取其他sentinel对master的判断，确定宕机后，进行客观下线；

3)、通过Raft算法选举一个sentinel作为leader；

（Raft 的核心思想：先到先得，少数服从多数。 Raft 算法演示： http://thesecretlivesofdata.com/raft/ ）

4)、该leader从salve中选出一个作为master；

5)、是其他slave节点成为新的master节点的从节点；

6)、将原来的master节点也改为新的master节点的从节点，这样在启动后，它将作为新master节点的salve节点。



4、查看sentinel的信息：

* 查看master信息

```shell
132.232.115.96:16380> sentinel masters
1)  1) "name"
    2) "mymaster"
    3) "ip"
    4) "132.232.115.96"
    5) "port"
    6) "6382"
    7) "runid"
    8) "54db1272beaa679ab3d2fa4ed8d4e7572b28dcf3"
   。。。。。。。。。。。。

```

* 查看slaves信息

```shell
132.232.115.96:16380> sentinel slaves mymaster
1)  1) "name"
    2) "132.232.115.96:6381"
    3) "ip"
    4) "132.232.115.96"
    5) "port"
    6) "6381"
    7) "runid"
    8) "aeb68bd611a9f3420a5976a3678376ed20746de1"
    9) "flags"
   10) "slave"
 。。。。。。。。。。。。。。。。。。
2)  1) "name"
    2) "132.232.115.96:6380"
    3) "ip"
    4) "132.232.115.96"
    5) "port"
    6) "6380"
    7) "runid"
    8) ""
    9) "flags"
   10) "s_down,slave,disconnected"
  。。。。。。。。。。。。。。。。。。。。。
```

通过上面的sentinel信息可以看出，当前master节点已经从6380成功转移到6382；

下面再看下Redis的6380的日志：

```shell
19641:S 10 Apr 2020 12:30:58.210 * Connecting to MASTER 132.232.115.96:6380
19641:S 10 Apr 2020 12:30:58.210 * MASTER <-> REPLICA sync started
19641:S 10 Apr 2020 12:30:58.213 # Error condition on socket for SYNC: Connection refused
19641:S 10 Apr 2020 12:30:59.223 * Connecting to MASTER 132.232.115.96:6380
19641:S 10 Apr 2020 12:30:59.223 * MASTER <-> REPLICA sync started
19641:S 10 Apr 2020 12:30:59.225 # Error condition on socket for SYNC: Connection refused
19641:S 10 Apr 2020 12:31:00.240 * Connecting to MASTER 132.232.115.96:6380
19641:S 10 Apr 2020 12:31:00.240 * MASTER <-> REPLICA sync started
19641:S 10 Apr 2020 12:31:00.243 # Error condition on socket for SYNC: Connection refused
19641:M 10 Apr 2020 12:31:01.016 # Setting secondary replication ID to b985e172b1a1edc76f88b04aad2e06e86d4b424c, valid up to offset: 94337. New replication ID is 9fbd24fad4319da8a4516aac12991e444e060435
19641:M 10 Apr 2020 12:31:01.016 * Discarding previously cached master state.
19641:M 10 Apr 2020 12:31:01.016 * MASTER MODE enabled (user request from 'id=12 addr=132.232.115.96:36502 fd=14 name=sentinel-5144e1f3-cmd age=242 idle=1 flags=x db=0 sub=0 psub=0 multi=3 qbuf=140 qbuf-free=32628 obl=36 oll=0 omem=0 events=r cmd=exec')
19641:M 10 Apr 2020 12:31:01.018 # CONFIG REWRITE executed with success.
19641:M 10 Apr 2020 12:31:01.912 * Replica 132.232.115.96:6381 asks for synchronization
19641:M 10 Apr 2020 12:31:01.912 * Partial resynchronization request from 132.232.115.96:6381 accepted. Sending 163 bytes of backlog starting from offset 94337.

```



### 5.2 重启原来故障的6380 master服务

```shell
`重启服务`
[root@m ~]# /usr/local/soft/redis-5.0.8/src/redis-server /usr/local/soft/master_6380/redis.conf
`查看sentinel的master信息`
[root@m ~]# /usr/local/soft/redis-5.0.8/src/redis-cli -h 132.232.115.96 -p 16380
132.232.115.96:16380> sentinel masters
1)  1) "name"
    2) "mymaster"
    3) "ip"
    4) "132.232.115.96"
    5) "port"
    6) "6382"
    7) "runid"
    8) "54db1272beaa679ab3d2fa4ed8d4e7572b28dcf3"
    9) "flags"
   10) "master"
   ............................
`查看sentinel的slaves信息`
132.232.115.96:16380> sentinel slaves mymaster
1)  1) "name"
    2) "132.232.115.96:6381"
    3) "ip"
    4) "132.232.115.96"
    5) "port"
    6) "6381"
    7) "runid"
    8) "aeb68bd611a9f3420a5976a3678376ed20746de1"
    9) "flags"
   10) "slave"
..................................
2)  1) "name"
    2) "132.232.115.96:6380"
    3) "ip"
    4) "132.232.115.96"
    5) "port"
    6) "6380"
    7) "runid"
    8) "858b148a5959a88d0ec96d7371a1b608728ff92e"
    9) "flags"
   10) "slave"
...................................  
```

从上面的信息可以看出，端口为6382的master不会变，原来6380的master变成一个salve节点。

最后我们到redis的6382实例中查看一下信息。如下：

```shell
[root@m ~]# /usr/local/soft/redis-5.0.8/src/redis-cli -h 132.232.115.96 -p 6382
132.232.115.96:6382> info replication
# Replication
role:master
connected_slaves:2
slave0:ip=132.232.115.96,port=6381,state=online,offset=686750,lag=0
slave1:ip=132.232.115.96,port=6380,state=online,offset=686750,lag=0
master_replid:9fbd24fad4319da8a4516aac12991e444e060435
master_replid2:b985e172b1a1edc76f88b04aad2e06e86d4b424c
master_repl_offset:686750
second_repl_offset:94337
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:234
repl_backlog_histlen:686517
```



### 5.3 master宕机

1、关闭6380从服务器：

```shell
[root@m sentinel_16382]# /usr/local/soft/redis-5.0.8/src/redis-cli -h 132.232.115.96 -p 6380 shutdown
```

2、查看master信息

```shell
[root@m sentinel_16382]# /usr/local/soft/redis-5.0.8/src/redis-cli -h 132.232.115.96 -p 6382
132.232.115.96:6382> info  replication
# Replication
role:master
`connected_slaves:1`
slave0:ip=132.232.115.96,port=6381,state=online,offset=4194170,lag=0
master_replid:9fbd24fad4319da8a4516aac12991e444e060435
master_replid2:b985e172b1a1edc76f88b04aad2e06e86d4b424c
master_repl_offset:4194170
second_repl_offset:94337
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:3145595
repl_backlog_histlen:1048576

```

此时发现只有一个salve了。



3、查看sentinel信息（3台相同）

```shell
28779:X 10 Apr 2020 17:59:48.732 # +sdown slave 132.232.115.96:6380 132.232.115.96 6380 @ mymaster 132.232.115.96 6382
```

直接下线salve节点。

4、启动salve服务器

```shell
[root@m sentinel_16382]# /usr/local/soft/redis-5.0.8/src/redis-server /usr/local/soft/master_6380/redis.conf

```

1>master主机日志

```shell
`发送数据同步请求`
19641:M 10 Apr 2020 18:07:20.898 * Replica 132.232.115.96:6380 asks for synchronization
19641:M 10 Apr 2020 18:07:20.898 * Partial resynchronization request from 132.232.115.96:6380 accepted. Sendin offset 4175145.

```



2>sentinel日志（3台相同）

```shell
28779:X 10 Apr 2020 18:07:21.863 * +reboot slave 132.232.115.96:6380 132.232.115.96 6380 @ mymaster 132.232.115.96 6382
28779:X 10 Apr 2020 18:07:21.947 # -sdown slave 132.232.115.96:6380 132.232.115.96 6380 @ mymaster 132.232.115.96 6382

```

3>查看master信息

```shell
[root@m sentinel_16382]# /usr/local/soft/redis-5.0.8/src/redis-cli -h 132.232.115.96 -p 6382
132.232.115.96:6382> info replication
# Replication
role:master
connected_slaves:2
slave0:ip=132.232.115.96,port=6381,state=online,offset=4533264,lag=0
slave1:ip=132.232.115.96,port=6380,state=online,offset=4533264,lag=1
master_replid:9fbd24fad4319da8a4516aac12991e444e060435
master_replid2:b985e172b1a1edc76f88b04aad2e06e86d4b424c
master_repl_offset:4533264
second_repl_offset:94337
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:3484689
repl_backlog_histlen:1048576

```

可以看到salve变为了两台。

4>查看sentinel信息

```shell
[root@m sentinel_16382]# /usr/local/soft/redis-5.0.8/src/redis-cli -h 132.232.115.96 -p 16380
132.232.115.96:16380> sentinel slaves  mymaster
1)  1) "name"
    2) "132.232.115.96:6381"
    3) "ip"
    4) "132.232.115.96"
    5) "port"
    6) "6381"
    7) "runid"
    8) "aeb68bd611a9f3420a5976a3678376ed20746de1"
    9) "flags"
   10) "slave"
。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。
2)  1) "name"
    2) "132.232.115.96:6380"
    3) "ip"
    4) "132.232.115.96"
    5) "port"
    6) "6380"
    7) "runid"
    8) "bfcd536783612a79b5928257d0f84dad243a2f5d"
    9) "flags"
   10) "slave"
  ..............................
```

上面可以看到两台slave的信息。



## 结语：

到此Redis的sentinel就搭建完成了。
