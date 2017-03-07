---
title: zookeeper-summary
date: 2016-09-15 07:53:03
tags:
---
## 认识
zookeeper去年看过文档,使用zkclient api写了watch的例子,每当了解其讲的功能时,比如watch,绝得并不是什么牛逼的技术,随便写几个  
tcp server实现都可以,配置管理,分布式锁也是,都可以实现,直到看了其ZAB原子广播协议,以即故障恢复模式,对其有了新的认识  

1. zookeeper类似NoSQL,而且Leader和Follower类似Master/slaver,数据结构为ZNode构成的树形结构,而不像关系数据库表和redis map等  
2. zookeeper也有事务日志,当leader收到一个消息,将会写日志并广播给Follower,如果超过半数ack,则广播commit.这里类似2pc,只是弱化  
   了,为了效率不用等到全部follower ack. 最主要还是每个follower是执行一样的工作,只要一个成功其它就应该成功,所以超过半数ack,  
   就commit是可以的.而在关系数据库分布式事务,每个事务参与者各自的事务不一样,不一定都能执行成功,所以需要等到全部ack才能commit  
3. 最核心还是故障恢复模式,2pc协调者挂了分布式事务就没法工作.而zookeeper leader挂了,集群会自动进行新的leader选举,ZAB会保证  
    在任意一台follower上的执行的事务都会在所有follower上执行,也会忽略只有在leader上存在的事务  

引用nileader那本书上一句话,zookeeper就是一个高可用的主备数据系统  

## 节点
zookeeper数据结构是znode构成的树形结构,分为持久化节点和临时节点,有点和MQ类似

## 超过半数(quorum=n/2+1)
zookeeper很多地方都有超过半数这个限制,因为一个集群要维护一致性,而分布式系统又有分区性,当集群产生分区就需要一个规则来判断以哪个  
分区数据为准,不然每个分区都能正常工作,然后数据就不一致.所以使用超过半数这个限制来决定当分区产生时采用那个分区作为工作分区,只有  
超过半数的分区才能正确工作.其它低于半数的分区就不能工作了.所以zookeeper集群如果有超过半数机器不能工作则该集群不能对外提供服务  

还可以给每台server一个权重,quorom就是超过权重之和一半  

## 协议描述术语
1. packet,a sequence of bytes sent through a FIFO channel, zookeeper使用tcp实现消息通讯
2. proposal,quorom服务器之间协商的提案,提案一般都包含messages,但是NEW_LEADER proposal不包含Message
3. Message,a sequence of bytes to be atomically broadcast to all ZooKeeper servers

## 其它术语
* quorom,集群中超过半数节点的集合

### ZAB节点三种状态
* following,当前节点是跟随者,服从leader命令
* leading, 当前节点是leader, 发布事务负责协调
* election/looking, 节点处于选举状态, 启动时和故障恢复开始时所有节点都处于这个状态

### 节点的持久状态
* history：当前节点接收到事务提议的 log
* acceptedEpoch：follower 已经接受的 leader 更改年号的 NEWEPOCH 提议
* currentEpoch：当前所处的年代
* lastZxid：history 中最近接收到的提议的 zxid （最大的）

## zxid
zookeeper需要保证消息的全局顺序,使用zookeeper transaction id(zxid),当proposal被提出的时候,proposal将被标记一个zxid,也就是  
一个全局顺序,proposals被发送到所有servers,当quorom server返回proposal ack时,proposal被commit.当收到proposal ack表示server  
已经记录该proposal到磁盘中.quorom server = (n/2 + 1), n is the number of servers make up the zookeeper service  

zxid由两部分构成,epoch+counter.长度为64为, we use the high order 32-bits for epoch and the low order 32-bits for counter  
epoch number代表leadership发生改变,每当有新的leader产生,它将有自己新的epoch number,没当产生一个新的proposal时,通过自增zxid  
保证每个proposal有唯一的zxid  

## zookeeper messaging system提供已下保证
1. 可靠的交付(reliable delivery),如果消息m交付到一台服务器,那么最终会被交付给所有服务器
2. 全局顺序(total order),如果消息a在消息b之前交付给一台服务器,那么消息a将在消息b之前交付给所有服务器
3. causal order,因果顺序,如果消息b在消息a之后又sender发送,那么b顺序一定在a之后

a. 已经在leader服务器上提交的事务最终都会被所有服务器提交  
b. 只在leader服务器上提出的事务会被丢弃(说明leader收到事务请求时没等ack就本地写了,所以故障恢复时可能需要丢弃)  

## ZooKeeper messaging consists of two phases
1. Leader activation  
In this phase a leader establishes the correct state of the system and gets ready to start making proposals.
 
2. Active messaging  
In this phase a leader accepts messages to propose and coordinates message delivery.

### leader选举
    当启动集群和集群发生故障需要恢复时,会进行leader election, 这个阶段很重要,当发生故障时,恢复需要保证数据一致性,这也是zookeeper  
高可用的核心.  
    zookeeper选举集群中拥有最大zxid proposal的server做为leader,说明新的leader拥有所有已提交的议案,新的leader可以省去检查proposal  
的提交和丢弃工作
#### Phase 0: Leader election（选举阶段）
节点在一开始都处于选举阶段，只要有一个节点得到超半数节点的票数，它就可以当选准 leader。只有到达 Phase 3 准 leader 才会成为真正的 leader。  
这一阶段的目的是就是为了选出一个准 leader，然后进入下一个阶段.  
协议并没有规定详细的选举算法，后面我们会提到实现中使用的 Fast Leader Election。  


#### Phase 1: Discovery（发现阶段）
在这个阶段，followers 跟准 leader 进行通信，同步 followers 最近接收的事务提议。这个一阶段的主要目的是发现当前大多数节点接收的最新提议，  
并且准 leader 生成新的 epoch，让 followers 接受，更新它们的 acceptedEpoch

发现阶段就是发现那个follower的事务是最新的,保证数据一致性.  

有个疑惑,如果拥有最新proposal的节点不在quorom中,虽然它的epoch一定不可能大于quorom中最大的epoch,但是可能该proposal被丢弃,但是这也  
和zookeeper提供的服务保证一致,因为zookeeper只保证在leader提交的事务最终都会被执行,在leader上被提交的proposal说明已经被quorom所  
提交,所以在发现阶段quorom一定包含被提交的proposal,而不在quorom中,拥有最新proposal的事务被丢弃也是合理的

*所以zookeeper故障恢复无法保障没有提交的事务最终能够提交,有概率成分,从事务的角度来看没有被commit的proposal被回滚也是合理的*

#### Phase 2: Synchronization（同步阶段）
根据phase 2获取到的最新历史proposal,同步到其它副本,当quorom都同步完成,准leader成为真正的leader,follower只会接受比自己lastZxid大的proposal  

新的leader zxid一定是最大的,就算在quorom之外的follower有最新的proposal,其epoch会比新leader的epoch小,所以zxid也小  

### 消息广播
选举出leader之后,zookeeper集群才能对外部提供服务,并且leader进行消息广播,有新节点加入时要对新节点进行同步

## 什么时候会发生选举?
1. 启动集群时
2. 当leader挂掉,或者集群产生分区,leader所在分区机器数量已经没有超过半数,那么超过半数的分区会产生新的leader  

所以当选举发生时leader一定不在了,但是有可能leader会再次加入到新的分区中  


## 协议实现
Fast Leader Election。FLE会选举拥有最大zxid的节点作为leader,省去了发现最新proposal的步骤,如果zxid一样则选举zoo.cfg中serverId最大的  
节点在选举开始时都给自己投票,接受其它节点选票时会根据epoch,zxid,serverId来更新自己的选票和发送选票给其它节点,比如收到投票发现其比自己  
还不符合条件就不投票给其它节点了,因为serverId不同,所以最终会选出一个最为leader  

## zookeeper与mysql主备复制比较
zookeeper能够保证在leader上commit的事务在故障恢复时最终被提交  
msyql replicaiton是异步的,当master提交事务时会写event到binlong中,但是不知道何时slave读取到event并处理他们    
mysql fully synchronous replicaiton,master commit事务需要等待所有slave commit之后才能真正提交,可能延时高,事务执行时间很长  
mysql semi-replicaiton居于上面两种同步机制之间,只要一个slave commit,master就提交事务,在数据完整性和效率之间折中考虑  
使用半同步,只要一个commit成功返回,表示data至少存在于两台机器上面,如果master commit但是crash发生了,然而通知可能还没到达slave  

mysql高可用也可以像zookeeper那样,同步超过半数,如果master挂了就采用最大binlog positon的slave作为leader,如果多个slave binlog postion  
一样也可以使用serverid来区分.zookeeper会丢弃只在leader上提交的proposal,mysql只在master上执行的事务,但是没有提交也应当丢弃,反正客户端  
也没有获得成功的响应,丢弃也是可以接受的.  

在mysql参与的分布式事务中,如果在2pc第二阶段,部分mysql参与者提交了事务,当协调者挂了后重新恢复,其它未提交事务的mysql参与者也应当提交事务,但是在故障  
恢复阶段,事务一直被挂起,需要协调者快速恢复来通知没有提交事务的mysql提交事务    


## 参考:  
http://blog.xiaohansong.com/2016/08/25/zab/  
http://www.tcs.hut.fi/Studies/T-79.5001/reports/2012-deSouzaMedeiros.pdf  






