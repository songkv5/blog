---
author: willis
date: 2020-05-23 23:34
---

# Zookeeper与ZAB协议

## 一、Zookeeper
### 1、定位
>Zookeeper是一个开放源代码的分布式协调服务，雅虎创建，是google chubby的开源实现。Zookeeper的设计目标是将那些复杂且容易出错的分布式一致性服务封装起来，构成一个可靠的原语集，兵役一些列简单易用的接口提供给用户使用。
>Zookeeper是一个典型的分布式数据一致性的解决方案，分布式应用程序可以基于它实现诸如数据发布订阅，负载均衡、命名服务、分布式协调/通知、集群管理、Master选举、分布式锁和分布式队列等功能。

### 2、一致性
- 顺序一致性
> 从同一个客户端发起的事务请求，最终将会严格按照其他发起顺序被应用到Zookeeper中

- 原子性
> 所有事物请求的处理结果在整个集群中所有机器上的应用情况是一致的，也就是说，要么整个集群所有机器都成功应用了某一个事物，要么都没有应用。

- 单一视图（Single System Image）
> 无论客户端连接的是哪个Zookeeper服务器，其看到的服务器数据模型都是一致的

- 可靠性
> 一旦服务端成功的应用了一个事务，并完成对客户端的响应，那么钙食物所引起的服务端状态变更将会一致保留下来，除非有另一个事务又对其进行了变更。

- 实时性
>一旦一个事务被成功应用，那么客户端能够立刻从服务端上读取到这个事务变更后的最新数据状态。但是，Zookeeper仅仅保证在【一定的时间内】，客户端最终一定能够从服务端上读取到最新的数据状态

### 3、zk的集群模式
![zk集群]({{ site.baseurl }}/images/2020-05-23/zk-group.png)

#### 3.1、集群角色
在zk中，没有沿用传统的Master/Slave概念，而是引入了Leader、Flollower和Observer三种角色。集群中的所有机器通过一个Leader选举过程来选定一台被称为leader的机器，支持读与写；follower和observer都能提供读服务，唯一区别在于observer机器不参与leader选举，也不参与过半策略。

#### 3.2、会话
即session。在zk中，通过tcp长链接维持会话，客户端通过心跳检测与服务端保持有效的会话。

#### 3.3、结点
1. 构成集群的机器，成为机器结点
2. 数据模型中的数据单元，成为数据结点（Znode）。
> zk将所有数据存储在内存中，数据模型是一棵树（Znode Tree），由斜杠/进行分割的路径，就是一个Znode，例如/foo/path1。每个znode都会保存自己的数据内容，同时还会保存一些里属性信息。
> 在zk中，znode可以分为持久结点和临时结点。持久结点一旦创建，除非主动移除znode，否则将一直存在于zk上；临时结点的生命周期和客户端会话（session）绑定，会话失效意味着临时结点会被移除。

#### 3.4、ACL（Acess control lists）权限控制，zk定义的权限如下
- CREATE：创建子节点
- READ：获取节点数据和子节点列表
- WRITE：更新节点数据
- DELETE：删除子节点
- ADMIN：设置结点ACL

## 二、ZAB
### 1、ZAB协议

>很多人可能认为Zookeeper就是paxos算法的一个实现，但事实上，zookeeper并没有完全采用Paxos算法，而是使用了一种称为zookeeper atomic broadcast（zab，zookeeper原子消息广播协议）的协议作为数据一致性的核心算法。
**zab协议是为分布式协调服务zookeeper专门设计的一种支持崩溃恢复的原子广播协议**

>基于zab协议，zk实现了一种主备模式的系统架构来保持集群中个副本之间的数据一致性。zk使用一个【单一的主进程】来接收并处理客户端的所有事务请求，并采用zab的原子广播协议，将服务器数据的状态变更以事务proposal的形式广播到所有的副本进程。

**zab要保证的问题：**一个全局的变更序列被顺序调用。zab协议需要保证如果一个状态变更已经被处理了，那么所有其以来的状态变更都应该已经被提前处理掉了。

**核心**：zab定义了对于那些会改变zk服务器数据状态的事务请求的处理方式，如下：
>所有事务请求必须有一个全局唯一的服务器来协调处理，这样的服务器成为leader服务器，而余下的其他服务器则成为follower服务器。leader服务器负责将一个客户端事务请求转换成一个事务proposal（提议），并将该proposal分发给集群中所有的follower服务器。之后leader服务器需要等待所有follower服务器的反馈，一旦超过半数的follower服务器进行了正确的反馈后，那么leader就会再次向所有的follower服务器分发commit消息，要求其将前一个proposal进行提交

#### 1.1、zab的两种工作模式

- 1、崩溃恢复模式

>当整个服务框架在【启动过程】中，或者是当【Leader服务器出现网络中断】、崩溃退出与重启等异常情况时，zab协议就会进入恢复模式并选举产生新的Leader服务器。

>当选举产生了新的Leader服务器，同时集群中已经有过半的机器与该leader服务器完成了状态同步之后，zab协议就会退出恢复模式。其中，所谓的状态同步是指数据同步，用来保证集群中存在过半的机器能够和leader服务器的数据保持一致。

>一个机器要成为新的leader，必须获得过半进程的支持，同时由于每个进程都有可能会奔溃，因此在zab协议运行的过程中，前后会出现多个leader，并且每个进程也有可能会多次成为leader。
>eg. 一个由3台机器组成的zab服务，通常由一个leader、2个follower服务器组成，某一个时刻，假如其中一个follower服务器挂了，整个zab集群是不会中断服务的，这时因为leader服务器依然能够获得过半机器（包括leader自己）的支持

**基本特性**

1)zab协议需要确保那些已经在leader服务器上提交的事务最终被所有服务器都提交
> 假如一个事务在leader服务器提交了，而且已经收到了过半follower的ack，但是在发送commit之前，leader挂了，如图
![ZAB崩溃恢复案例]({{ site.baseurl }}/images/2020-05-23/zab-2.png)

**说明：**
> px:proposalx

> cx: commit of proposalx

>如图这种案例，zab协议需要确保事务proposal2最终能在所有的服务器上都被提交成功，否则出现不一致。

2)zab协议需要丢弃那些只在leader服务器被提出的事务。
>同样上图，p3只在leader上被提出，其他follower都没有收到这个事务proposal。当leader再次回复过来加入到集群中的时候，zab协议需要确保丢弃proposal3这个事务。


- 2、消息广播模式
zab协议的消息广播过程使用的是一个**原子广播协议**，类似二阶段提交过程。

>针对客户端的事务请求，leader 服务器会为其生成对应的事务proposal，并将其发送给集群中其余所有的机器，然后再分别收集各自的选票，最后进行事务提交。如图
![ZAB广播协议]({{ site.baseurl }}/images/2020-05-23/zab-1.png)

>ZAB将二阶段提交中的中断逻辑移除，意味着我们可以在过半的follower服务器已经反馈Ack之后就开始提交事务proposal了，而不需要等待集群中所有的follower服务器都反馈响应。

>消息广播协议是基于具有fifo特性的tcp协议来进行网络通信的，因此能够很容易的保证消息广播过程中消息接收与发送的顺序性。

>整个消息广播过程中，leader服务器会为每个事物请求生成对应的proposal来进行广播，并且在广播事物proposal之前，leader服务器会首先为这个事务proposal分配一个全局单调递增的唯一ID，我们称之为事务ID（ZXID）。为了保证每个消息的严格的因果关系，必须将每一个事务proposal按照其ZXID的先后顺序来进行排序处理

### 2、主备服务器的数据同步

所有正常运行的服务器，要么成为leader，要么成为follower并和leader保持同步。

**新的follower如何加入到集群？**

>Leader服务器会为每一个follower服务器准备一个队列，并将那些没有被个follower服务器同步的事务以proposal消息的形式诸葛发送给follower服务器，并在每一个propsal消息后面紧接着在发送一个commit消息，以表示该事务已经被提交。等到follower服务器将所有其尚未同步的事务proposal都从leader服务器上同步过来并成功应用到本地数据库中，leader服务器就会将该follower服务器加入到真正可用follower列表中。

#### 2.1、ZXID的组成
![zxid结构]({{ site.baseurl }}/images/2020-05-23/zab-zxid-1.png)
>每当选举产生一个新的leader服务器，就会从这个leader服务器上取出其本地日志中最大事务proposal的xid，并从该zxid中解析出对应的epoch值（高32位），进行+1操作，之后以此编号作为新的epoch（所以新的epoch一定是有史以来值最大的）。

>zab协议通过epoch编号来区分leader周期变化的策略，可以有效避免不同的leader错误使用相同的zxid编号提出不一样的事务proposal的异常情况。

>基于类似的策略，当一个包含了上一个leader周期中尚未提交过的事务的proposal的服务器启动时，它肯定是无法成为leader。因为当前集群中包含一个Quorum集合，该集合中的机器一定包含了更高epoch的事务proposal，因此这台机器的事务proposal肯定不是最高，也就无法成为leader。

>**说明：**
>>Quorum: 过半原则。Quorum集合代表超过集合一半数量的子集。

### 3、zab协议的系统模型
_在由一个进程组∏ = {P1, P2, ..., Pn}组成的分布式系统中，每一个进程都有各自的存储设备，个进程通过相互通信来实现消息的传递。当集群中过半的处于UP状态的进程组成一个进程自己之后，就可以进行正常的消息广播了，将这样一个进程子集成为Quorum（简称Q），满足如下协议_

>∧ 任意的Q, Q ⊆ ∏；
>∧ 任意的Q1和Q2，Q1∩Q2 ≠ ∅

### 4、主进程周期
为了保证主进程每次广播出来的事务消息一致，必须却道zab协议只有在充分完成泵快恢复阶段之后，新的主进程才可以开始生成新的事务消息广播。为了实现这个目的，假设每个进程都实现了类似于ready（e）这样的一个函数，zab协议能够非常明确的告诉主进程和副本进程是否可以开始进行事务消息的广播。在调用ready之后，zab协议还需要为当前主进程设置一个实例值，用来唯一标志一个主进程周期，这个值就是上面提到的epoch值。zab协议保证在不同的主进程周期中是全局唯一的。参见[ZXID组成](#2.1、ZXID的组成)

### 5、事务组成
主进程每次进行事务广播都包含了两部分内容：
1. 事务内容
2. 事务标识
>事务标识又包含主进程周期epoch和当前主进程周期内的事务计数器counter

**在实际运行过程中，如果一个事务标识z优先于两一个事务标识z'，有两种可能：**
1. z的主进程周期前与z'的主进程周期：epoch(z) < epoch(z')
2. 主进程周期一致，但是事务z的事务计数器小于z'的事务计数器：epoch(z) < epoch(z') && counter(z) < counter(z')

### 6、 消息广播与崩溃恢复的三阶段描述
消息广播与崩溃恢复两个过程可以细分成3个阶段
1. 发现（Discovery）
2. 同步（Synchronization）
3. 广播（Broadcast）
组成zab协议的每一个分布式进程，会循环执行这个三个阶段，将这样一个循环成为一个**准进程周期**。

#### 6.1、发现
发现过程如图所示
![发现阶段执行过程]({{ site.baseurl }}/images/2020-05-23/zab-discovery-1.png)
**初始化事务集合**
准leader会从过半回复ack的服务器中选取一个Follower F，使它作为【初始化事务集合Ie'】。关于F的选取，对于Quorum中其他任意一个Follower F'，F需要满足一下两个条件中的一个
>1. Epoch(F'·p) < Epoch(F·p)。即F的leader周期值最大
>2. Epoch(F'·p) = Epoch(F·p) && F'·zxid ≤ F·zxid。即leader周期相同，但是事务计数器值大于等于F'的

#### 6.2、同步
同步的过程如图所示
![同步阶段执行过程]({{ site.baseurl }}/images/2020-05-23/zab-sync-1.png)

#### 6.3 广播
完成同步阶段之后，zab协议就可以正式开始接受客户端新的事务请求，并进行消息广播流程
![广播阶段执行过程]({{ site.baseurl }}/images/2020-05-23/zab-broadcast-1.png)
>正常情况下，zab协议会一致运行于阶段三来反复的进行消息广播流程。如果出现leader崩溃或其他原因导致leader确实，此时zab协议会再次进入阶段一

#### 6.4 名词说明

名词|说明
----:|:----
F·p|Follower f 处理的最后一个事务proposal
F·zxid|Follower f 处理过的历史事务Proposal中最后一个事务Proposal的事务标识ZXID
hf|每个follower已经处理过的事务序列。（一个follower会处理很多proposal）
Ie|初始化历史记录。每个主进程周期阶段一都会进行有初始化序列，阶段一完成时，hf就会标记为Ie


## 三、zk客户端

### 1、zkClient
依赖如下
```xml
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.6.1</version>
</dependency>
<dependency>
    <groupId>com.github.sgroschupf</groupId>
    <artifactId>zkclient</artifactId>
    <version>0.1</version>
</dependency>
```
>zkclient有很多实现，即基于zookeeper的java api接口实现的封装。


### 2、curator
>curator的意思是：馆长。zookeeper意为动物园看守者。用馆长来命名很贴切。
依赖如下
```xml
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-framework</artifactId>
    <version>5.0.0</version>
</dependency>

<!--下面这个包可选。这个包中提供了许多zk的使用参考-->
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-recipes</artifactId>
    <version>5.0.0</version>
</dependency>
<!--下面这个包可选。这个包中提供了许多便于测试的工具类-->
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-test</artifactId>
    <version>5.0.0</version>
</dependency>
```

-----
###### 参考文献《从paxos到Zookeeper分布式一致性原理与实践》