---
layout: post
read_time: true
show_date: true
title:  "Data storage" 
date:   2021-03-12 13:32:20 -0600
description: 有关数据存储的相关知识点总结、归纳
img: posts/20210228/MLLibrary.jpg 
tags: [Mysql, redis, MongoDB]
author: TM
github: https://github.com/Tomxiaobai
mathjax: yes
---

Mysql、Redis以及MongoDB都是我们工作中常见的数据存储的工具，本模块主要是对该三种数据存储做简单的介绍以及总结自己在工作中遇到的问题和基础知识，供自己学习，不用于任何商业化，有一些是博文的内容，仅作为复习用，先感谢各位博主大佬的知识分享。

<center><img src='./assets/img/posts/20210228/mysql_redis_mongo.png'></center>
## Mysql
* 在表结构设计时，主键的设计一定要尽可能地使用顺序值，这样才能保证在海量并发业务场景下的性能。

* 索引是提升查询速度的一种数据结构。

  索引之所以能提升查询速度，在于它在插入时对数据进行了排序（显而易见，它的缺点是影响插入或者更新的性能）。

* InnoDB 存储引擎支持的索引有 B+ 树索引、全文索引、R 树索引

* B+树索引的特点是： 基于磁盘的平衡树，但树非常矮，通常为 3~4 层，能存放千万到上亿的排序数据。树矮意味着访问效率高，从千万或上亿数据里查询一条数据，只用 3、4 次 I/O。

* 需要特别注意的是，在存储时间时，UUID 是根据时间位逆序存储， 也就是低时间低位存放在最前面，高时间位在最后，即 UUID 的前 4 个字节会随着时间的变化而不断“随机”变化，并非单调递增。而非随机值在插入时会产生离散 IO，从而产生性能瓶颈。这也是 UUID 对比自增值最大的弊端。

* 为了解决这个问题，MySQL 8.0 推出了函数 UUID_TO_BIN，它可以把 UUID 字符串：

  通过参数将时间高位放在最前，解决了 UUID 插入时乱序问题；

  去掉了无用的字符串"-"，精简存储空间；

  将字符串其转换为二进制值存储，空间最终从之前的 36 个字节缩短为了 16 字节。

* 而且由于 UUID 能保证全局唯一，因此使用 UUID 的收益远远大于自增ID。可能你已经习惯了用自增做主键，但在海量并发的互联网业务场景下，更推荐 UUID 这样的全局唯一值做主键。

  比如，我特别推荐游戏行业的用户表结构设计，使用 UUID 作为主键，而不是用自增 ID。因为当发生合服操作时，由于 UUID 全局唯一，用户相关数据可直接进行数据的合并，而自增 ID 却需要额外程序整合两个服务器 ID 相同的数据，这个工作是相当巨大且容易出错的。

* **三大日志**：

  * **binlog**

    * binlog 用于记录数据库执行的写入性操作(不包括查询)信息，以二进制的形式保存在磁盘中。 binlog 是 mysql的逻辑日志，并且由 Server 层进行记录，使用任何存储引擎的 mysql 数据库都会记录 binlog 日志。
    * **逻辑日志**： 可以简单理解为记录的就是sql语句 。
    * **物理日志**： mysql 数据最终是保存在数据页中的，物理日志记录的就是数据页变更 。
    * *binlog* 是通过追加的方式进行写入的，可以通过 *max_binlog_size* 参数设置每个 *binlog*
      文件的大小，当文件大小达到给定值之后，会生成新的文件来保存日志。
    * 使用场景：
      * 在实际应用中， binlog 的主要使用场景有两个，分别是 **主从复制** 和 **数据恢复** 。
      * **主从复制** ：在 Master端开启 *binlog*，然后将 *binlog* 发送到各个Slave端， Slave端重放 *binlog* 从而达到主从数据一致。
      * **数据恢复** ：通过使用 *mysqlbinlog* 工具来恢复数据。

    * **binlog刷盘时机**
      * 对于 *InnoDB* 存储引擎而言，只有在事务提交时才会记录 *binlog*，此时记录还在内存中，那么 *binlog*
        是什么时候刷到磁盘中的呢？ *mysql* 通过 *sync_binlog* 参数控制 *binlog* 的刷盘时机，取值范围是 *0-N*
        - 0：不去强制要求，由系统自行判断何时写入磁盘；
        - 1：每次 *commit*的时候都要将 *binlog*写入磁盘；
        - N：每N个事务，才会将 *binlog*写入磁盘。

  * *redo log*

    * **只要事务提交成功，那么对数据库做的修改就被永久保存下来了，不可能因为任何原因再回到原来的状态** 。

    * 那么 *mysql* 是如何保证一致性的呢？最简单的做法是在每次事务提交的时候，将该事务涉及修改的数据页全部刷新到磁盘中。但是这么做会有严重的性能问题，主要体现在两个方面：

      1. 因为 *Innodb* 是以 页 为单位进行磁盘交互的，而一个事务很可能只修改一个数据页里面的几个字节，这个时候将完整的数据页刷到磁盘的话，太浪费资源了！
      2. 一个事务可能涉及修改多个数据页，并且这些数据页在物理上并不连续，使用随机IO写入性能太差！

    * 因此 *mysql* 设计了 *redo log* ， **具体来说就是只记录事务对数据页做了哪些修改**，这样就能完美地解决性能问题了(相对而言文件更小并且是顺序IO)。

    * *redo log* 包括两部分：一个是内存中的日志缓冲(*redo log buffer*)，另一个是磁盘上的日志文件(*redo log file*).
    * mysql 每执行一条 DML 语句，先将记录写入 redo log buffer 
      ，后续某个时间点再一次性将多个操作记录写到 redo log file 。这种先写日志，再写磁盘的技术就是 MySQL
      里经常说到的 *WAL(Write-Ahead Logging)* 技术

    * mysql 支持三种将 redo log buffer 写入 redo log file `的时机，可以通过 innodb_flush_log_at_trx_commit参数配置，各参数值含义如下：
    <center><img src='./assets/img/posts/20210228/image-20211125002228267.png'></center>
    * *redo log* 实际上记录数据页的变更，而这种变更记录是没必要全部保存，因此 *redo log*
      实现上采用了大小固定，循环写入的方式，当写到结尾时，会回到开头循环写日志。
    <center><img src='./assets/img/posts/20210228/image-20211125003912942.png'></center>

    * 由 *binlog* 和 *redo log* 的区别可知： *binlog* 日志只用于归档，只依靠 *binlog* 是没有 *crash-safe* 能力的。但只有 *redo log* 也不行，因为 *redo log* 是 *InnoDB* 特有的，且日志上的记录落盘后会被覆盖掉。因此需要 *binlog* 和 *redo log* 二者同时记录，才能保证当数据库发生宕机重启时，数据不会丢失。
  
  
  * undo log
    * 数据库事务四大特性中有一个是 **原子性** ，具体来说就是 **原子性是指对数据库的一系列操作，要么全部成功，要么全部失败，不可能出现部分成功的情况**。实际上， **原子性** 底层就是通过 *undo log* 实现的。 *undo log* 主要记录了数据的逻辑变化，比如一条 *INSERT* 语句，对应一条 *DELETE* 的 *undo log* ，对于每个 *UPDATE* 语句，对应一条相反的 *UPDATE* 的 *undo log*，这样在发生错误时，就能回滚到事务之前的数据状态。
  
* **mysql优化**

  * **定位慢查询SQL**

    * 使用show processlist定位，查询正在执行的慢查询

    * MySQL 的慢查询日志记录的内容是：在 MySQL 中响应时间超过参数 **long_query_time**（单位秒，默认值 10）设置的值并且扫描记录数不小于 min_examined_row_limit（默认值0）的语句。

  > NOTE：默认情况下，慢查询日志中不会记录管理语句，如果需要记录的请做如下设置，设置log_slow_admin_statements = on 让管理语句中的慢查询也会记录到慢查询日志中。默认情况下，也不会记录查询时间不超过 long_query_time 但是不使用索引的语句，可通过配置 log_queries_not_using_indexes = on 让不使用索引的 SQL 都被记录到慢查询日志中（即使查询时间没超过 long_query_time 配置的值）。

  > NOTE：慢查询query time设置小技巧：线上业务一般建议把 long_query_time 设置为 1 秒，如果某个业务的 MySQL 要求比较高的 QPS，可设置慢查询为 0.1 秒。发现慢查询及时优化或者提醒开发改写。一般测试环境建议 long_query_time 设置的阀值比生产环境的小，比如生产环境是 1 秒，则测试环境建议配置成 0.5 秒。便于在测试环境及时发现一些效率低的 SQL。甚至某些重要业务测试环境 long_query_time 可以设置为 0，以便记录所有语句。并留意慢查询日志的输出，上线前的功能测试完成后，分析慢查询日志每类语句的输出，重点关注 Rows_examined（语句执行期间从存储引擎读取的行数），提前优化。
    
  * 通过 explain、show profile 和 trace 等诊断工具来分析慢查询
  
    * Explain 可以获取 MySQL 中 SQL 语句的执行计划，比如语句是否使用了关联查询、是否使用了索引、扫描行数等。可以帮我们选择更好地索引和写出更优的 SQL 。
  
* **MYsql MVCC解决幻读**

  * **隔离级别**： 读未提交(READ UNCOMMITED)->读已提交(READ COMMITTED)->可重复读(REPEATABLE READ)->序列化(SERIALIZABLE)。隔离级别依次增强，但是导致的问题是并发能力的减弱。myql默认为可重复读
  <center><img src='./assets/img/posts/20210228/image-20211216153355333.png'></center>

  * **Bin Log**:是mysql服务层产生的日志，常用来进行数据恢复、数据库复制，常见的mysql主从架构，就是采用slave同步master的binlog实现的

    **Redo Log**:记录了数据操作在物理层面的修改，mysql中使用了大量缓存，修改操作时会直接修改内存，而不是立刻修改磁盘，事务进行中时会不断的产生redo log，在事务提交时进行一次flush操作，保存到磁盘中。当数据库或主机失效重启时，会根据redo log进行数据的恢复，如果redo log中有事务提交，则进行事务提交修改数据。
  
    **Undo Log**: 除了记录redo log外，当进行数据修改时还会记录undo log，undo log用于数据的撤回操作，它记录了修改的反向操作，比如，插入对应删除，修改对应修改为原来的数据，**通过undo log可以实现事务回滚，并且可以根据undo log回溯到某个特定的版本的数据，实现MVCC**
  
  * **MVCC(Multi Version Concurrency Control的简称)，代表多版本并发控制**。与MVCC相对的，是基于锁的并发控制，Lock-Based Concurrency Control)。MVCC最大的优势：**读不加锁，读写不冲突**。
  
* Mysql数据一致性保证： **redo log 和 binlog**

  <center><img src='./assets/img/posts/20210228/mysql_innodb.png'></center>

  * redo log，他是一种偏向物理性质的重做日志，因为他里面记录的是类似这样的东西，“对哪个数据页中的什么记录，做了个什么修改”。
  
    而且redo log本身是属于InnoDB存储引擎特有的一个东西。
  
    而binlog叫做归档日志，他里面记录的是偏向于逻辑性的日志，类似于“对users表中的id=10的一行数据做了更新操 作，更新以后的值是什么”
  
    binlog不是InnoDB存储引擎特有的日志文件，是属于mysql server自己的日志文件。
  
  * 当我们把binlog写入磁盘文件之后，接着就会完成最终的事务提交，此时会把本次更新对应的binlog文件名称和这次 更新的binlog日志在文件里的位置，都写入到redo log日志文件里去，同时在redo log日志文件里写入一个commit标记。
  
  * **最后一步在redo日志中写入commit标记的意义是什么?**
  
    * 假设我们在提交事务的时候，一共有上图中的5、6、7三个步骤，必须是三个步骤都执行完毕，才算是提交了事务。那么在我们刚完成步骤5的时候，也就是redo log刚刷入磁盘文件的时候，mysql宕机了，此时怎么办?
  
      这个时候因为没有最终的事务commit标记在redo日志里，所以此次事务可以判定为不成功。不会说redo日志文件里 有这次更新的日志，但是binlog日志文件里没有这次更新的日志，不会出现数据不一致的问题。
  
      如果要是完成步骤6的时候，也就是binlog写入磁盘了，此时mysql宕机了，怎么办?
  
      同理，因为没有redo log中的最终commit标记，因此此时事务提交也是失败的。
  
      必须是在redo log中写入最终的事务commit标记了，然后此时事务提交成功，而且redo log里有本次更新对应的日 志，binlog里也有本次更新对应的日志 ，redo log和binlog完全是一致的。
  
  * 在MySQL内部，在事务提交时利用两阶段提交(内部XA的两阶段提交)很好地解决了上面提到的binlog和redo log的一致性问题：

    第一阶段： InnoDB Prepare阶段。此时SQL已经成功执行，并生成事务ID(xid)信息及redo和undo的内存日志。此阶段InnoDB会写事务的redo log，但要注意的是，此时redo log只是记录了事务的所有操作日志，并没有记录提交（commit）日志，因此事务此时的状态为Prepare。此阶段对binlog不会有任何操作。
    第二阶段：commit 阶段，这个阶段又分成两个步骤。第一步写binlog（先调用write()将binlog内存日志数据写入文件系统缓存，再调用fsync()将binlog文件系统缓存日志数据永久写入磁盘）；第二步完成事务的提交（commit），此时在redo log中记录此事务的提交日志（增加 commit 标签）。
    可以看出，此过程中是先写redo log再写binlog的。但需要注意的是，在第一阶段并没有记录完整的redo log（不包含事务的commit标签），而是在第二阶段记录完binlog后再写入redo log的commit 标签。还要注意的是，**在这个过程中是以第二阶段中binlog的写入与否作为事务是否成功提交的标志**。（记住不是根据redo log 中的commit标记作为是否成功的标志）
    **原文链接:[CSDN](https://blog.csdn.net/huangjw_806/article/details/100927097)**

  * 崩溃恢复过程如下：
    ​- 如果数据库在记录此事务的binlog之前和过程中发生crash。数据库在恢复后认为此事务并没有成功提交，则会回滚此事务的操作。与此同时，因为在binlog中也没有此事务的记录，所以从库也不会有此事务的数据修改。
    ​- 如果数据库在记录此事务的binlog之后发生crash。此时，即使是redo log中还没有记录此事务的commit 标签，数据库在恢复后也会认为此事务提交成功（因为在上述两阶段过程中，binlog写入成功就认为事务成功提交了）。它会扫描最后一个binlog文件，并提取其中的事务ID（xid），InnoDB会将那些状态为Prepare的事务（redo log没有记录commit 标签）的xid和Binlog中提取的xid做比较，如果在Binlog中存在，则提交该事务，否则回滚该事务。这也就是说，binlog中记录的事务，在恢复时都会被认为是已提交事务，会在redo log中重新写入commit标志，并完成此事务的重做（主库中有此事务的数据修改）。与此同时，因为在binlog中已经有了此事务的记录，所有从库也会有此事务的数据修改。

## Redis
  **相关文档**:
  - [极客时间--高并发](https://time.geekbang.org/column/intro/100035801)
  - [极客时间--Redis](https://time.geekbang.org/column/intro/100056701)
  * 数据结构
  <center><img src='./assets/img/posts/20210228/image-20211216122016176.png'></center>
   * **渐进式rehash**

  * 简单来说就是在第二步拷⻉数据时，Redis仍然正常处理客戶端请求，每处理一个请求时，从哈希表1中的第 一个索引位置开始，顺带着将这个索引位置上的所有entries拷⻉到哈希表2中;等处理下一个请求时，再顺 带拷⻉哈希表1中的下一个索引位置的entries。
  <center><img src='./assets/img/posts/20220414/image-20211216122233243.png'></center>

  * 压缩列表实际上类似于一个数组，数组中的每一个元素都对应保存一个数据。和数组不同的是，压缩列表在表头有三个字段zlbytes、zltail和zllen，分别表示列表⻓度、列表尾的偏移量和列表中的entry个数;压缩列 表在表尾还有一个zlend，表示列表结束
  <center><img src='./assets/img/posts/20220414/image-20211216122319447.png'></center>

  ​* 在压缩列表中，如果我们要查找定位第一个元素和最后一个元素，可以通过表头三个字段的⻓度直接定位， 复杂度是 O(1)。而查找其他元素时，就没有这么高效了，只能逐个查找，此时的复杂度就	是 O(N) 了。

  * 跳表在链表的基础上，**增加了多级索引，通过索引位置的几个跳转，实现数据的快速定位**
  <center><img src='./assets/img/posts/20220414/image-20211216122418910.png'></center>

  * **为什么单机Redis这么快？**

    * 以Get请求为例，SimpleKV为了处理一个Get请求，需要**监听客戶端请求(bind/listen)**，和**客戶端建立连接(accept)，从socket中读取请求(recv)**，**解析客戶端发送请求(parse)**，根据请求类型读取键值数据(get)，最后给客戶端返回结果，即向socket中写回数据(send)。

  <center><img src='./assets/img/posts/20220414/image-20220105003536340.png'></center>
  * 但是，在这里的网络IO操作中，有潜在的阻塞点，**分别是accept()和recv()**。当Redis监听到一个客戶端有连接请求，但一直未能成功建立起连接时，会阻塞在accept()函数这里，导致其他客戶端无法和Redis建立连接。类似的，**当Redis通过recv()从一个客戶端读取数据时，如果数据一直没有到达，Redis也会一直阻塞在 recv()。**
  这就导致Redis整个线程阻塞，无法处理其他客戶端请求，效率很低。不过，幸运的是，socket网络模型本身支持非阻塞模式。

  * **基于多路复用的高性能I/O模型**

  * 在Redis只运行单线程的情况下，**该机制允许内核中，同时存在多个监听套接字和已连接套接字**。内核会一 直监听这些套接字上的连接请求或数据请求。一旦有请求到达，就会交给Redis线程处理，这就实现了一个 Redis线程处理多个IO流的效果。

  <center><img src='./assets/img/posts/20220414/image-20220105003708152.png'></center>

  * 下图就是基于多路复用的Redis IO模型。图中的多个FD就是刚才所说的多个套接字。Redis网络框架调用 epoll机制，让内核监听这些套接字。此时，Redis线程不会阻塞在某一个特定的监听或已连接套接字上，也就是说，不会阻塞在某一个特定的客戶端请求处理上。正因为此，Redis可以同时和多个客戶端连接并处理 请求，从而提升并发性。
  * 为了在请求到达时能通知到Redis线程，select/epoll提供了**基于事件的回调机制**，即**针对不同事件的发生， 调用相应的处理函数**。

  * 那么，回调机制是怎么工作的呢?其实，select/epoll一旦监测到FD上有请求到达时，就会触发相应的事件。这些事件会被放进一个事件队列，Redis单线程对该事件队列不断进行处理。这样一来，**Redis无需一直轮询 是否有请求实际发生，这就可以避免造成CPU资源浪费。同时，Redis在对事件队列中的事件进行处理时， 会调用相应的处理函数**，这就**实现了基于事件的回调**。因为Redis一直在对事件队列进行处理，所以能及时响应客戶端请求，提升Redis的响应性能。

* SDS动态字符串
  <center><img src='./assets/img/posts/20220414/image-20220105203138723.png'></center>

  * 动态字符串的数据结构主要包括：free、len和buffer三个区域。其中free保存剩余的空间大小，len保存字符串长度大小，buffer保存字符串真实值
  * 为了降低字符串扩充和缩小的时间复杂度，SDS运用了空间预分配和惰性空间释放的优化策略。

* 为什么Redis的AOF采用写后日志，Mysql采用写前日志？

  * Redis在向AOF里面记录日志的时候，并不会先去对这些命令进行语法检 查。所以，如果先记日志再执行命令的话，日志中就有可能记录了错误的命令，Redis在使用日志恢复数据 时，就可能会出错。

    而写后日志这种方式，就是先让系统执行命令，只有命令能执行成功，才会被记录到日志中，否则，系统就 会直接向客戶端报错。所以，Redis使用写后日志这一方式的一大好处是，可以避免出现记录错误命令的情 况。

  * 它是在命令执行后才记录日志，所以**不会阻塞当前的写操作**

  * AOF存在的两个风险问题：

    * 可能会给下一个操作带来阻塞⻛险。这是因为，AOF日志也是 在主线程中执行的，如果在把日志文件写入磁盘时，磁盘写压力大，就会导致写盘很慢，进而导致后续的操 作也无法执行了。
    * 如果刚执行完一个命令，还没有来得及记日志就宕机了**，那么这个命令和相应的数据就有丢失的⻛ 险**。如果此时Redis是用作缓存，还可以从后端数据库重新读入数据进行恢复，但是，如果Redis是直接用作 数据库的话，此时，因为命令没有记入日志，所以就无法用日志进行恢复了。
    * 因此存在了三种写回策略：
      * **Always**，同步写回:每个写命令执行完，立⻢同步地将日志写回磁盘; 
      * **Everysec**，每秒写回:每个写命令执行完，只是先把日志写到AOF文件的内存缓冲区，每隔一秒把缓冲区中的内容写入磁盘;
      * **No**，操作系统控制的写回:每个写命令执行完，只是先把日志写到AOF文件的内存缓冲区，由操作系统 决定何时将缓冲区内容写回磁盘。

  * AOF过大导致的性能问题：

    * 一是，文件系统本身对文件大小有限制，无法保存过大的文 件;

    * 二是，如果文件太大，之后再往里面追加命令记录的话，效率也会变低;

    * 三是，如果发生宕机，AOF中 记录的命令要一个个被重新执行，用于故障恢复，如果日志文件太大，整个恢复过程就会非常缓慢，这就会 影响到Redis的正常使用。

* 解决方案：**AOF重写机制**

    * 写机制具有“多变一”功能。所谓的“多变一”，也就 是说，旧日志文件中的多条命令，在重写后的新日志中变成了一条命令。

    * 和AOF日志由主线程写回不同，**重写过程是由后台线程bgrewriteaof来完成的，这也是为了避免阻塞主线程**，导致数据库性能下降。

    * 重写的过程总结为“**一个拷⻉，两处日志**”。

      * “一个拷⻉”就是指，每次执行重写时，主线程fork出后台的bgrewriteaof子进程。此时，fork会把主线程 的内存拷⻉一份给bgrewriteaof子进程，这里面就包含了数据库的最新数据。然后，bgrewriteaof子进程就 可以在不影响主线程的情况下，逐一把拷⻉的数据写成操作，记入重写日志。

      * 因为主线程未阻塞，仍然可以处理新来的操作。此时，如果有写操作，第一处日志就是指正在使用的AOF日 志，Redis会把这个操作写到它的缓冲区。这样一来，即使宕机了，这个AOF日志的操作仍然是⻬全的，可 以用于恢复。

        而第二处日志，就是指新的AOF重写日志。这个操作也会被写到重写日志的缓冲区。这样，重写日志也不会 丢失最新的操作。等到拷⻉数据的所有操作记录重写完成后，重写日志记录的这些最新操作也会写入新的 AOF文件，以保证数据库最新状态的记录。此时，我们就可以用新的AOF文件替代旧文件了。

  * **总结来说**，每次AOF重写时，Redis会先执行一个内存拷⻉，用于重写; 然后，使用两个日志保证在重写过 程中，新写入的数据不会丢失。而且，因为Redis采用额外的线程进行数据重写，所以，这个过程并不会阻 塞主线程。


* **缓存策略**：

  * **Cache Aside(旁路缓存)策略**
    * **读策略的步骤是:**
      * 从缓存中读取数据; 如果缓存命中，则直接返回数据; 如果缓存不命中，则从数据库中查询数据; 查询到数据后，将数据写入到缓存中，并且返回给用户。
    * **写策略的步骤是:**
      * **更新数据库中的记录; 删除缓存记录**。
    * Cache Aside 存在的最大的问题是当写入比较频繁时，缓存中的数据会被频繁地清理，这 样会对缓存的命中率有一些影响。**如果你的业务对缓存命中率有严格的要求，那么可以考虑 两种解决方案:**
      * 一种做法是在更新数据时也更新缓存，只是在更新缓存前先加一个分布式锁，因为这样 在同一时间只允许一个线程更新缓存，就不会产生并发问题了。当然这么做对于写入的性能 会有一些影响
      * 另一种做法同样也是在更新数据时更新缓存，只是给缓存加一个较短的过期时间，这样 即使出现缓存不一致的情况，缓存的数据也会很快地过期，对业务的影响也是可以接受。

* **Read/Write Through(读穿 / 写穿)策略**
  * **Write Through 的策略是这样的:先查询要写入的数据在缓存中是否已经存在，如果已经 存在，则更新缓存中的数据，并且由缓存组件同步更新到数据库中，如果缓存中数据不存 在，我们把这种情况叫做“Write Miss(写失效)”。**
  * 一般来说，我们可以选择两种“Write Miss”方式:一个是“Write Allocate(按写分 配)”，做法是写入缓存相应位置，再由缓存组件同步更新到数据库中;另一个是“**No- write allocate(不按写分配)”，做法是不写入缓存中，而是直接更新到数据库中**。
  * **在 Write Through 策略中，我们一般选择“No-write allocate”方式，原因是无论采用哪 种“Write Miss”方式，我们都需要同步将数据更新到数据库中，而“No-write allocate”方式相比“Write Allocate”还减少了一次缓存的写入，能够提升写入的性能**。
  * Read Through 策略就简单一些，它的步骤是这样的:先查询缓存中数据是否存在，如果 存在则直接返回，如果不存在，则由缓存组件负责从数据库中同步加载数据。

  <center><img src='./assets/img/posts/20220414/image-20211121235720744.png'></center>

  * 我们看到 Write Through 策略中写数据库是同步的，这对于性能来说会有比较大的影响， 因为相比于写缓存，同步写数据库的延迟就要高很多了。那么我们可否异步地更新数据库? 这就是我们接下来要提到的“Write Back”策略。

* **Write Back(写回)策略**
  
  * 这个策略的核心思想是在写入数据时只写入缓存，并且把缓存块儿标记为“脏”的。而脏块儿只有被再次使用时才会将其中的数据写入到后端存储中。

  * **需要注意的是，**在“Write Miss”的情况下，我们采用的是“Write Allocate”的方式，也 就是在写入后端存储的同时要写入缓存，这样我们在之后的写请求中都只需要更新缓存即 可，而无需更新后端存储了，我将 Write back 策略的示意图放在了下面:
  <center><img src='./assets/img/posts/20220414/image-20211122000056312.png'></center>

  * **读取数据策略**：我们在读取缓存时如果发现 缓存命中则直接返回缓存数据。如果缓存不命中则寻找一个可用的缓存块儿，如果这个缓存 块儿是“脏”的，就把缓存块儿中之前的数据写入到后端存储中，并且从后端存储加载数据 到缓存块儿，如果不是脏的，则由缓存组件将后端存储中的数据加载到缓存中，最后我们将 缓存设置为不是脏的，返回数据就好了。
  <center><img src='./assets/img/posts/20220414/image-20211122000300636.png'></center>

  * **缓存击穿**
  
    * 可以在程序内部使用定时器，定时从下游数据库中获取最新数据，更新到本地内存缓存中
  
    * 如果说缓存穿透在软件正常运行时有可能发生，那么有一种情况，缓存穿透必定会发生，那就是在服务启动或重启的时候。
  
      可以在本地内存缓存与数据库之间加一层**网络内存缓存**，为秒杀接口服务各节点提供共享缓存。我上面提到的定时任务，就可以从网络内存缓存中获取最新的数据，并更新到本地内存缓存中，避免本地内存缓存穿透。
  
    * **服务端网络内存缓存**
  
      * 将最新的活动信息写入到 Redis 中。另外，为了防止 Redis 挂掉重启后缓存数据丢失，管理后台服务中可以实现一个定时器，定时将已发布的活动配置从数据库中取出，并写入到 Redis 中。
      * 由于管理后台是实时将最新数据写入到 Redis 中的，并且有定时任务作兜底方案，这也保障了在 Redis 正常运行的过程中不会有缓存穿透的问题。
  
* **总结**：

  Cache Aside 是我们在使用分布式缓存时最常用的策略，你可以在实际工作中直接拿来使用。

  Read/Write Through 和 Write Back 策略需要缓存组件的支持，所以比较适合你在实现 本地缓存组件的时候使用;

  Write Back 策略是计算机体系结构中的策略，不过写入策略中的只写缓存，异步写入后 端存储的策略倒是有很多的应用场景。

* 切片集群

  * Redis Cluster方案采用哈希槽(Hash Slot，接下来我会直接称之为Slot)，来处理数据和实例 之间的映射关系。在Redis Cluster方案中，一个切片集群共有16384个哈希槽，这些哈希槽类似于数据分 区，每个键值对都会根据它的key，被映射到一个哈希槽中。

  * 具体的映射过程分为两大步:首先根据键值对的key，按照CRC16算法计算一个16 bit的值;然后，再用这 个16bit值对16384取模，得到0~16383范围内的模数，每个模数代表一个相应编号的哈希槽。

  * 我们在部署Redis Cluster方案时，可以使用cluster create命令创建集群，此时，Redis会自动把这些槽平均 分布在集群实例上。例如，如果集群中有N个实例，那么，每个实例上的槽个数为16384/N个。
  
  当然， 我们也可以使用cluster meet命令手动建立实例间的连接，形成集群，再使用cluster addslots命 令，指定每个实例上的哈希槽个数。

  <center><img src='./assets/img/posts/20220414/image-20211230214756083.png'></center>

  ```java
    redis-cli -h 172.16.19.3 –p 6379 cluster addslots 0,1 
    redis-cli -h 172.16.19.4 –p 6379 cluster addslots 2,3 
    redis-cli -h 172.16.19.5 –p 6379 cluster addslots 4
  ```
   **在手动分配哈希槽时，需要把16384个槽都分配完，否则Redis集群无法正常 工作**。

  * **客戶端如何定位数据?**

    * 一般来说，客戶端和集群实例建立连接后，实例就会把哈希槽的分配信息发给客戶端。但是，在集群刚刚创 建的时候，每个实例只知道自己被分配了哪些哈希槽，是不知道其他实例拥有的哈希槽信息的。Redis实例会 把自己的哈希槽信息发给和它相连接的其它实例，来完成哈希槽分配信息的扩散。当实例之间相互连接后， 每个实例就有所有哈希槽的映射关系了。

  * 在集群中，实例和哈希槽的对应关系并不是一成不变的，最常⻅的变化有两个

    * 在集群中，实例有新增或删除，Redis需要重新分配哈希槽;

      为了**负载均衡**，Redis需要把哈希槽在所有实例上重新分布一遍。

    * 此时，实例之间还可以通过相互传递消息，获得最新的哈希槽分配信息，但是，客戶端是无法主动感知这些 变化的。这就会导致，它缓存的分配信息和最新的分配信息就不一致了

    * Redis Cluster方案提供了一种**重定向机制，**所谓的“重定向”，就是指，客戶端给一个实例发送数据读写操 作时，这个实例上并没有相应的数据，客戶端要再给一个新实例发送操作命令。那么，这个实例就会给客戶端返回下面的MOVED命 令响应结果，这个结果中就包含了新实例的访问地址。

    * ```go
      Get hello:key
      (error) MOVED 13320 ip:port // 13320 代表的是哈希槽
      ```

    * 画一张图来说明一下，MOVED重定向命令的使用方法。可以看到，由于负载均衡，Slot 2中的数据已经从 实例2迁移到了实例3，但是，客戶端缓存仍然记录着“Slot 2在实例2”的信息，所以会给实例2发送命令。 实例2给客戶端返回一条MOVED命令，把Slot 2的最新位置(也就是在实例3上)，返回给客戶端，客戶端就 会再次向实例3发送请求，同时还会更新本地缓存，把Slot 2与实例的对应关系更新过来。
    <center><img src='./assets/img/posts/20220414/image-20211230215349844.png'></center>
    * 需要注意的是，在上图中，当客戶端给实例2发送命令时，Slot 2中的数据已经全部迁移到了实例3。在实际 应用时，如果Slot 2中的数据比较多，就可能会出现一种情况:客戶端向实例2发送请求，但此时，Slot 2中 的数据只有一部分迁移到了实例3，还有部分数据没有迁移。在这种迁移部分完成的情况下，客戶端就会收 到一条ASK报错信息，如下所示:

      ```go
      Get hello:key
      (error) ASK 13320 ip:port
      ```

      这个结果中的ASK命令就表示，客戶端请求的键值对所在的哈希槽13320，在172.16.19.5这个实例上，但是 这个哈希槽正在迁移。此时，客戶端需要先给172.16.19.5这个实例发送一个ASKING命令。这个命令的意思 是，让这个实例允许执行客戶端接下来发送的命令。然后，客戶端再向这个实例发送GET命令，以读取数据。

      **ASK命令**表示两层含义:

      ​	第一，表明Slot数据还在迁移中;

      ​	第二，ASK命令把客戶端所请求数据的最新实例 地址返回给客戶端，此时，客戶端需要给实例3发送ASKING命令，然后再发送操作命令。
      <center><img src='./assets/img/posts/20220414/image-20211230220313412.png'></center>

      * 和MOVED命令不同，**ASK命令并不会更新客戶端缓存的哈希槽分配信息**。所以，在上图中，如果客戶端再次请求Slot 2中的数据，它还是会给实例2发送请求。这也就是说，ASK命令的作用只是让客戶端能给新实例 发送一次请求，而不像MOVED命令那样，会更改本地缓存，让后续所有命令都发往新实例。

## Redis源码分析
  ### Hash
  - HashTable的定义
  <center><img src='./assets/img/posts/20220414/hash_dict.png'></center>

  - 从HashTable的定义可以看到，其中主要的数据结构包括 *dictEntry* 和 已经使用的大小数量，以及和 *Refresh* 相关的字段。其中两个HashTable也是为了Refresh进行准备的。

  - dictEntry的定义：
  <center><img src='./assets/img/posts/20220414/dictEntry.png'></center>

 - Hash扩容流程：
    
    主要函数：*_dictExpandIfNeeded*, *dictExpand*
    ```c
    static int _dictExpandIfNeeded(dict *d)
    {
      /* Incremental rehashing already in progress. Return. */
      if (dictIsRehashing(d)) return DICT_OK;

      /* If the hash table is empty expand it to the initial size. */
      if (DICTHT_SIZE(d->ht_size_exp[0]) == 0) return dictExpand(d, DICT_HT_INITIAL_SIZE);

      /* If we reached the 1:1 ratio, and we are allowed to resize the hash
      * table (global setting) or we should avoid it but the ratio between
      * elements/buckets is over the "safe" threshold, we resize doubling
      * the number of buckets. */
      if (d->ht_used[0] >= DICTHT_SIZE(d->ht_size_exp[0]) &&
          (dict_can_resize ||
          d->ht_used[0]/ DICTHT_SIZE(d->ht_size_exp[0]) > dict_force_resize_ratio) &&
          dictTypeExpandAllowed(d))
      {
          return dictExpand(d, d->ht_used[0] + 1);
      }
      return DICT_OK;
    }
    ```
    从 *_dictExpandIfNeeded* 源码可以看出，hash扩容分为如下情况：

    1.初始化（也可以理解为不是扩容）2.当Hash已经使用的大小大于hash现有大小并且允许扩容 3. 负载因子（ *d->ht_used[0]/ DICTHT_SIZE(d->ht_size_exp[0]* )）的大小大于默认值并且hash类型允许扩容

    *dictExpand* 源码：
    ```c
    int _dictExpand(dict *d, unsigned long size, int* malloc_failed)
    {
      if (malloc_failed) *malloc_failed = 0;

      /* the size is invalid if it is smaller than the number of
      * elements already inside the hash table */
      if (dictIsRehashing(d) || d->ht_used[0] > size)
          return DICT_ERR;

      /* the new hash table */
      dictEntry **new_ht_table;
      unsigned long new_ht_used;
      signed char new_ht_size_exp = _dictNextExp(size);

      /* Detect overflows */
      size_t newsize = 1ul<<new_ht_size_exp;
      if (newsize < size || newsize * sizeof(dictEntry*) < newsize)
          return DICT_ERR;

      /* Rehashing to the same table size is not useful. */
      if (new_ht_size_exp == d->ht_size_exp[0]) return DICT_ERR;

      /* Allocate the new hash table and initialize all pointers to NULL */
      if (malloc_failed) {
          new_ht_table = ztrycalloc(newsize*sizeof(dictEntry*));
          *malloc_failed = new_ht_table == NULL;
          if (*malloc_failed)
              return DICT_ERR;
      } else
          new_ht_table = zcalloc(newsize*sizeof(dictEntry*));

      new_ht_used = 0;

      /* Is this the first initialization? If so it's not really a rehashing
      * we just set the first hash table so that it can accept keys. */
      if (d->ht_table[0] == NULL) {
          d->ht_size_exp[0] = new_ht_size_exp;
          d->ht_used[0] = new_ht_used;
          d->ht_table[0] = new_ht_table;
          return DICT_OK;
      }

      /* Prepare a second hash table for incremental rehashing */
      d->ht_size_exp[1] = new_ht_size_exp;
      d->ht_used[1] = new_ht_used;
      d->ht_table[1] = new_ht_table;
      d->rehashidx = 0;
      return DICT_OK;
    }
    ```
    在 *dictExpand* 中是直接调用的私有函数，从源码可以看到，每次扩容的时候是不停地乘以2直到达到目标大小,因此这个函数里有个计算扩容步伐的重要函数：*_dictNextExp*，同时其中分配内存的操作也是需要重点学习的地方。

    <center><img src='./assets/img/posts/20220414/dict_next_step.png'></center>
  因此，我们可以做一个主要的函数调用流程图：
    <center><img src='./assets/img/posts/20220414/dict_flow_one.png'></center>

### ReHash函数介绍
  dictRehash函数是Redis中时常被用到的函数，因为Redis是单线程的原因，因此在进行扩缩容是Redis是被阻塞住的，这时候其他的命令都是被阻塞的，因此，为了能够避免因为阻塞而影响业务，渐进式ReHash是基础数据结构中的重中之重。这里，主要介绍了rehash函数的步骤。
  ```cpp
  int dictRehash(dict *d, int n) {
    int empty_visits = n * 10; //避免一直是空bucket，导致阻塞
    if (!dictIsRehashing(d)) return 0;

    while (n -- && d->ht_used[0] != 0) {
      dictEntry *de, *nextde;

      assert(DICTHT_SIZE(d->ht_size_exp[0]) > (unsigned long)d->rehashidx);
      while(d->ht_table[0][d->rehashidx] == NULL) { // 找到第一个不为空的bucket，避免一直阻塞
        d->rehashidx++;
        if(--empty_visits == 0) return 1; //多次为空则直接返回1
      }
      de = d->ht_table[0][d->rehashidx];
      while(de) { // 解决链式冲突，将同一个bucket下的所有节点rehash到table[1]
        uint64_t h;

        nextde = de->next;
        h = dictHashKey(d, de->key) & DICTHT_SIZE_MASK(d->ht_size_exp[1]); // 计算新的hashIndex
        de->next = d->ht_table[1][h];
        d->ht_table[1][h] = de;
        d->ht_used[0]--;
        d->ht_used[1]--;
        de = nextde;
      }
      d->ht_table[0][d->rehashidx] = NULL; 
      d->rehashidx++;
    }
    if (d->ht_used[0] == 0) {
      zfree(d->ht_table[0]); //ht_table[0] 和 ht_table[1]替换
      d->ht_table[0] = d->ht_table[1];
      d->ht_used[0] = d->ht_used[1];
      _dictReset(d, 1); //重置ht[1]
      d->rehashidx = -1;
      return 0;
    }
    return 1;
  }
  ```
  该函数得调用链路如下：
   <center><img src='./assets/img/posts/20220414/dict_rehash.png'></center>











