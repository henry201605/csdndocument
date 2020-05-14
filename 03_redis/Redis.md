

## 1、安装环境

```shell
[root@m src]# lsb_release -a
LSB Version:	:core-4.1-amd64:core-4.1-noarch:cxx-4.1-amd64:cxx-4.1-noarch:desktop-4.1-amd64:desktop-4.1-noarch:languages-4.1-amd64:languages-4.1-noarch:printing-4.1-amd64:printing-4.1-noarch
Distributor ID:	CentOS
Description:	CentOS Linux release 7.7.1908 (Core)
Release:	7.7.1908
Codename:	Core

```

## 2、Redis下载

访问`http://download.redis.io/releases/`选择需要下载的版本，本文选择稳定版本：`redis-5.0.8.tar.gz`;

![image-20200326193258986](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200326193258986.png)

* 获取下载地址：

  http://download.redis.io/releases/redis-5.0.8.tar.gz



* 下载到本地

```shell
`进入到/usr/local/src/目录下`
[root@m src]# cd /usr/local/src/

`下载Reids`
[root@m src]# wget http://download.redis.io/releases/redis-5.0.8.tar.gz
```



## 3、安装

### 3.1、检查是否安装了gcc

由于 redis 是使用 C 语言开发的，在安装之前需要先确认一下是否安装了 gcc 环境，先来查看一下：

```shell
`查看gcc版本`
[root@m src]# gcc -v
Using built-in specs.
COLLECT_GCC=gcc
COLLECT_LTO_WRAPPER=/usr/libexec/gcc/x86_64-redhat-linux/4.8.5/lto-wrapper
Target: x86_64-redhat-linux
Configured with: ../configure --prefix=/usr --mandir=/usr/share/man --infodir=/usr/share/info --with-bugurl=http://bugzilla.redhat.com/bugzilla --enable-bootstrap --enable-shared --enable-threads=posix --enable-checking=release --with-system-zlib --enable-__cxa_atexit --disable-libunwind-exceptions --enable-gnu-unique-object --enable-linker-build-id --with-linker-hash-style=gnu --enable-languages=c,c++,objc,obj-c++,java,fortran,ada,go,lto --enable-plugin --enable-initfini-array --disable-libgcj --with-isl=/builddir/build/BUILD/gcc-4.8.5-20150702/obj-x86_64-redhat-linux/isl-install --with-cloog=/builddir/build/BUILD/gcc-4.8.5-20150702/obj-x86_64-redhat-linux/cloog-install --enable-gnu-indirect-function --with-tune=generic --with-arch_32=x86-64 --build=x86_64-redhat-linux
Thread model: posix
gcc version 4.8.5 20150623 (Red Hat 4.8.5-39) (GCC) 

```

如果未安装可执行如下语句：

`yum -y install gcc`



### 3.2、安装

```shell
`进入下载目录进行解压`
[root@m src]# cd /usr/local/src/
[root@m src]# tar -zvxf redis-5.0.8.tar.gz 

[root@m src]# ls
redis-5.0.8  redis-5.0.8.tar.gz

`进入到redis安装包中`
[root@m src]# cd redis-5.0.8/

`进行编译`
[root@m redis-5.0.8]# make

`安装到指定目录`
[root@m redis-5.0.8]# cd src
[root@m src]# make install prefix=/usr/local/redis

```

## 4、启动redis

### 4.1 启动

```shell
[root@m bin]# cd /usr/local/redis/bin
`启动redis` 
[root@m bin]# ./redis-server 

```

### 4.2 后台启动

```shell
`将redis.conf复制到安装目录下`
[root@m bin]# cd /usr/local/redis/bin
[root@m bin]# cp /usr/local/src/redis-5.0.8/redis.conf ./
```

修改`redis.conf`文件:

```shell
daemonize no
```

改为

```shell
daemonize yes
```

启动redis

```shell
`启动redis服务端`
[root@m bin]# redis-server ./redis.conf 
10115:C 26 Mar 2020 20:59:44.525 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
10115:C 26 Mar 2020 20:59:44.525 # Redis version=5.0.8, bits=64, commit=00000000, modified=0, pid=10115, just started
10115:C 26 Mar 2020 20:59:44.525 # Configuration loaded

`启动redis客户端`
[root@m bin]# ./redis-cli 
127.0.0.1:6379> 
```

## 5、开机启动

### 5.1、开机自启动脚本

#### 1）修改脚本

```shell
`复制开机启动脚本`
[root@m system]# cp /usr/local/src/redis-5.0.8/utils/redis_init_script /etc/init.d/redis
`修改脚本`
[root@m init.d]# vim  /etc/init.d/redis
```

脚本内容如下：

```shell
#!/bin/sh
# chkconfig: 2345 10 90
# description: Start and Stop redis
#
# Simple Redis init.d script conceived to work on Linux systems
# as it does use of the /proc filesystem.

### BEGIN INIT INFO
# Provides:     redis_6379
# Default-Start:        2 3 4 5
# Default-Stop:         0 1 6
# Short-Description:    Redis data structure server
# Description:          Redis data structure server. See https://redis.io
### END INIT INFO

REDISPORT=6379                       #端口号
EXEC=/usr/local/bin/redis-server     #服务端的命令
CLIEXEC=/usr/local/bin/redis-cli     #客户端的命令

PIDFILE=/var/run/redis_${REDISPORT}.pid   #进程文件生成的位置
CONF="/usr/local/redis/bin/redis.conf"    #配置文件所在的目录

case "$1" in
    start)
        if [ -f $PIDFILE ]
        then
                echo "$PIDFILE exists, process is already running or crashed"
        else
                echo "Starting Redis server..."
                $EXEC $CONF
        fi
        ;;
    stop)
        if [ ! -f $PIDFILE ]
        then
                echo "$PIDFILE does not exist, process is not running"
        else
                PID=$(cat $PIDFILE)
                echo "Stopping ..."
                $CLIEXEC -p $REDISPORT shutdown
                while [ -x /proc/${PID} ]
                do
                    echo "Waiting for Redis to shutdown ..."
                    sleep 1
                done
                echo "Redis stopped"
        fi
        ;;
    *)
        echo "Please use start or stop as first argument"
        ;;
esac
```

#### 2）添加权限

```shell
chmod 775 /etc/init.d/redis
```

#### 3）设置开机启动

```shell
`查看redis服务是否在服务配置中`
[root@m init.d]# chkconfig --list redis

Note: This output shows SysV services only and does not include native
      systemd services. SysV configuration data might be overridden by native
      systemd configuration.

      If you want to list systemd services use 'systemctl list-unit-files'.
      To see services enabled on particular target use
      'systemctl list-dependencies [target]'.

service redis supports chkconfig, but is not referenced in any runlevel (run 'chkconfig --add redis')


`若没有，则把redis注册为开机启动的服务`
[root@m init.d]# chkconfig --add redis

`设置为自动启动`
[root@m init.d]# chkconfig redis on

`查看redis服务是否在服务配置中`
[root@m init.d]# chkconfig --list redis
.........................
`redis          	0:off	1:off	2:on	3:on	4:on	5:on	6:off`


```

## 6、 服务停止与启动：

### 6.1、启动服务

```shell
[root@m init.d]# systemctl start redis.service
```

### 6.2、查看状态

```shell
`查看状态`
[root@m init.d]# systemctl status redis.service
● redis.service - LSB: Redis data structure server
   Loaded: loaded (/etc/rc.d/init.d/redis; bad; vendor preset: disabled)
   Active: `active` (exited) since Thu 2020-03-26 23:07:01 CST; 6s ago
     Docs: man:systemd-sysv-generator(8)
  Process: 19029 ExecStop=/etc/rc.d/init.d/redis stop (code=killed, signal=TERM)
  Process: 696 ExecStart=/etc/rc.d/init.d/redis start (code=exited, status=0/SUCCESS)

Mar 26 23:07:01 m systemd[1]: Starting LSB: Redis data structure server...
Mar 26 23:07:01 m redis[696]: /var/run/redis_6379.pid exists, process is already running or crashed
Mar 26 23:07:01 m systemd[1]: Started LSB: Redis data structure server.
                              [  OK  ]

```

### 6.3 、停止服务



```shell
[root@m init.d]# systemctl stop redis.service
`查看一下状态`
[root@m init.d]# systemctl status redis.service
● redis.service - LSB: Redis data structure server
   Loaded: loaded (/etc/rc.d/init.d/redis; bad; vendor preset: disabled)
   Active: `deactivating` (stop) since Thu 2020-03-26 23:09:30 CST; 1min 20s ago
     Docs: man:systemd-sysv-generator(8)
  Process: 696 ExecStart=/etc/rc.d/init.d/redis start (code=exited, status=0/SUCCESS)
  Control: 6555 (redis)
    Tasks: 2
   Memory: 308.0K
   CGroup: /system.slice/redis.service
           └─control
             ├─6555 /bin/sh /etc/rc.d/init.d/redis stop
             └─9879 sleep 1

```

### 6.4、重启服务

```shell
[root@m init.d]# systemctl restart redis.service

```

## 7、注意事项

### 7.1 redis配置文件

1、需要将`bind 127.0.0.1 `改成 `bind 0.0.0.0` 或注释，否则只能在本机访问；

2、如果需要密码访问，取消requirepass的注释

```
requirepass password
```

### 7.2、 关闭服务

停止redis（在客户端中）

```
redis> shutdown
```

或

```
ps -aux | grep redis
kill -9 xxxx
```

### 7.3 建立软连接

可以使用建立软连接的方式，方便操作使用；

```shell
[root@m system]# ln -s /usr/local/redis/bin/redis-cli /usr/bin/redis
[root@m system]# redis
127.0.0.1:6379> ping
PONG
```

### 7.4 客户端访问带密码的redis

```shell
[root@m system]# redis
127.0.0.1:6379> ping
(error) NOAUTH Authentication required.
`密码认证`
127.0.0.1:6379> auth henry    #(密码)
OK
127.0.0.1:6379> ping
PONG

```





### 至此，Redis安装大功告成！！！