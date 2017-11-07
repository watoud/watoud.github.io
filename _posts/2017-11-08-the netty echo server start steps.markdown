---
layout: post
title: 从EchoServer看Netty服务端启动过程
date: 2017-11-08
comments: true
archive: true
tag: netty
---
### EchoServer服务端代码
~~~~~java
	public static void main(String[] args) throws Exception
	{
		EventLoopGroup bossGroup = new NioEventLoopGroup(1);
		EventLoopGroup workerGroup = new NioEventLoopGroup();
		try
		{
			ServerBootstrap b = new ServerBootstrap();
			b.group(bossGroup, workerGroup).channel(NioServerSocketChannel.class)
					.option(ChannelOption.SO_BACKLOG, 100)
					.handler(new LoggingHandler(LogLevel.INFO))
					.childHandler(new ChannelInitializer<SocketChannel>()
					{
						public void initChannel(SocketChannel ch) throws Exception
						{
							ChannelPipeline p = ch.pipeline();
							p.addLast(new LoggingHandler(LogLevel.INFO), new EchoServerHandler());
						}
					});

			ChannelFuture f = b.bind(PORT).sync();
			f.channel().closeFuture().sync();
		}
		finally
		{
			bossGroup.shutdownGracefully();
			workerGroup.shutdownGracefully();
		}
	}
~~~~~
### 设置线程执行器
&emsp;&emsp;Netty没有使用jdk自带的线程池而是自己开发，**EchoServer**的代码中创建了两个**NioEventLoopGroup**，其中**bossGroup**用于处理客户端的连接请求，**workerGroup**用于处理读写等网络IO操作。**NioEventLoopGroup**里面有个数组字段**children**，这个**children**中的每一个元素就是具体的任务执行器**NioEventLoop**。**NioEventLoopGroup**有个方法**next()**返回**children**中的一个任务执行器，具体返回哪一个，由**EventExecutorChooser**决定。如果**children**的长度为2的幂次方，则chooser为**PowerOfTwoEventExecutorChooser**，否则为**GenericEventExecutorChooser**。**GenericEventExecutorChooser**获取下一个执行的方法为```executors[Math.abs(idx.getAndIncrement() % executors.length)]```，**PowerOfTwoEventExecutorChooser**获取下一个执行器的方法为```executors[idx.getAndIncrement() & executors.length - 1]```。创建**NioEventLoopGroup**时默认的**children**长度为**CPU**个数的两倍。
### Channel初始化及注册
1. 初始化一方面设置channel的选项和属性，另一方面设置ChannelPipeline的handler。ChannelPipeline创建时会创建HeadContext和TailContext两个handler，其中HeadContext既是Outbound类型又是Inbound类型，TailContext是Inbound类型。初始化后，ChannelPipeline上的handler链，**HeadContext<-->ChannelInitializer<-->TailContext**，其中ChannelInitializer在initChannel方法中添加了LoggingHandler和ServerBootstrapAcceptor。
2. 从group中选出一个执行线程，然后调用register方法。在注册过程中，首先把Channel上的eventLoop设置为当前选出的执行器，然后注册一个空的事件到当前的执行器关联的Selector，并把selectionKey保存到channel中。
3. 触发ChannelRegistered方法，从head开始执行查找Inbound类型的handler并依次执行channelRegistered方法，其中执行ChannelInitializer的channelRegistered方法时会调用它的initChannel方法，把handler添加到ChannelPipeline中来，并且把自己从ChannelPipeline中remote掉，最后再次触发整个pipeline上的channelRegistered方法。执行ChannelInitializer的channelRegistered方法之后，handler链如下，**HeadContext<-->LoggingHandler<-->ServerBootstrapAcceptor<-->TailContext**。其中LoggingHandler既是Inbound类型又是Outbound类型，ServerBootstrapAcceptor是Inbound类型。LoggingHandler在channelRegistered方法中打印日志并向下传递该方法。ServerBootstrapAcceptor的channelRegistered方法仅仅向后传递channelRegistered方法，而TailContext在
channelRegistered方法中什么都没有做。

### 服务端绑定
&emsp;&emsp;从pipeline的tail开始查找Outbound的handler，于是首先执行LoggingHandler的bind方法，打印日志并向前传递bind方法，于是执行HeadContext的bind方法。HeadContext首先调用JDK函数执行绑定操作，然后从head开始触发ChannelActive方法。从head开始依次查找Inbound handler并执行ChannelActive方法，其中head执行完ChannelActive方法之后会执行readIfIsAutoRead方法，在这个方法里面会触发
channel上的read方法，从tail开始查找Outbound类型的handler并执行read方法。需要注意的是，HeadContext在read方法中把SelectionKey.OP_ACCEPT事件添加到selector的事件集合中。    

### 检查发生的IO事件
&emsp;&emsp;把SelectionKey.OP_ACCEPT事件添加到selector的事件集合中后，可执行线程会循环检查是否有客户端发起连接，到这里服务端的启动就完成了。










