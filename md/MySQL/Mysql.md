# Mysql

## Mysql基础架构

### Server 层

- 连接器

	- 连接池

- 查询缓存

	- 查询缓存的失效非常频繁，只要有对一个表的更新，这个表上所有的查询缓存都会被清空。(Mysql8.0直接删除掉功能)

- 分析器

	- 词法分析/语法分析，生成“解析树”

- 优化器

	- 生成执行计划

- 执行器

### 存储引擎

- InnoDB、MyISAM、Memory
- MyISAM和InnoDB

### 日志系统

- Error log

	- 默认开启，show variables like "%log_error%"

- General query log

	- 记录一般查询语句，show variables like "%general%"

- show query log

	- 记录所有执行时间超时的sql，默认十秒

- bin log(服务层)

	- 只记录修改语句，用于数据库恢复和主从复制，默认关闭
	- show variables like "%log_bin%"
	- 记录模式

	- 写入机制

- redo log(innodb引擎)

	- 物理日志，记录的是数据页的物理修改;事务开始之前，生成log,事务结束后数据写入磁盘后，log可重写
- undo log

	- 逻辑日志，事务开始之前，修改的记录存到undo log
	- 作用

		- 实现事务的原子性
		- 实现多版本并发控制（MVCC）

	- 采用段(segment)的方式来记录的，每个undo操作在记录的时候占用一个undo log segment

## 事务

### 原子性(atomicity)

- 事务是一个完整的操作，各步操作是不可分的（原子的）

### 一致性(consistency)

- 当事务完成时，数据必须处于一致状态。

### 隔离性(isolation)

- 读未提交：一个事务还没提交时，它做的变更就能被别的事务看到
- 读提交：一个事务提交之后，它做的变更才会被其他事务看到
- 可重复读：一个事务执行过程中看到的数据，总是跟这个事务在启动时看到的数据一致。当然未提交变更对其他事务也是不可见的
- 串行化：读写锁串行

### 持久性(durability)

- 事务完成后，它对数据库的修改被永久保持，事务日志能够保持事务的永久性。

### 事务问题

- 脏读（读未提交）：一个事务读取到另外一个事务还没有提交的数据
- 不可重复读（读提交）：在同一个事务内，两次相同的查询返回了不同的结果（修改）
- 幻读（可重复读）：同一个事务内多次查询返回的结果集不一样（新增/删除）

## 锁

### 全局锁

- 对整个数据库实例加锁。
MySQL提供加全局读锁的方法：Flush tables with read lock(FTWRL)
- 使用场景：全库逻辑备份。
- 风险

	- 如果在主库备份，在备份期间不能更新，业务停摆
	- 如果在从库备份，备份期间不能执行主库同步的binlog，导致主从延迟

### 表级锁

- 表锁

	- 表锁的语法是 lock tables … read/write
	- 可以用unlock tables主动释放锁，也可以在客户端断开时自动释放

- 元数据锁（MDL)

	- MDL 不需要显式使用，在访问一个表的时候会被自动加上
	- 读读不互斥，读写/写写互斥

### 行级锁

- InnoDB支持，MyISAM不支持
-  InnoDB 事务中，行锁在需要时才加上的，但并不是不需要了就立刻释放，而是要等到事务结束时才释放。这个就是两阶段锁协议
- 事务中需要锁多个行，要把最可能造成锁冲突、最可能影响并发度的锁尽量往后放
- 死锁检测

	- 直接进入等待，直到超时。这个超时时间可以通过参数 innodb_lock_wait_timeout 来设置。（等待时间过长，不推荐）
	- 发起死锁检测，发现死锁后，主动回滚死锁链条中的某一个事务，让其他事务得以继续执行。将参数 innodb_deadlock_detect 设置为 on，表示开启这个逻辑。

## 索引

1. create index inde_name  on table (column)

2. alter table add index index_name on column

3. show index from table 	

### 概念

- 索引是什么

	- MySQL官方对索引的定义为：索引(Index)是帮助MySQL高效获取数据的数据结构。
可以得到索引的本质：索引是数据结构

	- 你可以简单理解为"索引是排好序的用于快速查找数据的数据结构"。
所以，索引会影响SQL语句中的where查找和order by 排序。

		- 详解

				- 特定查找算法：B树索引

					- 类似于折半查找，如果我们想要找72这个数字：
首先判断72是在30左边，还是30右边。
然后再进一步判断，是在40左边，还是在40右边。就这样一层一层的查找。

				- 说明

					- 索引是如何加快Col2的查找的？
比如查找91：select * from xxx where Col2=91
首先判断，91是在34左边还是右边，因为91是在34右边，因此找到了89；
然后判断91是在89的左边还是右边，因为91是在89右边，因此就找到了91。 
然后拿着91这个节点对应的物理地址，从而就找到了Col2中91对应的数据。
					- 左边是数据表，这个表在内存中，所以它对应的有物理地址。最右边的二叉树是这个表对应的索引。每个索引树上的节点都分别对应着数据表中的一条数据和该条数据的物理地址。
					- 如果对数据表里面的数据进行了修改或者删除，那么最右边的索引树种，有的节点就会失效。此时就需要重建索引。

		- 结论

			- 数据本身之外，数据库还维护着一个满足特定查找算法的数据结构，这些数据结构以某种方式指向数据，
这样就可以在这些数据结构的基础上实现高级查找算法，这种数据结构就是索引。

	- 索引存储位置

		- 一般来说索引本身也很大，不可能全部存储在内存中，因此索引往往以文件形式存储在硬盘上

	- 索引分类

		- 我们平时所说的索引，如果没有特别指明，都是指B树(多路搜索树，并不一定是二叉树)结构组织的索引。

其中聚集索引，次要索引，覆盖索引，复合索引，前缀索引，唯一索引默认都是使用B+树索引，统称索引。
当然,除了B+树这种类型的索引之外，还有哈希索引(hash index)等。

	- 一句话：优化SQL首先优化索引。索引对SQL性能的提高非常非常大。

- 优势

	- 类似大学图书馆建书目索引，提高数据检索效率，降低数据库的IO成本
	- 通过索引列对数据进行排序，降低数据排序成本，降低了CPU的消耗

- 劣势

	- 实际上索引也是一张表，该表保存了主键和索引字段，并指向实体表的记录，所以索引列也是要占用硬盘空间的。
	- 虽然索引大大提高了查询速度，同时却会降低更新表的速度,如果对表INSERT,UPDATE和DELETE。因为更新表时，MySQL不仅要不存数据，还要保存一下索引文件每次更新添加了索引列的字段，都会调整因为更新所带来的键值变化后的索引信息
	- 索引需要额外的维护成本；因为索引文件是单独存在的文件,对数据的增加,修改,删除,都会产生额外的对索引文件的操作,这些操作需要消耗额外的IO,会降低增/改/删的执行效率;
	- 索引只是提高效率的一个因素，如果你的MySQL有大数据量的表，就需要花时间研究建立优秀的索引，或优化查询语句

		- 注意：优秀的索引是需要程序员或者DBA花时间研究才能建立的，删了建，建了删，逐步优化出来的。

### 分类

- 单值索引
- 唯一索引
- 复合索引

	- 覆盖索引

	  查询的列刚好与创建的索引列的列名及顺序全部匹配或者部分匹配
	  
	  如索引 index_col1_col2  而查询的字段有
	  col1, col2 , col3 此时则不匹配

- 基本语法

	- 创建

		- 方式一：CREATE [UNIQUE] INDEX  indexName ON mytable(columnname(length));

			- 如果是CHAR,VARCHAR类型，length可以小于字段实际长度；
如果是BLOB和TEXT类型，必须指定length。
			-  [UNIQUE] ：中括号代表可以省略。如果加上这个字段，代表创建唯一索引。
			- 如果table后面只写了一个字段，就是单值索引，如果写了多个字段，就是复合索引。

		- 方式二：ALTER mytable ADD [UNIQUE]  INDEX [indexName] ON(columnname(length));

	- 删除

		- DROP INDEX [indexName] ON mytable;

	- 查看

		- SHOW INDEX FROM table_name;

	- 使用ALTER命令

### 索引结构

- B树索引
- Hash索引
- 全文索引
- R-Tree 索引

### 数据结构

- 二叉查找树

	- 查找耗时与树的深度相关的，最坏时间复杂度会退化成O(n)

- 平衡二叉树

	- 左右子树深度差绝对值不能超过 1
	- 插入/删除节点，需要频繁调整树结构，性能较差
	- 树的深度过深，磁盘IO开销较大

- 红黑树

	- 叶子节点全黑，一红带两黑，弱平衡性
	- 适用于删除插入操作较多的情况

- B树

	- 叶子节点，非叶子节点，都存储数据
	- 中序遍历，可以获得所有节点

- B+树

	- 非叶子节点不再存储数据，数据只存储在同一层的叶子节点上
	- 叶子之间，增加了链表，获取所有节点，不再需要中序遍历

### 性能分析

- mysql 查询优化器
- 常见瓶颈

	- CPU
	- IO

- explain

	- 能做什么

		- 表的读取顺序
		- 数据读取操作的操作类型
		- 可能用到哪些索引	
		- 实际用到哪些索引
		- 表之间的引用
		- 每张表有多少行被优化器查询

	- 列名解析

		- id

		  select 查询的序列号，包含一组数字， 表示查询中执行select 子句或操作表的顺序	

			- id相同

			  执行顺序从上到下

			- id 不同

			  如果是子查询，ID的序号会递增，ID值越大优先级越高，越先被执行
			  
- ID相同又不同
			
  序号大的先执行，序号相同的顺序执行
			  
			
	
- select_type
		
  查询的类型
		
	- Simple
		
	  简单的select查询，不包含子查询或者union查询
			  
		- Primary
	
		  最外层查询称为主查询
	
		- SubQuery
	
		  在select或where列表中包含子查询
	  
			- Derived
		
	  在from列表中包含的子查询被标记为derived(衍生)
		  mysql会递归这些子查询，将结果放到临时表中
	
			- Union
		
	  若第二个select出现在union之后，则被标记为union
		  若union包含在from子句的子查询中，外层select被标记为 DERIVED(衍生)
	
			- Union result
		
	  从union表获取结果的select
	
- table
		- type

		  性能从高到低：
		  system > const > equ_ref > ref > fulltext > ref_or_null > index_merge > unique_subquery > index_subquery > range > index > all

			- system
		
	  表只有一行记录，是const 类型特例
	
- const
			
		  表示通过一次索引就能找到，const用于primary key或者unique索引
	
		- equ_ref
		
		  唯一性索引扫描，对于每个索引键，表中只有一条记录与之匹配，常见于主键或唯一索引扫描
		
		- ref
		
		  非唯一性索引扫描，返回匹配某个单独值的所有行，本质上也是一种索引访问，它返回所有匹配某个单独值的行
		
		- range
		
		  只检索给定范围的行，使用一个索引来选择行。key列显示使用列哪个索引，一般在where子句中出现 between、> 、<、 in等的查询。
	  这种范围扫描索引的扫描方式比全表扫描好，因为它只需要开始于索引的某一个点，而结束于某一个点，而不用扫描全部索引
	
- index
			
		  全索引扫描
	
		- all
		
		  全表扫描
		
		- possible_keys
	
		  显示可能应用在这张表的索引，一个或多个。
  查询涉及到的字段上若存在索引，则该索引将被列出，但不一定被使用到
	
- key
		
		  实际使用的索引。
  若查询中使用列覆盖索引，则该索引仅出现在key列表中
	
- key_len
		
		  索引中使用的字节数，可通过该列计算查询中使用的索引长度。在不损失精确性的情况下，长度越短越好。
  
		  key_len显示的值为索引字段的最大可能长度，并非实际使用长度，即key_len是表定义计算而得，不是通过表内检索出
	
		- ref
		
		  显示索引的哪一列被使用，如果可能的话，是一个常数const
	
		- rows
	
		  根据表统计信息即索引选用情况，大致估算出找到记录所需要读取的行数
	
		- Extra
	
			- Using filesort
	
			  表示mysql会对数据使用一个外部的索引排序，而不是按照表内的索引顺序进行读取。
	  mysql无法利用索引完成的排序操作称为文件排序。
			  出现该情况时表示查询性能已经很差

			- Using temporary
		
			  使用列临时表来保存中间结果，常见与排序 order by 及分组 group by 查询.
	  性能最差，需要优化
		
	- Using index
		
			  表示相应的查询使用到了覆盖索引（Convering index）,避免访问表的数据行，效率不错。
	  如果同时出现using where , 表明索引被用来执行索引键值的查找；
			  如果没有同时出现using where，表明索引用来读取数据而非查询动作
	  
		- using where
			- using join buffer
			- impossible where
			
  
	如  where name='jim' and name ='jack'
			
		- select table optimized away
			- distinct

### 索引创建场景

- 哪些情况需要创建索引

	- 1.主键自动建立唯一索引
	- 2.频繁作为查询的条件的字段应该创建索引
	- 3.查询中与其他表关联的字段，外键关系建立索引
	- 4.频繁更新的字段不适合创建索引

		- 因为每次更新不单单是更新了记录还会更新索引，加重IO负担

	- 5.Where条件里用不到的字段不创建索引
	- 6.单一索引/复合索引的选择问题，平时选择哪一个？who？（在高并发下倾向创建组合索引）
	- 7.查询中排序的字段，排序字段若通过索引去访问将大大提高排序的速度

		- 比如你创建复合索引（name,age,address），那么排序的时候如果还按照name,age,address排序，速度会非常快。

	- 8.查询中统计或者分组字段

- 哪些情况不要创建索引

	- 1.表记录太少

		- mysql虽然官方说能撑得住500到800万，但是实际上，300万条数据，性能就开始下降。

	- 2.经常增删改的表
	- 3.数据重复且分布平均的表字段，因此应该只为经常查询和经常排序的数据列建立索引。 

		- 比如国籍，省市县，男女，这样的数据重复率高，这样的就不适合建索引。
		- 注意，如果某个数据列包含许多重复的内容，为它建立索引就没有太大的实际效果。

### 查询截取分析

- 查询优化

	- 永远是小表驱动大表

	  类似嵌套循环
	  
	  for(int i=5...){
	     for(int j = 10000...){
	        ....
	     }
	  }
	  
	  for(int i=10000...){
	     for(int j = 5...){
	        ....
	     }
	  }

	- order by

	  尽量使用索引排序，避免使用文件内排序 file sort。
	  
	  order by 满足两种情况会使用到索引
	  1. order by 子句使用到索引最左前列
	  2. 使用where子句与order by子句条件列组合满足索引最左前列	

		- 若排序的字段不在索引列上，
则mysql会启用两种filesort算法

		  在sort_buffer中，单路比多路要占用更多的空间，因为单路是把所有字段都取出，故取出的数据总大小超出了sort_buffer的容量，导致每次只能取sort_buffer容量大小的数据进行排序（创建临时表，多路合并），排序完后再取同样容量大小的数据再排序，如此循环...从而导致多次IO
		  
		  

			- 双路排序

			  mysql4.1之前使用的是双路排序，即两次扫描磁盘，最终得到数据。
			  读取行指针和orderby列，对他们进行排序，然后扫描已经排序好的列表，按照列表中的值重新从列表读取

			- 单路排序

			  从磁盘读取查询所需要的列，按照order by 列在buffer对他们进行排序，然后扫描排序后的列表对他们进行输出，它的效率更快一些，避免二次读取数据，并且把随机io变为顺序io , 但它会使用更多空间

		- 优化策略

		  1. 不要使用select *， 这是大忌，它可能导致两种结果：
		        1.1 当查询的字段大小总和小于max_length_for_sort_data而且排序字段不是text|blob时，会用改进后的算法--单路排序，否则使用多路排序
		         1.2 两种算法的数据都可能超出sort_buffer的容量，超出后会创建临时表进行合并排序，导致多次io,但是单路算法风险更大，所以要增加 sort_buffer_size参数的值
		  
		  2. 增大 sort_buffer_size
		  
		  3. 尝试提高 max_length_for_sort_data
		     该参数会增加改进算法的概率

			- 增大sort_buffer_size参数
			- 增大 max_length_for_sort_data 参数

	- group by

	  几乎与order by 原理一致
	  实质是先排序后分组

- 慢查询日志

  查看是否已经开启：
  
  show variables	 like '%slow_query_log%'；
  
  慢查询记录总数：
  
  show variables	 like '%slow_queries	%'；

	- 开启  set global slow_query_log=1;

	  只对本次有效，重启数据库后无效

	- 阀值 long_query_time

	  show variables	 like '%long_query_time%';
	  
	  // 设置慢查询时间阀值，需要重新登陆会话才可以看到结果
	  set global long_query_time=3;

	- 日志分析工具 mysqldumpslow

		- 相关参数

			- s 按照何种方式排序
			- c 访问次数
			- l 锁定时间
			- r 返回记录
			- t 查询时间
			- al 平均锁定时间
			- ar 平均返回记录数
			- at 平均查询时间
			- g 后面搭配一个正则表达式

- show profile

  show profile 相关参数：

  

  type： 

  

  all ：显示所有信息

  block io ： io相关开销

  context switches ：  上下文切换相关开销

  cpu ： CPU

  IPC ： 显示发送和接收相关开销

  memory： 内存开销

  page faults： 页面错误相关开销

  source  ：显示和source_function, source_file, source_line 相关开销

  

  swaps ： 显示交换次数相关开销

	- 分析步骤

		- 1. 查看数据库是否支持

		  show variables	 like '%profiling%'；
		  
		  set profiling = on;

		- 2. 运行 sql
		- 3.  show profiles; 查看执行的语句
		- 4. 诊断 sql , show profile cpu , block io for query N

		  show profile 相关参数：
		  
		  type： 
		  
		  all 显示所有信息
		  block io , io相关开销
		  context switches  上下文切换相关开销
		  cpu 
		  IPC  显示发送和接收相关开销
		  memory 内存开销
		  page faults 页面错误相关开销
		  source  显示和source_function, source_file, source_line 相关开销
		  
		  swaps  显示交换次数相关开销

	- 需要注意的结论

		- converting heap to myisam  
表示查询结果太大，内存不够用改用磁盘
		- creating tmp table 
		- copying to tmp table on disk 
把内存中的临时表复制到磁盘
		- locked

- 批量插入数据脚本

	- 建表

	  - dept表

	    CREATE TABLE `test`.`dept`  (
	      `id` int(32) NOT NULL AUTO_INCREMENT,
	    	deptno MEDIUMINT UNSIGNED not null default 0,
	    dname varchar(50) not null default '',
	    loc VARCHAR(13) not null DEFAULT '',
	    create_time datetime(3) ,
	      PRIMARY KEY (`id`) USING BTREE
	    ) ENGINE = InnoDB AUTO_INCREMENT = 1 CHARACTER SET = utf8 
	    COLLATE = utf8_general_ci ROW_FORMAT = Dynamic;

	  - emp表

	    CREATE TABLE `test`.`emp`  (

	      `id` int(32) NOT NULL AUTO_INCREMENT,

	    	empno MEDIUMINT UNSIGNED not null default 0,
    		
	    	deptno MEDIUMINT UNSIGNED not null default 0,
    		
	    	mgr MEDIUMINT UNSIGNED not null default 0,
    		
	    	ename varchar(20) not null default '',
    		
	    	job varchar(20) not null default '',
    		
	    	hiredate date ,
    		
	    	sal DECIMAL(7,2) default 0,
    		
	    	comm DECIMAL(7,2) default 0,
    		
	    	create_time datetime(3) ,

	
    ​	
	
    	email VARCHAR(30) default '',
	    	
    	mobile VARCHAR(20) DEFAULT '',
	    	
    	weight int DEFAULT 0,
	    	
    	married int DEFAULT 0,
	    	
    	pwd VARCHAR(20) DEFAULT '123456',
	

	    	
    
	      PRIMARY KEY (`id`) USING BTREE
    
	    ) ENGINE = InnoDB AUTO_INCREMENT = 1 CHARACTER SET = utf8 
    
	    COLLATE = utf8_general_ci ROW_FORMAT = Dynamic;

	  - 订单表

	    

	    CREATE TABLE `test`.`order`  (

	      `id` int(32) NOT NULL AUTO_INCREMENT,

	    	pay_status int not null default 0,
  		
	    	trade_status int not null default 0,

	      order_no VARCHAR(30) not null default '',

	      mobile VARCHAR(20) not null default '',

	      user_id int not null default 0,

	    	total DECIMAL(20,2) not null default 0,

	      real_total DECIMAL(20,2) not null default 0,

	      channel_id int not null default 0,

	      open_id VARCHAR(50) not null default '',

	      union_id VARCHAR(50) not null default '',

	      transaction_no VARCHAR(50) not null default '',

	      pay_time datetime(6),

	    	create_time datetime(3) ,

	      deleted bit(1) not null default 0,

	      PRIMARY KEY (`id`) USING BTREE

	    ) ENGINE = InnoDB AUTO_INCREMENT = 1 CHARACTER SET = utf8 

	    COLLATE = utf8_general_ci ROW_FORMAT = Dynamic;

	- 大数据插入注意事项
	
	  大数据插入mysql会报错，需要设置相关参数。
	  
	  show variables like '%log_bin_trust_function_creators%'
	  
  
	set global log_bin_trust_function_creators=1;
	
- 创建函数
	
	- 随机字符串函数
	
		  delimiter $$
		  create function rand_string(n int) returns varchar(255)
		  begin
		
		     declare chars_str varchar(100) default 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
		     declare return_str varchar(255) default '';
		     declare i int default 0;
		     while i < n do 
		         set return_str = concat(return_str, substring(chars_str,floor(1+rand()*52),1));
		         set i = i +1;
		     end while;
		     return return_str;
	  end $$
	
	- 随机部门编号
	
	  delimiter $$
	
	  create function rand_deptid() returns int(5)
	
	  begin
	
	     declare i int default 0;
	
	     set i = floor(1000+rand()*10);
	
	     return i;
	
	  end $$
	
	- 随机状态数字
	
	  delimiter $$
	
	  create function rand_num(max int) returns int
	
	  begin
	
	     return floor(rand()* max);
	
	  end $$
	
	- 随机指定位数数字字符串
	
	  delimiter $$
	
	  create function rand_num_str(length int) returns varchar(255)
	
	  begin
	
	  
	
	     declare chars_str varchar(100) default '0123456789';
	
	     declare return_str varchar(255) default '';
	
	     declare i int default 0;
	
	     while i < length do 
	
	         set return_str = concat(return_str, substring(chars_str,floor(1+rand()*10),1));
	        
	         set i = i +1;
	
	     end while;
	
	     return return_str;
	
	  end $$
	
	- 随机用户ID
	
	  delimiter $$
	
	  create function rand_userid() returns int(5)
	
	  begin
	
	     declare i int default 0;
	
	     set i = floor(1000000+rand()*10);
	
	     return i;
	
	  end $$
	
- 创建存储过程
	
  - 部门表dept
	
    delimiter $$
	
    create procedure insert_dept(in start int(10),in max_num int(10))
	
    begin
	
       declare i int default 0;
	
       set autocommit = 0;
	
       repeat
	
          set i = i+1;
	      
          insert into dept(deptno,dname,loc,create_time)values((start+1),concat( 'department_' , rand_string(10)), rand_string(12),now(3));
	      
          until i=max_num
	
    
	
       end repeat;
	
       commit;
	
    end $$
	
  - 员工表emp
	
    delimiter $$
	
    CREATE DEFINER=`test`@`%` PROCEDURE `insert_emp`(in start int(10),in max_num int(10))
	
    begin
	
       declare i int default 0;
	
       set autocommit = 0;
	
       repeat
	
          set i = i+1;
	      
          insert into emp(empno,ename,job,mgr,hiredate,sal,comm,
	      
        		deptno,email,mobile,weight,married,pwd,create_time)values
	      
        		((start+1), concat('employee',rand_string(7)), 'saleman', 0001, curdate(),2000,400,
	      
        		rand_deptid(), rand_email(),rand_mobile(),rand_num(100),i%2,rand_num_str(6),now(3));
	      
          until i=max_num
	
    
	
       end repeat;
	
       commit;
	
    end
	
    end $$
	
  - 订单表
	
    delimiter $$
	
    create procedure insert_orders(in max_num int(10))
	
    begin
	
       declare i int default 0;
	
       declare total decimal(20,2) default 0;
	
       declare real_total decimal(20,2) default 0;
	
       set autocommit = 0;
	
       repeat
	
          set i = i+1;
	      
          set total = rand()*1000;
	      
          set real_total = total - rand()*100;
	
    
	  ​	  
    
	        insert into `order`(pay_status,trade_status,order_no, mobile, 
        
	        user_id, total, real_total ,channel_id ,open_id, union_id, 
        
	        transaction_no , pay_time, create_time)
        
	        values( rand_num(4), rand_num(8),
        
	        concat( '4081' , date_format(sysdate(),'%Y%m%d%h%i%s'), rand_num_str(5)),
        
	        concat( '185' ,rand_num_str(8)), rand_userid(), total, real_total,
        
	        rand_deptid(),  rand_string(15), replace(UUID(),'-',''),
        
	        concat( '4200000' , rand_num_str(16)),
        
	        now(6), now(3));
    
	
    ​	  
	
          until i = max_num
	
    
	
       end repeat;
	
       commit;
	  
    end $$
	
	- 执行存储过程
	
	  call insert_dept(1000, 1000);
	  
	  call insert_emp(1000,1000000);
	  
	  
	  28.54

### 索引优化分析

- 性能下降SQL查询慢

	- 慢分为两方面

		- SQL执行时间长
		- SQL等待时间长

	- 原因

		- SQL查询语句写的烂

			- 各种子查询，各种连接导致没建索引，或者没有用上索引

		- 索引失效

			- 原因：索引失效的原因是建立了索引，但是没用上。
			- 分类

				- 单值索引

				- 复合索引

		- 关联查询太多join(设计缺陷或不得已的需求)
		- 服务器调优及各个参数设置(缓冲\线程数等)，设置不合理也会导致性能下降变慢
		- 保存数据库的硬盘空间不足

- 常见通用的join查询

	- SQL执行顺序

		- 程序员写的SQL语句（人写）

		- MySQL软件理解的SQL语句（机读）

				- MySQL软件拿到程序员写的SQL语句后，是按照这个顺序来执行的.
它是先从FROM 开始执行，先找到用到了哪些表，然后对表进行join连接，最后才查询具体的字段。【记住】

		- 总结

				- 这是个鱼骨图
				- 1、MySQL服务器，先从from开始执行，然后再执行on。【注意：from和on是有前后顺序的】
				- 2、然后同时执行join和on，然后依次执行，最后直到limit

	- join图

	- join原理

		- mysql中使用Nested Loop Join来实现join；

A JOIN B：
通过A表的结果集作为循环基础，一条一条的通过结果集中的数据作为过滤条件到下一个表中查询数据，然后合并结果；

	- 建表SQL
	
	  create DATABASE db0629;


​	  

​	  

	  DROP TABLE IF EXISTS tbl_dept;
	
	  CREATE TABLE `tbl_dept`(
	
	  	`id` INT NOT NULL AUTO_INCREMENT,
	
	  	`deptName` VARCHAR(20) DEFAULT NULL,
	
	  	`locAdd` VARCHAR(40) DEFAULT NULL,


​	  

	  	PRIMARY KEY (`id`)
	
	  )ENGINE=INNODB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;


​	  

​	  

	  CREATE TABLE `tbl_emp`(
	
	  	`id` INT NOT NULL AUTO_INCREMENT,
	
	  	`name` VARCHAR(20) DEFAULT NULL,
	
	  	`deptId` INT DEFAULT NULL,


​	  	

	  	PRIMARY KEY(`id`),
	
	  	KEY `fk_dept_id`(`deptId`) #CONSTRAINT `fk_dept_id` FOREIGN KEY (`deptId`) REFERENCES `tbl_dept`(`id`)
	
	  )ENGINE=INNODB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;


​	  

​	  

​	  

	  INSERT INTO tbl_dept(deptName,locAdd) VALUES('RD',11);
	
	  INSERT INTO tbl_dept(deptName,locAdd) VALUES('HR',12);
	
	  INSERT INTO tbl_dept(deptName,locAdd) VALUES('MK',13);
	
	  INSERT INTO tbl_dept(deptName,locAdd) VALUES('MIS',14);
	
	  INSERT INTO tbl_dept(deptName,locAdd) VALUES('FD',15); 


​	  

​	  

	  INSERT INTO tbl_emp(name,deptId) VALUES('z3',1);
	
	  INSERT INTO tbl_emp(name,deptId) VALUES('z4',1);
	
	  INSERT INTO tbl_emp(name,deptId) VALUES('z5',1);


​	  

	  INSERT INTO tbl_emp(name,deptId) VALUES('w5',2);
	
	  INSERT INTO tbl_emp(name,deptId) VALUES('w6',2);
	
	- 7种Join
	
		- 1、内连接
	
		- 2、外链接
	
			- 1、左外连接
	
			- 2、右外连接
	
			- 3、全外连接
	
				- 因为mysql不支持全外连接，所以可以通过这种方式来实现全外连接的效果。

- 索引优化

	- 索引分析

		- 单表

			-   建表SQL

			  CREATE TABLE IF NOT EXISTS `article`(

			  	`id` INT(10) UNSIGNED NOT NULL PRIMARY KEY AUTO_INCREMENT,
		
			  	`author_id` INT(10) UNSIGNED NOT NULL,
		
			  	`category_id` INT(10) UNSIGNED NOT NULL,
		
			  	`views` INT(10) UNSIGNED NOT NULL,#文章被查看的次数
		
			  	`comments` INT(10) UNSIGNED NOT NULL,#回复数
		
			  	`title` VARBINARY(255) NOT NULL,
		
			  	`content` TEXT NOT NULL

			  );

			  

			  

			  INSERT INTO `article`(`author_id`,`category_id`,`views`,`comments`,`title`,`content`)

			  VALUES

			  (1,1,1,1,'1','1'),

			  (2,2,2,2,'2','2'),

			  (3,3,3,3,'3','3');

			  

			  SELECT * FROM article;

			-   案例

			  #需求：查询category_id为1，且comments大于1的情况下，views（阅读量）最多的article_id。

			  

			  explain select id,author_id from article  where category_id=1 and comments>1 order by views desc limit 1;

			  

			  #结论：很显然，type是ALL ，即全表扫描，这是最坏的情况。Extra里面还出现了Using filesort，这也是最坏的情况。优化是必须的。

			  

			  

			  #开始优化：

			  

			  1、第一次优化：根据查询的3个条件建立索引：category_id、comments和views，取首字母简称cvv

			  

			  建立索引：create index idx_article_cvv on article(category_id,comments,views);

			  查看建立的索引：show index from article;

			  

			  再次explain，查看优化的结果：

			  explain select id,author_id from article  where category_id=1 and comments>1 order by views desc limit 1;

			  

			  #结论：

			  type这一次是Range，而不再是ALL了，这是可以忍受的，解决了全表扫描的问题。 

			  但是，仍然存在Using filesort 文件内排序的问题，仍然无法忍受。

			  我们已经根据3个查询条件建立索引了啊，怎么还会有Using filesort问题呢？难道建立的索引有什么问题吗？

			  这是因为按照BTree索引的工作原理：

			  先排序category_id，

			  如果遇到相同的category_id，则会再根据comments排序，如果遇到相同的comments，则再排序views。

			  当comments字段在复合索引里处于中间位置时候，

			  因为comments>1条件是一个范围值（所谓range）

			  MySQL无法利用索引再对comments后面的views部分进行检索，即range类型的查询字段后面的索引无效。

			  也即是，cvv 索引中，只有第一个字段起作用，最后一个字段无效了。

			  

			  

			  试一下，我们把where后面的查询条件comments>1   改为comments=1，再次explain，看看结果

			  explain select id,author_id from article  where category_id=1 and comments=1 order by views desc limit 1;

			  #结论：

			  我们发现，此时，type 变为了ref，而Using filesort也没有了，完美。 

			  这说明，该SQL完美的利用了我们建立的idx_article_cvv。这也说明我们在前面的结论成立。

			  

			  

			  2、第二次优化：

			  首先删除原来建立的索引：drop index idx_article_cvv on article;

			  

			  根据category_id和views两个字段，重新建立索引：

			  create index idx_article_cv on article(category_id,views);

			  

			  再次explain，查看优化的结果：

			  explain select id,author_id from article  where category_id=1 and comments>1 order by views desc limit 1;

			  

			  结论：

			  可以看到，type变为了ref，Extra中的 Using filesort也消失了，结果非常理想。这就完美了。

				- 需求

					- 查询category_id为1，且comments大于1的情况下，views（阅读量）最多的article_id。

				- SQL

					- select id,author_id from article  where category_id=1 and comments>1 order by views desc limit 1;

				- 优化

					- 没有优化之前

							- 结论

								- 很显然，type是ALL ，即全表扫描，这是最坏的情况。Extra里面还出现了Using filesort，这也是最坏的情况。优化是必须的。

					- 第一次优化

						- 优化：建立索引

							- 根据查询的3个条件建立索引：category_id、comments和views，取首字母简称cvv 

建立索引：create index idx_article_cvv on article(category_id,comments,views);

								- 查看建立的索引
	
									- 查看建立的索引：show index from article;
						- 查看优化结果
	
								- type这一次是Range，而不再是ALL了，这是可以忍受的，解决了全表扫描的问题。 
但是，仍然存在Using filesort 文件内排序的问题，仍然无法忍受。

						- 为什么本次优化有问题
	
							- 原因：

我们已经根据3个查询条件建立索引了啊，怎么还会有Using filesort问题呢？难道建立的索引有什么问题吗？
这是因为按照BTree索引的工作原理：
先排序category_id，
如果遇到相同的category_id，则会再根据comments排序，如果遇到相同的comments，则再排序views。
当comments字段在复合索引里处于中间位置时候，
因为comments>1条件是一个范围值（所谓range）
MySQL无法利用索引再对comments后面的views部分进行检索，即range类型的查询字段后面的索引无效。
也即是，cvv 索引中，只有第一个字段起作用，最后一个字段无效了。
							- 试一下：
我们把where后面的查询条件comments>1   改为comments=1，再次explain，看看结果
explain select id,author_id from article  where category_id=1 and comments=1 order by views desc limit 1;
#结论：
我们发现，此时，type 变为了ref，而Using filesort也没有了，完美。 
这说明，该SQL完美的利用了我们建立的idx_article_cvv。这也说明我们在前面的结论成立。

					- 第二次优化
	
						- 首先删除原来建立的索引
	
							- drop index idx_article_cvv on article;
	
						- 重新建立新索引
	
							- 根据category_id和views两个字段，重新建立索引：
create index idx_article_cv on article(category_id,views);

						- 再次查看优化结果
	
						- 结论
	
							- 可以看到，type变为了ref，Extra中的 Using filesort也消失了，结果非常理想。这就完美了。
	
		- 两表
	
			-   建表SQL
	
			  CREATE TABLE IF NOT EXISTS `class`(#商品类别
	
			  	`id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
	
			  	`card` INT(10) UNSIGNED NOT NULL,


​			  	

			  	PRIMARY KEY(`id`)
	
			  );


​			  

			  CREATE TABLE IF NOT EXISTS `book`(
	
			  	`bookid` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
	
			  	`card` INT(10) UNSIGNED NOT NULL,


​			  	

			  	PRIMARY KEY(`bookid`)
	
			  );


​			  

			  INSERT INTO class(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO class(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO class(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO class(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO class(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO class(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO class(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO class(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO class(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO class(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO class(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO class(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO class(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO class(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO class(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO class(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO class(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO class(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO class(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO class(card) VALUES(FLOOR(1+(RAND()*20)));


​			  

​			  

			  INSERT INTO book(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO book(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO book(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO book(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO book(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO book(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO book(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO book(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO book(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO book(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO book(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO book(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO book(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO book(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO book(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO book(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO book(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO book(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO book(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO book(card) VALUES(FLOOR(1+(RAND()*20)));
	
			-   案例
	
				- SQL：左连接
	
					- select * from class left join book on class.card = book.card;
	
				- 优化
	
					- 没优化之前
	
							- 结论：两个type都是ALL，都是全表扫描，不可接受。且查询行数是20*20行，笛卡尔积，必须优化。
	
					- 第一次优化
	
						- 尝试：左连接，索引建在左表上，即class上

建立索引：alter table class add index Y(`card`);
							- 结论：虽然id=1 的type变成了index，但是查询行数仍然是20*20，仍然是笛卡尔积，优化不成功。

					- 第二次优化
	
						- 删除上面建立的无效索引：drop index Y from class;
						- 尝试：左连接，索引建在右表上，即book上
建立索引： alter table book add index X(`card`);
							- 结论：type变成了ref，而且rows的优化效果非常明显。

				- 结论
	
					- 分析
	
						- 这是左连接的特性决定的，因为左连接查询的结果是：左表的全部，和右表的部分。
即，left join右边的条件，决定如何从右表搜索结果。
这就是说，左连接右表才算关键。右表决定了最终查询的结果。所以左连接要把索引建立在右表上。

					- 结论：
左连接，索引建在右表上。右连接，索引建在左表上。
即，左右连接，相反建。
并且，要把索引建在join条件字段上。

		- 三表
	
			-   建表SQL
	
			  CREATE TABLE IF NOT EXISTS `phone`(
	
			  	`phoneid` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
	
			  	`card` INT(10) UNSIGNED NOT NULL,


​			  	

			  	PRIMARY KEY(`phoneid`)
	
			  ) ENGINE = INNODB;


​			  

​			  

			  INSERT INTO phone(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO phone(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO phone(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO phone(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO phone(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO phone(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO phone(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO phone(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO phone(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO phone(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO phone(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO phone(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO phone(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO phone(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO phone(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO phone(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO phone(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO phone(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO phone(card) VALUES(FLOOR(1+(RAND()*20)));
	
			  INSERT INTO phone(card) VALUES(FLOOR(1+(RAND()*20)));
	
			-   案例
	
				- SQL
	
					- select * from class left join book on class.card = book.card left join phone on book.card = phone.card;
	
				- 没优化之前
	
				- 优化规则
	
					- 按照两表连接优化规则进行优化即可。
即，在book和phone两个表上建索引即可。

alter table book add index X(`card`);
alter table phone add index Z(`card`);
						- 结论：后2行的type都是ref，然后rows优化的效果也很好。

				- 总结
	
					- join语句的优化：

1、尽可能减少join语句中的NestedLoop的循环总次数：“永远用小结果集驱动大的结果集”。
2、优先优化NestedLoop的内层循环。因为内层是大结果集，内层的性能高了，整体的查询性能就高了。
3、保证join语句中被驱动表上join字段已经被索引。
4、当无法保证被驱动表的join条件字段被索引且内存资源充足的前提下，不要太吝啬MySQL配置文件中JoinBuffer的设置。
5、不管你有多少个索引,一次查询至多采用一个索引;(索引和索引之间是独立的)。为了解决这个问题，开发中尽量使用复合索引。在实际应用中,基本上都使用复合索引;

						- JoinBuffer说明
	
							- A表连接B表形成一个中间表，B表连接C表又形成一个中间表，这些中间表在连接的过程中都是放到缓存中。
如果join buffer越大，所有的中间表都可以放到内存中，所有的查询都可以在内存中完成。

如果join buffer小，连的表大，中间表的数据放不下，就会把连接的数据精简掉一些数据，
如果你要查询的数据刚好是被精简掉的数据，那么select还是要跑到数据库中重新查询一次，这样性能就比较低

	- 案例（索引失效）
	
		- 1、全值匹配我最爱
	
			- 全值匹配
	
					- 这3中情况都是很好的使用了索引。
	
			- 总结：where后面查询的字段个数和顺序，都和索引中的字段个数和顺序一致。这就是全值匹配。
	
		- 2、最佳左前缀法则
	
			- 如果索引了多例，要遵守最左前缀法则。指的是查询从索引的最左前列开始“并且不跳过索引中的列”。
			- 口诀
	
				- 1、带头大哥不能死。
	
						- 带头大哥死了，索引就全失效了。
	
				- 2、中间兄弟不能断。
	
						- 解读
	
							- 正常情况下，2个索引的key_len应该是78，而且ref后面应该有两个const。
所以，可以看出来，只用到了name的索引。pos的索引没有用到，失效了。
							- 就好比，建立的一个梯子，用来连接1楼，2楼，3楼，  
你不能把2楼的梯子抽走，否则就去不了3楼了。

这个时候，只能用到了1楼的索引，3楼的索引就失效了。

		- 3、不在索引列上做任何操作（计算、函数、（自动or手动）类型转换），会导致索引失效而转向全表扫描
	
		- 4、存储引擎不能使用索引中范围条件右边的列
	
				- 最后一条SQL：where name ='z4' and age>11 and pos='manager'

这里，索引只用到了name和age两个字段，而pos失效了。
name：用到索引了，用于查找；
age：用到索引了，用于排序。

			- 口诀：范围之后全失效
	
		- 5、尽量使用覆盖索引（只访问索引的查询（索引列和查询列一致）），减少select*
	
				- 解读
	
					- 第1、2两条SQL
覆盖索引比select * 在Extra中多了一个using index，这说明覆盖索引性能更优秀。
					- 第3条SQL
当select后面是覆盖索引的时候，如果where语句中，使用了范围age>25，
那么age和pos全失效。只有name一个索引有用
					- 最后2条SQL
要查的字段，最好在建的索引列之内，或者和索引列等同，这个时候Extra出现Using index，性能将非常优秀。

		- 6、mysql在使用不等于（！=或者<>）的时候无法使用索引，会导致全表扫描
	
			- 在开发中，有时候为了业务需要，即使是索引失效，也要写这样的SQL。具体问题具体分析。不要怕会导致索引失效，而束手束脚。
所以，你心里知道有这回事就行，开发中为了业务的需要，该怎么写就怎么写。

		- 7、is null,is not null 也无法使用索引
	
				- 既然知道有这个规则，开发中，我们在数据表中就尽量为每个字段设置一个default默认值，尽量不要在表数据中出现NULL。
	
		- 8、like以通配符开头（'%abc...'）mysql索引失效会变成全表扫描操作
	
			- 详解
	
					- %加在左边，有问题。会导致索引失效，且全表扫描。
%加在右边，没问题。

并且like 是个range，即范围

			- 问题
	
				- 解决like'%字符串%'索引不被使用的方法？？
即like中%导致索引失效的问题，如何解决？

					- 建表SQL
	
					  # DROP TABLE tbl_user;


					  CREATE TABLE `tbl_user`(


					  	`id` INT(11) NOT NULL AUTO_INCREMENT,


					  	`name` VARCHAR(20) DEFAULT NULL,


					  	`age` INT(11) DEFAULT NULL,


					  	`email` VARCHAR(20) DEFAULT NULL,


​					  


					  	PRIMARY KEY(`id`)


					  ) ENGINE=INNODB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;


​					  


​					  


					  INSERT INTO tbl_user(name,age,email) VALUES('1aa1',21,'b@163.com');


					  INSERT INTO tbl_user(name,age,email) VALUES('2aa2',21,'a@163.com');


					  INSERT INTO tbl_user(name,age,email) VALUES('3aa3',21,'c@163.com');


					  INSERT INTO tbl_user(name,age,email) VALUES('4aa4',21,'d@163.com');
	
					- 建立索引
	
						- CREATE INDEX idx_user_nameAge on tbl_user(name,age);
	
					- 解决方式：使用覆盖索引
	
						- id因为是主键，自带索引，自带光环。
explain select id from tbl_user where name like '%aa%'

explain select name from tbl_user where name like '%aa%'
explain select id,name from tbl_user where name like '%aa%'
explain select id,name,age from tbl_user where name like '%aa%'
explain select name,age from tbl_user where name like '%aa%'
			- 口诀

				- 百分% like  加右边。如果两边都有百分，使用覆盖索引解决失效。
	
		- 9、字符串不加单引号索引失效。
	
			- varchar类型，绝对不能省去单引号。【不写单引号，开发中是重罪】
				- 如果不写单引号，就会导致自动类型转换，违背索引失效：第3条禁忌，会导致索引失效且全表扫描。
	
		- 10、少用or，用它连接时会索引失效
	
		- 11、小总结
	
				- like KK%相当于=常量     %KK和%KK% 相当于范围
	
		- 12、优化总口诀
	
			- 全值匹配我最爱，最左前缀记心间。
带头大哥不能死，中间兄弟不能断。
索引列上不计算，like百分%最右边。
范围之后全失效，字符串要加引号。
多用覆盖不写星，不等!=（<>）、NULL、OR全完蛋。

	- 一般性建议
	
		- 对于单键索引，尽量选择针对当前query过滤性更好的索引
		- 在选择组合索引的时候，当前Query中过滤性最好的字段在索引字段顺序中，位置越靠前（即靠左）越好。
		- 在选择组合索引的时候，尽量选择可以能包含当前query中的where子句中更多字段的索引
		- 尽可能通过分析统计信息和调整query的写法来达到选择合适索引的目的

*XMind - Trial Version*