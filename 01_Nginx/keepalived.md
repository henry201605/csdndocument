Nat方式搭建

DR模式搭建



# LVS+KeepAlived+Nginx+Tomcat高可用解决方案

## 1、环境：

Centos7.6

| 服务器            | 软件           |
| :---------------- | :------------- |
| 192.168.0.13      | LVS+KeepAlived |
| 192.168.0.14      | LVS+KeepAlived |
| 192.168.0.15      | Nginx          |
| 192.168.0.16      | Nginx          |
|                   | Tomcat         |
|                   | Tomcat         |
| VIP 192.168.1.200 |                |

```shell
#!/bin/bash
# 检查脑裂的脚本，在备节点上进行部署
LB01_VIP=192.168.1.200
LB01_IP=192.168.0.13
LB02_IP=192.168.0.14
while true
do
  ping -c 2 -W 3 $LB01_VIP &>/dev/null
    if [ $? -eq 0 -a `ip add|grep "$LB01_VIP"|wc -l` -eq 1 ];then
        echo "ha is brain."
    else
        echo "ha is ok"
    fi
    sleep 5
done
```



## 2、安装lvs+keepalived

本文安装采用源代码编译方式进行安装。

### 2.1 Lvs

从2.4版本开始，linux内核默认支持LVS。要使用LVS的能力，只需安装一个LVS的管理工具：ipvsadm。

```shell
yum -y install ipvsadm
```



### 2.2 keepalived

同时在192.168.0.13和192.168.0.14两台服务器上操作：

2.2.1 下载

```shell
`进入到/usr/local/src目录下`
[root@henry004 ~]# cd /usr/local/src

`下载keepalived`
[root@henry004 src]# wget https://www.keepalived.org/software/keepalived-2.0.20.tar.gz

`解压缩`
[root@henry001 src]# tar -zxvf keepalived-2.0.20.tar.gz


```

2.2.1 安装

```shell
`在/usr/local目录下创建keepalived文件夹`
[root@henry001 keepalived-2.0.20]# mkdir /usr/local/keepalived

`将keepalived安装到/usr/local/keepalived下，conf配置文件指定到目录/etc下`
[root@henry001 keepalived-2.0.20]# ./configure --prefix=/usr/local/keepalived --sysconf=/etc

`编译安装`
[root@henry004 keepalived-2.0.20]# make && make install
```



编译过程中可能会出现如下常见问题：

1、缺少OpenSSL

```shell
`-------错误信息---------------`
hecking openssl/ssl.h usability... no
checking openssl/ssl.h presence... no
checking for openssl/ssl.h... no
configure: error: 
  !!! OpenSSL is not properly installed on your system. !!!
  !!! Can not include OpenSSL headers files.            !!!
  
`----- ---解决方案--------------------`
  yum -y install openssl-devel
```

2、缺少libnl/libnl-3

```shell
`--------错误信息---------------`
*** WARNING - this build will not support IPVS with IPv6. Please install libnl/libnl-3 dev libraries to support IPv6 with IPVS.

`-------解决方案--------------------`
yum -y install libnl libnl-devel
```



2.2.3 配置

```shell
`进入安装后的路径 cd /data/program/keepalived, 创建软连接`
[root@henry001 sbin]# ln -s /usr/local/keepalived/sbin/keepalived  /sbin/

`把 keepalived的启动文件复制到init.d下，加入开机启动项`
[root@henry001 keepalived-2.0.20]# cp /usr/local/src/keepalived-2.0.20/keepalived/etc/init.d/keepalived /etc/init.d

`添加keepalived到系统服务`
[root@henry001 sbin]# chkconfig –add keepalived
chkconfig version 1.7.4 - Copyright (C) 1997-2000 Red Hat, Inc.
This may be freely redistributed under the terms of the GNU Public License.

usage:   chkconfig [--list] [--type <type>] [name]
         chkconfig --add <name>
         chkconfig --del <name>
         chkconfig --override <name>
         chkconfig [--level <levels>] [--type <type>] <name> <on|off|reset|resetpriorities>
`检测是否添加成功`
[root@henry001 sbin]# chkconfig keepalived on
Note: Forwarding request to 'systemctl enable keepalived.service'.
Created symlink from /etc/systemd/system/multi-user.target.wants/keepalived.service to /usr/lib/systemd/system/keepalived.service.

```



2.2.4 启动keepalived

```shell
`启动keepalived服务`
[root@henry001 sbin]# systemctl start keepalived.service
`查看keepalived状态`
[root@henry001 sbin]# systemctl status keepalived.service
● keepalived.service - LVS and VRRP High Availability Monitor
   Loaded: loaded (/usr/lib/systemd/system/keepalived.service; enabled; vendor preset: disabled)
   Active: active (running) since Sat 2020-03-07 22:13:54 CST; 3s ago
  Process: 25684 ExecStart=/usr/local/keepalived/sbin/keepalived $KEEPALIVED_OPTIONS (code=exited, status=0/SUCCESS)
 Main PID: 25685 (keepalived)
   CGroup: /system.slice/keepalived.service
           ├─25685 /usr/local/keepalived/sbin/keepalived -D
           ├─25686 /usr/local/keepalived/sbin/keepalived -D
           └─25687 /usr/local/keepalived/sbin/keepalived -D

-------------------------------------------------
```

操作keepalived的命令有如下：

```shell
`----启动-----`
systemctl start keepalived.service
`----重启-----
systemctl restart keepalived.service
`----停止-----`
systemctl stop keepalived.service
`----查看状态-----`
systemctl status keepalived.service
```



### 2.3 防火墙

为方便测试，我直接关闭了防火墙，在实际应用中可以根据需要开启防火墙的端口，此外还要设置服务器的安全策略，我的是阿里云的服务器，就在阿里云服务器控制台设置了安全策略，开放了需要的端口。

关闭防火墙：

```shell
`关闭防火墙`
[root@henry001 sysconfig]# systemctl stop firewalld

`查看防火墙状态`
[root@henry001 sysconfig]# systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)

Mar 07 22:39:08 henry001 systemd[1]: Starting firewalld - dynamic firewall daemon...
Mar 07 22:39:08 henry001 systemd[1]: Started firewalld - dynamic firewall daemon.
Mar 07 22:39:23 henry001 systemd[1]: Stopping firewalld - dynamic firewall daemon...
Mar 07 22:39:24 henry001 systemd[1]: Stopped firewalld - dynamic firewall daemon.
```



## 3、安装Nginx

同时在192.168.0.15和192.168.0.16两台服务器上操作，对Nginx+Tomcat的安装请参考文章：



## 4、配置keepalived

### 4.1master

先来配置192.168.0.13的主机，指定其为master服务器；

```shell
`进入配置文件目录`
[root@henry001 ~]#  cd /etc/keepalived
[root@henry001 keepalived]# ls
keepalived.conf  samples
`编辑配置文件信息`
[root@henry001 keepalived]# vim keepalived.conf 


```

编辑keepalived.conf 文件

```shell
global_defs {
   #notification_email {
   #      edisonchou@hotmail.com
   #}
  # notification_email_from sns-lvs@gmail.com
  # smtp_server 192.168.80.1
   #smtp_connection_timeout 30
   router_id LVS_DEVEL  # 设置lvs的id，在一个网络内应该是唯一的
}
vrrp_instance VI_1 {
    state MASTER   #指定Keepalived的角色，MASTER为主，BACKUP为备 记得大写
    interface eth0  #网卡id 不同的电脑网卡id会有区别 可以使用:ip a查看
    virtual_router_id 51  #虚拟路由编号，主备要一致
    priority 100  #定义优先级，数字越大，优先级越高，主DR必须大于备用DR
    advert_int 1  #检查间隔，默认为1s
    authentication {   #这里配置的密码最多为8位，主备要一致，否则无法正常通讯
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.0.200  #定义虚拟IP(VIP)为192.168.1.200，可多设，每行一个
    }
}
# 定义对外提供服务的LVS的VIP以及port
virtual_server 192.168.0.200 80 {
    delay_loop 6 # 设置健康检查时间，单位是秒
    lb_algo rr # 设置负载调度的算法为wlc
    lb_kind DR # 设置LVS实现负载的机制，有NAT、TUN、DR三个模式
    #nat_mask 255.255.255.0
    persistence_timeout 0
    protocol TCP
    real_server 192.168.0.15 80 {  # 指定real server1的IP地址
        weight 3   # 配置节点权值，数字越大权重越高
        TCP_CHECK {
        connect_timeout 10
        nb_get_retry 3
        delay_before_retry 3
        connect_port 80
        }
    }
    real_server 192.168.0.16 80 {  # 指定real server2的IP地址
        weight 3  # 配置节点权值，数字越大权重越高
        TCP_CHECK {
        connect_timeout 10
        nb_get_retry 3
        delay_before_retry 3
        connect_port 80
        }
     }
}

```



### 4.1 backup

配置192.168.0.14的主机，指定其为backup服务器；

```shell
`进入配置文件目录`
[root@henry004 ~]#  cd /etc/keepalived
[root@henry004 keepalived]# ls
keepalived.conf  samples
`编辑配置文件信息`
[root@henry004 keepalived]# vim keepalived.conf 

```

编辑keepalived.conf 文件

```yaml

global_defs {
   #notification_email {
   #      edisonchou@hotmail.com
   #}
  # notification_email_from sns-lvs@gmail.com
  #  smtp_server 192.168.80.1
  #  smtp_connection_timeout 30
   router_id LVS_DEVEL  # 设置lvs的id，在一个网络内应该是唯一的
}
vrrp_instance VI_1 {
    state BACKUP #指定Keepalived的角色，MASTER为主，BACKUP为备 记得大写
    interface eth0  #网卡id 不同的电脑网卡id会有区别 可以使用:ip a查看
    virtual_router_id 51  #虚拟路由编号，主备要一致
    priority 50  #定义优先级，数字越大，优先级越高，主DR必须大于备用DR
    advert_int 1  #检查间隔，默认为1s
    authentication {   #这里配置的密码最多为8位，主备要一致，否则无法正常通讯
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.0.200  #定义虚拟IP(VIP)为192.168.1.200，可多设，每行一个
    }
}
# 定义对外提供服务的LVS的VIP以及port
virtual_server 192.168.0.200 80 {
    delay_loop 6 # 设置健康检查时间，单位是秒
    lb_algo rr # 设置负载调度的算法为wlc
    lb_kind DR # 设置LVS实现负载的机制，有NAT、TUN、DR三个模式
    #nat_mask 255.255.255.0
    persistence_timeout 0
    protocol TCP
    real_server 192.168.0.16 80 {  # 指定real server1的IP地址
        weight 3   # 配置节点权值，数字越大权重越高
        TCP_CHECK {
        connect_timeout 10
        nb_get_retry 3
        delay_before_retry 3
        connect_port 80
        }
    }
    real_server 192.168.0.15 80 {  # 指定real server2的IP地址
        weight 3  # 配置节点权值，数字越大权重越高
        TCP_CHECK {
        connect_timeout 10
        nb_get_retry 3
        delay_before_retry 3
        connect_port 80
        }
     }
}

```



## 5、



查看master服务器：

```shell
[root@henry001 keepalived]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:16:3e:30:cc:a2 brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.13/24 brd 192.168.0.255 scope global dynamic eth0
       valid_lft 315332386sec preferred_lft 315332386sec
    inet 192.168.1.200/32 scope global eth0
       valid_lft forever preferred_lft forever

```





查看backup服务器

```shell
[root@henry004 keepalived]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:16:3e:30:9f:0f brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.14/24 brd 192.168.0.255 scope global dynamic eth0
       valid_lft 315348979sec preferred_lft 315348979sec

```



下面我们停止掉master服务器上的keepalived,虚拟ip将会漂移到backup服务器上

```shell
[root@henry004 keepalived]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:16:3e:30:9f:0f brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.14/24 brd 192.168.0.255 scope global dynamic eth0
       valid_lft 315348845sec preferred_lft 315348845sec
       `虚拟ip`
    inet 192.168.0.200/32 scope global eth0
       valid_lft forever preferred_lft forever

```



此时发现在master和backup主机都发现了eth0网卡中出现了虚拟ip,也就是发生了脑裂的问题，解决方案请参考：





```shell
[root@henry001 ~]# tcpdump -i eth0 vrrp -n
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
11:45:59.820681 IP 192.168.0.13 > 224.0.0.18: VRRPv2, Advertisement, vrid 51, prio 100, authtype simple, intvl 1s, length 20
11:46:00.820717 IP 192.168.0.13 > 224.0.0.18: VRRPv2, Advertisement, vrid 51, prio 100, authtype simple, intvl 1s, length 20
11:46:01.820750 IP 192.168.0.13 > 224.0.0.18: VRRPv2, Advertisement, vrid 51, prio 100, authtype simple, intvl 1s, length 20
11:46:02.820784 IP 192.168.0.13 > 224.0.0.18: VRRPv2, Advertisement, vrid 51, prio 100, authtype simple, intvl 1s, length 20
11:46:03.820827 IP 192.168.0.13 > 224.0.0.18: VRRPv2, Advertisement, vrid 51, prio 100, authtype simple, intvl 1s, length 20
11:46:04.820868 IP 192.168.0.13 > 224.0.0.18: VRRPv2, Advertisement, vrid 51, prio 100, authtype simple, intvl 1s, length 20
11:46:05.820907 IP 192.168.0.13 > 224.0.0.18: VRRPv2, Advertisement, vrid 51, prio 100, authtype simple, intvl 1s, length 20

```





```shell
`查看日志`
[root@henry001 ~]# tail -f /var/log/messages

..................

Mar  8 11:48:26 henry001 Keepalived_vrrp[30034]: Opening file '/etc/keepalived/keepalived.conf'.
Mar  8 11:48:26 henry001 Keepalived_vrrp[30034]: Assigned address 192.168.0.13 for interface eth0
Mar  8 11:48:26 henry001 Keepalived_vrrp[30034]: Registering gratuitous ARP shared channel
Mar  8 11:48:26 henry001 Keepalived_vrrp[30034]: (VI_1) removing VIPs.
Mar  8 11:48:26 henry001 Keepalived_healthcheckers[30033]: Opening file '/etc/keepalived/keepalived.conf'.
Mar  8 11:48:26 henry001 Keepalived_healthcheckers[30033]: (Line 30) Unknown keyword 'nat_mask'
Mar  8 11:48:26 henry001 Keepalived_healthcheckers[30033]: (Line 31) number '0' outside range [1, 4294967295]
Mar  8 11:48:26 henry001 Keepalived_healthcheckers[30033]: (Line 31) persistence_timeout invalid
Mar  8 11:48:26 henry001 Keepalived_healthcheckers[30033]: (Line 37) Unknown keyword 'nb_get_retry'
Mar  8 11:48:26 henry001 Keepalived_healthcheckers[30033]: (Line 46) Unknown keyword 'nb_get_retry'
Mar  8 11:48:26 henry001 Keepalived_healthcheckers[30033]: Gained quorum 1+0=1 <= 6 for VS [192.168.1.200]:tcp:80
Mar  8 11:48:26 henry001 Keepalived_healthcheckers[30033]: Activating healthchecker for service [192.168.0.15]:tcp:80 for VS [192.168.1.200]:tcp:80
Mar  8 11:48:26 henry001 Keepalived_healthcheckers[30033]: Activating healthchecker for service [192.168.0.16]:tcp:80 for VS [192.168.1.200]:tcp:80
Mar  8 11:48:26 henry001 Keepalived_vrrp[30034]: (VI_1) Entering BACKUP STATE (init)
Mar  8 11:48:26 henry001 Keepalived_vrrp[30034]: VRRP sockpool: [ifindex(2), family(IPv4), proto(112), unicast(0), fd(11,12)]
Mar  8 11:48:28 henry001 Keepalived_healthcheckers[30033]: TCP connection to [192.168.0.15]:tcp:80 success.
Mar  8 11:48:30 henry001 Keepalived_vrrp[30034]: (VI_1) Receive advertisement timeout
Mar  8 11:48:30 henry001 Keepalived_vrrp[30034]: (VI_1) Entering MASTER STATE
Mar  8 11:48:30 henry001 Keepalived_vrrp[30034]: (VI_1) setting VIPs.
Mar  8 11:48:30 henry001 Keepalived_vrrp[30034]: Sending gratuitous ARP on eth0 for 192.168.1.200
Mar  8 11:48:30 henry001 Keepalived_vrrp[30034]: (VI_1) Sending/queueing gratuitous ARPs on eth0 for 192.168.1.200
Mar  8 11:48:30 henry001 Keepalived_vrrp[30034]: Sending gratuitous ARP on eth0 for 192.168.1.200
Mar  8 11:48:30 henry001 Keepalived_vrrp[30034]: Sending gratuitous ARP on eth0 for 192.168.1.200
Mar  8 11:48:30 henry001 Keepalived_vrrp[30034]: Sending gratuitous ARP on eth0 for 192.168.1.200
Mar  8 11:48:30 henry001 Keepalived_vrrp[30034]: Sending gratuitous ARP on eth0 for 192.168.1.200
Mar  8 11:48:31 henry001 Keepalived_healthcheckers[30033]: TCP connection to [192.168.0.16]:tcp:80 success.

.......................

```





backup机器



```shell
`查看日志`
[root@henry001 ~]# tail -f /var/log/messages

..................

Mar  8 11:51:18 henry004 Keepalived_vrrp[26647]: Assigned address 192.168.0.14 for interface eth0
Mar  8 11:51:18 henry004 Keepalived_vrrp[26647]: Registering gratuitous ARP shared channel
Mar  8 11:51:18 henry004 Keepalived_vrrp[26647]: (VI_1) removing VIPs.
Mar  8 11:51:18 henry004 Keepalived_healthcheckers[26646]: Opening file '/etc/keepalived/keepalived.conf'.
Mar  8 11:51:18 henry004 Keepalived_healthcheckers[26646]: (Line 31) Unknown keyword 'nat_mask'
Mar  8 11:51:18 henry004 Keepalived_healthcheckers[26646]: (Line 32) number '0' outside range [1, 4294967295]
Mar  8 11:51:18 henry004 Keepalived_healthcheckers[26646]: (Line 32) persistence_timeout invalid
Mar  8 11:51:18 henry004 Keepalived_healthcheckers[26646]: (Line 38) Unknown keyword 'nb_get_retry'
Mar  8 11:51:18 henry004 Keepalived_healthcheckers[26646]: (Line 47) Unknown keyword 'nb_get_retry'
Mar  8 11:51:18 henry004 Keepalived_healthcheckers[26646]: VS [192.168.1.200]:tcp:80: real server [192.168.0.16]:tcp:80 is duplicated - removing second rs
Mar  8 11:51:18 henry004 Keepalived_healthcheckers[26646]: Gained quorum 1+0=1 <= 3 for VS [192.168.1.200]:tcp:80
Mar  8 11:51:18 henry004 Keepalived_healthcheckers[26646]: Activating healthchecker for service [192.168.0.16]:tcp:80 for VS [192.168.1.200]:tcp:80
Mar  8 11:51:18 henry004 Keepalived_healthcheckers[26646]: Activating healthchecker for service [ᡰ05#001]:tcp:58493 for VS [192.168.1.200]:tcp:80
Mar  8 11:51:18 henry004 Keepalived_vrrp[26647]: (VI_1) Entering BACKUP STATE (init)
Mar  8 11:51:18 henry004 Keepalived_vrrp[26647]: VRRP sockpool: [ifindex(2), family(IPv4), proto(112), unicast(0), fd(11,12)]
Mar  8 11:51:22 henry004 Keepalived_vrrp[26647]: (VI_1) Receive advertisement timeout
Mar  8 11:51:22 henry004 Keepalived_vrrp[26647]: (VI_1) Entering MASTER STATE
Mar  8 11:51:22 henry004 Keepalived_vrrp[26647]: (VI_1) setting VIPs.
Mar  8 11:51:22 henry004 Keepalived_vrrp[26647]: Sending gratuitous ARP on eth0 for 192.168.1.200
Mar  8 11:51:22 henry004 Keepalived_vrrp[26647]: (VI_1) Sending/queueing gratuitous ARPs on eth0 for 192.168.1.200
Mar  8 11:51:22 henry004 Keepalived_vrrp[26647]: Sending gratuitous ARP on eth0 for 192.168.1.200
Mar  8 11:51:22 henry004 Keepalived_vrrp[26647]: Sending gratuitous ARP on eth0 for 192.168.1.200
Mar  8 11:51:22 henry004 Keepalived_vrrp[26647]: Sending gratuitous ARP on eth0 for 192.168.1.200
Mar  8 11:51:22 henry004 Keepalived_vrrp[26647]: Sending gratuitous ARP on eth0 for 192.168.1.200
Mar  8 11:51:24 henry004 Keepalived_healthcheckers[26646]: TCP connection to [192.168.0.16]:tcp:80 success.

...................
```





查看master服务器端口

```shell
[root@henry001 ~]# lsof -i:80
COMMAND    PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
AliYunDun 1222 root   22u  IPv4  17573      0t0  TCP henry001:53558->100.100.30.25:http (ESTABLISHED)

```



在backup服务器telnet一下

```shell
[root@henry004 ~]# telnet 192.168.0.13 80
Trying 192.168.0.13...
telnet: connect to address 192.168.0.13: Connection refused

```





```shell
global_defs {
   router_id lvs01          #router_id 机器标识，通常为hostname，但不一定非得是hostname。故障发生时，邮件通知会用到。
}
vrrp_instance VI_1 {            #vrrp实例定义部分
    state MASTER               #设置lvs的状态，MASTER和BACKUP两种，必须大写 
    interface eth0               #设置对外服务的接口
    virtual_router_id 100        #设置虚拟路由标示，这个标示是一个数字，同一个vrrp实例使用唯一标示 
    priority 100               #定义优先级，数字越大优先级越高，在一个vrrp——instance下，master的优先级必须大于backup
    advert_int 1              #设定master与backup负载均衡器之间同步检查的时间间隔，单位是秒
    authentication {           #设置验证类型和密码
        auth_type PASS         #主要有PASS和AH两种
        auth_pass 1111         #验证密码，同一个vrrp_instance下MASTER和BACKUP密码必须相同
    }
    virtual_ipaddress {         #设置虚拟ip地址，可以设置多个，每行一个
        192.168.0.100 
    }
}
virtual_server 192.168.0.100 80 {       #设置虚拟服务器，需要指定虚拟ip和服务端口
    delay_loop 6                 #健康检查时间间隔
    lb_algo wrr                  #负载均衡调度算法
    lb_kind DR                   #负载均衡转发规则
    persistence_timeout 50        #设置会话保持时间，对动态网页非常有用
    protocol TCP               #指定转发协议类型，有TCP和UDP两种
    real_server 192.168.0.15 80 {    #配置服务器节点1，需要指定real server的真实IP地址和端口
    weight 1               #设置权重，数字越大权重越高
    TCP_CHECK {              #realserver的状态监测设置部分单位秒
       connect_timeout 10       #连接超时为10秒
       retry 3             #重连次数
       delay_before_retry 3        #重试间隔
       connect_port 80        #连接端口为81，要和上面的保持一致
       }
    }
     real_server 192.168.0.16 80 {    #配置服务器节点1，需要指定real server的真实IP地址和端口
     weight 1                  #设置权重，数字越大权重越高
     TCP_CHECK {               #realserver的状态监测设置部分单位秒
       connect_timeout 10         #连接超时为10秒
       retry 3               #重连次数
       delay_before_retry 3        #重试间隔
       connect_port 80          #连接端口为81，要和上面的保持一致
       }
     }
}
```



```shell
global_defs {
   router_id lvs01          #router_id 机器标识，通常为hostname，但不一定非得是hostname。故障发生时，邮件通知会用到。
}
vrrp_instance VI_1 {            #vrrp实例定义部分
    state BACKUP              #设置lvs的状态，MASTER和BACKUP两种，必须大写 
    interface eth0           #设置对外服务的接口
    virtual_router_id 100         #设置虚拟路由标示，这个标示是一个数字，同一个vrrp实例使用唯一标示 
    priority 50             #定义优先级，数字越大优先级越高，在一个vrrp——instance下，master的优先级必须大于backup
    advert_int 1              #设定master与backup负载均衡器之间同步检查的时间间隔，单位是秒
    authentication {            #设置验证类型和密码
        auth_type PASS         #主要有PASS和AH两种
        auth_pass 1111         #验证密码，同一个vrrp_instance下MASTER和BACKUP密码必须相同
    }
    virtual_ipaddress {         #设置虚拟ip地址，可以设置多个，每行一个
        192.168.0.100 
    }
}
virtual_server 192.168.0.100 80 {      #设置虚拟服务器，需要指定虚拟ip和服务端口
    delay_loop 6             #健康检查时间间隔
    lb_algo wrr              #负载均衡调度算法
    lb_kind DR               #负载均衡转发规则
    persistence_timeout 50          #设置会话保持时间，对动态网页非常有用
    protocol TCP              #指定转发协议类型，有TCP和UDP两种
    real_server 192.168.0.16 80 {       #配置服务器节点1，需要指定real server的真实IP地址和端口
    weight 1                #设置权重，数字越大权重越高
    TCP_CHECK {              #realserver的状态监测设置部分单位秒
       connect_timeout 10         #连接超时为10秒
       retry 3                #重连次数
       delay_before_retry 3       #重试间隔
       connect_port 80           #连接端口为81，要和上面的保持一致
       }
    }
     real_server 192.168.0.15 80 {   #配置服务器节点1，需要指定real server的真实IP地址和端口
     weight 1                #设置权重，数字越大权重越高
     TCP_CHECK {              #realserver的状态监测设置部分单位秒
       connect_timeout 10          #连接超时为10秒
       retry 3             #重连次数
       delay_before_retry 3        #重试间隔
       connect_port 80         #连接端口为81，要和上面的保持一致
       }
     }
}
```





```shell
! Configuration File for keepalived global_defs { router_id LVS_DEVEL 运行keepalived服务器的标识，在一个网络内应该是唯一的 } vrrp_instance VI_1 { #vrrp实例定义部分 state MASTER #设置lvs的状态，MASTER和BACKUP两种，必须大写 interface ens33 #设置对外服务的接口 virtual_router_id 51 #设置虚拟路由标示，这个标示是一个数字，同一个vrrp实例使用唯一标示 priority 100 #定义优先级，数字越大优先级越高，在一个vrrp——instance下，master的优先级必须大于backup advert_int 1 #设定master与backup负载均衡器之间同步检查的时间间隔，单位是秒 authentication { #设置验证类型和密码 auth_type PASS auth_pass 1111 #验证密码，同一个vrrp_instance下MASTER和BACKUP密码必须相同 } virtual_ipaddress { #设置虚拟ip地址，可以设置多个，每行一个 192.168.11.100 } } virtual_server 192.168.11.100 80 { #设置虚拟服务器，需要指定虚拟ip和服务端口 delay_loop 6 #健康检查时间间隔 lb_algo rr #负载均衡调度算法 lb_kind NAT #负载均衡转发规则 persistence_timeout 50 #设置会话保持时间 protocol TCP #指定转发协议类型，有TCP和UDP两种 real_server 192.168.11.160 80 { #配置服务器节点1，需要指定real server的真实IP地址和端口 weight 1 #设置权重，数字越大权重越高
```

