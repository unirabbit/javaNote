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