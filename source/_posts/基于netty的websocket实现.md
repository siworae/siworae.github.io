---
title: 基于netty的websocket实现
abbrlink: 64c322b1
date: 2019-01-18 10:07:16
categories:
tags:
---

# 基于Netty的websocket简单实现

参考文献:[Netty 4.x 用户指南](https://waylau.com/netty-4-user-guide/)

netty是个啥?其实netty就是一个将JDK原生的NIO封装好的一个框架,可以让你不需要关注JDK原生NIO的各种概念便可以开发出一个网络应用.

官方点的说法就是Netty是由JBOSS提供的一个java开源框架。Netty提供异步的、事件驱动的网络应用程序框架和工具，用以快速开发高性能、高可靠性的网络服务器和客户端程序。

## Netty的优势

1. 使用JDK自带的NIO需要了解太多的概念，编程复杂，一不小心bug横飞
2. Netty底层IO模型随意切换，而这一切只需要做微小的改动，改改参数，Netty可以直接从NIO模型变身为IO模型
3. Netty自带的拆包解包，异常检测等机制让你从NIO的繁重细节中脱离出来，让你只需要关心业务逻辑
4. Netty解决了JDK的很多包括空轮询在内的bug
5. Netty底层对线程，selector做了很多细小的优化，精心设计的reactor线程模型做到非常高效的并发处理
6. 定制能力强，可以通过ChannelHandler对通信框架进行灵活地扩展
7. Netty社区活跃，遇到问题随时邮件列表或者issue
8. Netty已经历各大rpc框架，消息中间件，分布式通信中间件线上的广泛验证，健壮性无比强大

下面就是一个利用Netty来实现的网络聊天室

## DEMO

### 引入依赖

```xml
<!-- https://mvnrepository.com/artifact/io.netty/netty-all -->
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-all</artifactId>
    <version>4.1.30.Final</version>
</dependency>
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-core</artifactId>
    <version>2.7.0</version>
</dependency>
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.7.0</version>
</dependency>
```

### 客户端处理类,用于处理合HTTP请求

```java
/**
 * 客户端处理类
 * 扩展 SimpleChannelInboundHandler 用于处理 FullHttpRequest信息
 */
public class HttpRequestHandler extends SimpleChannelInboundHandler<FullHttpRequest> {
    private final String wsUri;
    private static final File INDEX;

    static {
        URL location = HttpRequestHandler.class.getProtectionDomain().getCodeSource().getLocation();
        try {
            String path = location.toURI() + "WebsocketChatClient.html";
            path = !path.contains("file:") ? path : path.substring(5);
            INDEX = new File(path);
        } catch (URISyntaxException e) {
            throw new IllegalStateException("Unable to locate WebsocketChatClient.html", e);
        }
    }

    public HttpRequestHandler(String wsUri) {
        this.wsUri = wsUri;
    }

    @Override
    public void channelRead0(ChannelHandlerContext ctx, FullHttpRequest request) throws Exception {
        //如果请求是WebSocket升级,递增引用计数器(保留)并且将它传递给在ChannelPipeline中的下个ChannelInboundHandler
        if (wsUri.equalsIgnoreCase(request.getUri())) {
            ctx.fireChannelRead(request.retain());
        } else {
            //处理符合 HTTP 1.1的 “100 Continue” 请求
            if (HttpHeaders.is100ContinueExpected(request)) {
                send100Continue(ctx);
            }

            //读取默认的 WebsocketChatClient.html 页面
            RandomAccessFile file = new RandomAccessFile(INDEX, "r");

            HttpResponse response = new DefaultHttpResponse(request.getProtocolVersion(), HttpResponseStatus.OK);
            response.headers().set(HttpHeaders.Names.CONTENT_TYPE, "text/html; charset=UTF-8");

            boolean keepAlive = HttpHeaders.isKeepAlive(request);

            //判断keepalive是否在请求头里面
            if (keepAlive) {
                response.headers().set(HttpHeaders.Names.CONTENT_LENGTH, file.length());
                response.headers().set(HttpHeaders.Names.CONNECTION, HttpHeaders.Values.KEEP_ALIVE);
            }
            //写HttpResponse到客户端
            ctx.write(response);

            //写index.html到客户端,判断SslHandler是否在ChannelPipeline来决定是使用DefaultFileRegion还是ChunkedNioFile
            if (ctx.pipeline().get(SslHandler.class) == null) {
                ctx.write(new DefaultFileRegion(file.getChannel(), 0, file.length()));
            } else {
                ctx.write(new ChunkedNioFile(file.getChannel()));
            }
            //写并刷新LastHttpContent到客户端，标记响应完成
            ChannelFuture future = ctx.writeAndFlush(LastHttpContent.EMPTY_LAST_CONTENT);
            //如果keepalive没有要求，当写完成时，关闭Channel
            if (!keepAlive) {
                future.addListener(ChannelFutureListener.CLOSE);
            }
            file.close();
        }
    }

    private static void send100Continue(ChannelHandlerContext ctx) {
        FullHttpResponse response = new DefaultFullHttpResponse(HttpVersion.HTTP_1_1, HttpResponseStatus.CONTINUE);
        ctx.writeAndFlush(response);
    }

	@Override
	public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause)
			throws Exception {
    	Channel incoming = ctx.channel();
		System.out.println("Client:"+incoming.remoteAddress()+"异常");
        // 当出现异常就关闭连接
        cause.printStackTrace();
        ctx.close();
	}
}
```

### 处理 WebSocket frame

```java
/**
 * TextWebSocketFrameHandler 继承自 SimpleChannelInboundHandler，这个类实现了 ChannelInboundHandler 接口，
 * ChannelInboundHandler 提供了许多事件处理的接口方法，然后你可以覆盖这些方法。
 * 现在仅仅只需要继承 SimpleChannelInboundHandler 类而不是你自己去实现接口方法。
 */
public class TextWebSocketFrameHandler extends
		SimpleChannelInboundHandler<TextWebSocketFrame> {
	
	public static ChannelGroup channels = new DefaultChannelGroup(GlobalEventExecutor.INSTANCE);

	/**
	 * 每当从服务端读到客户端写入信息时，将信息转发给其他客户端的 Channel。
	 * 其中如果你使用的是 Netty 5.x 版本时，需要把 channelRead0() 重命名为messageReceived()
	 * @param ctx
	 * @param msg
	 * @throws Exception
	 */
	@Override
	protected void channelRead0(ChannelHandlerContext ctx,TextWebSocketFrame msg) throws Exception {
		Channel incoming = ctx.channel();
		for (Channel channel : channels) {
            if (channel != incoming){
                channel.writeAndFlush(new TextWebSocketFrame("[" + incoming.remoteAddress() + "]" + msg.text()));
            } else {
            	channel.writeAndFlush(new TextWebSocketFrame("[you]" + msg.text() ));
            }
        }
	}

	/**
	 * 每当从服务端收到新的客户端连接时，客户端的 Channel 存入 ChannelGroup 列表中，并通知列表中的其他客户端 Channel
	 * @param ctx
	 * @throws Exception
	 */
	@Override
    public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
        Channel incoming = ctx.channel();
        
        // Broadcast a message to multiple Channels
        channels.writeAndFlush(new TextWebSocketFrame("[SERVER] - " + incoming.remoteAddress() + " 加入"));
        
        channels.add(incoming);
		System.out.println("Client:"+incoming.remoteAddress() +"加入");
    }

	/**
	 * 每当从服务端收到客户端断开时，客户端的 Channel 自动从 ChannelGroup 列表中移除了，并通知列表中的其他客户端 Channel
	 * @param ctx
	 * @throws Exception
	 */
	@Override
    public void handlerRemoved(ChannelHandlerContext ctx) throws Exception {  // (3)
        Channel incoming = ctx.channel();
        
        // Broadcast a message to multiple Channels
        channels.writeAndFlush(new TextWebSocketFrame("[SERVER] - " + incoming.remoteAddress() + " 离开"));
        
		System.out.println("Client:"+incoming.remoteAddress() +"离开");

        // A closed Channel is automatically removed from ChannelGroup,
        // so there is no need to do "channels.remove(ctx.channel());"
    }

	/**
	 * 服务端监听到客户端活动
	 * @param ctx
	 * @throws Exception
	 */
	@Override
	public void channelActive(ChannelHandlerContext ctx) throws Exception { // (5)
        Channel incoming = ctx.channel();
		System.out.println("Client:"+incoming.remoteAddress()+"在线");
	}

	/**
	 * 服务端监听到客户端不活动
	 * @param ctx
	 * @throws Exception
	 */
	@Override
	public void channelInactive(ChannelHandlerContext ctx) throws Exception { // (6)
        Channel incoming = ctx.channel();
		System.out.println("Client:"+incoming.remoteAddress()+"掉线");
	}

	/**
	 * 当出现 Throwable 对象才会被调用，即当 Netty 由于 IO 错误或者处理器在处理事件时抛出的异常时。
	 * 在大部分情况下，捕获的异常应该被记录下来并且把关联的 channel 给关闭掉。
	 * 然而这个方法的处理方式会在遇到不同异常的情况下有不同的实现，比如你可能想在关闭连接之前发送一个错误码的响应消息。
	 * @param ctx
	 * @param cause
	 * @throws Exception
	 */
	@Override
	public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause)	// (7)
			throws Exception {
    	Channel incoming = ctx.channel();
		System.out.println("Client:"+incoming.remoteAddress()+"异常");
        // 当出现异常就关闭连接
        cause.printStackTrace();
        ctx.close();
	}
}
```

### 初始化ChannelPipeline给channel

```java
public class WebsocketChatServerInitializer extends
		ChannelInitializer<SocketChannel> {

	@Override
    public void initChannel(SocketChannel ch) throws Exception {
		 ChannelPipeline pipeline = ch.pipeline();

        pipeline.addLast(new HttpServerCodec());
		pipeline.addLast(new HttpObjectAggregator(64*1024));
		pipeline.addLast(new ChunkedWriteHandler());
		pipeline.addLast(new HttpRequestHandler("/ws"));
		pipeline.addLast(new WebSocketServerProtocolHandler("/ws"));
		pipeline.addLast(new TextWebSocketFrameHandler());

    }
}
```

### 编写主方法启动服务

```java
public class WebsocketChatServer {

    private int port;

    public WebsocketChatServer(int port) {
        this.port = port;
    }

    public void run() throws Exception {
        
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
             .channel(NioServerSocketChannel.class)
             .childHandler(new WebsocketChatServerInitializer())
             .option(ChannelOption.SO_BACKLOG, 128)
             .childOption(ChannelOption.SO_KEEPALIVE, true);
            
    		System.out.println("WebsocketChatServer 启动了");
    		
            // 绑定端口，开始接收进来的连接
            ChannelFuture f = b.bind(port).sync();

            // 等待服务器  socket 关闭 。
            // 在这个例子中，这不会发生，但你可以优雅地关闭你的服务器。
            f.channel().closeFuture().sync();

        } finally {
            workerGroup.shutdownGracefully();
            bossGroup.shutdownGracefully();
            
    		System.out.println("WebsocketChatServer 关闭了");
        }
    }

    public static void main(String[] args) throws Exception {
        int port;
        if (args.length > 0) {
            port = Integer.parseInt(args[0]);
        } else {
            port = 8080;
        }
        new WebsocketChatServer(port).run();
    }
}
```

### 客户端

在程序的resources目录下新建WebsocketChatClient.html 页面来作为客户端

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>WebSocket Chat</title>
</head>
<body>
<script type="text/javascript">
    var socket;
    if (!window.WebSocket) {
        window.WebSocket = window.MozWebSocket;
    }
    if (window.WebSocket) {
        socket = new WebSocket("ws://127.0.0.1:8080/ws");
        socket.onmessage = function(event) {
            var ta = document.getElementById('responseText');
            ta.value = ta.value + '\n' + event.data
        };
        socket.onopen = function(event) {
            var ta = document.getElementById('responseText');
            ta.value = "连接开启!";
        };
        socket.onclose = function(event) {
            var ta = document.getElementById('responseText');
            ta.value = ta.value + "连接被关闭";
        };
    } else {
        alert("你的浏览器不支持 WebSocket！");
    }

    function send(message) {
        if (!window.WebSocket) {
            return;
        }
        if (socket.readyState == WebSocket.OPEN) {
            socket.send(message);
        } else {
            alert("连接没有开启.");
        }
    }
</script>
<form onsubmit="return false;">
    <h3>WebSocket 聊天室：</h3>
    <textarea id="responseText" style="width: 500px; height: 300px;"></textarea>
    <br>
    <input type="text" name="message"  style="width: 300px" value="hehe">
    <input type="button" value="发送消息" onclick="send(this.form.message.value)">
    <input type="button" onclick="javascript:document.getElementById('responseText').value=''" value="清空聊天记录">
</form>
<br>
<br>
</body>
</html>
```

先运行 WebsocketChatServer，再打开多个浏览器页面实现多个客户端访问 http://localhost:8080

那么,你就可以看到一个基于Netty实现的一个简单的网络应用了.