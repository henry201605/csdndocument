

MySQL集群搭建是用普通的还是Cluster

1、Haproxy(mysqlcluster)

keepalived 使用虚拟ip 

https://blog.csdn.net/csolo/article/details/87363388

2、pxc

percona XtralDB Cluster

使用的是Gelera

https://blog.csdn.net/poxiaonie/article/details/78626411

- 复制只适用于InnoDB存储引擎。任何写入其他类型的表，包括系统(mysql.*)表复制。