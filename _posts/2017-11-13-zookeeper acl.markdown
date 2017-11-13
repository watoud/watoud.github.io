---
layout: post
title: Zookeeper访问权限控制
date: 2017-11-13
comments: true
archive: true
tag: [zookeeper, ACL]
---
&emsp;&emsp; zookeeper访问控制通过**setAcl**、**getAcl**以及**addauth**三个命令进行操作，**setAcl**设置访问控制信息，**getAcl**查看访问信息，**addauth**添加访问权限信息。接下来通过操作示例介绍zookeeper的ACL命令及其实现，以及自定义权限访问控制。
### ACL命令
#### 示例
- 给节点添加权限控制
![set acl](/images/zkacl/createAcl.png)
- 重新登陆，添加权限，查看节点数据
![access data](/images/zkacl/aclAccess.png)
- 直接使用digest添加权限
![access data](/images/zkacl/setDigestAcl.png)
#### setAcl命令
&emsp;&emsp; **setAcl**命令给节点设置权限控制，命令格式为**setAcl {path} {scheme}:{expression}:{perms}**，其中path是节点的路径，scheme代表鉴权的类型，expression是与scheme格式相符合的字符串，perms代表权限。目前zookeeper实现的scheme有5种，**world**、**auth**、**digest**、**ip**和**x509**，内置的有**ip**和**digest**。权限也有5种，**CREATE**、**DELETE**、**WRITE**、*READ*以及**ADMIN**，可以用首字母小写代替。
#### getAcl命令
&emsp;&emsp;**getAcl**用于查看节点的权限控制信息
#### addauth命令
&emsp;&emsp;**addauth**命令添加鉴权信息，访问数据时会在请求中附带上鉴权信息。鉴权信息是跟网络连接关联在一起的，并且重连的时候会把之前添加的鉴权信息重新添加。
### ACL的实现
#### setAcl
&emsp;&emsp;**setAcl**是一个事务操作，事务操作首先需要发起投票，半数通过之后才会进行具体操作，相关代码如下。
~~~~~java
	public Stat setACL(String path, List<ACL> acl, int version)
			throws KeeperException.NoNodeException
	{
		Stat stat = new Stat();
		DataNode n = nodes.get(path);
		if (n == null)
		{
			throw new KeeperException.NoNodeException();
		}
		synchronized (n)
		{
			aclCache.removeUsage(n.acl);
			n.stat.setAversion(version);
			n.acl = aclCache.convertAcls(acl);
			n.copyStat(stat);
			return stat;
		}
	}
	
	public synchronized Long convertAcls(List<ACL> acls)
	{
		if (acls == null)
			return OPEN_UNSAFE_ACL_ID;

		// get the value from the map
		Long ret = aclKeyMap.get(acls);
		if (ret == null)
		{
			ret = incrementIndex();
			longKeyMap.put(ret, acls);
			aclKeyMap.put(acls, ret);
		}

		addUsage(ret);

		return ret;
	}
~~~~~
&emsp;&emsp;zookeeper内部把acl缓存起来并且关联到一个唯一的整数，然后把这个整数作为ACL信息保存到节点的状态信息中。
#### auth scheme
setAcl在命令在具体执行之前，还执行过fixupACL操作，对鉴权信息进行检查和转换，auth类型scheme比较奇特的地方也可以在这里找到。
~~~~~java
	private List<ACL> fixupACL(String path, List<Id> authInfo, List<ACL> acls)
			throws KeeperException.InvalidACLException
	{
		// check for well formed ACLs
		// This resolves https://issues.apache.org/jira/browse/ZOOKEEPER-1877
		List<ACL> uniqacls = removeDuplicates(acls);
		LinkedList<ACL> rv = new LinkedList<ACL>();
		if (uniqacls == null || uniqacls.size() == 0)
		{
			throw new KeeperException.InvalidACLException(path);
		}
		for (ACL a : uniqacls)
		{
			LOG.debug("Processing ACL: {}", a);
			if (a == null)
			{
				throw new KeeperException.InvalidACLException(path);
			}
			Id id = a.getId();
			if (id == null || id.getScheme() == null)
			{
				throw new KeeperException.InvalidACLException(path);
			}
			if (id.getScheme().equals("world") && id.getId().equals("anyone"))
			{
				rv.add(a);
			}
			else if (id.getScheme().equals("auth"))
			{
				// This is the "auth" id, so we have to expand it to the
				// authenticated ids of the requestor
				boolean authIdValid = false;
				for (Id cid : authInfo)
				{
					AuthenticationProvider ap = ProviderRegistry.getProvider(cid.getScheme());
					if (ap == null)
					{
						LOG.error("Missing AuthenticationProvider for " + cid.getScheme());
					}
					else if (ap.isAuthenticated())
					{
						authIdValid = true;
						rv.add(new ACL(a.getPerms(), cid));
					}
				}
				if (!authIdValid)
				{
					throw new KeeperException.InvalidACLException(path);
				}
			}
			else
			{
				AuthenticationProvider ap = ProviderRegistry.getProvider(id.getScheme());
				if (ap == null || !ap.isValid(id.getId()))
				{
					throw new KeeperException.InvalidACLException(path);
				}
				rv.add(a);
			}
		}
		return rv;
	}
~~~~~
从上面的代码中可以看到，当scheme类型为auth时，系统会从当前请求消息携带的鉴权信息中，根据scheme找到对应的**AuthenticationProvider**，如果该对象的**isAuthenticated()**方法返回**true**，则把当前的鉴权信息作为**setAcl**命令需要设置的鉴权信息。
![access data](/images/zkacl/auth.png)
#### getAcl
&emsp;&emsp;**getAcl**是非事务操作，直接从后台数据中查找相应的ACL信息并返回。
~~~~~java
	public List<ACL> getACL(String path, Stat stat) throws KeeperException.NoNodeException
	{
		DataNode n = nodes.get(path);
		if (n == null)
		{
			throw new KeeperException.NoNodeException();
		}
		synchronized (n)
		{
			n.copyStat(stat);
			return new ArrayList<ACL>(aclCache.convertLong(n.acl));
		}
	}
~~~~~
#### addauth
&emsp;&emsp;**addauth**给当前的连接添加鉴权信息，此后通过当前连接的数据访问请求都会携带该鉴权信息，并且连接闪断后重连，会把连接的鉴权信息重新添加。
~~~~~java
		if (h.getType() == OpCode.auth)
		{
			LOG.info("got auth packet " + cnxn.getRemoteSocketAddress());
			AuthPacket authPacket = new AuthPacket();
			ByteBufferInputStream.byteBuffer2Record(incomingBuffer, authPacket);
			String scheme = authPacket.getScheme();
			AuthenticationProvider ap = ProviderRegistry.getProvider(scheme);
			Code authReturn = KeeperException.Code.AUTHFAILED;
			if (ap != null)
			{
				try
				{
					authReturn = ap.handleAuthentication(cnxn, authPacket.getAuth());
				}
				catch (RuntimeException e)
				{
					LOG.warn("Caught runtime exception from AuthenticationProvider: " + scheme
							+ " due to " + e);
					authReturn = KeeperException.Code.AUTHFAILED;
				}
			}
			if (authReturn == KeeperException.Code.OK)
			{
				if (LOG.isDebugEnabled())
				{
					LOG.debug("Authentication succeeded for scheme: " + scheme);
				}
				LOG.info("auth success " + cnxn.getRemoteSocketAddress());
				ReplyHeader rh = new ReplyHeader(h.getXid(), 0, KeeperException.Code.OK.intValue());
				cnxn.sendResponse(rh, null, null);
			}
			else
			{
				if (ap == null)
				{
					LOG.warn("No authentication provider for scheme: " + scheme + " has "
							+ ProviderRegistry.listProviders());
				}
				else
				{
					LOG.warn("Authentication failed for scheme: " + scheme);
				}
				// send a response...
				ReplyHeader rh = new ReplyHeader(h.getXid(), 0,
						KeeperException.Code.AUTHFAILED.intValue());
				cnxn.sendResponse(rh, null, null);
				// ... and close connection
				cnxn.sendBuffer(ServerCnxnFactory.closeConn);
				cnxn.disableRecv();
			}
			return;
		}
~~~~~
### 自定义scheme
&emsp;&emsp;用户可以自己定制鉴权插件，只需要实现**AuthenticationProvider**接口并添加相关参数信息，如**-Dzookeeeper.authProvider.X=com.f.MyAuth**。AuthenticationProvider的获取代码如下。
~~~~~java
		synchronized (ProviderRegistry.class)
		{
			if (initialized)
				return;
			IPAuthenticationProvider ipp = new IPAuthenticationProvider();
			DigestAuthenticationProvider digp = new DigestAuthenticationProvider();
			authenticationProviders.put(ipp.getScheme(), ipp);
			authenticationProviders.put(digp.getScheme(), digp);
			Enumeration<Object> en = System.getProperties().keys();
			while (en.hasMoreElements())
			{
				String k = (String) en.nextElement();
				if (k.startsWith("zookeeper.authProvider."))
				{
					String className = System.getProperty(k);
					try
					{
						Class<?> c = ZooKeeperServer.class.getClassLoader().loadClass(className);
						AuthenticationProvider ap = (AuthenticationProvider) c.newInstance();
						authenticationProviders.put(ap.getScheme(), ap);
					}
					catch (Exception e)
					{
						LOG.warn("Problems loading " + className, e);
					}
				}
			}
			initialized = true;
		}
~~~~~
### 阅读资料
- http://zookeeper.apache.org/doc/r3.5.2-alpha/zookeeperProgrammers.html#sc_ZooKeeperAccessControl









