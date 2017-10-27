---
layout: post
title: zookeeper事务一致性
date: 2017-10-17
comments: true
archive: true
tag: zookeeper
---
&emsp;&emsp;zookeeper集群运行起来之后，向其中一个节点发送创建命令之后，其他节点如何能够做出同样的操作，保持整个集群数据的一致性？接下来看下zookeeper事务操作的过程。
### 事务操作
~~~~~java
	protected boolean needCommit(Request request)
	{
		switch (request.type)
		{
		case OpCode.create:
		case OpCode.create2:
		case OpCode.createContainer:
		case OpCode.delete:
		case OpCode.deleteContainer:
		case OpCode.setData:
		case OpCode.reconfig:
		case OpCode.multi:
		case OpCode.setACL:
			return true;
		case OpCode.sync:
			return matchSyncs;
		case OpCode.createSession:
		case OpCode.closeSession:
			return !request.isLocalSession();
		default:
			return false;
		}
	}
~~~~~
&emsp;&emsp;从**CommitProcessor**的**needCommit**方法中列举除了可能的事务请求，对于会话，只有全局的会话的创建和关闭才是事务操作。

### 事务处理
&emsp;&emsp;请求操作的处理都是**RequestProcessor**的**processRequest**方法处理的，而且是多个processor依次处理。Observer、follower和leader的processor处理链如下所示。

- leader
> LeaderRequestProcessor-->PrepRequestProcessor-->ProposalRequestProcessor-->
> CommitProcessor-->ToBeAppliedRequestProcessor-->FinalRequestProcessor
- follower
> FollowerRequestProcessor-->CommitProcessor-->FinalRequestProcessor
> SyncRequestProcessor-->SendAckRequestProcessor
- observer
> ObserverRequestProcessor-->CommitProcessor-->FinalRequestProcessor

#### follower对客服端请求的处理
follower端处理    
1. 判断客户端请求是否是事务操作
2. 如果不是事务操作，则直接由本节点处理
3. 如果是事务操作，首先向leader节点发送**Leader.REQUEST**消息，然后等待
4. 收到**Leader.PROPOSAL**消息后则在本地记录事务操作日志，然后向leader发送**Leader.ACK**消息
5. 收到**Leader.COMMIT**消息后在本地执行请求操作
6. 向客户端返回处理结果

leader端处理     
1. 收到follower的**Leader.REQUEST**之后向所有的follower发送**Leader.PROPOSAL**消息
2. 判断是否由接收到超过半数的**Leader.ACK**消息
3. 收到超过半数的**Leader.ACK**消息之后向所有follower发送**Leader.COMMIT**，向所有observer发送**Leader.INFORM**消息
4. 在本地执行请求操作

#### observer对客服端请求的处理
1. 判断客户端请求是否是事务操作
2. 如果不是事务操作，则直接由本节点处理
3. 如果是事务操作，首先向leader节点发送**Leader.REQUEST**消息，然后等待
4. 收到**Leader.INFORM**消息之后在本地执行请求操作
5. 向客户端返回处理结果

























