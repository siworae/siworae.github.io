---
title: WebSocket实现Java后台消息推送
categories:
  - Java
  - websocket
tags:
  - websocket
abbrlink: 1e2cf77e
---

# WebSocket实现Java后台消息推送

## 1、WebSocket是什么

WebSocket是一种在单个TCP连接上进行全双工通信的协议,WebSocket协议的出现使得客户端和服务器之间的数据交换变得更加简单，允许服务端主动向客户端推送数据.

## 2、WebSocket与其他方式（Ajax轮询和long poll）

之前大多数的后台消息推送都是使用ajax轮询实现的,也就是利用异步js和Ajax在一定的时间间隔向服务器发起HTTP请求,将服务器的信息主动的拉取过来,但是这种方式本质上还是HTTP,不断的发起请求,断开请求,效率低下,而且非常消耗资源.

long poll其实和Ajax轮询差不多,也是浏览器发起请求,但是连接不断开,一直保存着,直到服务器有新的消息了,才带着新信息响应给浏览器,

以上这两种方式,对服务器都有一定的要求,都是不断的在建立HTTP连接,然后等待服务器处理请求

简单来说,这两种方式就相当于,服务器不会主动的联系(发起请求)浏览器,但是boss有命令,不管你多累,你都得接待好这些客人(请求).

Ajax轮询要求服务器有很快的处理速度,而long poll则要求服务器要有很高的并发了.

而WebSocket在建立连接之后,这种连接状态会一直保持,而且服务器也可以主动给浏览器发送消息,而不是被动的等待浏览器请求了之后才能给浏览器发送消息.

Ajax轮询的连接示意图

{% asset_img 2.jpg 图片上传失败 %}

long poll连接示意图

{% asset_img 4.png 图片上传失败 %}

WebSocket连接示意图

{% asset_img 3.jpg 图片上传失败 %}

只需要经过一次HTTP请求，就可以做到源源不断的信息传送了.(在程序设计中,这种设计叫做回调,即:你有信息了再来通知我,而不是我傻乎乎的每次跑来问你 )

websocket协议解决了之前协议的同步有延迟,并且非常消耗资源的情况,但是这样看来是不是觉得websocket也一样是连接一直建立着啊,不和long poll一样么?为什么它就能减少服务器资源的消耗呢?

这里就要探讨一下程序的访问过程了

我们经常进行的访问实际上是要经过两成代理的,浏览器发起HTTP请求,在经过Nginx等服务器的解析下,在发送给对应的handler处理请求.有点类似于我们打客服电话,打进去之后有一个智能系统(Nginx)根据不同的业务转接给不同的客服(handler)处理.

智能系统的处理速度是很快的,但是我们每次打客服电话的时候是卡在哪里呢?等客服接听对不对,这就相应地智能系统是能够满足需求的,但是由于客服的处理速度太慢,导致客服数量不足,满足不了接待需求.

这就相当于传统的Ajax轮询和long poll方式,而websocket就解决了这个问题.

我们打客服电话的时候,我们一直和智能系统保存着连接,不挂电话,当客服处理好了问题或者有消息的时候,客服只要通知智能系统就可以,而智能系统统一的通知我们,这样就能解决客服处理过慢的问题了

而且,websocket比传统方式还有一个优势,因为HTTP协议是无状态的,当你断开连接后服务器就不认识你是谁了,你每次连接都需要重新发送一个鉴别信息给服务器,然后服务器解析这个信息来知道你是谁,

虽然智能系统的处理速度很快,但是你每次接通电话之后都需要balabal的说一大堆和按键跳转来告诉系统你是谁和要处理什么业务,然后系统不断的把这些信息转交给客服,这样不但浪费智能系统和客服的处理时间,在转交的过程中也会占用时间.

而websocket不一样,只需要第一次连接的时候需要发送鉴别信息,之后连接状态一直保持着,这样就避免了HTTP协议的无状态性.

这样就解决了你需要不断的告诉系统你是谁,你要干什么.直到你挂断电话之前,系统都能知道你是谁,要干什么.

而且处理方式也从需要我们不断的问客服处理的怎么样了转变成客服有消息了就会通知我们,没有消息的时候这个通话连接就交给智能系统处理,避免了占用客服.

## ３、WebSocket与HTTP

Websocket是基于HTTP协议的，或者说借用了HTTP的协议来完成一部分握手.

{% asset_img 1.jpg 图片上传失败 %}

WebSocket也是HTTP5定义的一个协议,它和HTTP协议有关系,但是和HTTP协议本身没有关系,WebSocket是独立并且依赖于HTTP协议.

WebSocket与HTTP到底有什么区别和联系呢?

在HTTP协议中,1.0版本中最明显的特点就是无状态无连接,而在1.1的版本中新增了所谓的 `keep-alive`,也就是把多个HTTP请求合并为一个,但是WebSocker是一个新的协议,只是利用了HTTP完成了握手而已.

webSocker和HTTP有交集,但不是全部.

下面我们对比一下WebSocket和HTTP的请求头信息

这个是基于WebSocket的客户端请求(此处感谢wikipedia)

```
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: x3JJHMbDL1EzLkh9GBhXDw==
Sec-WebSocket-Protocol: chat, superchat
Sec-WebSocket-Version: 13
Origin: http://example.com
```

这个是基于WebSocket的服务器响应

```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: HSmrc0sMlYUkAGmm5OPpG2HaGWk=
Sec-WebSocket-Protocol: chat
```

HTTP的请求头

```
GET /item/WebSocket HTTP/1.1
Host: baike.baidu.com
Connection: keep-alive
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/68.0.3440.106 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8
```

HTTP的响应头

```
HTTP/1.1 200 OK
Content-Language: zh-CN
Content-Type: text/html; charset=utf-8
Server: Jetty(6.1.25)
```

在WebSocket请求中比HTTP请求中多了几个东西

```
Upgrade: websocket
Connection: Upgrade
```

这个就是WebSocket中主要也是核心的东西,告诉服务器发起的是WebSocket请求,而不是HTTP请求.

```
Sec-WebSocket-Key: x3JJHMbDL1EzLkh9GBhXDw==
Sec-WebSocket-Protocol: chat, superchat
Sec-WebSocket-Version: 13
```

Sec-WebSocket-Key : 是一个base64 encode的值,这个值是浏览器随机生成的,用来验证服务器是否切换至WebSocket协议

Sec-WebSocket-Protocol : 这个是用户自定义的一个字符串,用来定义同一个URL下不同的服务使用不同的协议.

Sec-WebSocket-Version : 这个是告诉服务器当前协议的版本.

服务器接收到请求之后返回的响应头也多了些东西,此时连接已经建立了.

```
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: HSmrc0sMlYUkAGmm5OPpG2HaGWk=
Sec-WebSocket-Protocol: chat
```

Upgrade和Connection : 告诉浏览器,我已经切换到websocket协议了

Sec-WebSocket-Accept : 这个是服务器确认并且加密过的Sec-WebSocket-Key,告诉浏览器我用的真的是webSocket协议,不是骗你的哈!

Sec-WebSocket-Protocol : 告诉浏览器最终使用的协议版本

当服务器返回响应后,HTTP的工作已经全部结束了,浏览器和服务器就可以抛弃HTTP，开始使用WebSocket协议来工作了

## 4、demo

### 1)、创建maven工程

项目目录一览

{% asset_img 5.png 图片上传失败 %}

### 2)、项目添加依赖

主要是在spring项目中新增websocket依赖,下面贴出完整的xml配置文件

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>cn.justsoul.study</groupId>
    <artifactId>websocket_demo</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>war</packaging>
    <properties>
        <spring.version>4.3.10.RELEASE</spring.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-core</artifactId>
            <version>${spring.version}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-websocket</artifactId>
            <version>${spring.version}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-webmvc</artifactId>
            <version>${spring.version}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-web</artifactId>
            <version>${spring.version}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context-support</artifactId>
            <version>${spring.version}</version>
        </dependency>
        <dependency>
            <groupId>javax.servlet</groupId>
            <artifactId>javax.servlet-api</artifactId>
            <version>3.1.0</version>
            <scope>provided</scope>
        </dependency>
        <!-- 日志打印相关的jar -->
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
    </dependencies>
    <build>
        <finalName>websocket</finalName>
        <plugins>
            <!-- 配置Tomcat插件 -->
            <plugin>
                <groupId>org.apache.tomcat.maven</groupId>
                <artifactId>tomcat7-maven-plugin</artifactId>
                <configuration>
                    <port>8080</port>
                    <path>/ws</path>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

### 3)、配置xml

spring.xml

```
<context:annotation-config/>
<context:component-scan base-package="com.siworae.websocket"/>
```

spring_mvc.xml

```
<context:component-scan base-package="com.siworae.websocket" />
<mvc:annotation-driven/>
<mvc:resources mapping="/js/**" location="/js/"/>
<bean id="viewResolver" class="org.springframework.web.servlet.view.InternalResourceViewResolver">
    <property name="prefix" value="/WEB-INF/jsp/"/>
    <property name="suffix" value=".jsp"/>
</bean>
```

web.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
         version="4.0">
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:spring.xml</param-value>
    </context-param>

    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    <servlet>
        <servlet-name>dispatcherServlet</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath:spring_mvc.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
        <async-supported>true</async-supported>
    </servlet>
    <servlet-mapping>
        <servlet-name>dispatcherServlet</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>

</web-app>
```

### 4)、创建websocket拦截器

新建一个MyHandshakeInterceptor类,并继承HttpSessionHandshakeInterceptor.

拦截器主要处理的是在握手建立连接前后需要处理的事情,这个demo只是简单的在控制台输出了一句话.

在握手建立连接前做了一个处理,用来区分WebSocketHandler

```
public class MyHandshakeInterceptor extends HttpSessionHandshakeInterceptor {
    @Override
    public boolean beforeHandshake(ServerHttpRequest request, ServerHttpResponse response, WebSocketHandler wsHandler, Map<String, Object> attributes) throws Exception {
        System.out.println("beforeHandshake");
        if (request instanceof ServletServerHttpRequest) {
            ServletServerHttpRequest servletRequest = (ServletServerHttpRequest) request;
            HttpSession session = servletRequest.getServletRequest().getSession(false);
            if (session != null) {
                //使用userName区分WebSocketHandler，以便定向发送消息
                String userName = (String) session.getAttribute("SESSION_USERNAME");
                if (userName == null) {
                    userName = "system-" + session.getId();
                }
                attributes.put("WEBSOCKET_USERNAME", userName);
            }
        }
        return true;
    }
    @Override
    public void afterHandshake(ServerHttpRequest request, ServerHttpResponse response, WebSocketHandler wsHandler, Exception ex) {
        System.out.println("afterHandshake");
    }
}
```

### 5)、创建websocket处理器

在有了拦截器之后,就需要有处理器来处理建立连接之后的事了.

websocket处理器继承了TextWebSocketHandler类,当然你也可以继承AbstractWebSocketHandler

TextWebSocketHandler继承自AbstractWebSocketHandler类用来处理文本消息。
BinaryWebSocketHandler继承自AbstractWebSocketHandler类用来处理二进制消息。

```
public class MyWebSocketHandler extends TextWebSocketHandler {

    //已建立连接的用户
    private static final ArrayList<WebSocketSession> users = new ArrayList<WebSocketSession>();

    /**
     * 当新连接建立的时候，被调用
     * 连接成功时候，会触发页面上onOpen方法
     * @param session
     * @throws Exception
     */
    @Override
    public void afterConnectionEstablished(WebSocketSession session) throws Exception {
        System.out.println("afterConnectionEstablished");
        users.add(session);
        System.out.println("zxrs:---->"+users.size());
        super.afterConnectionEstablished(session);
    }

    /**
     * 处理前端发送的文本信息
     * js调用websocket.send时候，会调用该方法
     * @param session
     * @param message
     * @throws Exception
     */
    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) throws Exception {
        String username = (String) session.getAttributes().get("WEBSOCKET_USERNAME");
        // 获取提交过来的消息详情
        System.out.println("收到用户 " + username + "的消息:" + message.toString());
        //回复一条信息，
        session.sendMessage(new TextMessage("reply msg:" + message.getPayload()));
    }

    /**
     * 当连接关闭时被调用
     * @param session
     * @param status
     * @throws Exception
     */
    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) throws Exception {
        System.out.println("afterConnectionClosed");
        users.remove(session);
        System.out.println("zxrs:---->"+users.size());
        super.afterConnectionClosed(session, status);
    }

    /**
     * 给所有在线用户发送消息
     * 自定义方法
     * @param message
     */
    public void sendMessageToUsers(TextMessage message) {
        System.out.println(users.toString());
        for (WebSocketSession user : users) {
            try {
                if (user.isOpen()) {
                    user.sendMessage(message);
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    /**
     * 给某个用户发送消息
     * 自定义方法
     * @param userName
     * @param message
     */
    public void sendMessageToUser(String userName, TextMessage message) {
        for (WebSocketSession user : users) {
            if (user.getAttributes().get("WEBSOCKET_USERNAME").equals(userName)) {
                try {
                    if (user.isOpen()) {
                        user.sendMessage(message);
                    }else {
                        System.out.println("------------>连接关闭了!");
                    }
                } catch (IOException e) {
                    e.printStackTrace();
                }
                break;
            }
        }
    }
}
```

### 6)、创建websocket配置类

我们使用它来加载咱们的拦截器和处理器,使能建立websocket连接.

这里使用了一种备用方案,就额是sockjs,这个是可以在不支持websocket的浏览器中只用这个协议来连接

SockJS是WebSocket 技术的一种模拟.SockJS会尽可能对应WebSocket API,但如果WebSocket技术不可用的话,就会选择另外的通信方式协议.

```
@Configuration
@EnableWebSocket
public class WebSocketConfig extends WebMvcConfigurerAdapter implements WebSocketConfigurer {

    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {

        String websocket_url = "/websocket/socketServer.do";
        registry.addHandler(new MyWebSocketHandler(), websocket_url).addInterceptors(new MyHandshakeInterceptor());

        String sockjs_url = "/sockjs/socketServer.do";
        registry.addHandler(new MyWebSocketHandler(), sockjs_url).addInterceptors(new MyHandshakeInterceptor()).withSockJS();

    }
}
```

7)、创建controller

```
@Controller
public class WebSocketController {

    @Bean//这个注解会从Spring容器拿出Bean
    public MyWebSocketHandler infoHandler() {
        return new MyWebSocketHandler();
    }

    @RequestMapping(value = "/index", method = RequestMethod.GET)
    public ModelAndView goIndex(){
        ModelAndView mv = new ModelAndView();
        mv.setViewName("index");
        return mv;
    }

    @RequestMapping(value = "/login1", method = RequestMethod.GET)
    public ModelAndView index(){
        ModelAndView mv = new ModelAndView();
        mv.setViewName("login");
        return mv;
    }

    @RequestMapping("/login")
    public ModelAndView login(HttpServletRequest request) {
        String username = request.getParameter("username");
        System.out.println(username + "dl");
        HttpSession session = request.getSession();
        session.setAttribute("SESSION_USERNAME", username);
        ModelAndView mv = new ModelAndView();
        mv.setViewName("websocket");
        return mv;
    }

    @RequestMapping("/send")
    @ResponseBody
    public String send(HttpServletRequest request) {
        String username = request.getParameter("username");
        System.out.println("---->"+username);
        infoHandler().sendMessageToUser(username, new TextMessage("你好，欢迎测试！！！！"));
        return null;
    }
}
```

7)、创建前台测试页面

index.jsp

用来简单的测试websocket连接和sockjs连接

```
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
  <head>
    <title>Websocket Demo</title>
  </head>
  <body>
  Websocket Demo
  <hr>
  <button id="ws">使用ws创建连接</button>
  <button id="sockjs">使用sockjs创建连接</button>
  <button id="close">关闭websocket连接</button>

  </body>
</html>
<script type="text/javascript" src="js/jquery-3.2.1.min.js"></script>
<script type="text/javascript" src="js/sockjs-1.1.0.min.js"></script>
<script type="text/javascript">
  $(function () {
      $("#ws").click(function () {
           ws_connect();
      });
      $("#sockjs").click(function () {
          sockjs_connect();
      });
      $("#close").click(function () {
          websocket.close();
      });

  });
  function ws_connect() {
      websocket = new WebSocket("ws://localhost:8080/ws/websocket/socketServer.do");
  }
  function sockjs_connect() {
      websocket = new SockJS("http://localhost:8080/ws/sockjs/socketServer.do");
  }

</script>
```

login.jsp

用来实现登陆

```
<%@ page language="java" contentType="text/html; charset=utf-8" pageEncoding="utf-8" %>
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">
<html>
<body>
<h2>Wellcome</h2>

<body>
<form action="login">
    登录名：<input type="text" name="username"/>
    <input type="submit" value="登录"/>
</form>
</body>
</body>
</html>
```

websocket.jsp

登陆后跳转此页面,进行前后台的文本消息发送

```
<%@ page language="java" contentType="text/html; charset=utf-8"
         pageEncoding="utf-8" %>
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">
<html>
<head>
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8">
    <title>Java API for WebSocket (JSR-356)</title>
</head>
<body>
<script type="text/javascript" src="http://cdn.bootcss.com/jquery/3.1.0/jquery.min.js"></script>
<script type="text/javascript" src="http://cdn.bootcss.com/sockjs-client/1.1.1/sockjs.js"></script>
<script type="text/javascript">
    var websocket = null;
    if ('WebSocket' in window) {
        //Websocket的连接
        websocket = new WebSocket("ws://localhost:8080/ws/websocket/socketServer.do");//WebSocket对应的地址
    }
    else if ('MozWebSocket' in window) {
        //Websocket的连接
        websocket = new MozWebSocket("ws://localhost:8080/ws/websocket/socketServer.do");//SockJS对应的地址
    }
    else {
        //SockJS的连接
        websocket = new SockJS("http://localhost:8080/ws/sockjs/socketServer.do");    //SockJS对应的地址
    }
    websocket.onopen = onOpen;
    websocket.onmessage = onMessage;
    websocket.onerror = onError;
    websocket.onclose = onClose;

    function onOpen(openEvt) {
        //alert(openEvt.Data);
    }

    function onMessage(evt) {
        alert(evt.data);
    }
    function onError() {
    }
    function onClose() {
    }

    function doSend() {
        if (websocket.readyState == websocket.OPEN) {
            var msg = document.getElementById("inputMsg").value;
            websocket.send(msg);//调用后台handleTextMessage方法
            alert("发送成功!");
        } else {
            alert("连接失败!");
        }
    }

    window.close = function () {
        websocket.onclose();
    }


</script>
请输入：<textarea rows="3" cols="100" id="inputMsg" name="inputMsg"></textarea>
<button onclick="doSend();">发送</button>

</body>
</html>
```

到此,spring框架实现websocket的消息推送已经实现了.

websocket请求地址一定要结合你自己的环境配置正确设置.不然可能会出现404

完整的项目代码可以去我的GitHub上下载[siworae](https://github.com/siworae/websocket)