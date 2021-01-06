# 一 .RocketMQ 的架构原理

## 1.1 选型比较

### 1.1.1 kafka

首先Kafka的吞吐量几乎是行业里最优秀的，在常规的机器配置下，一台机器可以达到每秒十几万的QPS，相当的强悍。
Kafka性能也很高，基本上发送消息给Kafka都是**毫秒级**的性能。可用性也很高，Kafka是可以支持集群部署的，其中部分机器宕机是可以继续运行的。

但是Kafka比较为人诟病的一点，似乎是**丢数据**方面的问题，因为Kafka收到消息之后会写入一个磁盘缓冲区里，并没有直接落地到物理磁盘上去，所以要是机器本身故障了，可能会导致磁盘缓冲区里的数据丢失。
而且Kafka另外一个比较大的缺点，就是**功能非常的单一**，主要是支持发送消息给他，然后从里面消费消息，其他就没有什么额外的高级功能了。所以基于Kafka有限的功能，可能适用的场景并不是很多。

因此综上所述，以及查阅了Kafka技术在各大公司里的使用，基本行业里的一个标准，是把Kafka用在用户行为日志的采集和传输上，比如大数据团队要收集APP上用户的一些行为日志，这种日志就是用Kafka来收集和传输的。
因为那种日志适当丢失数据是没有关系的，而且一般量特别大，要求吞吐量要高，一般就是收发消息，不需要太多的高级功能，所以Kafka是非常适合这种场景的。  

### 1.1.2 RabbitMq

在RocketMQ出现之前，国内大部分公司都从ActiveMQ切换到RabbitMQ来使用，包括很多一线互联网大厂，而且直到现在都有很多中小型公司在使用RabbitMQ。
RabbitMQ的优势在于可以**保证数据不丢失，也能保证高可用性**，即集群部署的时候部分机器宕机可以继续运行，然后支持部分高级功能，比如说死信队列，消息重试之类的，这些是他的优点。
但是也有一些缺点:

1. RabbitMQ的吞吐量是比较低的，一般就是每秒几万的级别，所以如果遇到特别特别高并发的情况下，支撑起来是有点困难的。
2. 进行集群扩展的时候（也就是加机器部署）比较麻烦。
3. 另外还有一个较为致命的缺陷，其开发语言是erlang，二次开发与维护比较麻烦。

### 1.1.3 RocketMQ  

RocketMQ是阿里开源的消息中间件，久经沙场，非常的靠谱。他几乎同时解决了Kafka和RabbitMQ的缺陷。
RocketMQ的吞吐量也同样很高，单机可以达到10万QPS以上，而且可以保证高可用性，性能很高，而且支持通过配置保证数据绝对不丢失，可以部署大规模的集群，还支持各种高级的功能，比如说延迟消息、事务消息、消息回溯、死信队列、消息积压，等等。
而且RocketMQ是基于Java开发的，符合国内大多数公司的技术栈，很容易就可以阅读他的源码，甚至是修改他的源码。  

## 1.2 基本构成

Producer：消息的发送者，支持分布式集群方式部署。Producer通过MQ的负载均衡模块选择相应的Broker集群队列进行消息投递，投递的过程支持快速失败并且低延迟。举例：发信者

Consumer：消息接收者，支持分布式集群方式部署。支持以push推，pull拉两种模式对消息进行消费。同时也支持集群方式和广播方式的消费，它提供实时消息订阅机制，可以满足大多数用户的需求。举例：收信者

Broker：暂存和传输消息；举例：邮局

NameServer：管理Broker，Topic路由注册中心，其角色类似Dubbo中的zookeeper，支持Broker的动态注册与发现。主要包括两个功能：举例：各个邮局的管理机构

- Broker管理，NameServer接受Broker集群的注册信息并且保存下来作为路由信息的基本数据。然后提供心跳检测机制，检查Broker是否还存活；
- 路由信息管理，每个NameServer将保存关于Broker集群的整个路由信息和用于客户端查询的队列信息。然后Producer和Conumser通过NameServer就可以知道整个Broker集群的路由信息，从而进行消息的投递和消费。

Topic：区分消息的种类；一个发送者可以发送消息给一个或者多个Topic；一个消息的接收者可以订阅一个或者多个Topic消息

Message Queue：相当于是Topic的分区；用于并行发送和接收消息

![RocketMQ角色](https://gitee.com/adambang/pic/raw/master/RocketMQ%E8%A7%92%E8%89%B2.jpg)

## 1.3 集群原理

**NameServer**是一个几乎无状态节点，可集群部署（保证中心的高可用），节点之间无任何信息同步。

**Broker**部署相对复杂，Broker分为Master与Slave，一个Master可以对应多个Slave，但是一个Slave只能对应一个Master，Master与Slave的对应关系通过指定相同的BrokerName，不同的BrokerId来定义，BrokerId为0表示Master，非0表示Slave。Master也可以部署多个。每个Broker与NameServer集群中的所有节点建立长连接，**定时注册Topic信息（TCP长连接）到所有NameServer（路由信息同步,心跳） **。

**Producer**与NameServer集群中的其中一个节点（随机选择）建立长连接，**定期从NameServer取Topic路由信息**，并向提供Topic服务的Master建立长连接，且定时向Master发送心跳。Producer完全无状态，可集群部署。

**Consumer**与NameServer集群中的其中一个节点（随机选择）建立长连接，**定期从NameServer取Topic路由信息**，并向提供Topic服务的Master、Slave建立长连接，且定时向Master、Slave发送心跳。Consumer既可以从Master订阅消息，也可以从Slave订阅消息，订阅规则由Broker配置决定。

![image-20210105105514602](C:\Users\zhaozhixiang\AppData\Roaming\Typora\typora-user-images\image-20210105105514602.png)

## 1.4 集群搭建

### 1.4.1 NameServer集群化部署  

NameServer的设计是采用的Peer-to-Peer的模式来做的，也就是可以集群化部署，但是里面任何一台
机器都是独立运行的，跟其他的机器没有任何通信。
每台NameServer实际上都会有完整的集群路由信息，包括所有的Broker节点信息，我们的数据信息，等等。所以只要任何一台NameServer存活下来，就可以保证MQ系统正常运行，不会出现故障。  

<img src="https://gitee.com/adambang/pic/raw/master/20210105165110.png" alt="image-20210105165110394" style="zoom: 80%;" />

### 1.4.2  基于Dledger的Broker主从架构部署  

Dledger技术（基于raft协议）是要求至少得是一个Master带两个Slave，这样有三个Broke组成一个Group，也就是作为一个分组来运行。一旦Master宕机，他就可以从剩余的两个Slave中选举出来一个新的Master对外提供服务 。

<img src="https://gitee.com/adambang/pic/raw/master/20210105181527.png" alt="image-20210105181526925" style="zoom:67%;" />

# 二. Broker

## 2.1 数据存储

消息存储是RocketMQ中最为复杂和最为重要的一部分，决定了生产者消息写入的吞吐量，决定了消息不能丢失，决定了消费者获取消息的吞吐量。

RocketMQ消息的存储是由ConsumeQueue和CommitLog配合完成 的，消息真正的物理存储文件是CommitLog，ConsumeQueue是消息的逻辑队列，类似数据库的索引文件，存储的是指向物理存储的地址。每 个Topic下的每个Message Queue都有一个对应的ConsumeQueue文件。

![在这里插入图片描述](https://gitee.com/adambang/pic/raw/master/20210106103912.png)

### 2.1.1 消息存储整体架构

消息存储架构图中主要有下面三个跟消息存储相关的文件构成。

1. CommitLog：消息主体以及元数据的存储主体，存储Producer端写入的消息主体内容,消息内容不是定长的。单个文件大小默认1G ，文件名长度为20位，左边补零，剩余为起始偏移量。消息主要是顺序写入日志文件，当文件满了，写入下一个文件；

2. ConsumeQueue：消息消费队列，引入的目的主要是提高消息消费的性能，由于RocketMQ是基于主题topic的订阅模式，消息消费是针对主题进行的，如果要遍历commitlog文件中根据topic检索消息是非常低效的。Consumer即可根据ConsumeQueue来查找待消费的消息。其中，ConsumeQueue（逻辑消费队列）作为消费消息的索引，保存了指定Topic下的队列消息在CommitLog中的起始物理偏移量offset，消息大小size和消息Tag的HashCode值。consumequeue文件可以看成是基于topic的commitlog索引文件。

3.  IndexFile：IndexFile（索引文件）提供了一种可以通过key或时间区间来查询消息的方法。Index文件的存储位置是：HOME\store\indexHOME\store\index{fileName}，文件名fileName是以创建时的时间戳命名的，固定的单个IndexFile文件大小约为400M，一个IndexFile可以保存 2000W个索引，IndexFile的底层存储设计为在文件系统中实现HashMap结构，故rocketmq的索引文件其底层实现为hash索引。

![image-20210106134657055](https://gitee.com/adambang/pic/raw/master/20210106134657.png)

### 2.1.2  CommitLog写入性能

Broker是基于OS操作系统的**PageCache**和**顺序写**两个机制，来提升CommitLog写入性能的。

首先Broker是以顺序的方式将消息写入CommitLog磁盘文件的，也就是每次写入在文件末位追加一条记录即可。对文件顺序写比随机写的性能要高很多。

另外，数据写入CommitLog文件时，并未直接写入底层的物理磁盘文件，而是先进入OS操作系统的**Page Cache**内存缓存中，然后后续由OS的后台线程选一个时间，异步化的将OS PageCache内存缓冲中的数据刷入底层的磁盘文件（刷盘策略分为同步刷盘和异步刷盘，吞吐量与数据丢失风险的平衡）。

![image-20210106140340334](https://gitee.com/adambang/pic/raw/master/20210106140340.png)

## 2.2 主从同步

如果要让Broker实现高可用，那么必须有一个Broker组，里面有一个是Leader Broker可以写入数据，然后让
Leader Broker接收到数据之后，直接把数据同步给其他的Follower Broker 。

![image-20210106145105590](https://gitee.com/adambang/pic/raw/master/20210106145105.png)

一条数据就会在三个Broker上有三份副本，此时如果Leader Broker宕机，那么就直接让其他的Follower Broker自动切换为新的Leader Broker，继续接受客户端的数据写入就可以了。 

如果基于DLedger技术来实现Broker高可用架构，实际上就是用DLedger先替换掉原来Broker自己管理的CommitLog，由DLedger来管理CommitLog 。

- 同时DLedger可以基于raft协议，在master节点挂了以后，进行自动选主，将slave节点提升至master节点。 
- DLedger在进行同步的时候是采用Raft协议进行两阶段完成的多副本同步的  

> 1. Leader Broker上的DLedger收到一条数据之后，会标记为uncommitted状态，然后他会通过自己的DLedgerServer组件把这个uncommitted数据发送给Follower Broker的DLedgerServer。  
> 2. 接着Follower Broker的DLedgerServer收到uncommitted消息之后，必须返回一个ack给Leader Broker的DLedgerServer，然后如果Leader Broker收到超过半数的Follower Broker返回ack之后，就会将消息标记为committed状态。
> 3. 最后Leader Broker上的DLedgerServer就会发送commited消息给Follower Broker机器的DLedgerServer，让他们也把消息标记为comitted状态。  

## 2.3 高性能读写优化

### 2.3.1 传统IO多次拷贝问题

Linux操作系统分为【用户态】和【内核态】，文件操作、网络操作需要涉及这两种形态的切换，免不了进行数据复制。

一台服务器 把本机磁盘文件的内容发送到客户端，一般分为两个步骤：

1）read；读取本地文件内容； 

2）write；将读取的内容通过网络发送出去。

这两个看似简单的操作，实际进行了4 次数据复制，分别是：

1. 从磁盘复制数据到内核态内存；
2. 从内核态内存复 制到用户态内存；
3. 然后从用户态 内存复制到网络驱动的内核态内存；
4. 最后是从网络驱动的内核态内存复 制到网卡中进行传输。

![文件操作和网络操作](https://gitee.com/adambang/pic/raw/master/%E6%96%87%E4%BB%B6%E6%93%8D%E4%BD%9C%E5%92%8C%E7%BD%91%E7%BB%9C%E6%93%8D%E4%BD%9C.png)

### 2.3.2 mmap与PageCache

mmap相当于将磁盘文件与虚拟内存地址（PageCache）建立一个映射关系，所以可以减少一次cpu的内存拷贝。

如下图，broker写入与读取消息都只需要进行一次地址拷贝。

<img src="https://gitee.com/adambang/pic/raw/master/20210106162221.png" alt="image-20210106162221000" style="zoom:67%;" />