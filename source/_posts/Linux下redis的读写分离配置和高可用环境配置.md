---
title: Linux下redis的读写分离配置和高可用环境配置
categories:
- Java
- redis
tags: 
- redis
abbrlink: 9227d01
date: 2019-01-04 20:36:40
---

# Linux下redis的读写分离配置和高可用环境配置

因为高可用和主备切换需要多台服务器,准备不了那么多服务器所以本文中的多台服务器是利用一台服务器的多端口模拟多台服务器实现的

redis 服务器三台

```
192.168.230.221	6379(master-主服务器)
192.168.230.221	6380(slave-从服务器)
192.168.230.221	6381(slave-从服务器)
```

Sentinel 哨兵服务器三台

```
192.168.230.221	26379
192.168.230.221	26380
192.168.230.221	26381
```

## 1、redis的安装

redis官方网站:http://www.redis.cn

### 1)、下载并解压redis

进入到指定目录,这里我下载到/usr/soft/redis目录

```
cd /usr/soft/redis
wget http://download.redis.io/releases/redis-5.0.3.tar.gz
tar zxf redis-5.0.3.tar.gz
```

解压之后redis目录下就会出现redis-5.0.3文件夹,cd进入这个文件夹,

因为redis是用C语言写的,所以在运行前需要对源码文件进行编译才能运行

如果你的系统里面已经有了GCC的编译环境,那么你可以直接编译,没有的话就需要安装GCC的编译环境进行对redis的源码进行编译,

```
yum install gcc
```

安装成功之后就可以进行编译了,一定要在redis的根目录进行编译

```
make
```

如果在编译过程中出现了`command not found`错误,则是没有安装GCC编译环境,重新安装完编译环境后一定要删除解压后的文件重新解压编译,不然会出错

编译完成后,src目录中就会出现这些可执行文件就说明你的redis服务器安装成功了

{% asset_img 1.jpg 图片上传失败 %}

## 2、配置redis主服务器

redis的配置文件是存放在根目录的redis.conf文件启动redis之前我们修改一些配置文件

```
vim redis.conf
```

```
#bind 127.0.0.1		#注释此配置,允许该服务器所有IP都能访问到Redis服务
port 6379			#Redis服务的端口,默认是6379。可不修改
daemonize yes		#以守护进程模式启动redis,这样启动之后reids就不会占用当前连接
databases 16		#默认redis服务器的库数量
requirepass "123456"	#设置redis的访问密码
masterauth "123456"		#设置从服务器访问主服务器的访问密码
```

## 3、配置redis从服务器

由于是单机多端口模拟多台服务器,所以需要将redis的文件夹拷贝两份

```
cp -r redis-5.0.3 redis-6380
cp -r redis-5.0.3 redis-6381
```

分别进入这两个文件夹进行配置文件修改

```
cd redis-6380
vim redis.conf
```

```
#bind 127.0.0.1		#注释此配置,允许该服务器所有IP都能访问到Redis服务
port 6380			#Redis服务的端口,进入redis-6381文件夹时端口改为6381,确保端口不冲突
daemonize yes		#以守护进程模式启动redis,这样启动之后reids就不会占用当前连接
requirepass "123456"	#设置redis的访问密码
masterauth "123456"		#设置从服务器访问主服务器的访问密码
replicaof 192.168.230.220 6379	#设置主服务器IP地址及端口
```

低版本的redis设置服务器的配置项是:slaveof 192.168.230.220 6379,经测试,高版本设置这个属性也可以正常连接

保存退出,然后在各个redis的根目录启动redis服务器

启动redis服务器就是执行src文件夹中redis-server.sh的可以执行文件进行启动,同时显式的指定配置文件

## 4、启动服务器

启动主服务器

```
cd redis-5.0.3
src/redis-server redis.conf
```

启动两个从服务器

```
cd ../redis-6380
src/redis-server redis.conf
cd ../redis-6381
src/redis-server redis.conf
```

查看服务器启动情况

```
ps -ef|grep redis
```

{% asset_img 3.jpg 图片上传失败 %}

此时你的redis服务已经启动成功了,然后就可以用客户端进行连接了

接着执行客户端启动脚本,连接的时候利用-a指定密码,-p指定端口(如果是默认端口6379可不写)

```
src/redis-cli -a 123456 -p 6379
```

{% asset_img 2.jpg 图片上传失败 %}

当你看着这个的时候说明redis连接成功了,此时你可以在这里输入reids命令进行操作

因为客户端会占用当前连接,所以我们需要另开一个ssh连接进行从服务器的客户端连接

此时在主服务器客户端(也就是6379端口的服务器)中查看从服务器的连接情况

```
info replication
```

{% asset_img 4.jpg 图片上传失败 %}

此时说明两台从服务器已经连接上了主服务器了,当然你也可以在从服务器的客户端查看主服务器的情况

```
info replication
```

{% asset_img 5.jpg 图片上传失败 %}

```
master_host:192.168.230.221   主服务器IP
master_port:6379	主服务器端口
master_link_status:up   主服务器是否在线(up为在线,down为离线)
```

此时redis服务器的读写分离环境已经搭建完成

主服务器读写测试

{% asset_img 10.jpg 图片上传失败 %}

从服务器读测试,此时从服务器可以直接读取到在主服务器添加的数据,

{% asset_img 11.jpg 图片上传失败 %}

而在从服务器添加数据,则会提示从服务器是只读的

{% asset_img 12.jpg 图片上传失败 %}

## 5、准备Sentinel 哨兵服务器

回到redis根目录的上一级,我的目录结构是:/usr/soft/redis/,redis目录用来存放各个redis服务器.所以回到redis目录拷贝

```
cp -r redis-6379 redis-26379
cp -r redis-6379 redis-26380
cp -r redis-6379 redis-26381
```

{% asset_img 6.jpg 图片上传失败 %}

最终你的redis文件夹应该有这六个文件,说明你的sentinel服务器已经准备好了.剩下的就是修改配置了

哨兵服务器用的配置文件和redis服务器不一样,哨兵服务器用的是sentinel.conf

依次cd进入各个文件夹修改配置文件

```
cd redis-26379
vim sentinel.conf
```

```
port 26379		# 哨兵服务器端口
# mymaster为redis主服务器别名,192.168.230.221为主服务器IP,6379为主服务器端口,2为哨兵切换主备投票的最低票
# 数,计算公式为(哨兵服务器数量+1)/2
sentinel monitor mymaster 192.168.230.221 6379 2
sentinel auth-pass mymaster 123456	# mymaster为上面设置的别名,123456为连接主服务器需要的访问密码
# sentinel在该配置值内未能完成failover操作（即故障时master/slave自动切换),则认为本次failover失败
# 默认值为900000毫秒,可根据实际情况修改
sentinel failover-timeout mymaster 900000
sentinel down-after-milliseconds mymaster 30000		# sentinel认为redis服务器已经断线所需的毫秒数
```

配置到这里,基本上sentinel服务器已经配置好了,剩下的就是依次进入26380和26381文件夹修改对应的配置,注意更换端口,不然启动时会端口冲突

配置完成之后,启动就变得很简单了,由于没有配置sentinel的后台启动,所以每个服务器都会占用一个连接,就需要多开连接进行了

```
cd /redis-26379
src/redis-sentinel sentinel.conf
```

{% asset_img 7.jpg 图片上传失败 %}

如果你能看到这个经典的启动界面,说明你的sentinel服务器已经启动了,你离成功也就差几个命令了

在新开的连接窗口,进入sentinel服务器根目录

```
cd /usr/soft/redis/redis-26380
src/redis-sentinel sentinel.conf
```

启动成功之后,你能看到启动界面,并且能打印之前我们启动的那台sentinel服务器信息,说明这台sentinel也已经启动成功了

{% asset_img 9.jpg 图片上传失败 %}

并且之前启动的端口号为26379的那台sentinel服务器也会打印这台服务器信息

{% asset_img 8.jpg 图片上传失败 %}

说明两天sentinel服务器已经正常监听了6379的redis主服务器,并且两台sentinel也建立了通信

如果新开的sentinel服务器显示redis主服务器离线,并且也没有打印26379的服务器信息,说明服务器的配置处理问题重点检查是否配置了redis主服务器的验证密码,redis主服务器的IP端口是否正确

{% asset_img 212.jpg 图片上传失败 %}

剩下的就是按照以上步骤,启动端口为26381的sentinel服务器.

```
cd /usr/soft/redis/redis-26381
src/redis-sentinel sentinel.conf
```

如果你能在之前启动的两台服务器日志里看到26381的日志打印信息,说明服务器启动成功.

这时,你的Linux下的redis读写分离和高可用的环境已经搭建完成,当有主服务器宕机之后,sentinel服务器会自动选举出新的服务器并且进行切换,宕机的服务器重新上线后会自动作为从服务器加入集群环境.