---
title: 基于zookeeper的Dubbo框架搭建
categories:
  - Java
  - Dubbo
tags:
  - Dubbo
abbrlink: 1d70d5b2
date: 2019-01-06 13:15:03
---

# 基于zookeeper的Dubbo框架搭建

工具:idea

环境:jdk1.8

本文采用了Dubbo、zookeeper及spring框架整合,实现了单机zookeeper的远程方法调用

## 1、Dubbo的基本介绍

Dubbo官方网站:http://dubbo.apache.org

Dubbo是阿里巴巴在2011年开源出来基于Java的分布式远程调用框架, dubbo是基于定义服务的概念,指定可以通过参数和返回类型远程调用的方法。在服务器端,服务器实现这个接口,并运行一个dubbo服务器来处理客户端调用。在客户端,客户机有一个存根,它提供与服务器相同的方法。

简单来说,dubbo就是:

- 一款分布式服务框架
- 高性能和透明化的RPC远程服务调用方案
- SOA服务治理方案

Dubbo为我们提供了三个核心功能:

- 基于接口的远程调用
- 容错和负载均衡
- 服务的自动注册和发现

## 2、Dubbo的结构

{% asset_img 1.png 图片上传失败 %}

Provider: 暴露服务的服务提供方。 
Consumer: 调用远程服务的服务消费方。 
Registry: 服务注册与发现的注册中心。 
Monitor: 统计服务的调用次数和调用时间的监控中心。

调用流程 
0.服务容器负责启动，加载，运行服务提供者。 
1.服务提供者在启动时，向注册中心注册自己提供的服务。 
2.服务消费者在启动时，向注册中心订阅自己所需的服务。 
3.注册中心返回服务提供者地址列表给消费者，如果有变更，注册中心将基于长连接推送变更数据给消费者。 
4.服务消费者，从提供者地址列表中，基于软负载均衡算法，选一台提供者进行调用，如果调用失败，再选另一台调用。 
5.服务消费者和提供者，在内存中累计调用次数和调用时间，定时每分钟发送一次统计数据到监控中心

## 3、Dubbo的支持的注册中心

### 1)、Multicast注册中心

Multicast 注册中心不需要启动任何中心节点，只要广播地址一样，就可以互相发现.但这种注册方式受网络结构限制,只适合小规模应用或开发阶段使用。

使用Multicast注册中心可使用的组播地址段: 224.0.0.0 - 239.255.255.255

### 2、zookeeper注册中心

Zookeeper是Apacahe Hadoop的子项目,是一个树型的目录服务

### 3、Redis注册中心

是一套基于redis实现的注册中心

### 4、Simple注册中心

Simple注册中心本身就是一个普通的 Dubbo 服务,可以减少第三方依赖.使整体通讯方式一致。

在这里我们采用的是第二种也就是zookeeper注册中心

具体的注册中心介绍可以参照官网文档http://dubbo.apache.org/zh-cn/docs/user/quick-start.html

### 4、Dubbo的优缺点

优点:

1. 透明化的远程方法调用

   像调用本地方法一样调用远程方法；只需简单配置，没有任何API侵入

2. 软负载均衡及容错机制

   可在内网替代nginx lvs等硬件负载均衡器

3. 服务注册中心自动注册和自动发现

   不需要写死服务提供者地址，注册中心基于接口名自动查询提供者IP

4. 服务接口监控与治理 

   Dubbo-admin与Dubbo-monitor提供了完善的服务接口管理与监控功能,针对不同应用的不同接口,可以进行多版本,多协议,多注册中心管理

但是缺点也很明显,就是只支持Java语言

## 4、zookeeper注册中心

### 1)、zookeeper结构

{% asset_img 2.jpg 图片上传失败 %}

Root:为zookeeper的根节点

Service:注册的服务存储节点

Type:消费方和提供方的储存节点

URL:具体的url地址

流程说明：

- 服务提供者启动时:向/dubbo/com.foo.BarService/providers目录下写入自己的URL地址
- 服务消费者启动时:订阅/dubbo/com.foo.BarService/providers目录下的提供者URL地址,并向/dubbo/com.foo.BarService/consumers目录下写入自己的URL地址
- 监控中心启动时:订阅/dubbo/com.foo.BarService目录下的所有提供者和消费者URL地址

### 2)、zookeeper客户端

Dubbo支持zkclient和curator两种Zookeeper客户端实现

#### zkclient客户端

从2.2.0版本开始缺省为zkclient实现,以提升zookeeper客户端的健状性。zkclient 是 Datameer 开源的一个 Zookeeper 客户端实现.

具体配置见后面的demo

#### curator客户端

从2.3.0版本开始支持可选curator实现.Curator是Netflix开源的一个Zookeeper客户端实现

因为这里使用的是zkclient客户端,所以对于curator客户端就不做阐述了

## 5、zookeeper安装

本文的服务器系统使用的是centOS6.7.Windows系统下的安装步骤一样,只是启动脚本改为.cmd后缀的

### 1)、下载与解压

zookeeper官方地址http://zookeeper.apache.org/

```
wget http://mirror.bit.edu.cn/apache/zookeeper/zookeeper-3.4.11/zookeeper-3.4.13.tar.gz
tar -zxvf zookeeper-3.4.13.tar.gz
```

解压完成之后会在下载的目录下多出一个zookeeper-3.4.13.tar.gz目录,cd进入这个目录

### 2)、zookeeper配置

在任意目录创建两个文件用来存放zookeeper的数据和日志文件夹

```
mkdir /usr/soft/zookeeper/data
mkdir /usr/soft/zookeeper/log
```

cd进入conf目录,拷贝zoo_sample.cfg文件为zoo.cfg.因为zookeeper启动默认加载zoo.cfg配置文件

```
cd conf
cp cp zoo_sample.cfg zoo.cfg
vim zoo.cfg
```

修改配置

```
dataDir=/usr/soft/zookeeper/data		# 数据保存路径,填写的是你之前创建的路径
dataLogDir=/usr/soft/zookeeper/log		# 新增日志保存路径,填写的是你之前创建的路径
clientPort=2181						   # 端口号,因为这里不是集群环境,所以保持默认就好
```

保存退出后进入到bin目录,启动zookeeper

### 3)、启动zookeeper

```
cd bin
./zkServer.sh start
```

看到这个提示信息说明你的zookeeper启动成功了

{% asset_img 3.jpg 图片上传失败 %}

你可以使用客户端连接zookeeper查看

```
./zkCli.sh
```

如果连接中出现这个,直接回车就好了

{% asset_img 4.jpg 图片上传失败 %}

连接成功之后可以输入`ls /`查看

{% asset_img 5.jpg 图片上传失败 %}

出现这个说明你的zookeeper已经启动成功了

因为是在虚拟机中测试,所有想要在本机中连接上zookeeper,最简单的办法就是将虚拟机的防火墙关掉,不然会连接超时

```
service iptables stop
```

## 6、Java Demo

系统要求:jdk1.6以上,maven3.0以上

### 1)、创建maven项目

项目结构:

- dubbo_api:服务接口
- dubbo_consumer:服务消费方
- dubbo_provider:服务提供方

{% asset_img 6.jpg 图片上传失败 %}

### 2)、导入依赖

a.在父模块中引入依赖,因为是提供方和消费方都需要的依赖,避免重复导入,所以在父模块中引入重复的依赖

```xml
<!-- 单元测试 -->
<dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.12</version>
    <scope>test</scope>
</dependency>
<!-- dubbo -->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>dubbo</artifactId>
    <version>2.5.6</version>
</dependency>
<!-- zookeeper -->
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.3.3</version>
</dependency>
<!-- zookeeper客户端 -->
<dependency>
    <groupId>com.github.sgroschupf</groupId>
    <artifactId>zkclient</artifactId>
    <version>0.1</version>
</dependency>
```

b.服务消费方和服务提供方都需要引入dubbo_api的依赖

```xml
<dependency>
    <groupId>com.siworae</groupId>
    <artifactId>dubbo_api</artifactId>
    <version>1.0-SNAPSHOT</version>
</dependency>
```

### 3)、接口定义实现

{% asset_img 7.jpg 图片上传失败 %}

```java
public interface IUserService {
    public Integer queryUserByUserId(Integer userId);
}
```

### 4)、provider服务提供方实现

{% asset_img 8.jpg 图片上传失败 %}

```java
@Service
public class UserServiceImpl implements IUserService {

    @Override
    public Integer queryUserByUserId(Integer userId) {
        System.out.println("收到请求,查询参数是:"+userId);
        return 1;
    }
}
```

定义一个测试类,用来发布服务

```java
public class publish {
    public static void main(String[] args) throws IOException {
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("spring-provider.xml");
        context.start();
        System.in.read();
    }
}
```

### 5)、consumer服务实现

{% asset_img 9.jpg 图片上传失败 %}

```java

```

### 6)、xml配置

a.consumer消费方配置

在main下新建资源文件夹resources,新建文件spring_consumer.xml

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context.xsd
        http://code.alibabatech.com/schema/dubbo
        http://code.alibabatech.com/schema/dubbo/dubbo.xsd">
    <!--启动包扫描-->
    <context:component-scan base-package="com.siworae"/>
    <!--应用名称-->
    <dubbo:application name="dubbo-consumer"/>
    <!--注册中心 设置为你们自己的zookeeper服务器的IP和端口-->
    <dubbo:registry address="zookeeper://192.168.230.220:2181"/>
    <!--配置订阅服务-->
    <dubbo:reference id="userService" interface="com.siworae.service.IUserService"/>
</beans>
```

b.provider提供方配置

在main下新建资源文件夹resources,新建文件spring_provider.xml

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context.xsd
        http://code.alibabatech.com/schema/dubbo
        http://code.alibabatech.com/schema/dubbo/dubbo.xsd">

    <!-- 配置包扫描 -->
    <context:component-scan base-package="com.siworae"/>
    <!--
        服务配置
            注册中心
            服务列表
            应用名称
    -->
    <!--应用名称-->
    <dubbo:application name="dubbo_provider"/>
    <!--注册中心-->
    <!--<dubbo:registry address="multicast://224.5.6.7:1234"/>-->
    <!--<dubbo:registry address="zookeeper://127.0.0.1:2181"/>-->
    <dubbo:registry address="zookeeper://192.168.230.220:2181"/>
    <!--框架协议-->
    <dubbo:protocol name="dubbo" port="20880"/>
    <!--配置注册的服务-->
    <dubbo:service interface="com.siworae.service.IUserService" ref="userServiceImpl"/>

</beans>

```

同时在provider和consumer的资源文件夹中添加log4j配置文件,不然运行会报异常

```xml
# Global logging configuration
log4j.rootLogger=DEBUG, stdout
# Console output...
log4j.appender.stdout=org.apache.log4j.ConsoleAppender
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
log4j.appender.stdout.layout.ConversionPattern=%5p [%t] - %m%n
```

### 7)、测试

在consumer新建测试类,测试是否可以调用成功,如果控制台成功打印数字"1".则说明调用成功

{% asset_img 10.jpg 图片上传失败 %}

```java
public class UserControllerTest {
    public static void main(String[] args){
        ApplicationContext context = new ClassPathXmlApplicationContext("spring-consumer.xml");
        UserController userController = (UserController) context.getBean("userController");
        Integer user = userController.queryUserByUserId(1);
        System.out.println(user);
    }
}
```

