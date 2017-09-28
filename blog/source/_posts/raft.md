---
title: 由Consul谈到Raft
date: 2017-9-25 16:06:59
categories: Note
tags:
- consul
- Cluster
- Consensus
---
在前一篇文章[consul配置与实战](http://blueskykong.com/2017/09/16/consul/)中，介绍了consul的一些内幕及consul配置相关，并对项目中的一些实际配置进行展示。这篇文章重点介绍consul中所涉及到的一致性算法raft。

## 1. 背景
分布式系统的一致性是相当重要的，即为CAP理论中的C(Consistency)。一致性又可以分为强一致性和最终一致性。这篇文章重点讨论强一致性算法raft。   

Lamport发表Paxos一致性算法从90年提出到现在已经有二十几年了，直到2006年Google的三篇论文初现“云”的端倪，其中的Chubby Lock服务使用Paxos作为Chubby Cell中的一致性算法，Paxos的人气从此一路狂飙。而Paxos流程太过于繁杂实现起来也比较复杂，虽然现在很广泛使用的Zookeeper也是基于Paxos算法来实现，但是Zookeeper使用的ZAB（Zookeeper Atomic Broadcast）协议对Paxos进行了很多的改进与优化，复杂性是制约他发展的一个重要原因。Raft的设计初衷就是易于理解性。

> Raft是斯坦福的Diego Ongaro、John Ousterhout两个人以易懂（Understandability）为目标设计的一致性算法，在2013年发布了论文：《In Search of an Understandable Consensus Algorithm》

从2013年发布到现在不过只有两年，到现在已经有了十多种语言的Raft算法实现框架，较为出名的有etcd。

## 2. Raft详解

强调的是易懂（Understandability），Raft和Paxos一样只要保证n/2+1节点正常就能够提供服务；众所周知但问题较为复杂时可以把问题分解为几个小问题来处理，Raft也使用了分而治之的思想把算法流程分为三个子问题：选举（Leader election）、日志复制（Log replication）、安全性（Safety）三个子问题。
### 2.1 raft基本概念
1. states   
	一个raft集群拥有多个server，通常会有5台，这样可以允许系统中两台server宕机。在任何情况下，所有的server只有如下三种状态之一：

	- Leader，负责Client交互和log复制，同一时刻系统中最多存在1个
	- Follower，被动响应请求RPC，从不主动发起请求RPC
	- Candidate，由Follower向Leader转换的中间状态

	在正常的操作流程中，集群中有且只有一个server，其他所有的server都是follower。follower是被动的，他们只是被动地相应Candidate和leader的请求。。leader处理所有的客户端请求，follower自己不处理而是转发给leader。第三种状态是Candidate，用来选举一个新的leader。

   ![角色转换](http://ovci9bs39.bkt.clouddn.com/Raft%E8%BD%AC%E6%8D%A2%E5%9B%BE%E7%8A%B6%E6%80%81.jpg "state转换图")
   
2. Terms   
 	Raft将时间划分成任意的长度周期。Terms可以理解为逻辑周期，用连续的整数表示。在分布式环境中，时间同步很重要，同时是一个难题。在Raft中使用了一个可以理解为周期（任期）的概念，用Term作为一个周期，每个Term都是一个连续递增的编号，每一轮选举都是一个Term周期，在一个Term中只能产生一个Leader。   
 	每个term伴随着一次election，一个或多个Candidate试图成为leader，如上图的状态转换。如果某个Candidate赢得了这次election，它将升级为剩余server的leader。在某些election的情形中，会产生耗票（Split Votes）的结果 ，即投票结果无效，随后一次新的term开始。raft确保在某个term至多有一个leader。   
 	![terms](http://ovci9bs39.bkt.clouddn.com/terms-raft.jpg "terms")
 如上图所示，时间被划分成多个terms，每个term随着一次election。election完成后，一个leader节点管理整个集群，直至这个term结束。有些election失败了，未能产生一个leader。
 	
### 2.2 Leader election
所有节点都是以follower启动。一个最小的 Raft 集群需要三个参与者，这样才可能投出多数票。初始状态 都是 Follower，然后发起选举这时有三种可能情形发生。如果每方都投给了自己，结果没有任何一方获得多数票。之后每个参与方随机休息一阵（Election Timeout）重新发起投票直到一方获得多数票。这里的关键就是随机 timeout，最先从timeout中恢复发起投票的一方向还在 timeout 中的另外两方请求投票，这时它们就只能投给对方了，很快达成一致。   
Raft的选举由定时器来触发，每个节点的选举定时器时间都是不一样的，开始时状态都为Follower某个节点定时器触发选举后Term递增，状态由Follower转为Candidate，向其他节点发起RequestVote RPC请求，这时候有三种可能的情况发生：   
　　1.该RequestVote请求接收到n/2+1（过半数）个节点的投票，从Candidate转为Leader，向其他节点发送heartBeat以保持Leader的正常运转。   
　　2.在此期间如果收到其他节点发送过来的AppendEntries RPC请求，如该节点的Term大则当前节点转为Follower，否则保持Candidate拒绝该请求。   
　　3.Election timeout发生则Term递增，重新发起选举。   
　　在一个Term期间每个节点只能投票一次，所以当有多个Candidate存在时就会出现每个Candidate发起的选举都存在接收到的投票数都不过半的问题，这时每个Candidate都将Term递增、重启定时器并重新发起选举，由于每个节点中定时器的时间都是随机的，所以就不会多次存在有多个Candidate同时发起投票的问题。

引用一张网上的图片，比较形象，如下图。
![vote](http://ovci9bs39.bkt.clouddn.com/voterequest.png "vote")

### 2.3 Log replication
 
日志复制主要是用于保证节点的一致性，这阶段所做的操作也是为了保证一致性与高可用性；当Leader选举出来后便开始负责客户端的请求，所有事务（更新操作）请求都必须先经过Leader处理，这些事务请求或说成命令也就是这里说的日志，我们都知道要保证节点的一致性就要保证每个节点都按顺序执行相同的操作序列，日志复制（Log Replication）就是为了保证执行相同的操作序列所做的工作；在Raft中当接收到客户端的日志（事务请求）后先把该日志追加到本地的Log中，然后通过heartbeat把该Entry同步给其他Follower，Follower接收到日志后记录日志然后向Leader发送ACK，当Leader收到大多数（n/2+1）Follower的ACK信息后将该日志设置为已提交并追加到本地磁盘中，通知客户端并在下个heartbeat中Leader将通知所有的Follower将该日志存储在自己的本地磁盘中。

![log](http://ovci9bs39.bkt.clouddn.com/log-replication.png "log")

上图中，当leader选出来之后，follower的logs场景很可能出现在上图中。follower有可能丢失entries、有未提交的entries、有额外的entries等等场景。Raft中，leader通过强制followers复制自己的logs来处理不一致。这意味着，在follower中logs冲突的entries将会被leader logs中的覆写。

### 2.4 Safety
安全性是用于保证每个节点都执行相同序列的安全机制，如当某个Follower在当前Leader commit Log时变得不可用了，稍后可能该Follower又会倍选举为Leader，这时新Leader可能会用新的Log覆盖先前已committed的Log，这就是导致节点执行不同序列；Safety就是用于保证选举出来的Leader一定包含先前 commited Log的机制；

- Election Safety   
每个Term只能选举出一个Leader，假设某个Term同时选举产生两个LeaderA和LeaderB，根据选举过程定义，A和B必须同时获得超过半数节点的投票，至少存在节点N同时给予A和B投票，因此矛盾。
- Leader Completeness      
这里所说的完整性是指Leader日志的完整性，当Log在Term1被Commit后，那么以后Term2、Term3…等的Leader必须包含该Log；Raft在选举阶段就使用Term的判断用于保证完整性：当请求投票的该Candidate的Term较大或Term相同Index更大则投票，否则拒绝该请求；
- Leader Append-Only   
Leader从不“重写”或者“删除”本地Log，仅仅“追加”本地Log。Raft算法中Leader权威至高无上，当Follower和Leader产生分歧的时候，永远是Leader去覆盖修正Follower。
- Log Matching   
如果两个节点上的日志项拥有相同的Index和Term，那么这两个节点[0, Index]范围内的Log完全一致。
- State Machine Safety   
一旦某个server将某个日志项应用于本地状态机，以后所有server对于该偏移都将应用相同日志项。

## 3. 总结
本文主要讲解了Raft算法的基本概念，以及算法中涉及到的leader选举，日志同步，安全性。Raft是以易理解性作为其设计的一个目标，对于一个学习的新手来说，确实比Paxos易于理解很多。虽然 Raft 的论文比 Paxos 简单版论文容易读，论文依然有很多地方需要深刻体会与理解，笔者也还是搞了好几天。


---
### 参考
1. [Raft Animate Demo](http://thesecretlivesofdata.com/raft/)   
2. [Raft Paper](https://ramcloud.stanford.edu/raft.pdf)
3. [Raft Website](https://raft.github.io/#implementations)
4. [Raft 为什么是更易理解的分布式一致性算法](http://www.cnblogs.com/mindwind/p/5231986.html)
