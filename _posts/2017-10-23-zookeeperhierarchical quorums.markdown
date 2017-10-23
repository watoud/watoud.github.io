---
layout: post
title: zookeeper hierarchical quorums
date: 2017-10-23
comments: true
archive: true
tag: zookeeper
---
&emsp;&emsp;zookeeper集群在判断一个投票是否有效的时候有两种方法，一种是**majority quorums**，另一种是**hierarchical quorums**。**majority quorums**是比较常见的一种策略，只要大部分（超过一半）节点同意，投票就有效。在**hierarchical quorums**方式中，zookeeper服务器分在不同的组，并且每个服务器被赋予不同的权重，只要大部分（超过一半）的组有超过一半权重的节点都赞同，那么投票就有效。使用**hierarchical quorums**的场景之一就是对zookeeper服务器进行分组隔离，比如把对外提供服务的zookeeper分为一组，并且全部配置为observer，这一部分节点就只提供对外服务，不参与集群选举，就算这部分节点出了问题也不影响集群的稳定性。
### hierarchical quorums 配置示例
> server.1=192.168.41.128:9210:9211    
> server.2=192.168.41.129:9210:9211    
> server.3=192.168.41.130:9210:9211    
> server.1=192.168.41.131:9210:9211    
> server.2=192.168.41.132:9210:9211    
> server.3=192.168.41.133:9210:9211    
> server.1=192.168.41.134:9210:9211    
> server.2=192.168.41.135:9210:9211    
> server.3=192.168.41.136:9210:9211    
>     
> group.1=1:2:3    
> group.2=4:5:6    
> group.3=7:8:9    
>     
> weight.1=1    
> weight.2=1    
> weight.3=1    
> weight.4=1    
> weight.5=1    
> weight.6=1    
> weight.7=1    
> weight.8=1    
> weight.9=1  
  
&emsp;&emsp;**hierarchical quorums**相比于常用的**majority quorums**，在配置上多了分组以及权重配置。如上所示，把1、2、3分为一组，4、5、6分为一组，7、8、9分为一组，并且每个服务器的权重都为1。

### hierarchical quorums 配置解析
~~~~~java
	private void parse(Properties quorumProp) throws ConfigException
	{
		for (Entry<Object, Object> entry : quorumProp.entrySet())
		{
			String key = entry.getKey().toString();
			String value = entry.getValue().toString();

			if (key.startsWith("server."))
			{
				int dot = key.indexOf('.');
				long sid = Long.parseLong(key.substring(dot + 1));
				QuorumServer qs = new QuorumServer(sid, value);
				allMembers.put(Long.valueOf(sid), qs);
				if (qs.type == LearnerType.PARTICIPANT)
					participatingMembers.put(Long.valueOf(sid), qs);
				else
				{
					observingMembers.put(Long.valueOf(sid), qs);
				}
			}
			else if (key.startsWith("group"))
			{
				int dot = key.indexOf('.');
				long gid = Long.parseLong(key.substring(dot + 1));

				numGroups++;

				String parts[] = value.split(":");
				for (String s : parts)
				{
					long sid = Long.parseLong(s);
					if (serverGroup.containsKey(sid))
						throw new ConfigException("Server " + sid + "is in multiple groups");
					else
						serverGroup.put(sid, gid);
				}

			}
			else if (key.startsWith("weight"))
			{
				int dot = key.indexOf('.');
				long sid = Long.parseLong(key.substring(dot + 1));
				serverWeight.put(sid, Long.parseLong(value));
			}
			else if (key.equals("version"))
			{
				version = Long.parseLong(value, 16);
			}
		}

		for (QuorumServer qs : allMembers.values())
		{
			Long id = qs.id;
			if (qs.type == LearnerType.PARTICIPANT)
			{
				if (!serverGroup.containsKey(id))
					throw new ConfigException("Server " + id + "is not in a group");
				if (!serverWeight.containsKey(id))
					serverWeight.put(id, (long) 1);
			}
		}

		computeGroupWeight();
	}
	
	private void computeGroupWeight()
	{
		for (long sid : serverGroup.keySet())
		{
			Long gid = serverGroup.get(sid);
			if (!groupWeight.containsKey(gid))
				groupWeight.put(gid, serverWeight.get(sid));
			else
			{
				long totalWeight = serverWeight.get(sid) + groupWeight.get(gid);
				groupWeight.put(gid, totalWeight);
			}
		}

		/*
		 * Do not consider groups with weight zero
		 */
		for (long weight : groupWeight.values())
		{
			LOG.debug("Group weight: " + weight);
			if (weight == ((long) 0))
			{
				numGroups--;
				LOG.debug("One zero-weight group: " + 1 + ", " + numGroups);
			}
		}
	}
~~~~~

- server解析，以**server.**开头的配置信息。server信息保存在**allMembers**、**participatingMembers
**和**observingMembers**这3个HashMap中，key为sid，value为QuorumServer。
- group解析，以**group.**开头的配置信息。每解析一组group，numGroups计数器加1。解析完的分组信息保存到HashMap **serverGroup**中，key为sid，value为gid，即每一组的id。每一个zookeeper服务器只允许在一个分组里面，不能出现在多个分组中。
- weight解析，以**weight**开发的配置信息。解析的权重信息保存在HashMap **serverWeight**中，key为sid，value为权重。没有配置权重时默认为1，**OBSERVER**不参与权重计算。另外，如果配置的**PARTICIPANT**角色的服务器不在**serverGroup**中将会报错。
- 计算组权重。 每一组中所有服务器权重的总和为这一组的组权重，保存到**groupWeight**中，如果某一个组的权重为0，则代表这个组不参与投票了，对应的组计数器numGroups减1。

### hierarchical quorums 投票计算
~~~~~java
	public boolean containsQuorum(Set<Long> set)
	{
		HashMap<Long, Long> expansion = new HashMap<Long, Long>();

		/*
		 * Adds up weights per group
		 */
		if (set.size() == 0)
			return false;
		else
			LOG.debug("Set size: " + set.size());

		for (long sid : set)
		{
			Long gid = serverGroup.get(sid);
			if (gid == null)
				continue;
			if (!expansion.containsKey(gid))
				expansion.put(gid, serverWeight.get(sid));
			else
			{
				long totalWeight = serverWeight.get(sid) + expansion.get(gid);
				expansion.put(gid, totalWeight);
			}
		}

		/*
		 * Check if all groups have majority
		 */
		int majGroupCounter = 0;
		for (long gid : expansion.keySet())
		{
			LOG.debug(
					"Group info: " + expansion.get(gid) + ", " + gid + ", " + groupWeight.get(gid));
			if (expansion.get(gid) > (groupWeight.get(gid) / 2))
				majGroupCounter++;
		}

		LOG.debug("Majority group counter: " + majGroupCounter + ", " + numGroups);
		if ((majGroupCounter > (numGroups / 2)))
		{
			LOG.debug("Positive set size: " + set.size());
			return true;
		}
		else
		{
			LOG.debug("Negative set size: " + set.size());
			return false;
		}
	}
~~~~~

1. 计算收到的投票中每一组的投票权重，比如收到的投票中只有1、4、6、7、8、9，那么组1的投票权重为1，组2的投票权重为2，组3的投票权重为3。
2. 计算有大部分投票权重的组的个数。组1的投票权重为1，少于大部分（3/2）；组2的投票权重为2，占大部分；组3的投票权重为3，组内的服务器都投票了。有大部分投票权重的组的个数为2。
3. 有大部分投票权重的组是否占所有分组的半数以上。如上所示，组2、组3组内投票权重都占了大多数，超过半数分组（3个分组），所以投票有效。

### 阅读资料
- http://zookeeper.apache.org/doc/r3.4.6/zookeeperHierarchicalQuorums.html























