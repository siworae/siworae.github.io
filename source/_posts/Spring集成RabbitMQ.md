---
title: Spring集成RabbitMQ
categories:
  - Spring
  - RabbitMQ
tags:
  - RabbitMQ
abbrlink: ee9f7bfc
date: 2019-01-18 15:53:47
---

# Spring集成RabbitMQ

## 1、RabbitMQ

MQ ,也就是消息队列是一种进程间通信或同一进程的不同线程间的通信方式,软件的贮列用来处理一系列的输入,通常是来自用户.消息队列提供了异步的通信协议,每一个贮列中的纪录包含详细说明的数据,包含发生的时间,输入设备的种类,以及特定的输入参数,也就是说,消息的发送者和接收者不需要同时与消息队列互交.消息会保存在队列中,直到接收者取回它。

目前,市面上拥有MQ的有很多,比如RabbMQ,ActiveMQ,ZeroMQ,Kafka等,今天我们使用的是RabbitMQ,至于为什么选用RabbitMQ,可以参考这篇博文:https://blog.csdn.net/linsongbin1/article/details/47781187

### 消息队列的使用场景

在项目中,将一些无需即时返回且耗时的操作提取出来,进行了异步处理,而这种异步处理的方式大大的节省了服务器的请求响应时间,从而提高了系统的吞吐量.

## 2、RabbitMQ的下载与安装

### 1)、Erlang环境准备

因为RabbitMQ是依赖与Erlang的,所有需要先安装Erlang的,

#### 1.1、Erlang下载与安装

[官方下载地址](http://www.erlang.org/downloads)

根据自己的电脑系统版本下载对于版本的安装包,一直默认安装就行

### 2)、RabbitMQ Service下载与安装

[RabbitMQ官方下载](http://www.rabbitmq.com/download.html)

根据自己的电脑系统版本下载对于版本的安装包,全部默认安装就行

### 3)、测试

打开命令行,进入RabbitMQ的安装目录下的sbin目录,输入`rabbitmqctl.bat status`

如果出现以下界面,说明是安装成功的.并且此时rabbitmq已经正常启动了

{% asset_img 1.jpg 图片上传失败 %}

如果启动报错,可以尝试将C:\Windows目录下的`.erlang.cookie`文件拷贝覆盖到你的用户目录下`C:\Users\userName`

### 4)、管理界面

在命令行输入`rabbitmq-plugins.bat list`

{% asset_img 2.jpg 图片上传失败 %}

 在输入`rabbitmq-plugins.bat enable rabbitmq_management`开启管理界面插件

{% asset_img 3.jpg 图片上传失败 %}

管理界面的默认端口是15672,所以在浏览器输入127.0.0.1:15672即可进入管理界面

{% asset_img 4.jpg 图片上传失败 %}

然后你就可以用默认的账户和密码登陆了

账号:`guest`

密码:`guest`

这里对管理界面的一些操作就不做详细阐述了

## 3、RabbitMQ快速入门

在RabbitMQ的官网中给了六种快速入门的demo.[有兴趣的可以去官网看看](http://www.rabbitmq.com/getstarted.html)

这里只介绍四种队列的模式:work queues,publish/subscribe,routing,topics

### 1)、work queues

自动回执和手动回执

在工作队列中,mq提供了两种回执方式,默认是自动回执,

其中设置不同回执方式就是在将队列加入到监听中指定

```java
channel.basicConsume(QUEUE_NAME,true,consumer);
```

1. 第一个参数 QUEUE_NAME : 队列的名称
2. 第二个参数 指定回执方式 : 默认true(自动回执),false(手动回执)
3. 第三个参数 consumer : consumer的实现类

PS:在设置为手动回执之后,我们就需要在consumer执行完之后手动发送回执消息给mq服务器了

```
channel.basicAck(envelope.getDeliveryTag(),false);
```

完整的demo代码请参考我的[GitHub](https://github.com/siworae/RabbitMQ/tree/master/src/main/java/com/siworae/workQueues)

### 2)、publish/subscribe

在发布与订阅模式中,在生产者和队列中新增了一个角色,也就是交换机的概念,交换机有四种类型,也就对应后面的四种模式.其中最常用的是fanout, direct ,topic.

以下是三种模式下的示意图

fanout

{% asset_img 5.jpg 图片上传失败 %}

direct 

{% asset_img 6.jpg 图片上传失败 %}

topic

{% asset_img 7.jpg 图片上传失败 %}

在生产者发送消息至交换机中时,会一起发送一个key值给交换机,交换机就会根据这个key值去匹配队列和交换机绑定的时候设置的key,如果匹配就将消息发送给队列.

可以理解为交换机是一个保安,守护着队列对外的入口,当生产者生产出的消息想要进入队列的时候,就需要带着key,也就是门票了,然后保安就会检查这个门票,看看这个门票符合那个就放它进哪个队列,如果不符合就不让进,也就是将这个消息丢弃了.

三种模式的区别是:

fanout : 不进行key校验,只要队列绑定了交换机我就将消息发送给你

direct : key值为固定值,通用性较差

topic : key值可以使用通配符模式

```
.:分割字符串
*:匹配一个单词 不能为空
#:匹配多个单词 可为空
```

完整的demo代码请参考我的[GitHub](https://github.com/siworae/RabbitMQ/tree/master/src/main/java/com/siworae/subscrible)

## 4、Spring集成RabbitMQ

1)、引入依赖

```xml
<dependency>
    <groupId>org.springframework.amqp</groupId>
    <artifactId>spring-rabbit</artifactId>
    <version>1.6.5.RELEASE</version>
</dependency>
```

2)、模板类

```java
public class TestSpringRabbitmq {
    public static void main(String[] args) {
        ApplicationContext ac=new ClassPathXmlApplicationContext("spring-rabbitmq.xml");
        AmqpTemplate amqpTemplate= (AmqpTemplate) ac.getBean("amqpTemplate");
        //amqpTemplate.convertAndSend("hello spring rabbitmq");
        amqpTemplate.convertAndSend("springExchange02","query.queryById","hello spring rabbitmq");
    }
}
```

3)、配置消费类

```java
public class ConsumerService {

    public  void test(Object obj){
        System.out.println("消息者01收到消息-->"+obj);
    }
    public  void test02(Object obj){
        System.out.println("消息者02收到消息-->"+obj);
    }
}
```

4)、设置配置文件

resources资源目录新建spring-rabbitmq.xml

```xml
<?xml version="1.0" encoding="utf-8" ?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:rabbit="http://www.springframework.org/schema/rabbit"
       xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context.xsd
	    http://www.springframework.org/schema/rabbit
	    http://www.springframework.org/schema/rabbit/spring-rabbit.xsd" >
    <context:component-scan base-package="com.siworae"/>
    <!--
       配置工厂类
    -->
    <rabbit:connection-factory id="connectionFactory" host="127.0.0.1" port="5672" virtual-host="/siworae"sername="siworae"password="123456"></rabbit:connection-factory>

    <rabbit:admin connection-factory="connectionFactory"></rabbit:admin>
    <!--
      配置交换机
    -->
    <rabbit:topic-exchange name="springExchange">
        <rabbit:bindings>
            <rabbit:binding pattern="spring.#" queue="spring_test"></rabbit:binding>
        </rabbit:bindings>
    </rabbit:topic-exchange>

    <rabbit:topic-exchange name="springExchange02">
        <rabbit:bindings>
            <rabbit:binding pattern="query.*" queue="spring_test_02"></rabbit:binding>
        </rabbit:bindings>
    </rabbit:topic-exchange>
    <!--
       配置队列
    -->
    <rabbit:queue name="spring_test"></rabbit:queue>

    <rabbit:queue name="spring_test_02"></rabbit:queue>
    <!--
       配置监听容器
    -->
    <rabbit:listener-container connection-factory="connectionFactory">
        <rabbit:listener ref="consumerService" method="test" queue-names="spring_test"></rabbit:listener>
        <rabbit:listener ref="consumerService" method="test02" queue-names="spring_test_02"></rabbit:listener>
    </rabbit:listener-container>
    <!--
         配置amqpTemplate
    -->
    <rabbit:template id="amqpTemplate" connection-factory="connectionFactory" exchange="springExchange" routing-key="spring.test"></rabbit:template>
</beans>
```

完整demo可以clone[GitHub](https://github.com/siworae/RabbitMQ/tree/master/src/main/java/com/siworae/spring_rabbitmq)