---
title: HDFS分布式文件系统的伪分布式搭建
categories:
  - HADOOP
  - HDFS
tags:
  - HDFS
abbrlink: b033a6ab
date: 2019-01-29 18:33:04
---

# 分布式文件存储系统HDFS

## 1、什么是HDFS

HDFS是Hadoop分布式文件存储系统,主要用来解决海量数据的存储问题.

HDFS具有高容错性,存入的数据会自动保存多个副本,在副本丢失的情况下会自动恢复副本;同时在大数据批处理的情况下也能将数据位置暴露给计算框架,移动计算而非移动数据,但是HDFS也有一些缺点,比如说只能支持到秒级别的响应,暂时还不能支持到毫秒级别的响应,小文件存储时占用内容过高的问题.

## 2、HDFS功能模块

### 2.1、HDSF架构图

{% asset_img 1.jpg 图片上传失败 %}

### 2.2、功能模块

#### 2.2.1、HDFS数据存储单元(block)

所有上传至HDFS的文件都会被自动切分成固定大小的数据块(block),

在2.x的版本中,默认的大小为128MB,且可自定义配置大小,在1.x的版本中默认的大小为64MB,

若文件大小不足于128MB时也会单独打包成一个block,

每个被切分的block块都会被存储在不同的节点上,每个block默认存在三个副本,并且这个三个副本会被存储在不同的节点上,

Block大小和副本数通过Client端上传文件时设置，文件上传成功后副本数可以变更，Block Size不可变更,

HDFS会自动的线性切分上传的文件,每个block块都会有一个偏移量(offset),用来记录文件切分的位置,HDFS可根据offset来将各个block块合并成完整的文件

block存储模拟图

{% asset_img 2.jpg 图片上传失败 %}

#### 2.2.2、NameNode(NN)

NameNode的主要功能是接受客户端的读/写服务,并且收集各个block利用心跳机制发送的block列表信息

NameNode是存在于内存中的,基于内存存储,不会和磁盘发送交互

NameNode保存了metadata的元数据信息,包括文件的owership(归属)和permissions(权限),文件的大小和创建时间,block列表和block的偏移量

NameNode的metadata数据也会进行持久化,metadata持久化之后保存到磁盘的文件名是`fsimage`,这个文件类似于系统的快照,保存着最新的元数据信息,在HDFS启动时,会将fsimage文件加载到内存中

每次客户端对文件进行了修改之后,editslog会保存客户端的操作记录,只有当客户端操作成功之后才会去更新metadata的信息,这就相当于`metadata = editslog + fsimage`,而,HDFS会将客户端的每次操作都记录到edits里面.

#### 2.2.3、DataNode(DN)

DataNode是以文件的形式来存储数据(block) 的,并且保存着block的元信息文件,相当于是直接操作数据文件的角色,在DataNode启动的时候会向NameNode发送其节点的block信息,并且之后每3秒一次与NameNode保存心跳联系,如果NameNode十分钟没有收到DataNode的心跳信息,就会认为此DataNode节点挂了,会将存储在这个节点的block复制到其他DataNode节点上,维持其block的副本数量,这个就是HDFS的副本恢复策略.

#### 2.2.4、SecondaryNameNode(SNN)

SNN的主要功能就是帮助NameNode去合并editslog和fsimage文件,以此减少NameNode的启动时间.

至于为什么说会减少NameNode的启动时间,是因为在HDFS系统启动的时候,SecondaryNameNode就已经将各个DataNode的editslog + fsimage文件合并为最新的metadata文件了,那么NameNode只需要直接读取这个metadata文件就可以了,而不用去合并读取每个DataNode的editslog + fsimage.

SNN的合并时间和机制

1. 根据配置文件设置的时间间隔fs.checkpoint.period 默认3600秒。
2. 根据配置文件设置edits log大小 fs.checkpoint.size 规定edits文件的最大值默认是64MB

SecondaryNameNode SNN合并流程

{% asset_img 3.jpg 图片上传失败 %}

首先是NN中的Fsimage和edits文件通过网络拷贝，到达SNN服务器中，拷贝的同时，用户的实时在操作数据，那么NN中就会从新生成一个edits来记录用户的操作，而另一边的SＮＮ将拷贝过来的edits和fsimage进行合并，合并之后就替换NN中的fsimage。之后NN根据fsimage进行操作（当然每隔一段时间就进行替换合并，循环）。当然新的edits与合并之后传输过来的fsimage会在下一次时间内又进行合并

#### 2.2.5、HDFS读写流程

HDFS写流程

{% asset_img 4.jpg 图片上传失败 %}

1.Client调用create方法,通过distributedFileSystem实例去调用NameNode去预先创建一些空的没有关联的block块

2.Client通过FSDataOutputStream流写入数据

3.FSDataOutputStream会构建一个write packet管道流,数据包会构成一个datamq(数据队列),然后通过流的方式写入DataNode

4.DataNode成功写入block块之后,会根据副本存放策略去复制block块的副本存放到其他的DataNode节点上

5.当所有的block块副本写入成功之后,开始写入第二个block块

6.ack packet相当于是一个监管者的身份,它会留存一份datamq中的数据,同时还会监听DataNode的状态,当在写入的过程中DataNode宕机了或者其他无法写入的情况下,ack packet会通知FSDataOutputStream关闭当前管道流并在其他DataNode节点上开闭新的管道流,并将之前留存的数据发送到新的管道流的datamq中,重新将block块写入到新的DataNode节点中.如果写入过程中没有发生问题,那么ack packet会被直接丢弃

HDFS读流程

{% asset_img 5.jpg 图片上传失败 %}

1.Client调用open方法,通过distributedFileSystem实例去读取NameNode的metadata元信息

2.NameNode返回一批block locations信息给客户端

3.Client获取到block的信息之后,通过FSDataInputStream去读取DataNode中的block文件(读取过程中会按照DataNode的拓扑网络结构图,就近原则去读取DataNode中的数据,当读到的block构成了一个完整的文件时,读取就结束了)

## 3、Hadoop搭建

### 1、环境配置

#### 1.1、jdk安装

jdk的具体安装步骤就不阐述了,网上很多教程,随便找找都有

#### 1.2、安装Hadoop

官网下载[Hadoop](https://hadoop.apache.org/releases.html)压缩包

上传hadoop.tar.gz到服务器并解压至/usr/soft/目录下(此目录自行定义)

#### 1.3、配置环境变量

配置root用户下的环境变量

HADOOP_HOME配的路径为你解压后的根目录

```
vim /root/.bash_profile

HADOOP_HOME=//usr/soft/hadoop-2.6.5
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
```

#### 1.4、修改配置文件

```
cd /usr/soft/hadoop-2.6.5/etc/hadoop
```

修改hadoop-env.sh

```
vim hadoop-env.sh
## 配置的是你安装的jdk的根目录,也就是你的jdk的环境变量
export JAVA_HOME=/usr/soft/jdk1.8.0_191
```

然后重新加载环境变量

```
source ~/.bash_profile
```

修改core-site.xml

```
vim core-site.xml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        ## 此处配置的是你的服务器的IP和端口
        <value>hdfs://192.168.230.220:9000</value>
    </property>
    <property>
        <name>hadoop.tmp.dir</name>
        ## 配置你的数据存放的路径
        <value>/var/hadoop/local</value>
     </property>
</configuration>
```

修改hdfs-site.xml

```
vim hdfs-site.xml 

<configuration>
<property>
	 ## 配置block块的副本数量,默认为1
      <name>dfs.replication</name>
      <value>3</value>
 </property>
 <property>
 	## 配置服务器的IP和nameNode的端口
     <name>dfs.namenode.secondary.http-address</name>
    <value>192.168.230.220:50090</value>
 </property>
</configuration>
```

修改 slaves(配置datanode节点)

```
vim slaves
```

文件内直接添加服务器的节点IP,这里因为是用单机模拟分布式,所以只填写了192.168.230.220一个IP,正常分布式环境应该填写多个IP地址

格式化HDFS

```
hdfs namenode -format
```

启动

```
start-dfs.sh
```

查看HDFS进程是否启动

```
jps
```

{% asset_img 6.jpg 图片上传失败 %}

出现以上进程说明HDFS进程已经启动了,但是进程启动了并不代表HDFS系统已经启动了

你可以在浏览器访问`192.168.230.220:50070`HDFS的管理界面

{% asset_img 7.jpg 图片上传失败 %}

你可以进入如上页面并且状态为active存活状态,则说明HDFS已经启动成功了

如果不能访问请关闭虚拟机的防火墙