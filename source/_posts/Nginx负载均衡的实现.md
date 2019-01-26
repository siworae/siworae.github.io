---
title: Nginx负载均衡的实现
categories:
  - Java
  - Nginx
tags:
  - Nginx
abbrlink: fffa5954
date: 2019-01-26 16:55:48
---

# Nginx负载均衡的实现

## 1、Nginx安装

### 1.1 安装依赖

```
yum install gcc openssl-devel pcre-devel zlib-devel -y
```

### 1.2 下载与安装

[官方网站下载](http://nginx.org/en/download.html)

根据自己的系统版本下载对应版本的压缩包

上传至服务器

解压缩

```
tar -zxvf nginx-1.8.1.tar.gz
```

进入解压后的目录,设置安装的目录,`/usr/soft/nginx`为你要安装的路径

```
./configure --prefix=/usr/soft/nginx
```

编译并安装

```
make && make install
```

安装完成之后就会在`/usr/soft/`目录下生成nginx目录,这个就是安装后的nginx了

### 1.3 配置nginx

进入`/etc/rc.d/init.d/`目录,新建nginxd文件

在nginxd文件中写入以下内容

```
#!/bin/sh
#
# nginx - this script starts and stops the nginx daemon
#
# chkconfig:   - 85 15 
# description:  Nginx is an HTTP(S) server, HTTP(S) reverse \
#               proxy and IMAP/POP3 proxy server
# processname: nginx
# config:      /etc/nginx/nginx.conf
# config:      /etc/sysconfig/nginx
# pidfile:     /var/run/nginx.pid
 
# Source function library.
. /etc/rc.d/init.d/functions
 
# Source networking configuration.
. /etc/sysconfig/network
 
# Check that networking is up.
[ "$NETWORKING" = "no" ] && exit 0
 
nginx="/usr/soft/nginx/sbin/nginx"
prog=$(basename $nginx)
 
NGINX_CONF_FILE="/usr/soft/nginx/conf/nginx.conf"
 
[ -f /etc/sysconfig/nginx ] && . /etc/sysconfig/nginx
 
lockfile=/var/lock/subsys/nginx
 
make_dirs() {
   # make required directories
   user=`nginx -V 2>&1 | grep "configure arguments:" | sed 's/[^*]*--user=\([^ ]*\).*/\1/g' -`
   options=`$nginx -V 2>&1 | grep 'configure arguments:'`
   for opt in $options; do
       if [ `echo $opt | grep '.*-temp-path'` ]; then
           value=`echo $opt | cut -d "=" -f 2`
           if [ ! -d "$value" ]; then
               # echo "creating" $value
               mkdir -p $value && chown -R $user $value
           fi
       fi
   done
}
 
start() {
    [ -x $nginx ] || exit 5
    [ -f $NGINX_CONF_FILE ] || exit 6
    make_dirs
    echo -n $"Starting $prog: "
    daemon $nginx -c $NGINX_CONF_FILE
    retval=$?
    echo
    [ $retval -eq 0 ] && touch $lockfile
    return $retval
}
 
stop() {
    echo -n $"Stopping $prog: "
    killproc $prog -QUIT
    retval=$?
    echo
    [ $retval -eq 0 ] && rm -f $lockfile
    return $retval
}
 
restart() {
    configtest || return $?
    stop
    sleep 1
    start
}
 
reload() {
    configtest || return $?
    echo -n $"Reloading $prog: "
    killproc $nginx -HUP
    RETVAL=$?
    echo
}
 
force_reload() {
    restart
}
 
configtest() {
  $nginx -t -c $NGINX_CONF_FILE
}
 
rh_status() {
    status $prog
}
 
rh_status_q() {
    rh_status >/dev/null 2>&1
}
 
case "$1" in
    start)
        rh_status_q && exit 0
        $1
        ;;
    stop)
        rh_status_q || exit 0
        $1
        ;;
    restart|configtest)
        $1
        ;;
    reload)
        rh_status_q || exit 7
        $1
        ;;
    force-reload)
        force_reload
        ;;
    status)
        rh_status
        ;;
    condrestart|try-restart)
        rh_status_q || exit 0
            ;;
    *)
        echo $"Usage: $0 {start|stop|status|restart|condrestart|try-restart|reload|force-reload|configtest}"
        exit 2
esac
```

然后修改nginxd文件的权限,使其拥有可执行权限

```
chmod +x nginxd
```

添加进系统服务

```
chkconfig --add nginxd
```

查看是否添加成功

```
chkconfig --list nginxd
```

出现以下则说明添加成功

{% asset_img 1.jpg 图片上传失败 %}

接下来就可以启动nginxd了

```
service nginxd start
```

启动后在浏览器输入你的服务器IP+80端口就可以看到以下界面了

{% asset_img 2.jpg 图片上传失败 %}

至此,你的nginx已经安装成功.

## 2、Nginx的配置

nginx的负载均衡配置主要有四种:

1. 轮询负载均衡:对应用程序服务器的请求以循环方式分发
2. 加权负载均衡:根据对应服务器的权重进行分发
3. 最少连接数:将下一个请求分配给活动连接数最少的服务器
4. ip-hash:哈希函数用于确定下一个请求（基于客户端的IP地址）应该选择哪个服务器

### 2.1 轮询配置

- `siworae`:设置自定义的访问路径
- `server`:设置业务服务器的IP

```
http { 
    upstream siworae{ 
        server srv1.example.com; 
        server srv2.example.com; 
        server srv3.example.com; 
    } 
    server { 
        listen 80; 
	server_name  localhost;
        location / {
            proxy_pass http://siworae;
        }
    } 
}
```

### 2.2  加权配置

通过使用服务器权重，还可以进一步影响nginx负载均衡算法，谁的权重越大，分发到的请求就越多

```
http { 
    upstream siworae{ 
        server srv1.example.com weight=3; 
        server srv2.example.com; 
        server srv3.example.com; 
    } 
    server { 
        listen 80; 
	server_name  localhost;
        location / {
            proxy_pass http://siworae;
        }
    } 
}
```

### 2.3 最少连接配置

在连接负载最少的情况下，nginx会尽量避免将过多的请求分发给繁忙的应用程序服务器，而是将新请求分发给不太繁忙的服务器，避免服务器过载

```
http { 
    upstream siworae{ 
    	least_conn;
        server srv1.example.com;
        server srv2.example.com; 
        server srv3.example.com; 
    } 
    server { 
        listen 80; 
	server_name  localhost;
        location / {
            proxy_pass http://siworae;
        }
    } 
}
```

### 2.4 ip-hsah

使用ip-hash，客户端的IP地址将用作散列键，以确定应该为客户端的请求选择服务器组中的哪台服务器。此方法可确保来自同一客户端的请求将始终定向到同一台服务器，除非此服务器不可用.

```
http { 
    upstream siworae{ 
    	ip_hash;
        server srv1.example.com;
        server srv2.example.com; 
        server srv3.example.com; 
    } 
    server { 
        listen 80; 
	server_name  localhost;
        location / {
            proxy_pass http://siworae;
        }
    } 
}
```

上述的循环或最少连接数的负载平衡方法，每个后续客户端的请求都可能被分发到不同的服务器。不能保证相同的客户端总是定向到相同的服务器。如果需要将客户端绑定到特定的应用程序服务器，换句话说，就是始终选择相同的服务器而言，就要使客户端的会话“粘滞”或“持久” ，ip-hash负载平衡机制就是有这种特性。