-bash: lsb_release: command not found

## 1、问题：

今天在使用lsb_release -a进行系统版本查询的时候，发现报如下错误： lsb_release: command not found.

```shell
[root@m src]# lsb_release -a
-bash: lsb_release: command not found

```

## 2、安装lsb_release：

需要进行lsb_release的安装，方法如下：

```shell
[root@m src]# yum install -y redhat-lsb
Loaded plugins: fastestmirror, langpacks
Repository epel is listed more than once in the configuration
Determining fastest mirrors
docker-ce-stable                                                                                                             | 3.5 kB  00:00:00     
epel                                                                                                                         | 5.3 kB  00:00:00     
extras                                                                                                                       | 2.9 kB  00:00:00     
kubernetes                                                                                                                   | 1.4 kB  00:00:00     
os                                                                                                                           | 3.6 kB  00:00:00     
updates                                                                                                                      | 2.9 kB  00:00:00     
(1/7): epel/7/x86_64/group_gz                                                                                                |  95 kB  00:00:00     
(2/7): kubernetes/primary                                                                                                    |  65 kB  00:00:00     
(3/7): epel/7/x86_64/updateinfo                                                                                              | 1.0 MB  00:00:00     
(4/7): extras/7/x86_64/primary_db                                                                                            | 164 kB  00:00:00     
(5/7): epel/7/x86_64/primary_db                                                                                              | 6.7 MB  00:00:00     
(6/7): updates/7/x86_64/primary_db                                                                                           | 7.6 MB  00:00:01     
(7/7): docker-ce-stable/x86_64/primary_db                                                                                    |  41 kB  00:00:05  

...........省略..................

`Complete!`
```

## 3、查验系统版本

```shell
[root@m src]# lsb_release -a
LSB Version:	:core-4.1-amd64:core-4.1-noarch:cxx-4.1-amd64:cxx-4.1-noarch:desktop-4.1-amd64:desktop-4.1-noarch:languages-4.1-amd64:languages-4.1-noarch:printing-4.1-amd64:printing-4.1-noarch
Distributor ID:	CentOS
Description:	CentOS Linux release 7.7.1908 (Core)
Release:	7.7.1908
Codename:	Core

```

