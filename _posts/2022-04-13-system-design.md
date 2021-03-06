---
layout: post
read_time: true
show_date: true
title: "system design"
desc: "常见系统架构的学习和总结"
date: 2021-03-24
img: posts/20210324/starting_adventure.jpg
tags: [system design]
author: TM
description: "系统架构设计"
---
## 系统设计
### 秒杀常见问题以及解决方案

* 高并发的解决方案：

  * **降低和分散流量**
    * **nginx做负载均衡**
    * **资源静态化，把前端的模板页面放到CDN服务器中**
    * **页面的按钮，按一次后置灰X秒**（防止一直用户一直点,同一个用户重复点击，虽然不会再卖给他，但是请求还是会到后端给系统压力，需要在前端按钮上做限制，比如点一下限制五秒内不能点击。
    * **同一个uid，限制访问频度，做页面缓存，x秒内到达站点层的请求，均返回同一页面**（用来防止其他程序员跳过按钮，直接用for循环不断发起http请求，具体的话可以请求每次进来都去redis查有没有对应的id的key值，没有就在redis中设置X秒过期的key）
    * **读请求用redis集群顶住**
    * 预热
    * **对于写请求，用消息队列**（比如商品有一万件，就放一万个请求进来，当然要做好每秒几个的限制，不能一秒内全放进来，都成功了就继续放下一批，没成功就剩下的请求全部返回失败）

* 链接暴露的解决方案

  * **在秒杀时间到的时候，才能获得url**。

    * ①页面中有一个计时模块，是访问秒杀页面的时候去从服务器里拿的，计时结束，显示秒杀的按钮。    **问题：（为什么不直接取用户的时间？）**用户的本机时间是可以修改的。
      ②点击秒杀按钮后，再次请求服务器时间，与秒杀的时间对比，如果秒杀进行中，返回一个由加密过的秒杀url  **问题：（为什么还要再次请求服务器时间？怎么加密url?）**

      虽然第一次请求到了服务器时间，但是时间的倒计时操作是页面来完成的，比如在几天前就打开这个秒杀页面等待倒计时，等秒杀开始时可能与服务器时间存在几秒的误差。

      通过一个无序的String字符串，也就是我们常说的盐值字符串，进行MD5加密得到哈希值，在步骤②中点击秒杀按钮返回url的时候，返回这个MD5值进行url的拼接。

      ③通过这个加密过的url来进行支付、减库存操作。

* 恶意请求的解决方案

  * **怎么限制让一个人只能秒杀一件商品**
    * 数据库肯定有一张订单表（包含用户id字段、商品id字段），只要每次去查一下**用户的id和这件商品的id**有没有在这张表里就行了，存在说明该用户买过这个商品了，就不给买了。
    * 一般支付成功扣款和把订单信息加入到订单表中这两个操作设为一个事务。
       把**用户id**和**商品id**作为**联合主键索引**存储到数据库中，下次如果再想插入一样用户id和商品id的行，是会报错的。（主键索引的唯一性）,事务中出现错误，支付操作自然也失败了

* 数据库层面的解决方案

  * **用消息队列来削峰**
    * 比如一秒钟进来1W个写请求，我们数据库只能顶住一秒5000个，那我们就每秒放出来4000个。
  * **用缓存来顶住大量的查询请求**
    * 引入redis，如果redis中有要查询的数据，就直接返回，如果没有，从数据库查询的时候，把查询的结果放到redis中，以后查询都会落到redis层。
  * **数据库层面读写分离**
    * 设置主从数据库，主数据库负责写，从数据库负责读，可以极大程度的缓解X锁和S锁争用。

* 超卖问题的解决方案

  * **悲观锁**

    * 将减少库存的操作加入到事务中，（注意此时数据库的引擎需要是Innodb的只有innodb才支持事务和行级锁，而事务和行级锁这两个概念，一般也是同时出现），此时一个线程开始修改这一行的时候，会先获得排他锁，获得锁后开始修改，没有获得锁的线程会阻塞。
    * **可以思考下先做insert还是update**？
       答案应该是**先insert而后update**，因为insert的锁的行不会有人竞争，而update的排它锁的行会出现大量竞争，而**事务锁释放的时机是整个事务完成，而不是这个方法执行完毕的时候**，所以需要后update，尽量减少库存行锁的获取时间，来提高并发效率。
  * **乐观锁**
  
    * **需要在数据库表中加入版本号字段**，sql语句如下，先查询出版本号，在下次更新的时候通过判断版本号是否更改和库存是否不为0来决定是否update。这种方式采用了版本号的方式，其实也就是**CAS**的原理。
  * **redis**
    * 利用redis的单线程预减库存

## Redis支撑秒杀场景的关键技术和实践[总结](https://time.geekbang.org/column/intro/100056701)
 - 秒杀场景的特征：
    - 瞬时并发量且量级很高
    - 读多写少，且读操作较简单
- 秒杀场景分析
  - 秒杀前
    - 秒杀前用户会不断的刷新商品详情页，这样会导致详情页的访问量暴增，因此通常情况下会把商品详情页元素静态化，然后使用CDN或者浏览器把这些静态元素缓存起来。这样一来，秒杀前的大量请求可以直接由CDN或是浏览器缓存服务，不会到达服务器端了。
  - 秒杀中
    - 秒杀中主要的操作也是核心操作为：查库存、库存扣减以及订单处理。其中查库存和库存扣减需要放在redis中进行处理，订单处理可以放到数据库侧进行处理。但是，由于并发量的问题，如何保证查库存和库存扣减是原子性是解决秒杀问题的关键。至于订单处理，因为会过滤掉很多请求，此时数据库侧也能够解决该量级了。
    - 如果库存扣减的操作放到数据库执行，会带来两个问题：

      1. 额外开销。Redis中保存了库存量，而库存量的最新值又是数据库在维护，所以数据库更新后，还需要 和Redis进行同步，这个过程增加了额外的操作逻辑，也带来了额外的开销
      2. 下单量超过实际库存量，出现超售。由于数据库的处理速度较慢，不能及时更新库存余量，这就会导致 大量库存查验的请求读取到旧的库存值，并进行下单。此时，就会出现下单数量大于实际的库存量，导 致出现超售，这就不符合业务层的要求了。
    - 如果不能保证查询库存和扣减库存的原子性，也会导致超买的问题。因此，这也是秒杀设计中的核心要点。
  - 秒杀后
    - 秒杀后的场景请求量已经能够正常处理了，如订单撤回、退款或者其他操作在此就不再展开。

**秒杀场景下的核心问题**
  - 高并发
    - 对于高并发问题，如果有多个秒杀商品，我们也可以使用切片集群，用不同的实例保存不同商品的库存，这样就避免，使用单个实例导致所有的秒杀请求都集中在一个实例上的问题。当使用切片集群时，我们要先用CRC算法计算不同秒杀商品key对应的Slot，然后，我们在分配Slot和实例对应关系时，才能把不同秒杀商品 对应的Slot分配到不同实例上保存。
  - 原子性
    - 原子性的保证可以通过Redis原生命令、分布式锁以及Lua脚本实现。由于查询库存和扣减库存不同通过一个命令实现，因此，我们只能够通过分布式锁以及Lua脚本实现。
    - Lua脚本Demo
    
    ```lua
    #获取商品库存信息
    local count = redis.call("MGET", KEYS[1], "total", "ordered");

    local total = tonumber(count[1]);

    local ordered = tonumber(count[2]);

    if ordered + k <= total then
      redis.call("HINCRBY", KEYS[1], "ordered", k);
      return k;
    end
    return 0;
    ```
    - 分布式锁
      1. 客户端向Redis申请分布式锁
      2. 只有拿到锁的客户端才能进行秒杀操作
    
  ``` java
  //使用商品ID作为key
  key = itemID
  //使用客戶端唯一标识作为value
  val = clientUniqueID
  //申请分布式锁，Timeout是超时时间
  lock = acquireLock(key, val, Timeout)
  //当拿到锁后，才能进行库存查验和扣减
  if (lock == true) {
    // 查验库存扣减后的结果
    availSock = DECR(key, k)
    // 如果结果小于0， 则直接返回扣减失败
    if (availSock < 0) {
      releaseLock(key, val)
      return error
    }
    // 扣减成功后继续执行后续步骤
    else {
      releaseLock(key, val)
      //订单处理
    }
  }
  else {
    return 
  }
  ```
## 秒杀中需要注意的其他事项
- **前端静态⻚面的设计:** 秒杀⻚面上能静态化处理的⻚面元素，我们都要尽量静态化，这样可以充分利用 CDN或浏览器缓存服务秒杀开始前的请求。
- **请求拦截和流控:** 在秒杀系统的接入层，对恶意请求进行拦截，避免对系统的恶意攻击，例如使用黑名单禁止恶意IP进行访问。如果Redis实例的访问压力过大，为了避免实例崩溃，我们也需要在接入层进行限流，控制进入秒杀系统的请求数量。
- **库存信息过期时间处理:** Redis中保存的库存信息其实是数据库的缓存，为了避免缓存击穿问题，我们不要给库存信息设置过期时间。
- **数据库订单异常处理:** 如果数据库没能成功处理订单，可以增加订单重试功能，保证订单最终能被成功处理。

## Feeds流系统设计
  - 新鲜事系统设计 News Feed 典型的有Facebook、Twitter和朋友圈之后看到的信息流
  - 核心因素：1、关注与被关注 2、每个人看到的新鲜事都是不同的
  - 读扩散（pull）与写扩散（push）

   - 读扩散： 
    1. 算法：用户查看News Feed时，获取每个好友的前100条信息，然后合并出前100条News Feed。使用的算法：K路归并算法 Merge K Sorted Array
    2. 复杂度分析：假如有N个关注对象，则为N次DB Reads的时间 + K路归并时间（可忽略）。N次用户DB Reads非常慢，且发生在用户获得News Feed的请求过程中。
    3. 简单的表格主要包含：1、FriendShipTable 2、Tweet Table。首先，用户通过web服务器查询自己关注的用户ID，后来根据用户ID去查询该用户的Tweet信息，然后对所有的信息进行合并，抽选前N条的信息进行展示。

   