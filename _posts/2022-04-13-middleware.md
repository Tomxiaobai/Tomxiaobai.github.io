---
layout: post
read_time: true
show_date: true
title: "Middleware"
date: 2021-04-20
img: posts/20210420/post8-rembrandt.jpg
tags: [kafka, rocketmq, rabbitmq, elasticsearch]
category: opinion
author: TM
description: "消息中间件的学习记录"
---
## MQ
## Kafka
  
  * 在 Kafka 中，Topic 是一个存储消息的逻辑概念，可以认为是一个消息集合。每条消息发送到 Kafka 集群的消息都有一个类别。物理上来说，不同的 Topic 的消息是分开存储的，每个 Topic 可以有多个生产者向它发送消息，也可以有多个消费者去消费其中的消息。
  
  * 每个 Topic 可以划分多个分区（每个 Topic 至少有一个分区），同一 Topic 下的不同分区包含的消息是不同的。每个消息在被添加到分区时，都会被分配一个 offset，它是消息在此分区中的唯一编号，Kafka 通过 offset 保证消息在分区内的顺序，offset 的顺序不跨分区，即 Kafka 只保证在同一个分区内的消息是有序的。消息是每次追加到对应的 Partition 的后面
  
    <center><img src='https://img-blog.csdnimg.cn/2019011811343340.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RvbmdndWFiYWk=,size_16,color_FFFFFF,t_70'></center>

  * 一条消息其实是由 Key + Value 组成的，在发送一条消息时，我们可以指定这个 Key，那么 Producer 会根据 Key 和 partition 机制来判断当前这条消息应该发送并存储到哪个 partition 中（这个就跟分片机制类似）。我们可以根据需要进行扩展 Producer 的 partition 机制。
  
  * 如果 Consumer 数量比 partition 数量多，会有的 Consumer 闲置无法消费，这样是一个浪费。如果 Consumer 数量小于 partition 数量会有一个 Consumer 消费多个 partition。**Kafka 在 partition 上是不允许并发的**。**Consuemr 数量建议最好是 partition 的整数倍。 还有一点，如果 Consumer 从多个 partiton 上读取数据，是不保证顺序性的，Kafka 只保证一个 partition 的顺序性，跨 partition 是不保证顺序性的。增减 Consumer、broker、partition 会导致 Rebalance**
  
  * 在 Kafka 中，同一个 Group 中的消费者对于一个 Topic 中的多个 partition 存在一定的分区分配策略。
  
    * **Range strategy**（**范围分区**）
  
      * Range 策略是对每个主题而言的，首先对同一个主题里面的分区按照序号进行排序，并对消费者按照字母顺序进行排序。假设我们有10个分区，3个消费者，排完序的分区将会是0,1,2,3,4,5,6,7,8,9；消费者线程排完序将会是C1-0, C2-0, C3-0。然后将 partitions 的个数除于消费者线程的总数来决定每个消费者线程消费几个分区。**如果除不尽，那么前面几个消费者线程将会多消费一个分区。**
  
        假如在 Topic1 中有 10 个分区，3 个消费者线程，10/3 = 3，而且除不尽，那么消费者线程 C1-0 将会多消费一个分区，所以最后分区分配的结果是这样的：
  
        C1-0 将消费 0,1,2,3 分区
        C2-0 将消费 4,5,6 分区
        C3-0 将消费 7,8,9 分区
  
        假如在 Topic1 中有 11 个分区，那么最后分区分配的结果看起来是这样的：
  
        C1-0 将消费 0,1,2,3 分区
        C2-0 将消费 4, 5, 6, 7 分区
        C3-0 将消费 8,9,10 分区
  
        假如有两个 Topic：Topic1 和 Topic2，都有 10 个分区，那么最后分区分配的结果看起来是这样的：
  
        C1-0 将消费 Topic1 的 0,1,2,3 分区和 Topic2 的 0,1,2,3 分区
        C2-0 将消费 Topic1 的 4,5,6 分区和Topic2 的 4,5,6 分区 
        C3-0 将消费 Topic1 的 7,8,9 分区和Topic2 的 7,8,9 分区
  
        其实这样就会有一个问题，C1-0 就会多消费两个分区，这就是一个很明显的弊端。
        
  
    * **RoundRobin strategy(轮询分区)**
  
      * 轮询分区策略是把所有 partition 和所有 Consumer 线程都列出来，然后按照 hashcode 进行排序。最后通过轮询算法分配partition 给消费线程。如果所有 Consumer 实例的订阅是相同的，那么 partition 会均匀分布。
  
        假如按照 hashCode 排序完的 Topic / partitions组依次为
  
        T1-5, T1-3, T1-0, T1-8, T1-2, T1-1, T1-4，T1-7, T1-6, T1-9
  
        消费者线程排序为 C1-0，C1-1，C2-0，C2-1，
  
        最后的分区分配的结果为：
  
        C1-0 将消费 T1-5, T1-2, T1-6分区
        C1-1 将消费 T1-3, T1-1, T1-9分区
        C2-0 将消费 T1-0, T1-4分区
        C2-1 将消费 T1-8, T1-7分区
  
        使用**轮询分区策略**必须满足两个条件
        1.**每个主题的消费者实例具有相同数量的流**
        2.**每个消费者订阅的主题必须是相同的**
  
    * **什么时候会触发这个策略**
  
      * **同一个 Consumer group内新增了消费者**
        **消费者离开当前所属的 Consumer group，比如主动停机或者宕机**
        **Topic 新增了分区（也就是分区数量发生了变化）**
      * **Consumer group 的分区分配方案是在客户端执行的！Kafka 将这个权利下放给客户端主要是因为这样做可以有更好的灵活性。**
  
* Sendfile 零拷贝 、mmap和DMA：
  
  * 相关链接： [DMA](https://blog.csdn.net/z69183787/article/details/104760890)
  * 总结： DMA利用java nio的相关知识，直接将数据根据socket描述符信息发送至网卡处。
  
* **相关面试题[kafka](https://developer.aliyun.com/article/740170)**