---
layout: post
title: Netty 客户端与服务端的通信过程
date: 2017-11-08
comments: true
archive: true
tag: netty
---
&emsp;&emsp;前面一篇博客简要分析了netty服务端的启动过程，接下来简要描述一下客户端与服务端端额通信过程
### EchoClient客户端代码
~~~~~java
	public static void main(String[] args) throws Exception
	{
		EventLoopGroup group = new NioEventLoopGroup();
		try
		{
			Bootstrap b = new Bootstrap();
			b.group(group).channel(NioSocketChannel.class)//
					.option(ChannelOption.TCP_NODELAY, true)
					.handler(new ChannelInitializer<SocketChannel>()
					{
						@Override
						public void initChannel(SocketChannel ch) throws Exception
						{
							ChannelPipeline p = ch.pipeline();
							p.addLast(new EchoClientHandler());
						}
					});
			ChannelFuture f = b.connect(HOST, PORT).sync();
			f.channel().closeFuture().sync();
		}
		finally
		{
			group.shutdownGracefully();
		}
	}
~~~~~
&emsp;&emsp;相比于服务端，客户端只设置了一个执行线程组，设置好选项、channel和handler之后就执行connect方法。

### 客服端发起连接过程
#### channel的初始化和注册
&emsp;&emsp;客户端在初始化时设置了channel的选项和属性，并且把config中的handler添加到pipeline上；然后在执行器关联的selector上注册一个空的事件，并把返回的SelectionKey保存在channel中，用于后面重新设置感兴趣的事件集合；最后触发ChannelRegistered事件。
#### 客户端建立连接
&emsp;&emsp;从ChannelPipeline的tail contexgt开始查找Outbound类型的handler并执行connect方法，ChannelPipeline上的handler链：HeadContext<-->EchoClientHandler<-->TailContext，其中EchoClientHandler跟TailContext都是Inbound类型的，所以就只在HeadContext上执行connect方法。HeadContext在connect方法中回去连接服务端，连接成功后会触发ChannelActive事件，然后在处理ChannelActive事件的时候，HeadContext在channelActive方法中会把SelectionKey.OP_READ加入到监听事件集合中，EchoClientHandler向服务端发送消息。
### 服务端接收连接过程
&emsp;&emsp;上一篇博客中讲到服务端启动后会在selector上监听SelectionKey.OP_ACCEPT事件，所以一旦有客户端建立连接请求时，首先会执行accept方法创建连接，然后触发ChannelRead方法，从head开始查找Inbound handler并执行channelRead方法。在ServerBootstrapAcceptor的channelRead方法中会对新创建的Channel进行注册，然后触发ChannelRegistered和ChannelActive方法，HeadContext在channelActive方法中会把SelectionKey.OP_READ加入到监听事件集合中。
### 服务端与客户端的交互
&emsp;&emsp;Echo服务器在接收到客户端的消息之后立即把消息回写给客户端，客户端接收到消息关闭连接。
### 连接的关闭
#### 客户端的关闭
&emsp;&emsp;首先调用ChannelHandlerContext.close方法，在这个方法中首先从当前Handler往前查找Outbound类型的handler，也就是HeadContext，然后调用它的close方法。先关闭SocketChannel，解注册，然后触发ChannelInactive和ChannelUnregistered事件。
#### 服务端的关闭
&emsp;&emsp;客户端关闭时，服务端会检测到一个SelectionKey.OP_READ事件，读取时返回值为负数，发现对方已经关闭连接，触发ChannelReadComplete事件之后执行关闭操作。接下来的关闭操作跟客户端后面的操作一样。

























