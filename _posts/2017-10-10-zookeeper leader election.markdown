---
layout: post
title: zookeeper集群选举
date: 2017-10-10
comments: true
archive: true
tag: zookeeper
---
### zookeeper集群选举基本概念
#### 节点的角色
&emsp;&emsp;在zookeeper的配置文件中可以通过**peerType**设置节点的类型，值只能是**observer**或者**participant**，系统默认值为**participant**，相关的配置解析代码如下所示。
![peer type parse call hierarchy](/images/zkLeaderElection/peerTypeParse.png)
~~~~java
	protected LearnerType peerType = LearnerType.PARTICIPANT;
	
	else if (key.equals("peerType"))
	{
		if (value.toLowerCase().equals("observer"))
		{
			peerType = LearnerType.OBSERVER;
		}
		else if (value.toLowerCase().equals("participant"))
		{
			peerType = LearnerType.PARTICIPANT;
		}
		else
		{
			throw new ConfigException("Unrecognised peertype: " + value);
		}
	}
~~~~
要使Observer配置生效，还需要在集群信息配置中添加**:observer**，例如：    
**<center>server.1:localhost:2181:3181:observer</center>**

&emsp;&emsp;那么observer节点有什么不同，官方文档描述如下。
> Observers are non-voting members of an ensemble which only hear the results of votes, not the agreement protocol that leads up to them. Other than this simple distinction, Observers function exactly the same as Followers - clients may connect to them and send read and write requests to them. Observers forward these requests to the Leader like Followers do, but they then simply wait to hear the result of the vote.

Observer节点不参与集群投票，包括集群选举以及事务投票，除此之外，Observer跟follower一样。当集群非常大的时候，增加Observer节点的好处在于，可以减少投票操作，提升集群写操作性能。另外，Observer由于不参与投票，所以即使该节点挂掉了或者与集群断开了，也不会影响集群的稳定性。

&emsp;&emsp;**于是我们知道，zookeeper集群中的节点可以分为observer和participant，只有participant才参与集群选举，参与选举的节点中，除了主节点leader之外，其他的都是follower。所以在一个运行的zookeeper集群中，zookeeper节点可能是leader、follower或者observer。**

#### 节点的状态
&emsp;&emsp;zookeeper集群中的节点可能存在四中状态。

- LOOKING：尚未加入集群，zookeeper刚启动时就处于该状态
- LEADING：集群选举结束后，主节点处于该状态
- FOLLOWING：集群选举结束后，参与集群选举的非主节点处于该状态
- OBSERVING： 集群选举结束后，没有参与选举的节点处于该状态

#### sid、epoch、zxid
&emsp;&emsp;在集群选举的投票信息中，有几个关键信息，leaderId、peerEpoch以及zxid。其中leaderId是被推举为leader的节点的id，集群中每个节点都有一个唯一的id，可以在配置文件中的data目录下的myid文件中看到。peerEpoch为被推举节点所处的纪元，每次重新选举都会新生成一个递增的epoch。zxid为被推举节点的事务ID，zookeeper每更改一次都会对应唯一的一个zxid，并且zxid是递增的。zxid包含该两部分，高32位为epoch，低32位用于递增计数。zxid的生成以及提取epoch相关信息见类**ZxidUtils**。

### 集群选举流程
#### 创建并启动选举算法
&emsp;&emsp;在类**QuorumPeer**的**startLeaderElection**方法中创建并启动了选举方法，默认使用FastLeaderElection，在这个方法里面启动两个线程**WorkerSender**和**WorkerReceiver**，一个用于处理发送消息，一个用于处理接收的消息，这里着重分析一下WorkerReceiver的处理。     
&emsp;&emsp;WorkerReceiver线程不断地从通信管理接收队列中获取poll接收到的消息并解析，首先判断发送该消息的节点是否参与选举投票，如果不是则直接向该节点返回本节点当前的投票信息，比如不参与选举投票observer节点。如果自身的状态为**LOOKING**，那么把收到的消息加入到本地接收队列中，如果对方的状态为**LOOKING**并且他的选举epoch比当前节点小，那么给对方发送当前节点的投票信息。如果当前节点不是**LOOKING**状态，但是对方处于**LOOKING**状态，则向对方发送主节点信息。详细流程如下所示。

![receiver process](/images/zkLeaderElection/rvcProcess.png)

#### 选举主节点
&emsp;&emsp;集群选举过程中发送的投票信息主要包含节点id、zxid以及epoch，节点启动加入选举时，会更新投票信息，把投票信息设置为自己的sid、zxid以及epoch。当一个节点收到其他节点发来的投票消息时，会与自己当前的投票信息进行比价，首先比较epoch，然后再是zxid，最后比较sid，然后把当前的投票信息设置为最优的投票，如果有更新则广播新的投票信息。当本节点发现推举的节点占多数时，则认为该节点为主节点，选主阶段结束。    
~~~~~java
return ((newEpoch > curEpoch) || //
				((newEpoch == curEpoch) && ((newZxid > curZxid) //
						|| ((newZxid == curZxid) && (newId > curId)))));
~~~~~
&emsp;&emsp;observer在选举的过程中虽然也向其他参与选举的节点发送了投票信息，但是这些投票信息都被无视了，而是直接返回了那些节点自身认为的主节点信息。而一个节点只有在选出主节点之后才会去更新主节点信息，所以observer节点必然是在其他节点已经选出了主节点的情况下才获得主节点信息。     
&emsp;&emsp;集群选举流程如下图所示。
![receiver process](/images/zkLeaderElection/zkelection.jpg)

#### 数据同步
&emsp;&emsp;选出主节点之后，接着就是节点之间的数据同步。
- 1. **followr节点注册**  leader节点为每个连接关联一个**LearnerHandler**，并启动消息处理线程。follower节点连上leader之后，立即发送一个**FOLLOWERINFO**信息过去，包含了自身的节点ID以及上一轮接受的epoch。leader等待半数以上的follower都发来**FOLLOWERINFO**消息之后，计算出当前轮次的epoch，相关代码如下。     

接着，leader给follower发送**LEADERINFO**消息，该消息包含了**newLeaderZxid**，该值由两部分组成，高32位为上一步得到的epoch，低32位为0。follower接收到该消息后会更新自身当前的epoch，然后向leader发送**ACKEPOCH**，该消息中包含了最新的zxid。leader等待半数以上的follower节点都发送了**ACKEPOCH**消息之后，接着就进入了数据同步阶段。
- 2. **数据同步** leader根据follower节点发送的**ACKEPOCH**消息中的zxid来确定如何跟对应的follower来同步数据。如果follower跟leader已经是同步的，即follower的lastZxid与leader的lastZxid相同，这个时候，leader只需要发送一个空的**DIFF**消息。如果follower的lastZxid大于leader的lastZxid，则说明follower的数据超前了，leader发送一个**TRUNC**消息。如果如果follower的lastZxid大于leader的lastZxid，则先发一个**DIFF**消息，接着将leader中多的事务消息一次发送给follower，其他一些特殊情况则发送**SNAP**消息。最后发送一个**NEWLEADER**消息，然后leader开始等待follower的**ACK**消息。follower根据消息不同，分别处理**TRUNC**、**DIFF**和**SNAP**消息，之后向leader返回**ACK**消息。最后leader等待半数以上的follower返回**ACK**之后，接着向follower发送**UPTODATE**消息，这个时候数据同步结束。

### 阅读资料
> - http://zookeeper.apache.org/doc/r3.5.3-beta/zookeeperObservers.html























