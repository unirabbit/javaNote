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

# 二.  Producer/Comsumer

## 2.1 Producer 启动流程

Producer启动时，需要指定Namesrv的地址，从Namesrv集群中选一台建立长连接。如果该Namesrv宕机，会自动连其他Namesrv。直到有可用的Namesrv为止。生产者每30秒从Namesrv获取Topic跟Broker的映射关系，更新到本地内存中。再跟Topic涉及的所有Broker建立长连接，每隔30秒发一次心跳。

## 2.2 Producer发送流程

消息发送流程主要的步骤：验证消息、查找路由、消息发送（包含异常处理机制）。

**三种发送方式**

- 同步：在广泛的场景中使用可靠的同步传输，如重要的通知信息、短信通知、短信营销系统等。
- 异步：异步发送通常用于响应时间敏感的业务场景，发送出去即刻返回，利用回调做后续处理。
- 一次性：一次性发送用于需要中等可靠性的情况，如日志收集，发送出去即完成，不用等待发送结果，回调等等。

## 2.3 Comsumer消费流程

RMQ利用“长轮询”来实现拉模式,client发送请求到server，server有数据返回，没有数据请求挂起不断开连接.

消息消费有两种模式：广播模式与集群模式。

广播模式比较简单，每一个消费者需要去拉取订阅主题下所有消费队列的消息。

集群消费：一个topic可以由同一个ID下所有消费者分担消费。当使用集群消费模式时，消息队列 RocketMQ 认为任意一条消息只需要被集群内的任意一个消费者处理即可。

## 2.3 负载均衡

生产者发送时，会自动轮询当前所有可发送的broker，一条消息发送成功，下次换另外一个broker发送，以达到消息平均落到所有的broker上。

在集群消费模式下（clustering）相同的group中的每个消费者只消费topic中的一部分内容，group中的所有消费者都参与消费过程，每个消费者消费的内容不重复，从而达到负载均衡的效果。

# 三. Broker

## 3.1 数据存储

消息存储是RocketMQ中最为复杂和最为重要的一部分，决定了生产者消息写入的吞吐量，决定了消息不能丢失，决定了消费者获取消息的吞吐量。

RocketMQ消息的存储是由ConsumeQueue和CommitLog配合完成 的，消息真正的物理存储文件是CommitLog，ConsumeQueue是消息的逻辑队列，类似数据库的索引文件，存储的是指向物理存储的地址。每 个Topic下的每个Message Queue都有一个对应的ConsumeQueue文件。

![在这里插入图片描述](https://gitee.com/adambang/pic/raw/master/20210106103912.png)

### 3.1.1 消息存储整体架构

消息存储架构图中主要有下面三个跟消息存储相关的文件构成。

1. CommitLog：消息主体以及元数据的存储主体，存储Producer端写入的消息主体内容,消息内容不是定长的。单个文件大小默认1G ，文件名长度为20位，左边补零，剩余为起始偏移量。消息主要是顺序写入日志文件，当文件满了，写入下一个文件；
2. ConsumeQueue：消息消费队列，引入的目的主要是提高消息消费的性能，由于RocketMQ是基于主题topic的订阅模式，消息消费是针对主题进行的，如果要遍历commitlog文件中根据topic检索消息是非常低效的。Consumer即可根据ConsumeQueue来查找待消费的消息。其中，ConsumeQueue（逻辑消费队列）作为消费消息的索引，保存了指定Topic下的队列消息在CommitLog中的起始物理偏移量offset，消息大小size和消息Tag的HashCode值。consumequeue文件可以看成是基于topic的commitlog索引文件。
3.  IndexFile：IndexFile（索引文件）提供了一种可以通过key或时间区间来查询消息的方法。Index文件的存储位置是：HOME\store\indexHOME\store\index{fileName}，文件名fileName是以创建时的时间戳命名的，固定的单个IndexFile文件大小约为400M，一个IndexFile可以保存 2000W个索引，IndexFile的底层存储设计为在文件系统中实现HashMap结构，故rocketmq的索引文件其底层实现为hash索引。

> RocketMQ通过使用内存映射文件来提高IO访问性能，无论是CommitLog、ConsumeQueue还是IndexFile，单个文件都被设计为固定长度，如果一个文件写满以后再创建一个新文件，文件名就为该文件第一条消息对应的全局物理偏移量。

![image-20210106134657055](https://gitee.com/adambang/pic/raw/master/20210106134657.png)

### 3.1.2  CommitLog写入性能

Broker是基于OS操作系统的**PageCache**和**顺序写**两个机制，来提升CommitLog写入性能的。

首先Broker是以顺序的方式将消息写入CommitLog磁盘文件的，也就是每次写入在文件末位追加一条记录即可。对文件顺序写比随机写的性能要高很多。

另外，数据写入CommitLog文件时，并未直接写入底层的物理磁盘文件，而是先进入OS操作系统的**Page Cache**内存缓存中，然后后续由OS的后台线程选一个时间，异步化的将OS PageCache内存缓冲中的数据刷入底层的磁盘文件（**刷盘策略**分为同步刷盘和异步刷盘，吞吐量与数据丢失风险的平衡）。

![image-20210106140340334](https://gitee.com/adambang/pic/raw/master/20210106140340.png)

## 3.2 主从同步

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

## 3.3 高性能读写优化

### 3.3.1 传统IO多次拷贝问题

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

### 3.3.2 mmap与PageCache

mmap相当于将磁盘文件与虚拟内存地址（PageCache）建立一个映射关系，所以可以减少一次cpu的内存拷贝。

如下图，broker写入与读取消息都只需要进行一次地址拷贝。

<img src="https://gitee.com/adambang/pic/raw/master/20210106162221.png" alt="image-20210106162221000" style="zoom:67%;" />

### 3.3.3 零拷贝

RocketMQ读写过程中，使用了零拷贝，零拷贝包含以下两种方式

1. 使用 mmap + write 方式
   优点：即使频繁调用，使用小块文件传输，效率也很高
   缺点：不能很好的利用 DMA 方式，会比 sendfile 多消耗 CPU，内存安全性控制复杂，需要避免 JVM Crash问题。
2. 使用 sendfile 方式
   优点：可以利用 DMA 方式，消耗 CPU 较少，大块文件传输效率高，无内存安全新问题。
   缺点：小块文件效率低于 mmap 方式，只能是 BIO 方式传输，不能使用 NIO。
   RocketMQ 选择了第一种方式， mmap+write 方式，因为有小块数据传输的需求，效果会比 sendfile 更好。  

# 三 常见问题

## 3.1. 消息零丢失方案

### 3.1.1 producer零丢失方案

1. 方案一 同步发送消息+反复多次重试
2. 方案二 事务消息发送机制

**Half Message(半消息)**

是指暂不能被Consumer消费的消息。Producer 已经把消息成功发送到了 Broker 端，但此消息被标记为`暂不能投递`状态，处于该种状态下的消息称为半消息。需要 Producer

对消息的`二次确认`后，Consumer才能去消费它。

**消息回查**

由于网络闪段，生产者应用重启等原因。导致 **Producer** 端一直没有对 **Half Message(半消息)** 进行 **二次确认**。这是**Brock**服务器会定时扫描`长期处于半消息的消息`，会

主动询问 **Producer**端 该消息的最终状态(**Commit或者Rollback**),该消息即为 **消息回查**。

![图片](https://gitee.com/adambang/pic/raw/master/20210107163522.webp)

1. A服务先发送个Half Message给Brock端，消息中携带 B服务 即将要+100元的信息。
2. 当A服务知道Half Message发送成功后，那么开始第3步执行本地事务。
3. 执行本地事务(会有三种情况1、执行成功。2、执行失败。3、网络等原因导致没有响应)
4. 如果本地事务成功，那么Product向Brock服务器发送Commit,这样B服务就可以消费该message。
5. 如果本地事务失败，那么Product向Brock服务器发送Rollback,那么就会直接删除上面这条半消息。
6. 如果因为网络等原因迟迟没有返回失败还是成功，那么会执行RocketMQ的回调接口,来进行事务的回查。

### 3.1.2 broker消息零丢失

同步刷盘+raft协议主从同步

### 3.1.3 comsumer消息零丢失

offset手动提交，消息同步处理，在消费者执行完业务逻辑后在给broker返回成功。

## 3.2 消费幂等

消息队列 RocketMQ 消费者在接收到消息以后，有必要根据业务上的唯一 Key 对消息做幂等处理的必要性。

### 3.2.1 消费幂等的必要性

在互联网应用中，尤其在网络不稳定的情况下，消息队列 RocketMQ 的消息有可能会出现重复，这个重复简单可以概括为以下情况：

- 发送时消息重复

  当一条消息已被成功发送到服务端并完成持久化，此时出现了网络闪断或者客户端宕机，导致服务端对客户端应答失败。 如果此时生产者意识到消息发送失败并尝试再次发送消息，消费者后续会收到两条内容相同并且 Message ID 也相同的消息。

- 投递时消息重复

  消息消费的场景下，消息已投递到消费者并完成业务处理，当客户端给服务端反馈应答的时候网络闪断。 为了保证消息至少被消费一次，消息队列 RocketMQ 的服务端将在网络恢复后再次尝试投递之前已被处理过的消息，消费者后续会收到两条内容相同并且 Message ID 也相同的消息。

- 负载均衡时消息重复（包括但不限于网络抖动、Broker 重启以及订阅方应用重启）

  当消息队列 RocketMQ 的 Broker 或客户端重启、扩容或缩容时，会触发 Rebalance，此时消费者可能会收到重复消息。

### 3.2.2 处理方式

保证每条消息都有唯一编号(**比如唯一流水号)**，且保证消息处理成功与去重表的日志同时出现。

建立一个消息表，拿到这个消息做数据库的insert操作。给这个消息做一个唯一主键（primary key）或者唯一约束，那么就算出现重复消费的情况，就会导致主键冲突，那么就不再处理这条消息。

用redis记录消息的状态，每次收到消息都从redis查询消息状态。

## 3.3 消息重试

### 3.3.1 顺序消息的重试

对于顺序消息，当消费者消费消息失败后，消息队列 RocketMQ 会自动不断进行消息重试（每次间隔时间为 1 秒），这时，应用会出现消息消费被阻塞的情况。因此，在使用顺序消息时，务必保证应用能够及时监控并处理消费失败的情况，避免阻塞现象的发生。

顺序消息消费失败不能走重试队列。

### 3.3.2 无序消息的重试

对于无序消息（普通、定时、延时、事务消息），当消费者消费消息失败时，您可以通过设置返回状态达到消息重试的结果。

无序消息的重试只针对集群消费方式生效；广播方式不提供失败重试特性，即消费失败后，失败消息不再重试，继续消费新的消息。

Consumer处理消息失败，可以给Broker返回RECONSUME_LATER状态。可以过段时间进行消息发送重试（消息放入ComsumerGroup消费组中的重试队列中）。

![image-20210108145148272](https://gitee.com/adambang/pic/raw/master/20210108145148.png)

## 3.4 死信队列

当一条消息初次消费失败，消息队列 RocketMQ 会自动进行消息重试；达到最大重试次数后，若消费依然失败，则表明消费者在正常情况下无法正确地消费该消息，此时，消息队列 RocketMQ 不会立刻将消息丢弃，而是将其发送到该消费者对应的特殊队列中。

在消息队列 RocketMQ 中，这种正常情况下无法被消费的消息称为死信消息（Dead-Letter Message），存储死信消息的特殊队列称为**死信队列**（Dead-Letter Queue）

默认重试16次之后，消息就变成死信，放入死信队列中。

可以后台单独开一个线程单处处理死信队列中的信息。

![image-20210108145539554](https://gitee.com/adambang/pic/raw/master/20210108145539.png)

**死信队列特性：**

死信消息具有以下特性

- 不会再被消费者正常消费。
- 有效期与正常消息相同，均为 3 天，3 天后会被自动删除。因此，请在死信消息产生后的 3 天内及时处理。

死信队列具有以下特性：

- 一个死信队列对应一个 Group ID， 而不是对应单个消费者实例。
- 如果一个 Group ID 未产生死信消息，消息队列 RocketMQ 不会为其创建相应的死信队列。
- 一个死信队列包含了对应 Group ID 产生的所有死信消息，不论该消息属于哪个 Topic。

## 3.5 消息顺序

**一个topic下有多个队列**，为了保证发送有序，**RocketMQ**提供了**MessageQueueSelector**队列选择机制。可使用**Hash取模法**，让同一个订单发送到同一个队列中，再使用同步发送，只有同个订单的创建消息发送成功，再发送支付消息。这样，我们保证了发送有序。

有序消息处理失败不能走重试队列，会使消息乱序。

消息发送有序+发送到同一个MessageQueue中+处理失败，内部等待处理

binlog消息全局有序性保证如下：

<img src="https://gitee.com/adambang/pic/raw/master/20210108152948.png" alt="image-20210108152948432" style="zoom:80%;" />