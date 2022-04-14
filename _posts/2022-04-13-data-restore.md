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


