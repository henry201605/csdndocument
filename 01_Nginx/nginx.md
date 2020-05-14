Nat方式搭建

DR模式搭建



# 配置Nginx+Tomcat解决方案

## 1、环境：

Centos7.6

| 服务器         | 软件           |
| :------------- | :------------- |
| 192.168.0.13   | LVS+KeepAlived |
| 192.168.0.14   | LVS+KeepAlived |
| 192.168.0.15   | Nginx          |
| 182.92.220.145 | Nginx          |
| 123.206.63.51  | Tomcat         |
| 132.232.115.96 | Tomcat         |



## 2、安装Nginx

### 2.1 下载

```shell
`进入到/usr/local/src目录下`
[root@henry004 ~]# cd /usr/local/src

`下载nginx`
[root@henry004 src]# wget http://nginx.org/download/nginx-1.16.1.tar.gz

`解压缩`
[root@henry003 src]# tar -zxvf nginx-1.16.1.tar.gz


```

### 2.2安装

```shell
`进入到/usr/local/src/nginx-1.16.1目录中`
[root@henry003 nginx-1.16.1]# cd /usr/local/src/nginx-1.16.1
[root@henry003 nginx-1.16.1]# ./configure --prefix=/usr/local/nginx

`编译安装`
[root@henry004 nginx-1.16.1]# make && make install
```



编译过程中可能会出现如下常见问题：

1、缺少PCRE

```shell
`-------错误信息---------------`
./configure: error: the HTTP rewrite module requires the PCRE library.
You can either disable the module by using --without-http_rewrite_module
option, or install the PCRE library into the system, or build the PCRE library
statically from the source with nginx by using --with-pcre=<path> option.
  
`----- ---解决方案--------------------`
  yum -y install pcre-devel
```

2、缺少zlib

```shell
`--------错误信息---------------`
./configure: error: the HTTP gzip module requires the zlib library.
You can either disable the module by using --without-http_gzip_module
option, or install the zlib library into the system, or build the zlib library
statically from the source with nginx by using --with-zlib=<path> option.

`-------解决方案--------------------`
yum -y install zlib-devel
```



2.2.4 启动nginx

```shell
`启动nginx服务`
[root@henry003 nginx]# cd /usr/local/nginx/sbin
[root@henry003 sbin]# ./nginx
`查看nginx启动情况`
[root@henry003 sbin]# ps -ef|grep nginx
root     16374     1  0 23:12 ?        00:00:00 nginx: master process ./nginx
nobody   16375 16374  0 23:12 ?        00:00:00 nginx: worker process
root     16377 11301  0 23:12 pts/0    00:00:00 grep --color=auto nginx

```

### 2.3 防火墙

为方便测试，我直接关闭了防火墙，在实际应用中可以根据需要开启防火墙的端口，此外还要设置服务器的安全策略，我的是阿里云的服务器，就在阿里云服务器控制台设置了安全策略，开放了需要的端口。

关闭防火墙：

```shell
`关闭防火墙`
[root@henry003 sbin]# systemctl stop firewalld

`查看防火墙状态`
[root@henry003 sbin]# systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)

Mar 07 22:39:08 henry003 systemd[1]: Starting firewalld - dynamic firewall daemon...
Mar 07 22:39:08 henry003 systemd[1]: Started firewalld - dynamic firewall daemon.
Mar 07 22:39:23 henry003 systemd[1]: Stopping firewalld - dynamic firewall daemon...
Mar 07 22:39:24 henry003 systemd[1]: Stopped firewalld - dynamic firewall daemon.
```



### 2.4 访问

访问后可以看到nginx熟悉的界面。

![image-20200307231901845](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200307231901845.png)

## 3 、Tomcat

tomcat安装就不在此介绍了，我已在另外两台服务器分别安装了tomcat。接下来就进行nginx的配置，来实现负载均衡。

## 4、配置nginx



```shell
`进入到nginx配置目录下`
[root@henry002 conf]# cd /usr/local/nginx/conf
`编辑nginx.conf`
[root@henry002 conf]# vim nginx.conf
```

nginx.conf文件如下：

```shell
#user  nobody;
worker_processes  1;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                     '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

	#引入文件
    include ../extra/*.conf;   
    gzip on;
    gzip_min_length 5k;
    gzip_comp_level 3;
    gzip_types application/javascript image/jpeg image/svg+xml image/png;
    gzip_buffers 4 32k;
    gzip_vary on;

}
```

创建extra文件夹，并创建文件

```shell
[root@henry002 nginx]# mkdir /usr/local/nginx/extra
[root@henry002 nginx]# cd extra/
[root@henry002 nginx]# vim tomcat.conf
```



```shell
upstream tomcat {
    server 123.206.63.51:8090 max_fails=2 fail_timeout=60s;
    server 132.232.115.96:8090;
}
server {
    listen 80;
    server_name localhost;
    location / {
        proxy_pass http://tomcat;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_next_upstream error timeout http_500 http_503;
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
        add_header 'Access-Control-Allow-Origin' '*';
        add_header 'Access-Control-Allow-Methods' 'GET,POST,DELETE';
        add_header 'Aceess-Control-Allow-Header' 'Content-Type,*';
	}
	#location ~ .*\.(js|css|png|svg|ico|jpg)$ {
		#防盗链
    #    valid_referers none blocked 123.206.63.51;
    #    if ($invalid_referer) {
    #    return 404;
    #    }
    #    root static-resource;
    #    expires 1d;
	#}
}
```



## 5、访问

为了区分两个tomcat，修改了其中一个tomcat的首页：

![image-20200308002315048](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200308002315048.png)

访问Nginx地址

![image-20200308002059886](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200308002059886.png)





![image-20200308002115619](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200308002115619.png)

结语：

到此nginx的配置就已经完成了，要想了解LVS+KeepAlived+Nginx+Tomcat高可用解决方案，可以访问以下地址。