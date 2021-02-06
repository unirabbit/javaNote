# 一.  kafka架构

## 1.1 架构总览

Kafka 是一个分布式的基于发布/订阅模式的消息队列（Message Queue），主要应用于大数据实时处理领域。

![image-20210125154729903](https://gitee.com/adambang/pic/raw/master/20210125154730.png)

1）**Producer** ：消息生产者，就是向 kafka broker 发消息的客户端； 

2）**Consumer** ：消息消费者，向 kafka broker 取消息的客户端； 

3）**Consumer Group** **（CG）**：消费者组，由多个 consumer 组成。消费者组内每个消费者负责消费不同分区的数据，一个分区只能由一个组内消费者消费；消费者组之间互不影响。所有的消费者都属于某个消费者组，即消费者组是逻辑上的一个订阅者。 

4）**Broker** ：一台 kafka 服务器就是一个 broker。一个集群由多个 broker 组成。一个 broker 可以容纳多个 topic。 

5）**Topic** ：可以理解为一个队列，生产者和消费者面向的都是一个 topic； 

6）**Partition**：为了实现扩展性，一个非常大的 topic 可以分布到多个 broker（即服务器）上， 一个 topic 可以分为多个 partition，每个 partition 是一个有序的队列； 

7）**Replica**：副本，为保证集群中的某个节点发生故障时，该节点上的 partition 数据不丢失，且 kafka 仍然能够继续工作，kafka 提供了副本机制，一个 topic 的每个分区都有若干个副本， 一个 leader 和若干个 follower。

8）**leader**：每个分区多个副本的“主”，生产者发送数据的对象，以及消费者消费数据的对象都是 leader。 

9）**follower**：每个分区多个副本中的“从”，实时从 leader 中同步数据，保持和 leader 数据 的同步。leader 发生故障时，某个 follower 会成为新的 follower。

## 1.2 工作流程及文件存储机制

Kafka 中消息是以 topic 进行分类的，生产者生产消息，消费者消费消息，都是面向 topic 的。 topic 是逻辑上的概念，而 partition 是物理上的概念，每个 partition 对应于一个 log 文 件，该 log 文件中存储的就是 producer 生产的数据。Producer 生产的数据会被不断追加到该 log 文件末端，且每条数据都有自己的 offset。消费者组中的每个消费者，都会实时记录自己 消费到了哪个 offset，以便出错恢复时，从上次的位置继续消费。

![image-20210125160318913](https://gitee.com/adambang/pic/raw/master/20210125160319.png)

由于生产者生产的消息会不断追加到 log 文件末尾，为防止 log 文件过大导致数据定位 效率低下，Kafka 采取了分片和索引机制，将每个 partition 分为多个 segment。每个 segment 对应两个文件——“.index”文件和“.log”文件。这些文件位于一个文件夹下，该文件夹的命名 规则为：topic 名称+分区序号。

![image-20210125160532195](https://gitee.com/adambang/pic/raw/master/20210125160532.png)

# 二.Kafka 生产者

## 2.1 总体流程

消息在通过 send（）方法发往 broker 的过程中， 有可能需要经过拦截器（ Interceptor）、序列
化器（ Serializer）和分区器（ Partitioner ）的一系列作用之后才能被真正地发往 broker。

发送消息主要有三种模式(producer.type)： 发后即忘（fire-and-forget〕、同步（ sync）及异步 ( async ）。

当设置为 async，会大幅提升性能，因为生产者会在本地缓冲消息，并适时批量发送。一般是在 send（）方法里指定一个 Callback 的回调函数，Kafka 在返回响应时调用该函数来实现异步的发送确认。

如果对可靠性要求高，那么这里可以设置为 sync 同步发送。

![image.png](https://cdn.nlark.com/yuque/0/2021/png/671987/1612585524511-ff9541cc-79c1-425c-986d-6a05a7beba3e.png)

## 2.2 分区策略

分区的原因 ：

1. 便在集群中扩展，每个 Partition 可以通过调整以适应它所在的机器，而一个 topic
   又可以有多个 Partition 组成，因此整个集群就可以适应任意大小的数据了； 
2. 可以提高并发，因为可以 Partition 为单位读写了。

分区的原则：

需要将 producer 发送的数据封装成一个 ProducerRecord 对象。

1. 指明 partition 的情况下，直接将指明的值直接作为 partiton 值； 
2. 没有指明 partition 值但有 key 的情况下，将 key 的 hash 值与 topic 的 partition数进行取余得到 partition 值； 
3. 既没有 partition 值又没有 key 值的情况下，第一次调用时随机生成一个整数（后面每次调用在这个整数上自增），将这个值与 topic 可用的 partition 总数取余得到 partition 值，也就是常说的 round-robin 算法。

## 2.3 数据可靠性保证

为保证 producer 发送的数据，能可靠的发送到指定的 topic，topic 的每个partition收到producer 发送的数据后，都需要向 producer 发送 ack（acknowledgement 确认收到），如果 producer 收到 ack，就会进行下一轮的发送，否则重新发送数据。

![image-20210205170855939](https://gitee.com/adambang/pic/raw/master/20210205170856.png)

### 2.3.1 副本数据同步策略

![image-20210205171350106](https://gitee.com/adambang/pic/raw/master/20210205171350.png)

Kafka 选择了第二种方案，原因如下：

1. 同样为了容忍 n 台节点的故障，第一种方案需要 2n+1 个副本，而第二种方案只需要 n+1 个副本，而Kafka 的每个分区都有大量的数据，第一种方案会造成大量数据的冗余。 
2. 虽然第二种方案的网络延迟会比较高，但网络延迟对Kafka 的影响较小。

### 2.3.2 ISR

Leader 维护了一个动态的 in-sync replica set (ISR)，意为和 leader 保持同步的 follower 集合。当 ISR 中的 follower 完成数据的同步之后，leader 就会给 follower 发送 ack。如果 follower 长时间未向 leader 同步数据，则该 follower 将被踢出 ISR ，该时间阈值由replica.lag.time.max.ms 参数设定。Leader 发生故障之后，就会从 ISR中选举新的 leader。

### 2.3.3 ack应答机制

对于某些不太重要的数据，对数据的可靠性要求不是很高，能够容忍数据的少量丢失，所以没必要等 ISR中的 follower 全部接收成功。 所以 Kafka 为用户提供了三种可靠性级别，用户根据对可靠性和延迟的要求进行权衡，
选择以下的acks配置：

> 0：producer 不等待 broker 的 ack，这一操作提供了一个最低的延迟，broker 一接收到还没有写入磁盘就已经返回，当 broker 故障时有可能丢失数据；
>
> 1 At Most Once：producer 等待 broker 的 ack，partition 的 leader 落盘成功后返回 ack，如果在 follower 同步成功之前leader 故障，那么将会丢失数据；
>
> -1（all）At Least Once ：producer 等待 broker 的 ack，partition 的 leader 和 follower 全部落盘成功后才返回 ack。但是如果在 follower 同步完成后，broker 发送 ack 之前，leader 发生故障，那么会造成数据重复。

0.11 版本的Kafka，引入了一项重大特性：幂等性。所谓的幂等性就是指 Producer 不论向 Server 发送多少次重复数据，Server 端都只会持久化一条。幂等性结合 At Least Once 语 义，就构成了Kafka 的 Exactly Once 语义。即： 
$$
At Least Once + 幂等性 = Exactly Once
$$

> 要启用幂等性，只需要将 Producer 的参数中 enable.idompotence 设置为true即可。Kafka的幂等性实现其实就是将原来下游需要做的去重放在了数据上游。开启幂等性的Producer在 初始化的时候会被分配一个 PID，发往同一Partition的消息会附带 Sequence Number。而 Broker 端会对<PID, Partition, SeqNumber>做缓存，当具有相同主键的消息提交时，Broker 只会持久化一条。 但是 PID重启就会变化，同时不同的 Partition 也具有不同主键，所以幂等性无法保证跨分区跨会话的 Exactly Once。

### 2.3.4 故障处理细节

![image-20210206001039206](https://gitee.com/adambang/pic/raw/master/20210206001039.png)

**LEO**：指的是每个副本最大的 offset；

**HW**：指的是消费者能见到的最大的 offset，ISR队列中最小的LEO。 

1. follower 故障 follower 发生故障后会被临时踢出 ISR，待该 follower 恢复后，follower 会读取本地磁盘
   记录的上次的HW，并将 log 文件高于HW的部分截取掉，从HW开始向 leader 进行同步。等该 follower 的LEO大于等于该 Partition 的HW，即 follower 追上 leader 之后，就可以重新加入ISR了。

2. leader 故障 leader 发生故障之后，会从 ISR 中选出一个新的 leader，之后，为保证多个副本之间的数据一致性，其余的 follower 会先将各自的 log 文件高于HW的部分截掉，然后从新的 leader 同步数据。 

<u>注意：这只能保证副本之间的数据一致性，并不能保证数据不丢失或者不重复。</u>

## 2.3 消息压缩

在 Kafka 中，压缩可能发生在两个地方：生产者端和 Broker 端。

Producer 端压缩、Broker 端保持（Broker端写入时也会进行解压缩，执行验证）、Consumer 端解压缩。

# 三. Kafka 消费者

consumer 采用 pull（拉）模式从 broker 中读取数据。pull 模式则可以根据 consumer 的消费能力以适当的速率消费消息。 

pull 模式不足之处是，如果 kafka 没有数据，消费者可能会陷入循环中，一直返回空数据。针对这一点，Kafka 的消费者在消费数据时会传入一个时长参数 timeout，如果当前没有 数据可供消费，consumer 会等待一段时间之后再返回，这段时长即为 timeout。

## 3.1 分区分配策略

一个 consumer group 中有多个 consumer，一个 topic 有多个 partition，所以必然会涉及到 partition 的分配问题，即确定那个 partition 由哪个 consumer 来消费。 Kafka 有两种分配策略，一是 RoundRobin，一是Range。

## 3.2 offset 的维护

由于 consumer 在消费过程中可能会出现断电宕机等故障，consumer 恢复后，需要从故障前的位置的继续消费，所以 consumer 需要实时记录自己消费到了哪个offset，以便故障恢复后继续消费。

Kafka 0.9 版本之前，consumer 默认将 offset 保存在 Zookeeper 中，从 0.9 版本开始，consumer 默认将 offset 保存在Kafka 一个内置的 topic 中，该 topic 为__consumer_offsets。

# 四.kafka Broker

一个独立的Kafka 服务器被称为broker。

broker 为消费者提供服务，对读取分区的请求作出响应，返回已经提交到磁盘上的消息。

1. 如果某topic有N个partition，集群有N个broker，那么每个broker存储该topic的一个partition。
2. 如果某topic有N个partition，集群有(N+M)个broker，那么其中有N个broker存储该topic的一个partition，剩下的M个broker不存储该topic的partition数据。
3. 如果某topic有N个partition，集群中broker数目少于N个，那么一个broker存储该topic的一个或多个partition。在实际生产环境中，尽量避免这种情况的发生，这种情况容易导致Kafka集群数据不均衡。

broker 是集群的组成部分。每个集群都有一个broker 同时充当了集群控制器的角色（自动从集群

的活跃成员中选举出来）。

控制器负责管理工作，包括将分区分配给broker 和监控broker。

在集群中，一个分区从属于一个broker，该broker 被称为分区的首领。

![image-20210206233515295](https://gitee.com/adambang/pic/raw/master/20210206233515.png)

## 4.1 优先副本的选举

分区使用多副本机制来提升可靠性，但只有leader副本对外提供读写服务，而 follower 副本只负责在内部进行消息的同步。如果一个分区的 leader 副本不可用，那么就意味着整个分区变得不可用，此时就需要kafka 从剩余的 follower 副本中挑选一个新的 leader 副本来继续对外提供服务。虽然不够严谨，但从某种程度上说， broker 节点中 leader 副本个数的多少决定了这个节点负载的高低。

所谓的**优先副本**的选举是指通过一定的方式促使优先副本选举为 leader 副本，以此来促进集群的负载均衡， 这一行为也可以称为“**分区平衡**” 。leader副本恢复后作为follower副本加入集群。

## 4.2 集群Broker选举

Kafka 集群中有一个 broker 会被选举为 Controller，负责管理集群 broker 的上下线，所有 topic 的分区副本分配和 leader 选举等工作。 Controller 的管理工作都是依赖于 Zookeeper 的。

由于controller是临时节点，当控制器broker挂机之后，就会断开与zookeeper的会话连接，临时节点也会消失，其它节点监听到controller节点消失后，就会重新争取controller节点。

<img src="https://gitee.com/adambang/pic/raw/master/20210207034151.png" alt="image-20210207034151337" style="zoom: 80%;" />

以下为 partition 的 leader 选举过程：

当controller感知到副本leader所在的broker宕机之后，会在当前副本所在的副本列表中选出第一个副本所在的broker作为副本leader，并且要保证这个broker一定要在副本的ISR(存活的副本broker)集合中，如果第一个不存在，则继续尝试第二个第三个，直到满足。

![image-20210207034034689](https://gitee.com/adambang/pic/raw/master/20210207034034.png)

## 4.3 分区重分配

所谓协调者，在 Kafka 中 对应的术语是 Coordinator，它专门为 Consumer Group 服务，负责为 Group 执行 Rebalance 以及提供位移管理和组成员管理等。

Consumer 端应用程序在提交位移时，其实是向 Coordinator 所在的 Broker 提交位移。同样地，当 Consumer 应用启动时，也是向 Coordinator 所在的 Broker 发送 各种请求，然后由 Coordinator 负责执行消费者组的注册、成员管理记录等元数据管理操作。

Rebalance 的触发条件有 3 个：

1. 组成员数发生变更
2. 订阅主题数发生变更
3. 订阅主题的分区数发生变更

在 Rebalance 过程中，**所有 Consumer 实例都会停止消费**，等待 Rebalance 完成。

# 五.kafka 核心

## 5.1 kafka如何保证消息不丢失

### 5.1.1 Producer 端

1. **Producer 选用同步或者异步的发送方式，永远要使用带有回调通知的发送 API**。也就是说不要使用 producer.send(msg)，而要使用 producer.send(msg, callback)。
2. **设置 acks = all（-1）**。acks 是 Producer 的一个参数，代表了你对“已提交”消息的定义。 如果设置成 all，则表明所有副本 Broker 都要接收到消息，该消息才算是“已提交”。 这是最高等级的“已提交”定义。
3. **设置 retries 为一个较大的值。**这里的 retries 同样是 Producer 的参数，对应前面提到 的 Producer 自动重试。当出现网络的瞬时抖动时，消息发送可能会失败，此时配置了retries > 0 的 Producer 能够自动重试消息发送，避免消息丢失。

### 5.1.2 Broker端

1. **设置 unclean.leader.election.enable = false。**这是 Broker 端的参数，它控制的是哪 些 Broker 有资格竞选分区的 Leader。如果一个 Broker 落后原先的 Leader 太多，那么 它一旦成为新的 Leader，必然会造成消息的丢失。故一般都要将该参数设置成 false， 即不允许这种情况的发生。
2. **设置 replication.factor >= 3。**这也是 Broker 端的参数。其实这里想表述的是，最好将 消息多保存几份，毕竟目前防止消息丢失的主要机制就是冗余。
3. **设置 min.insync.replicas > 1**。这依然是 Broker 端参数，控制的是消息至少要被写入到多少个副本才算是“已提交”。设置成大于 1 可以提升消息持久性。在实际环境中千万不要使用默认值 1。
4. **确保 replication.factor > min.insync.replicas**。如果两者相等，那么只要有一个副本挂机，整个分区就无法正常工作了。我们不仅要改善消息的持久性，防止数据丢失，还要在不降低可用性的基础上完成。推荐设置成 replication.factor = min.insync.replicas + 1。

### 5.1.3 Comsumer端

关闭自动提交位移，在消息被完整处理之后再手动提交位移。



