---
author: willis
date: 2020-06-21 12:53
---

# zk里的投票
zk中的投票，涉及到leader选举投票与实务proposal投票。

## 一、Leader选举投票
>zk会根据zoo.config的配置，创建响应的leader选举算法。从3.4.0版本开始，zk废弃了LeaderElection和AuthFastLeaderElection，只支持FastLeaderElection选举算法了。
### 1、服务器启动时期的leader选举
>当有一台服务器启动的时候，他是无法完成leader选举的，是无法进行leader选举的。当第二台机器也启动后，此时这两台机器已经能够进行互相通信，每台机器都试图找到一个leader，就会进入leader选举流程。

_选举过程如下_

- 1、每个server发出一个投票
>对于server1和server2来说，**都会将自己作为leader服务器来进行投票**，每次投票的基本元素有：所选举的服务器的myid和ZXID（myid，ZXID）。初始阶段，无论是server1还是server2，都会投给自己，即server1投票为（1,0），server2投票为（2,0），然后各自将这个投票发给集群中的其他所有机器

- 2、接收来自各个服务器的投票
>每个服务器都会接收来自其他服务器的投票，集群中的每个服务器在接收到投票后，首先会判断该投票的有效性，包括检查是否是本轮投票，是否来自LOOKING状态的服务器

- 3、处理投票
>在接收到来自其他服务器的投票后，针对每一个投票，服务器都需要将别人的投票和自己的投票进行PK
>- 有限检查zxid。zxid比较大的服务器有限作为leader
>- 如果zxid相同，就比较myid。myid比较大的作为leader

- 4、统计投票
>每次投票后，服务器都会统计所有的投票，判断是否已经有**过半**的机器接收到相同的投票信息。对于server1和server2服务器来说，都统计出已经有两台机器接受了（2,0）这个投票信息。

- 5、改变服务器状态
>一旦确定了leader，每个服务器就会更新自己的状态：如果是Follower，那么就变更为FOLLOWING，如果是LEADER，那么久变更为LEADING


### 2、服务器运行期间的Leader选举
>在zk集群正常运行过程中，一旦选举出一个leader，那么所有服务器的集群角色一般不会发生变化。也就是说，leader服务器将一直作为集群的leader，即使集群中有非leader集群挂了或是有新机器加入集群也不会影响Leader。但是一旦Leader所在机器挂了，那么整个集群将暂时无法对外服务，而是进入新一轮的leader选举。服务器运行期间的Leader选举和启动时期的Leader选举基本过程是一致的。

__选举过程__

- 1、变更状态
>当Leader挂了之后，余下的非Observer服务器都会将自己的服务器状态变更为LOOKING，然后开始进入Leader选举流程

- 2、每个Server会发出一个投票

- 3、接收来自各个服务器的投票

- 4、处理投票

- 5、统计投票

- 6、改变服务器状态

### 3、选举算法

#### 3.1、选举的触发条件
- 服务器初始化启动时
- 服务器运行期间无法和Leader保持连接

当一台机器进入Leader选举流程时，当前集群也可能会处于以下两种状态
- 集群中本来就已经存在一个Leader
>这种情况通常是集群中的某一台机器启动比较晚，在他启动之前，集群已经可以正常工作，即已经存在一台Leader服务器。
- 集群中确实不存在Leader

#### 3.2、算法过程

- 1、开始第一次投票
>两种情况导致集群不存在leader：1)整个服务器刚刚初始化启动时，此时尚未产生一台Leader服务器。2）运行期间当前集群Leader服务器挂了。无论哪种情况，此时集群中所有机器都处于一种师徒选举出一个Leader的状态，成为LOOKING状态。当一台机器处于LOOKING状态时，那么他就会向集群中所有其他机器发送消息，我们成这个消息为“投票”。

>投票消息中包含两个最基本信息：（SID，ZXID）

- 2、变更投票
>集群中每台机器发出自己的投票后，也会接收到来自集群中其他机器的投票。相关概念
>- vote_sid: 接收到的投票中所推举Leader服务器的sid
>- vote_zxid: 接收到投票中所推举Leader服务器的zxid
>- self_sid: 当前服务器自己的sid
>- self_zxid: 当前服务器自己的zxid
>服务器每次对投票的处理，都是对（vote_sid,vote_zxid）(self_sid,self_zxid)的处理。

>**处理规则如下**
>1. 如果vote_zxid大于self_zxid,就认可当前收到的投票，并在此将该投票发送出去
>2. 如果vote_zxid小于self_zxid，那么就坚持自己的投票，不做任何变更
>3. 如果vote_zxid等于self_zxid,就比较sid。如果vote_sid>self_sid,那么就认可收到的投票，并在此将该投票发送出去
>4. 如果vote_zxid等于self_zxid,兵器vote_sid < self_sid,那么就坚持自己的投票不做变更。

**变更投票的前后过程**

eg： 5台服务器：server 1~5.其中 server1和server2挂掉，如图
![投票过程]({{ site.baseurl }}/images/2020-06-21/zk-vote-1.png)

- 对于server-3来说，他接受到(4,8)和(5,8)这两个投票。对比后，自己的zxid大于接收到的zxid，不做任何变更
- 对于server-4来说，它接收到了收到(3,9)和(5,8)两个投票，对比后，由于收到(3,9)这个投票的zxid大于自己，因此需要变更投票为收到(3,9)，然后继续将这个投票发送给另外两台机器
- 对于server-5来说，他收到收到(3,9)和(4,8).对比后，由于（3,9）这个投票的zxid大于自己，因此需要将投票变更为（3,9），然后继续将这个投票发送给另外两台机器。

- 3、 确定Leader
经过第二次投票后，集群中的每台机器都会再次收到其他机器的投票，然后开始统计投票。如果一台机器收到了超过半数相同的投票，那么整个投票对应的sid机器即为Leader。
在上述例子中，server-3、server-4、server-5都投票（3,9），因此确定server-3为Leader


## 二、事务处理
在zk中，每一个事务请求都需要集群中过半机器投票认可才能被真正应用到zk的内存数据库中。

### proposal处理流程

1）发起投票
>如果当前请求是事务请求，那么Leader服务器就会发起一轮事务投票。在发起投票之前，首先会检查当前服务端的zxid是否可用。如果不可用，将会抛出异常。

2）生成提案 proposal
>如果当前服务端的zxid可用，那么就可以开始事务投票了。zk会将之前创建的请求头和事务体，以及zxid和请求本身序列化到proposal对象中。

3）广播提议
>生成提议后，Leader服务器会以zxid作为标识，将该提议放入投票箱outstandingProposals中，同时会将该提议广播给所有Follower服务器

4）收集投票
>Follower服务器在接收到Leader发来的这个提议后，会进入Sync流程来进行事务日志的记录，一旦日志记录完成后，就会发送**ack消息**给Leader服务器，Leader服务器根据这些ACK消息来统计每个提议的投票情况。
当一个提议获得了集群中过半机器的投票，那么就认为该提议通过，接下去就可以进入提起的commit阶段。

5）将请求放入toBeApplied队列。
>在该提议提交之前，zookeeper首先将其放入toBeApplied队列。

6）广播commit消息
>一旦zk确认一个提议已经可以被提交了，那么Leader服务器就会向Follower和Observer服务器发送commit消息，以便所有服务器都能够提交该提议。zk对Observer服务器广播commit消息会区别对待，Leader会想起发送一种被称为Inform的消息，该消息中包含了当前提议的内容。对Follower服务器，由于已经保存了有关该提议的信息，leader服务器只需要向其发送zxid即可。


### Commit流程（P349）

1）请求交付给CommitProcessor处理器。
>将请求放图queuedRequests队列。commitProcessor会处理PreRequestProcessor中流转下来的请求
2）处理queuedRequests队列请求
>特有线程检测请求队列，逐个取出进行处理
3）标记nextPending
>如果请求是事务请求，需要投票，当前请求标记为nextPending。保证事务顺序性，同时方便知道当前集群是否正在进行事务请求投票。
4）等待Proposal投票
>发起投票后需要等待投票结果
5）投票通过
>如果一个提议获得了过半机器的投票认可，将会进入请求提交阶段。zk将该请求放入commitedRequests队列，同时唤醒Commit流程
6）提交请求
>一旦commitedRequests队列有可以提交的请求，commit流程就开始提交请求了。提交之前，commit流程会对比之前标记的nextPending和commitedRequests队列中第一个请求是否一致。如果检查通过，commit流程会将请求放入toProcess队列中，交付给下一个请求处理器：FinalRequestProcessor.