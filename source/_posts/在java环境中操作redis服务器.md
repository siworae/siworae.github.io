---
title: 在java环境中操作redis服务器
categories:
  - Java
  - redis
tags:
  - redis
abbrlink: b5f3cf78
date: 2019-01-06 12:49:40
---

# 在java环境中操作redis服务器

工具:idea

环境:jdk1.8

## 1、环境配置

在ieda中新建maven工程,导入依赖

```xml
<!-- 单元测试 -->
<dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.12</version>
    <scope>test</scope>
</dependency>
<!-- spring测试 -->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-test</artifactId>
    <version>4.3.7.RELEASE</version>
</dependency>
<!-- spring-data-redis -->
<dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-redis</artifactId>
    <version>1.8.1.RELEASE</version>
</dependency>
<!-- jedis -->
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>2.9.0</version>
</dependency>
<!-- log4j日志打印 -->
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-log4j12</artifactId>
    <version>1.7.2</version>
</dependency>
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-api</artifactId>
    <version>1.7.2</version>
</dependency>
```

## 2、配置文件

在main下面新建资源文件夹resources.在resources下新建spring配置文件spring.xml文件

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:p="http://www.springframework.org/schema/p"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd">
    
    <!-- 连接池配置 -->
    <bean id="jedisPoolConfig" class="redis.clients.jedis.JedisPoolConfig">
        <!-- 最大连接数  -->
        <property name="maxTotal" value="1024"/>
        <!-- 最大 空闲连接数 -->
        <property name="maxIdle" value="200"/>
        <!-- 获取连接时最大等待毫秒数 -->
        <property name="maxWaitMillis" value="10000"/>
        <!-- 在获取连接时检查有效性 -->
        <property name="testOnBorrow" value="true"/>
    </bean>
    <!-- 设置redis的连接 -->
    <bean id="redisSentinelConfiguration" class="org.springframework.data.redis.connection.RedisSentinelConfiguration">
        <property name="master">
            <bean class="org.springframework.data.redis.connection.RedisNode">
                <property name="name" value="mymaster"></property>
            </bean>
        </property>
        <!-- 设置sentinel -->
        <property name="sentinels">
            <set>
                <bean class="org.springframework.data.redis.connection.RedisNode">
                    <!--value改为你们自己的sentinel服务器的IP和端口 -->
                    <constructor-arg name="host" value="192.168.230.221"></constructor-arg>
                    <constructor-arg name="port" value="26379"></constructor-arg>
                </bean>
                <bean class="org.springframework.data.redis.connection.RedisNode">
                    <constructor-arg name="host" value="192.168.230.221"></constructor-arg>
                    <constructor-arg name="port" value="26380"></constructor-arg>
                </bean>
                <bean class="org.springframework.data.redis.connection.RedisNode">
                    <constructor-arg name="host" value="192.168.230.221"></constructor-arg>
                    <constructor-arg name="port" value="26381"></constructor-arg>
                </bean>
            </set>
        </property>
    </bean>

    <!-- 客户端连接工厂 -->
    <bean id="jedisConnFactory" class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory">
        <!-- 连接池引用 -->
        <constructor-arg name="poolConfig" ref="jedisPoolConfig"/>
        <constructor-arg name="sentinelConfig" ref="redisSentinelConfiguration"/>
        <!-- value改为你们自己redis服务器的访问密码 -->
        <property name="password" value="123456"></property>
    </bean>
	<!-- redis模板类配置 -->
    <bean id="redisTemplate" class="org.springframework.data.redis.core.RedisTemplate"
          p:connection-factory-ref="jedisConnFactory">
        <!-- 配置序列化操作,指定key值和hashkey的序列化方式 -->
        <property name="keySerializer">
            <bean class="org.springframework.data.redis.serializer.StringRedisSerializer"></bean>
        </property>
        <property name="hashKeySerializer">
            <bean class="org.springframework.data.redis.serializer.StringRedisSerializer"></bean>
        </property>
    </bean>
</beans>
```

在resources下新建log4j配置文件log4j.properties

这个配置文件基本通用,可以不用改,直接拷过去就能用了

```xml
# Global logging configuration
log4j.rootLogger=DEBUG, stdout
# Console output...
log4j.appender.stdout=org.apache.log4j.ConsoleAppender
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
log4j.appender.stdout.layout.ConversionPattern=%5p [%t] - %m%n
```

## 3、启动项目

然后我们借助spring的测试环境测试

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = {"classpath:spring.xml"})
public class TestSpringDataRedis {
    @Resource(name = "redisTemplate")
    private ValueOperations<String,Object> valueOperations;
    @Test
    public void test(){
//        valueOperations.set("a","123");
        System.out.println(valueOperations.get("a"));
    }
}
```