# Redis

## 总体概述

![img](https://gitee.com/adambang/pic/raw/master/20201125142504.jpeg)

“两大维度”就是指系统维度和应用维度，“三大主线”也就是指高性能、高可靠和高可扩展（可以简称为“三高”）。

问题画像图：

![img](https://gitee.com/adambang/pic/raw/master/20201125144844.jpeg)

## 基础概念

Redis本质上是一个 Key-Value 类型的内存数据库,支持String、List、Set、Sorted Set、hashes。

Redis 的通信协议是 Redis 序列化协议，简称 RESP。它具有如下特征：1.在 TCP 层；2.二进制安全；3.基于请求 - 响应模式。

Redis 会将事务中的多个命令一次性、按顺序一次执行，在执行期间可以保证不会中断事务去执行其他命令。Redis 的事务满足一致性和隔离性，但不支持原子性和持久性，事务不会回滚。

### 键值对存储结构

为了实现从键到值的快速访问，Redis 使用了一个哈希表来保存所有键值对。一个哈希表，其实就是一个数组，数组的每个元素称为一个哈希桶。所以，我们常说，一个哈希表是由多个哈希桶组成的，每个哈希桶中保存了键值对数据。

在下图中，可以看到，哈希桶中的 entry 元素中保存了*key和*value指针，分别指向了实际的键和值，这样一来，即使值是一个集合，也可以通过*value指针被查找到。

<img src="https://gitee.com/adambang/pic/raw/master/20201125164041.jpeg" alt="img" style="zoom:40%;" />

### Hash冲突

Redis 解决哈希冲突的方式，就是链式哈希。链式哈希也很容易理解，就是指同一个哈希桶中的多个元素用一个链表来保存，它们之间依次用指针连接。

<img src="https://gitee.com/adambang/pic/raw/master/20201125180302.jpeg" alt="img" style="zoom:40%;" />

如果哈希表里写入的数据越来越多，哈希冲突可能也会越来越多，这就会导致某些哈希冲突链过长，进而导致这个链上的元素查找耗时长，效率降低。

所以，Redis 会对哈希表做 rehash 操作。rehash 也就是增加现有的哈希桶数量，让逐渐增多的 entry 元素能在更多的桶之间分散保存，减少单个桶中的元素数量，从而减少单个桶中的冲突。那具体怎么做呢？

其实，为了使 rehash 操作更高效，Redis 默认使用了两个全局哈希表：哈希表 1 和哈希表 2。一开始，当你刚插入数据时，默认使用哈希表 1，此时的哈希表 2 并没有被分配空间。随着数据逐步增多，Redis 开始执行 rehash，这个过程分为三步：

1. 给哈希表 2 分配更大的空间，例如是当前哈希表 1 大小的两倍；
2. 把哈希表 1 中的数据重新映射并拷贝到哈希表 2 中；
3. 释放哈希表 1 的空间。

> 这个rehash动作并不是一次性、集中式地完成的，而是分多次、渐进式地完成的。
> 这样做的原因在于，如果ht[0]里只保存着四个键值对，那么服务器可以在瞬间就将这些键值对全部rehash到ht[1]；但是，如果哈希表里保存的键值对数量不是四个，而是四百万、四千万甚至四亿个键值对，那么要一次性将这些键值对全部rehash到ht[1]的话，庞大的计算量可能会导致服务器在一段时间内停止服务。
> 因此，为了避免rehash对服务器性能造成影响，服务器不是一次性将ht[0]里面的所有键值对全部rehash到ht[1]，而是分多次、渐进式地将ht[0]里面的键值对慢慢地rehash到ht[1]。
> 以下是哈希表渐进式rehash的详细步骤：
>
> 1. 为ht[1]分配空间，让字典同时持有ht[0]和ht[1]两个哈希表。
> 2. 在字典中维持一个索引计数器变量rehashidx，并将它的值设置为0，表示rehash工作正式开始。
> 3. 在rehash进行期间，每次对字典执行添加、删除、查找或者更新操作时，程序除了执行指定的操作以外，还会顺带将ht[0]哈希表在rehashidx索引上的所有键值对rehash到ht[1]，当rehash工作完成之后，程序将rehashidx属性的值增一。
> 4. 随着字典操作的不断执行，最终在某个时间点上，ht[0]的所有键值对都会被rehash至ht[1]，这时程序将rehashidx属性的值设为-1，表示rehash操作已完成。
>    渐进式rehash的好处在于它采取分而治之的方式，将rehash键值对所需的计算工作均摊到对字典的每个添加、删除、查找和更新操作上，从而避免了集中式rehash而带来的庞大计算量。

### 底层数据结构

1. **字符串**：redis没有直接使用C语言传统的字符串表示，而是自己实现的叫做简单动态字符串SDS的抽象类型。C语言的字符串不记录自身的长度信息，而SDS则保存了长度信息，这样将获取字符串长度的时间由O(N)降低到了O(1)，同时可以避免缓冲区溢出和减少修改字符串长度时所需的内存重分配次数。
2. **链表linkedlist**：redis链表是一个双向无环链表结构，很多发布订阅、慢查询、监视器功能都是使用到了链表来实现，每个链表的节点由一个listNode结构来表示，每个节点都有指向前置节点和后置节点的指针，同时表头节点的前置和后置节点都指向NULL。
3. **字典hashtable**：用于保存键值对的抽象数据结构。redis使用hash表作为底层实现，每个字典带有两个hash表，供平时使用和rehash时使用，hash表使用链地址法来解决键冲突，被分配到同一个索引位置的多个键值对会形成一个单向链表，在对hash表进行扩容或者缩容的时候，为了服务的可用性，rehash的过程不是一次性完成的，而是渐进式的。
4. **跳跃表skiplist**：跳跃表是有序集合的底层实现之一，redis中在实现有序集合键和集群节点的内部结构中都是用到了跳跃表。redis跳跃表由zskiplist和zskiplistNode组成，zskiplist用于保存跳跃表信息（表头、表尾节点、长度等），zskiplistNode用于表示表跳跃节点，每个跳跃表的层高都是1-32的随机数，在同一个跳跃表中，多个节点可以包含相同的分值，但是每个节点的成员对象必须是唯一的，节点按照分值大小排序，如果分值相同，则按照成员对象的大小排序。
5. **整数集合intset**：用于保存整数值的集合抽象数据结构，不会出现重复元素，底层实现为数组。
6. **压缩列表ziplist**：压缩列表是为节约内存而开发的顺序性数据结构，他可以包含多个节点，每个节点可以保存一个字节数组或者整数值。所以，所有可以用压缩列表实现的数据类型都会优先使用压缩列表，数据量大了后再使用别的编码转换。

![img](https://gitee.com/adambang/pic/raw/master/20201125155149.jpeg)

### 数据类型

#### **String**

用途：

> 适用于简单 key-value 存储、setnx key value 实现分布式锁、计数器 (原子性)、分布式全局唯一 ID。

<u>字符串对象的编码可以是int、raw或者embstr。</u>

- 整数值->int编码;

- 字符串值(长度大于32)->raw编码（SDS，两次内存分配）

- 字符串值(长度小于等于32)->embstr（SDS，一次内存分配）


SDS动态字符串，二进制安全，最大512M

1. 开发者不用担心字符串变更造成的内存溢出问题。

2. 常数时间复杂度获取字符串长度len字段。

3. 空间预分配free字段，会默认留够一定的空间防止多次重分配内存。

#### **List**

用途：

> 一般可以用来做简单的消息队列，并且当数据量小的时候可能用到独有的压缩列表来提升性能。当然专业点还是要 [RabbitMQ](https://mp.weixin.qq.com/s?__biz=MzI4NjI1OTI4Nw==&mid=2247488121&idx=1&sn=1ca9adc665b9ba0fc68c2d647b967d7c&scene=21#wechat_redirect)、ActiveMQ 等。

<u>列表对象的编码可以是ziplist或者linkedlist</u>

双向链表上扩展了头、尾节点、元素数等属性

1. 可以直接获得头、尾节点。

2. 常数时间复杂度得到链表长度。

3. 双向链表

#### **hash**

哈希对象的编码可以是ziplist或者hashtable。

ziplist编码的哈希对象使用压缩列表作为底层实现，每当有新的键值对要加入到哈希对象时，程序会先将保存了键的压缩列表节点推入到压缩列表表尾，然后再将保存了值的压缩列表节点推入到压缩列表表尾。

hashtable在数组+链表的基础上，进行了一些rehash优化等

1.Redis的Hash采用链地址法来处理冲突

2.哈希表节点采用单链表结构。

3.rehash优化,渐进式rehash

#### **set**

集合对象的编码可以是intset或者hashtable。

intset->整数集合

无序的自动去重

#### **zset**

zset为有序（有限score排序，score相同则元素字典序），自动去重的集合数据类型，其底层实现为 字典（dict） + 跳表（skiplist），当数据比较少的时候用ziplist编码结构存储。

同时满足以下两个条件采用ziplist存储：

- 有序集合保存的元素数量小于默认值128个
- 有序集合保存的所有元素的长度小于默认值64字节

skiplist编码内部使用 HashMap 和跳跃表（skipList）

HashMap 里放的是成员到 Score 的映射。而跳跃表里存放的是所有的成员

### 内存回收

因为C语言并不具备自动内存回收功能，所以Redis在自己的对象系统中构建了一个引用计数（reference counting）技术实现的内存回收机制，通过这一机制，程序可以通过跟踪对象的引用计数信息，在适当的时候自动释放对象并进行内存回收。

### 集合统计模式

1. **聚合统计**

所谓的聚合统计，就是指统计多个集合元素的聚合结果，包括：统计多个集合的共有元素（交集统计）；把两个集合相比，统计其中一个集合独有的元素（差集统计）；统计多个集合的所有元素（并集统计）。

Set 的差集、并集和交集的计算复杂度较高，在数据量较大的情况下，如果直接执行这些计算，会导致 Redis 实例阻塞。所以，可以从主从集群中选择一个从库，让它专门负责聚合计算，或者是把数据读取到客户端，在客户端来完成聚合统计，这样就可以规避阻塞主库实例和其他从库实例的风险了

2. **排序统计**

在 Redis 常用的 4 个集合类型中（List、Hash、Set、Sorted Set），List 和 Sorted Set 就属于**有序集合**。List 是按照元素进入 List 的顺序进行排序的，而 Sorted Set 可以根据元素的权重来排序，我们可以自己来决定每个元素的权重值。

3. **二值状态统计**

第三个场景：二值状态统计。这里的二值状态就是指集合元素的取值就只有 0 和 1 两种。在签到打卡的场景中，我们只用记录签到（1）或未签到（0），所以它就是非常典型的二值状态。

Bitmap 本身是用 String 类型作为底层数据结构实现的一种统计二值状态的数据类型。String 类型是会保存为二进制的字节数组，所以，Redis 就把字节数组的每个 bit 位利用起来，用来表示一个元素的二值状态。你可以把 Bitmap 看作是一个 bit 数组。

4. **基数统计**

基数统计就是指统计一个集合中不重复的元素个数。

HyperLogLog 是一种用于统计基数的数据集合类型，它的最大优势就在于，当集合元素数量非常多时，它计算基数所需的空间总是固定的，而且还很小。

在 Redis 中，每个 HyperLogLog 只需要花费 12 KB 内存，就可以计算接近 2^64 个元素的基数。你看，和元素越多就越耗费内存的 Set 和 Hash 类型相比，HyperLogLog 就非常节省空间。

不过，有一点需要你注意一下，HyperLogLog 的统计规则是基于概率完成的，所以它给出的统计结果是有一定误差的，标准误算率是 0.81%。这也就意味着，你使用 HyperLogLog 统计的 UV 是 100 万，但实际的 UV 可能是 101 万。虽然误差率不算大，但是，如果你需要精确统计结果的话，最好还是继续用 Set 或 Hash 类型。



各种数据类型支持的场景如下：

<img src="https://gitee.com/adambang/pic/raw/master/20201201153912.png" alt="image-20201201153912012" style="zoom:50%;" />

## 线程模型

Redis 在单线程下还可以支持高并发的一个重要原因就是 Redis 的线程模型：基于非阻塞的IO多路复用机制。(epoll/select/poll)。

Redis 是单线程，主要是指 Redis 的网络 IO 和键值对读写是由一个线程来完成的，这也是 Redis 对外提供键值存储服务的主要流程。但 Redis 的其他功能，比如持久化、异步删除、集群数据同步等，其实是由额外的线程执行的。

redis 内部使用文件事件处理器 file event handler，这个文件事件处理器是单线程的，所以 redis 才叫做单线程的模型。它采用 IO 多路复用机制同时监听多个 socket，根据 socket 上的事件来选择对应的事件处理器进行处理。

### 避免堵塞的异步机制

Redis 实例在运行时，要和许多对象进行交互，这些不同的交互就会涉及不同的操作，下面我们来看看和 Redis 实例交互的对象，以及交互时会发生的操作。

- **客户端**：网络 IO，键值对增删改查操作，数据库操作；

- **磁盘**：生成 RDB 快照，记录 AOF 日志，AOF 日志重写；
- **主从节点**：主库生成、传输 RDB 文件，从库接收 RDB 文件、清空数据库、加载 RDB 文件；
- **切片集群实例**：向其他实例传输哈希槽信息，数据迁移。

 4 类交互对象和具体的操作之间的关系如下图：

<img src="https://gitee.com/adambang/pic/raw/master/20201201164731.jpeg" alt="img" style="zoom: 18%;" />

阻塞点：

- 集合全量查询和聚合操作；
- bigkey 删除；
- 清空数据库；
- AOF 日志同步写；
- 从库加载 RDB 文件

### 文件事件处理器

1. **多个 socket**

2. **IO 多路复用程序**

   Linux 中的 IO 多路复用机制是指一个线程处理多个 IO 流，就是我们经常听到的 select/epoll 机制。简单来说，在 Redis 只运行单线程的情况下，该机制允许内核中，同时存在多个监听套接字和已连接套接字。内核会一直监听这些套接字上的连接请求或数据请求。一旦有请求到达，就会交给 Redis 线程处理，这就实现了一个 Redis 线程处理多个 IO 流的效果。

   下图就是基于多路复用的 Redis IO 模型。图中的多个 FD 就是刚才所说的多个套接字。Redis 网络框架调用 epoll 机制，让内核监听这些套接字。此时，Redis 线程不会阻塞在某一个特定的监听或已连接套接字上，也就是说，不会阻塞在某一个特定的客户端请求处理上。正因为此，Redis 可以同时和多个客户端连接并处理请求，从而提升并发性。

   <img src="https://gitee.com/adambang/pic/raw/master/20201218164503.jpeg" alt="img" style="zoom: 15%;" />

   为了在请求到达时能通知到 Redis 线程，select/epoll 提供了基于**事件的回调机制**，即针对不同事件的发生，调用相应的处理函数。

- select/poll

  单个进程能够监视的文件描述符的数量存在最大限制，通常是1024，当然可以更改数量，但由于select采用轮询的方式扫描文件描述符，文件描述符数量越多，性能越差；

  内核/用户空间内存拷贝问题，select需要复制大量的句柄数据结构，产生巨大的开销；

  select返回的是含有整个句柄的**数组**，应用程序需要遍历整个数组才能发现哪些句柄发生了事件；

  select的触发方式是水平触发，应用程序如果没有完成对一个已经就绪的文件描述符进行IO，那么之后再次select调用还是会将这些文件描述符通知进程。

  相比于select模型，poll使用**链表**保存文件描述符，因此没有了监视文件数量的限制，但其他三个缺点依然存在。

  拿select模型为例，假设我们的服务器需要支持100万的并发连接，则在_FD_SETSIZE为1024的情况下，则我们至少需要开辟1k个进程才能实现100万的并发连接。除了进程间上下文切换的时间消耗外，从内核/用户空间大量的无脑内存拷贝、数组轮询等，是系统难以承受的。因此，基于select模型的服务器程序，要达到10万级别的并发访问，是一个很难完成的任务。

- epoll

  epoll向内核注册了一个文件系统，用于存储所有被监控socket，该epoll文件系统是基于红黑树结构的。
  
  - EPOLLLT和EPOLLET两种触发模式
  - 没有最大并发连接的限制，能打开的FD的上限远大于1024
  - 效率提升，不是轮询的方式，不会随着FD数目的增加效率下降。只有活跃可用的FD才会调用callback函数
  
  > 1.select用数组保存文件描述符，能保存的文件描述符数量有限；
  > 2.select/poll查看文件描述符的状态需要进行轮询，消耗系统资源；
  > 3.select/poll每次调用都需要将全部描述符从应用进程缓冲区复制到内核缓冲区，消耗资源；
  > 4.select/poll事件触发方式是水平触发，就绪的文件描述符如果没有进行io，下次事件还是会重新触发。

3. **文件事件分派器**

4. **事件处理器**

- 连接应答处理器
- 命令请求处理器
- 命令回复处理器

多个 socket 可能会并发产生不同的操作，每个操作对应不同的文件事件，但是 IO 多路复用程序会监听多个 socket，会将 socket 产生的事件放入队列中排队，事件分派器每次从队列中取出一个事件，把该事件交给对应的事件处理器进行处理。

来看客户端与 redis 的一次通信过程：

![为什么 redis 单线程却能支撑高并发？（面试34讲）](https://gitee.com/adambang/pic/raw/master/20201126090644.jpeg)

客户端 socket01 向 redis 的 server socket 请求建立连接，此时 server socket 会产生一个 AE_READABLE 事件，IO 多路复用程序监听到 server socket 产生的事件后，将该事件压入队列中。文件事件分派器从队列中获取该事件，交给连接应答处理器。连接应答处理器会创建一个能与客户端通信的 socket01，并将该 socket01 的 AE_READABLE 事件与命令请求处理器关联。

假设此时客户端发送了一个 set key value 请求，此时 redis 中的 socket01 会产生 AE_READABLE 事件，IO 多路复用程序将事件压入队列，此时事件分派器从队列中获取到该事件，由于前面 socket01 的 AE_READABLE 事件已经与命令请求处理器关联，因此事件分派器将事件交给命令请求处理器来处理。命令请求处理器读取 socket01 的 key value 并在自己内存中完成 key value 的设置。操作完成后，它会将 socket01 的 AE_WRITABLE 事件与命令回复处理器关联。

如果此时客户端准备好接收返回结果了，那么 redis 中的 socket01 会产生一个 AE_WRITABLE 事件，同样压入队列中，事件分派器找到相关联的命令回复处理器，由命令回复处理器对 socket01 输入本次操作的一个结果，比如 ok，之后解除 socket01 的 AE_WRITABLE 事件与命令回复处理器的关联。

## 高并发/可用

### 持久化

Redis 的持久化主要有两大机制，即 AOF（Append Only File）日志和 RDB 快照。

#### AOF

AOF 即 Append-only file：把所有对 Redis 服务器进行修改的命令保存到 aof 文件中，命令的集合。（命令写入/文件同步/文件重写）。

AOF持久化功能的实现可以分为命令追加（append）、文件写入、文件同步（sync）三个步骤。

> **AOF整个流程分两步**：第一步是命令的实时写入，不同级别可能有1秒数据损失。命令先追加到`aof_buf`然后再同步到AO磁盘，**如果实时写入磁盘会带来非常高的磁盘IO，影响整体性能**。
>
> 第二步是对aof文件的**重写**，目的是为了减少AOF文件的大小，可以自动触发或者手动触发(**BGREWRITEAOF**)，是Fork出子进程操作，期间Redis服务仍可用。

AOF 是写后日志，“写后”的意思是 Redis 是先执行命令，把数据写入内存，然后才记录日志，如下图所示：

<img src="https://static001.geekbang.org/resource/image/40/1f/407f2686083afc37351cfd9107319a1f.jpg" alt="img" style="zoom:16%;" />

写后日志这种方式，就是先让系统执行命令，只有命令能执行成功，才会被记录到日志中，否则，系统就会直接向客户端报错。所以，Redis 使用写后日志这一方式的一大好处是，可以**避免出现记录错误命令的情况**。

除此之外，AOF 还有一个好处：**它是在命令执行后才记录日志，所以不会阻塞当前的写操作**。

AOF 也有两个**潜在的风险**。

首先，如果刚执行完一个命令，还没有来得及记日志就宕机了，那么这个**命令和相应的数据就有丢失的风险**。如果此时 Redis 是用作缓存，还可以从后端数据库重新读入数据进行恢复，但是，如果 Redis 是直接用作数据库的话，此时，因为命令没有记入日志，所以就无法用日志进行恢复了。

其次，AOF 虽然避免了对当前命令的阻塞，但**可能会给下一个操作带来阻塞风险**。这是因为，AOF 日志也是在主线程中执行的，如果在把日志文件写入磁盘时，磁盘写压力大，就会导致写盘很慢，进而导致后续的操作也无法执行了。

##### 刷盘策略

这两个风险都是与系统刷盘有关的，AOF 机制提供了三个刷盘策略，也就是 AOF 配置项 appendfsync 的三个可选值。

- Always，同步写回：每个写命令执行完，立马同步地将日志写回磁盘；
- Everysec，每秒写回：每个写命令执行完，只是先把日志写到 AOF 文件的内存缓冲区，每隔一秒把缓冲区中的内容写入磁盘；
- No，操作系统控制的写回：每个写命令执行完，只是先把日志写到 AOF 文件的内存缓冲区，由操作系统决定何时将缓冲区内容写回磁盘。

优缺点如下：

<img src="https://gitee.com/adambang/pic/raw/master/20201126111345.jpeg" alt="img" style="zoom: 25%;" />

##### AOF 重写机制

简单来说，AOF 重写机制就是在重写时，Redis 根据数据库的现状创建一个新的 AOF 文件，也就是说，读取数据库中的所有键值对，然后对每一个键值对用一条命令记录它的写入。

为什么**重写机制可以把日志文件变小**呢? 实际上，重写机制具有“多变一”功能。所谓的“多变一”，也就是说，旧日志文件中的多条命令，在重写后的新日志中变成了一条命令。

AOF 日志由主线程写回不同，重写过程是由后台子进程 bgrewriteaof 来完成的，这也是为了**避免阻塞主线程**，导致数据库性能下降。

<img src="https://gitee.com/adambang/pic/raw/master/20201126112756.jpeg" alt="img" style="zoom:20%;" />

总结来说，每次 AOF 重写时，Redis 会先执行一个内存拷贝，用于重写；然后，使用两个日志保证在重写过程中，新写入的数据不会丢失。而且，因为 Redis 采用额外的线程进行数据重写，所以，这个过程并不会阻塞主线程。

#### RDB内存快照

Redis 的数据都在内存中，为了提供所有数据的可靠性保证，它执行的是全量快照，也就是说，把内存中的所有数据都记录到磁盘中。这样做的好处是，一次性记录了所有数据，一个都不少。

Redis 提供了两个命令来生成 RDB 文件，分别是 save 和 bgsave。

- save：在主线程中执行，会导致阻塞；
- bgsave：创建一个子进程，专门用于写入 RDB 文件，避免了主线程的阻塞，这也是 Redis RDB 文件生成的默认配置。

**处理复制时新产生的数据**

Redis 会借助操作系统提供的写时复制技术（Copy-On-Write, COW），在执行快照的同时，正常处理写操作。

简单来说，bgsave 子进程是由主线程 fork 生成的，可以共享主线程的所有内存数据。bgsave 子进程运行后，开始读取主线程的内存数据，并把它们写入 RDB 文件。

此时，如果主线程对这些数据也都是读操作（例如图中的键值对 A），那么，主线程和 bgsave 子进程相互不影响。但是，如果主线程要修改一块数据（例如图中的键值对 C），那么，这块数据就会被复制一份，生成该数据的副本。然后，bgsave 子进程会把这个副本数据写入 RDB 文件，而在这个过程中，主线程仍然可以直接修改原来的数据。

<img src="https://gitee.com/adambang/pic/raw/master/20201126140544.jpeg" alt="img" style="zoom:20%;" />

这既保证了快照的完整性，也允许主线程同时对数据进行修改，避免了对正常业务的影响。



#### 优缺点

**AOF:**

优点：

1. 相比于 RDB，AOF 更加安全，默认同步策略为每秒同步一次，最差就失去一秒的数据。
2. 根据关注点不同，AOF 提供了不同的同步策略。
3. AOF 文件是以 append-only 方式写入，相比如RDB 全量写入的方式，它没有任何磁盘寻址的开销，写入性能非常高。

缺点：

1. 由于 AOF 日志文件是命令级别的，所以相比于 RDB 紧致的二进制文件而言它的加载速度会慢些。
2. AOF 开启后，支持的写 QPS 会比 RDB 支持的写 QPS 低。

**RDB**

优点：

1. 由于 RDB 文件是一个非常紧凑的二进制文件，所以加载的速度回快于 AOF 方式；
2. fork 子进程(bgsave)方式，不会阻塞

RDB 文件代表着 Redis 服务器的某一个时刻的全量数据，所以它非常适合做冷备份和全量复制的场景

缺点：没办法做到实时持久化，会存在丢数据的风险。定时执行持久化过程，如果在这个过程中服务器崩溃了，则会导致这段时间的数据全部丢失。

触发机制

- 全量复制
- debug reload:进行一个debug级别的重启 不需要清空内存 并且在该过程仍会触发RDB文件的生成
- shutdown:进行关闭的时候会进行 RDB文件的生成

### 事务

redis通过MULTI、EXEC、WATCH等命令来实现事务机制，事务执行过程将一系列多个命令按照顺序一次性执行，并且在执行期间，事务不会被中断，也不会去执行客户端的其他请求，直到所有命令执行完毕。事务的执行过程如下：

1. 服务端收到客户端请求，事务以MULTI开始
2. 如果客户端正处于事务状态，则会把事务放入队列同时返回给客户端QUEUED，反之则直接执行这个命令。这些命令就是 Redis 本身提供的数据读写命令，例如 GET、SET 等。不过，这些命令虽然被客户端发送到了服务器端，但 Redis 实例只是把这些命令暂存到一个命令队列中，并不会立即执行
3. 当收到客户端EXEC命令时，WATCH命令监视整个事务中的key是否有被修改，如果有则返回空回复到客户端表示失败，否则redis会遍历整个事务队列，执行队列中保存的所有命令，最后返回结果给客户端。

WATCH的机制本身是一个CAS的机制，被监视的key会被保存到一个链表中，如果某个key被修改，那么REDIS_DIRTY_CAS标志将会被打开，这时服务器会拒绝执行事务。

**Redis 的事务保证了 ACID 中的一致性（C）和隔离性（I），但并不保证原子性（A）和持久性（D）。**

1.原子性（一个事务所有操作要么全部成功，要么全部失败）

redis的单个命令是原子性的，但是事务中的的redis命令，就算后面的命令失败/终止了（比如KILL进程，主机宕机等），此时事务失败，前面执行的命令也不会回滚。

2.一致性

（1）不正确入队命令的事务不会被执行，不会影响数据库的一致性

（2）命令执行中的错误，将错误包含在事务结果中，不会中断事务，不影响已执行结果，也不影响后面命令的执行，对事务的一致性也没有影响。


### 多机实现

#### 主从复制（保障高并发）

默认情况下，Redis所有节点都是主节点，节点之间互不干涉，而主从复制的节点则是划分了主节点（master）和从节点（slave）。

Redis 提供了主从库模式，以保证数据副本的一致，主从库之间采用的是读写分离的方式。

读操作：主库、从库都可以接收；

写操作：首先到主库执行，然后，主库将写操作同步给从库。

**主从库间如何进行第一次同步？**

当我们启动多个 Redis 实例的时候，它们相互之间就可以通过 replicaof（Redis 5.0 之前使用 slaveof）命令形成主库和从库的关系，之后会按照三个阶段完成数据的第一次同步。

主从同步的原理如下：

1. slave发送sync命令到master
2. master收到sync之后，执行bgsave，生成RDB全量文件
3. master把slave的写命令记录到缓存
4. bgsave执行完毕之后，发送RDB文件到slave，slave执行
5. master发送缓存中的写命令到slave，slave执行

![63d18fd41efc9635e7e9105ce1c33da1](https://gitee.com/adambang/pic/raw/master/20201126144342.jpg)

下图会更加简明清晰：

<img src="https://gitee.com/adambang/pic/raw/master/da902664d2684bdc9fd852c623647d36~tplv-k3u1fbpfcp-zoom-1.image" alt="img" style="zoom: 67%;" />

##### 主从同步问题

###### 主从数据不一致

主从数据不一致，就是指客户端从从库中读取到的值和主库中的最新值并不一致。**因为主从库间的命令复制是异步进行的。**

一方面，主从库间的网络可能会有传输延迟，所以从库不能及时地收到主库发送的命令，从库上执行同步命令的时间就会被延后。

另一方面，即使从库及时收到了主库的命令，但是，也可能会因为正在处理其它复杂度高的命令（例如集合操作命令）而阻塞。此时，从库需要处理完当前的命令，才能执行主库发送的命令操作，这就会造成主从数据不一致。而在主库命令被滞后处理的这段时间内，主库本身可能又执行了新的写操作。这样一来，主从库间的数据不一致程度就会进一步加剧。

**解决方法：**

1. 在硬件环境配置方面，我们要尽量保证主从库间的网络连接状况良好。
2. 我们还可以开发一个外部程序来监控主从库间的复制进度。

###### 读取过期数据

读取到过期数据一般由 Redis 的过期数据删除策略引起的。

Redis 同时使用了两种策略来删除过期的数据，分别是惰性删除策略和定期删除策略。

> 首先，虽然定期删除策略可以释放一些内存，但是，Redis 为了避免过多删除操作对性能产生影响，每次随机检查数据的数量并不多。如果过期数据很多，并且一直没有再被访问的话，这些数据就会留存在 Redis 实例中。业务应用之所以会读到过期数据，这些留存数据就是一个重要因素。
>
> 其次，惰性删除策略实现后，数据只有被再次访问时，才会被实际删除。如果客户端从主库上读取留存的过期数据，主库会触发删除操作，此时，客户端并不会读到过期数据。但是，从库本身不会执行删除操作，如果客户端在从库中访问留存的过期数据，从库并不会触发数据删除。

解决方法：

1.使用 Redis 3.2 及以上版本，避免主库删除过期数据后，从库可以读到过期数据；

2.可以使用 EXPIREAT/PEXPIREAT 命令设置过期时间，避免从库上的数据过期时间滞后。不过，这里有个地方需要注意下，因为 EXPIREAT/PEXPIREAT 设置的是时间点，所以，主从节点上的时钟要保持一致，具体的做法是，让主从节点和相同的 NTP 服务器（时间服务器）进行时钟同步。

###### 不合理配置项导致的服务挂掉

这里涉及到的配置项有两个，分别是 protected-mode 和 cluster-node-timeout。

1.protected-mode 配置项

这个配置项的作用是限定哨兵实例能否被其他服务器访问。当这个配置项设置为 yes 时，哨兵实例只能在部署的服务器本地进行访问。当设置为 no 时，其他服务器也可以访问这个哨兵实例。

2.cluster-node-timeout 配置项

这个配置项设置了 Redis Cluster 中实例响应心跳消息的超时时间。

如果执行主从切换的实例超过半数，而主从切换时间又过长的话，就可能有半数以上的实例心跳超时，从而可能导致整个集群挂掉。所以，我建议你将 cluster-node-timeout 调大些（例如 10 到 20 秒）。

![img](https://gitee.com/adambang/pic/raw/master/9fb7a033987c7b5edc661f4de58ef093.jpg)

#### 哨兵机制

##### **基本流程**

哨兵其实就是一个运行在特殊模式下的 Redis 进程，主从库实例运行的同时，它也在运行。(1).哨兵集群至少要 3 个节点，来确保自己的健壮性。(2).redis主从 + sentinel的架构，是不会保证数据的零丢失的，它是为了保证redis集群的高可用。哨兵主要负责的就是三个任务：**监控、选主（选择主库）和通知**。

1. **监控**是指哨兵进程在运行时，周期性地给所有的主从库发送 PING 命令，检测它们是否仍然在线运行。如果从库没有在规定时间内响应哨兵的 PING 命令，哨兵就会把它标记为“下线状态”；同样，如果主库也没有在规定时间内响应哨兵的 PING 命令，哨兵就会判定主库下线，然后开始自动切换主库的流程。
2. **选主**。 主库挂了以后，哨兵就需要从很多个从库里，按照一定的规则选择一个从库实例，把它作为新的主库。这一步完成后，现在的集群里就有了新主库。
3. **通知**。在执行通知任务时，哨兵会把新主库的连接信息发给其他从库，让它们执行 replicaof 命令，和新主库建立连接，并进行数据复制。同时，哨兵会把新主库的连接信息通知给客户端，让它们把请求操作发到新主库上。

三项任务与目标如下图：

<img src="https://gitee.com/adambang/pic/raw/master/20201126144901.jpeg" alt="img" style="zoom: 20%;" />

另一个角度阐释来说：

功能

1.集群监控，即时刻监控着redis的master和slave进程是否是在正常工作。

2.消息通知，就是说当它发现有redis实例有故障的话，就会发送消息给管理员。

3.故障自动转移

- 1.多个sentinel发现master有问题
- 2.选举一个sentinel作为领导
- 3.选举一个slave作为master
- 4.通知其它slave作为新的master的slave
- 5.通知客户端主从变化
- 6.等待老的master复活成为新的master的slave

4.充当配置中心，如果发生了故障转移，它会通知将master的新地址写在配置中心告诉客户端。

##### 下线判断

哨兵对主库的下线判断有“主观下线”和“客观下线”两种。

哨兵进程会使用 PING 命令检测它自己和主、从库的网络连接情况，用来判断实例的状态。如果哨兵发现主库或从库对 PING 命令的响应超时了，那么，哨兵就会先把它标记为“**主观下线**”

在判断主库是否下线时，不能由一个哨兵说了算，只有大多数的哨兵实例，都判断主库已经“主观下线”了，主库才会被标记为“**客观下线**”

##### 新库选择

一般来说，把哨兵选择新主库的过程称为“筛选 + 打分”。简单来说，我们在多个从库中，先按照一定的筛选条件，把不符合条件的从库去掉。然后，我们再按照一定的规则，给剩下的从库逐个打分，将得分最高的从库选为新主库，如下图所示：

<img src="https://gitee.com/adambang/pic/raw/master/20201126145916.jpeg" alt="img" style="zoom:15%;" />

#### 哨兵集群

##### 基于 pub/sub 机制的哨兵集群组成

哨兵实例之间可以相互发现，要归功于 Redis 提供的 pub/sub 机制，也就是发布 / 订阅机制。哨兵只要和主库建立起了连接，就可以在主库上发布消息了，比如说发布它自己的连接信息（IP 和端口）。同时，它也可以从主库上订阅消息，获得其他哨兵发布的连接信息。当多个哨兵实例都在主库上做了发布和订阅操作后，它们之间就能知道彼此的 IP 地址和端口。然后多个哨兵实例之间就可以相互建立连接。

<img src="https://gitee.com/adambang/pic/raw/master/20201126151113.jpeg" alt="img" style="zoom:20%;" />

**哨兵是如何知道从库的 IP 地址和端口的呢**？这是由哨兵向主库发送 INFO 命令来完成的。就像下图所示，哨兵 2 给主库发送 INFO 命令，主库接受到这个命令后，就会把从库列表返回给哨兵。接着，哨兵就可以根据从库列表中的连接信息，和每个从库建立连接，并在这个连接上持续地对从库进行监控。哨兵 1 和 3 可以通过相同的方法和从库建立连接。

<img src="https://static001.geekbang.org/resource/image/88/e0/88fdc68eb94c44efbdf7357260091de0.jpg" alt="img" style="zoom:16%;" />

##### 基于 pub/sub 机制的客户端事件通知

从本质上说，哨兵就是一个运行在特定模式下的 Redis 实例，只不过它并不服务请求操作，只是完成监控、选主和通知的任务。所以，每个哨兵实例也提供 pub/sub 机制，客户端可以从哨兵订阅消息。哨兵提供的消息订阅频道有很多，不同频道包含了主从库切换过程中的不同关键事件。

<img src="https://static001.geekbang.org/resource/image/4e/25/4e9665694a9565abbce1a63cf111f725.jpg" alt="img" style="zoom:23%;" />

可以让客户端从哨兵这里订阅消息了。具体的操作步骤是，客户端读取哨兵的配置文件后，可以获得哨兵的地址和端口，和哨兵建立网络连接。然后，我们可以在客户端执行订阅命令，来获取不同的事件消息。

哨兵leader选举，在投票过程中，任何一个想成为 Leader 的哨兵，要满足两个条件：第一，拿到半数以上的赞成票；第二，拿到的票数同时还需要大于等于哨兵配置文件中的 quorum 值。以 3 个哨兵为例，假设此时的 quorum 设置为 2，那么，任何一个想成为 Leader 的哨兵只要拿到 2 张赞成票，就可以了。

#### 切片集群

当数据量过大一个主机放不下的时候，就需要对数据进行分区，将key按照一定的规则进行计算，并将key对应的value分配到指定的Redis实例上，这样的模式简称Redis集群。

主从节点的分布式器群，具有复制分片和高可用特性

针对海量数据+高并发+高可用的场景

数据分布算法

- hash算法（大量缓存重建）
- 一致性hash算法（自动缓存迁移）+虚拟节点（自动负载均衡）
- redis cluster的hash slot算法

步骤

1. 将各个独立的节点连接起来，构成一个包含多个节点的集群。

   节点的握手过程：

   > 1. 节点A收到客户端的cluster meet命令
   >
   > 2. A根据收到的IP地址和端口号，向B发送一条meet消息
   >
   > 3. 节点B收到meet消息返回pong
   >
   > 4. A知道B收到了meet消息，返回一条ping消息，握手成功
   >
   > 5. 最后，节点A将会通过gossip协议把节点B的信息传播给集群中的其他节点，其他节点也将和B进行握手


   ![img](https://gitee.com/adambang/pic/raw/master/f0565e88dd304956bb393d43bc819222~tplv-k3u1fbpfcp-zoom-1.image)

2. 集群的整个数据库被分为16384个槽（slot），将槽分配给不同节点。

3. 接收命令的节点会计算出命令要处理的数据库键属于哪个槽，并检查这个槽是否指派给了自己。若为否返回MOVED错误，指引客户端转向（redirect）至正确的节点。

切片集群，也叫分片集群，就是指启动多个 Redis 实例组成一个集群，然后按照一定的规则，把收到的数据划分成多份，每一份用一个实例来保存。回到我们刚刚的场景中，如果把 25GB 的数据平均分成 5 份（当然，也可以不做均分），使用 5 个实例来保存，每个实例只需要保存 5GB 数据。如下图所示：

<img src="https://gitee.com/adambang/pic/raw/master/20201126154948.jpeg" alt="img" style="zoom:16%;" />

Redis Cluster 方案采用哈希槽（Hash Slot，接下来我会直接称之为 Slot），来处理数据和实例之间的映射关系。在 Redis Cluster 方案中，一个切片集群共有 16384 个哈希槽，这些哈希槽类似于数据分区，每个键值对都会根据它的 key，被映射到一个哈希槽中。

具体的映射过程分为两大步：首先根据键值对的 key，按照CRC16 算法计算一个 16 bit 的值；然后，再用这个 16bit 值对 16384 取模，得到 0~16383 范围内的模数，每个模数代表一个相应编号的哈希槽。

**哈希槽分配：**

我们在部署 Redis Cluster 方案时，可以使用 cluster create 命令创建集群，此时，Redis 会自动把这些槽平均分布在集群实例上。例如，如果集群中有 N 个实例，那么，每个实例上的槽个数为 16384/N 个。

当然， 我们也可以使用 cluster meet 命令手动建立实例间的连接，形成集群，再使用 cluster addslots 命令，指定每个实例上的哈希槽个数。

**客户端如何定位数据？**

Redis 实例会把自己的哈希槽信息发给和它相连接的其它实例，来完成哈希槽分配信息的扩散。当实例之间相互连接后，每个实例就有所有哈希槽的映射关系了。

客户端收到哈希槽信息后，会把哈希槽信息缓存在本地。当客户端请求键值对时，会先计算键所对应的哈希槽，然后就可以给相应的实例发送请求了。

但是，在集群中，实例和哈希槽的对应关系并不是一成不变的，最常见的变化有两个：

- 在集群中，实例有新增或删除，Redis 需要重新分配哈希槽；
- 为了负载均衡，Redis 需要把哈希槽在所有实例上重新分布一遍。

实例之间还可以通过相互传递消息，获得最新的哈希槽分配信息，但是，客户端是无法主动感知这些变化的。这就会导致，它缓存的分配信息和最新的分配信息就不一致了。

Redis Cluster 方案提供了一种重定向机制，所谓的“重定向”，就是指，客户端给一个实例发送数据读写操作时，这个实例上并没有相应的数据，客户端要再给一个新实例发送操作命令。

具体细节：接收命令的节点会计算出命令要处理的数据库键属于哪个槽，并检查这个槽是否指派给了自己。若为否返回MOVED错误，指引客户端转向（redirect）至正确的节点。

<img src="https://gitee.com/adambang/pic/raw/master/20201126160623.jpeg" alt="img" style="zoom:16%;" />

## 缓存场景

场景的缓存模式，一般有3种：

- Cache Aside（旁路缓存）：同时更新缓存和数据库

- Read/Write Through（读写穿透）：先更新缓存，缓存负责同步更新数据库
- Write Behind Caching（异步缓存写入）：先更新缓存，缓存定时异步更新数据库

### 缓存的类型

按照 Redis 缓存是否接受写请求，我们可以把它分成只读缓存和读写缓存。

**只读缓存**

当 Redis 用作只读缓存时，应用要读取数据的话，会先调用 Redis GET 接口，查询数据是否存在。而所有的数据写请求，会直接发往后端的数据库，在数据库中增删改。对于删改的数据来说，如果 Redis 已经缓存了相应的数据，应用需要把这些缓存的数据删除，Redis 中就没有这些数据了。

当应用再次读取这些数据时，会发生缓存缺失，应用会把这些数据从数据库中读出来，并写到缓存中。这样一来，这些数据后续再被读取时，就可以直接从缓存中获取了，能起到加速访问的效果。

只读缓存直接在数据库中更新数据的**好处**是，所有最新的数据都在数据库中，而数据库是提供数据可靠性保障的，这些数据不会有丢失的风险。当我们需要缓存图片、短视频这些用户只读的数据时，就可以使用只读缓存这个类型了。

**读写缓存**

对于读写缓存来说，除了读请求会发送到缓存进行处理（直接在缓存中查询数据是否存在)，所有的写请求也会发送到缓存，在缓存中直接对数据进行增删改操作。此时，得益于 Redis 的高性能访问特性，数据的增删改操作可以在缓存中快速完成，处理结果也会快速返回给业务应用，这就可以提升业务应用的响应速度。

但是，和只读缓存不一样的是，在使用读写缓存时，最新的数据是在 Redis 中，而 Redis 是内存数据库，一旦出现掉电或宕机，内存中的数据就会丢失。这也就是说，应用的最新数据可能会丢失，给应用业务带来风险。

所以，根据业务应用对数据可靠性和缓存性能的不同要求，我们会有同步直写和异步写回两种策略。其中，同步直写策略优先保证数据可靠性，而异步写回策略优先提供快速响应。学习了解这两种策略，可以帮助我们根据业务需求，做出正确的设计选择。如下图：

<img src="https://gitee.com/adambang/pic/raw/master/009d055bb91d42c28b9316c649f87f66.jpg" alt="img" style="zoom: 20%;" />

### 缓存一致性

删除更新失败产生的缓存一致性问题场景如下：

<img src="https://gitee.com/adambang/pic/raw/master/20201202100314.jpeg" alt="img" style="zoom: 18%;" />

解决方法可以采用重试机制：

具体来说，可以把要删除的缓存值或者是要更新的数据库值暂存到消息队列中（例如使用 Kafka 消息队列）。当应用没有能够成功地删除缓存值或者是更新数据库值时，可以从消息队列中重新读取这些值，然后再次进行删除或更新。

如果能够成功地删除或更新，我们就要把这些值从消息队列中去除，以免重复操作，此时，我们也可以保证数据库和缓存的数据一致了。否则的话，我们还需要再次进行重试。如果重试超过的一定次数，还是没有成功，我们就需要向业务层发送报错信息了。

两种缓存更新机制并发问题分情况如下：

**情况一：先删除缓存，再更新数据库。**

假设线程 A 删除缓存值后，还没有来得及更新数据库（比如说有网络延迟），线程 B 就开始读取数据了，那么这个时候，线程 B 会发现缓存缺失，就只能去数据库读取。这会带来两个问题：

1. 线程 B 读取到了旧值；
2. 线程 B 是在缓存缺失的情况下读取的数据库，所以，它还会把旧值写入缓存，这可能会导致其他线程从缓存中读到旧值。

<img src="https://gitee.com/adambang/pic/raw/master/20201202102605.jpeg" alt="img" style="zoom:16%;" />

解决方法为“延时双删”：

> 在线程 A 更新完数据库值以后，我们可以让它先 sleep 一小段时间，再进行一次缓存删除操作。

伪代码如下：

```
redis.delKey(X)
db.update(X)
Thread.sleep(N)
redis.delKey(X)
```

**情况二：先更新数据库值，再删除缓存值。**

如果线程 A 删除了数据库中的值，但还没来得及删除缓存值，线程 B 就开始读取数据了，那么此时，线程 B 查询缓存时，发现缓存命中，就会直接从缓存中读取旧值。

不过，在这种情况下，如果其他线程并发读缓存的请求不多，那么，就不会有很多请求读取到旧值。而且，线程 A 一般也会很快删除缓存值，这样一来，其他线程再次读取时，就会发生缓存缺失，进而从数据库中读取最新值。所以，这种情况对业务的影响较小。

总结：

<img src="https://gitee.com/adambang/pic/raw/master/20201202104116.png" alt="image-20201202104116187" style="zoom: 50%;" />

### 缓存异常

#### 缓存雪崩

缓存雪崩是指缓存中数据大批量到过期时间，而查询数据量巨大，引起数据库压力过大甚至down机。（Redis崩溃导致的缓存失效也包括在内）

解决方法

- 事前

  1.缓存数据的过期时间设置随机，防止同一时间大量数据过期现象发生。

  2.如果缓存数据库是分布式部署，将热点数据均匀分布在不同Redis库中。

  3.redis高可用，主从+哨兵，redis cluster，避免全盘崩溃

- 事中

  本地ehcache缓存 + hystrix限流&降级，避免MySQL被打死

- 事后

  redis持久化，快速恢复缓存数据

#### 缓存击穿

缓存击穿是指缓存中没有但数据库中有的数据（热点key），这时由于并发用户特别多，同时读缓存没读到数据，又同时去数据库去取数据，引起数据库压力瞬间增大，造成过大压力。

解决方法：1.设置热点数据永远不过期。2.加互斥锁。

#### 缓存穿透

缓存穿透是指缓存和数据库中都没有的数据，而用户不断发起请求 。

解决方法

- 1.接口层增加校验，如用户鉴权校验，id做基础校验，id<=0的直接拦截；
- 2.空值设置缓存，key-null；
- 3.布隆过滤器。

<img src="https://gitee.com/adambang/pic/raw/master/20201202105524.jpeg" alt="img" style="zoom: 20%;" />

### 过期策略

**定时删除**

**定期删除**

- redis 默认是每隔 100ms 就随机抽取一些设置了过期时间的 key，检查其是否过期，如果过期就删除。

**惰性删除**

- 获取 key 的时候，如果此时 key 已经过期，就删除，不会返回任何东西

### 内存淘汰机制

内存淘汰机制如下图：

![img](https://gitee.com/adambang/pic/raw/master/20201202094231.jpeg)

- allkeys-lru：当内存不足以容纳新写入数据时，在键空间中，移除最近最少使用的 key
- allkeys-random：当内存不足以容纳新写入数据时，在键空间中，随机移除某个 key
- volatile-lru：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，移除最近最少使用的 key
- volatile-random：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，随机移除某个 key
- volatile-ttl：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，有更早过期时间的 key 优先移除。



```java
class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int CACHE_SIZE;

/**

 * 传递进来最多能缓存多少数据

 * @param cacheSize 缓存大小

 */
    public LRUCache(int cacheSize) {
        // true 表示让 linkedHashMap 按照访问顺序来进行排序，最近访问的放在头部，最老访问的放在尾部。

        super((int) Math.ceil(cacheSize / 0.75) + 1, 0.75f, true);

        CACHE_SIZE = cacheSize;

}

    @Override

    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {

    // 当 map中的数据量大于指定的缓存个数的时候，就自动删除最老的数据。

    return size() > CACHE_SIZE;

    }
}
```

### 并发访问

#### **无锁的原子操作**

为了实现并发控制要求的临界区代码互斥执行，Redis 的原子操作采用了两种方法：

1. 把多个操作在 Redis 中实现成一个操作，也就是单命令操作；

> 虽然 Redis 的单个命令操作可以原子性地执行，但是在实际应用中，数据修改时可能包含多个操作，至少包括读数据、数据增减、写回数据三个操作，这显然就不是单个命令操作了。
>
> 对此，Redis 提供了 INCR/DECR 命令，把这三个操作转变为一个原子操作了。INCR/DECR 命令可以对数据进行增值 / 减值操作，而且它们本身就是单个命令操作，Redis 在执行它们时，本身就具有互斥性。

2. 把多个操作写到一个 Lua 脚本中，以原子性方式执行单个 Lua 脚本。

> Redis 会把整个 Lua 脚本作为一个整体执行，在执行的过程中不会被其他命令打断，从而保证了 Lua 脚本中操作的原子性。

#### **分布式锁**

**基于单个 Redis 节点实现分布式锁**

可以用 SETNX 和 DEL 命令组合来实现加锁和释放锁操作。下面的伪代码示例显示了锁操作的过程。

```
// 加锁
SETNX lock_key 1
// 业务逻辑
DO THINGS
// 释放锁
DEL lock_key
```

上述如果加锁后系统异常，锁会无法释放。

解决方法：给锁加一个过期时间。（jedis有原子操作的函数）

在基于单个 Redis 实例实现分布式锁时，对于加锁操作，我们需要满足三个条件。

- 加锁包括了读取锁变量、检查锁变量值和设置锁变量值三个操作，但需要以原子操作的方式完成，所以，我们使用 SET 命令带上 NX 选项来实现加锁；
- 锁变量需要设置过期时间，以免客户端拿到锁后发生异常，导致锁一直无法释放，所以，我们在 SET 命令执行时加上 EX/PX 选项，设置其过期时间；
- 锁变量的值需要能区分来自不同客户端的加锁操作，以免在释放锁时，出现误释放操作，所以，我们使用 SET 命令设置锁变量值时，每个客户端设置的值是一个唯一值，用于标识客户端。

**基于多个 Redis 节点实现高可靠的分布式锁**

当我们要实现高可靠的分布式锁时，就不能只依赖单个的命令操作了，我们需要按照一定的步骤和规则进行加解锁操作，否则，就可能会出现锁无法工作的情况。“一定的步骤和规则”是指啥呢？其实就是分布式锁的算法。

为了避免 Redis 实例故障而导致的锁无法工作的问题，Redis 的开发者 Antirez 提出了分布式锁算法 Redlock。

Redlock 算法的基本思路，是让客户端和多个独立的 Redis 实例依次请求加锁，如果客户端能够和半数以上的实例成功地完成加锁操作，那么我们就认为，客户端成功地获得分布式锁了，否则加锁失败。这样一来，即使有单个 Redis 实例发生故障，因为锁变量在其它实例上也有保存，所以，客户端仍然可以正常地进行锁操作，锁变量并不会丢失。

具体看下 Redlock 算法的执行步骤。Redlock 算法的实现需要有 N 个独立的 Redis 实例。接下来，我们可以分成 3 步来完成加锁操作。

1. 客户端获取当前时间。
2. 客户端按顺序依次向 N 个 Redis 实例执行加锁操作。
3. 一旦客户端完成了和所有 Redis 实例的加锁操作，客户端就要计算整个加锁过程的总耗时。

> 客户端只有在满足下面的这两个条件时，才能认为是加锁成功。
>
> 条件一：客户端从超过半数（大于等于 N/2+1）的 Redis 实例上成功获取到了锁；
>
> 条件二：客户端获取锁的总耗时没有超过锁的有效时间。

