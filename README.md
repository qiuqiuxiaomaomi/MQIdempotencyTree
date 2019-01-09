# MQIdempotencyTree
消息幂等性技术研究


<pre>
消息中间件一般由三部分组成：
     1）Producer
     2) Consumer
     3) Broker（消息存储）

     为了保证消息的可达性，采取了超时重传机制，但是这可能导致消息总线或者业务方收到重复消息。
</pre>

![](https://i.imgur.com/mHKjKQd.png)

<pre>
上半场流程：
     1）发送端MQ-Client 将消息发给服务端MQ-Server
     2) 服务端MQ-Server将消息落地
     3）服务端MQ-Server回ACK给发送端 MQ-Client

     如果 3）丢失，发送端MQ-Client超时后会重新发送，可能MQ-Server重复收到消息。

     为了避免消息重复发送，MQ系统内部必须要有一个内部消息ID，作为去重和幂等的依据，这个内部
     消息ID的特性是：
         1）全局唯一
         2）MQ生成，具备业务无关性。

     这样才能保证幂等
</pre>

<pre>
下半场流程：
     4）服务端MQ-Server将消息发送给接收端MQ-Client
     5) 接收端MQ-Client回ACK给服务端
     6）服务端MQ-Server将落地消息删除

     如果 5）丢失，服务端MQ-Server超时后重复发送消息，可能导致MQ-Client收到重复的消息。

     为保证业务幂等性，必须有一个biz-id，作为去重和幂等的依据，这个业务ID的特性是：
         1）对于同一个业务场景，全局唯一
         2）有业务消息发送发生成，业务相关，对MQ透明
         3）有业务消息消费方负责判重，以保证幂等。

     常见的如订单ID，支付ID等
</pre>

<pre>
MQ为了保证消息必达，消息上下半场均可能发送重复消息，如何保证消息的幂等性呢？

    上半场
         MQ-client生成inner-msg-id，保证上半场幂等。
         这个ID全局唯一，业务无关，由MQ保证。

    下半场
         业务发送方带入biz-id，
         业务接收方去重保证幂等。
         这个ID对单业务唯一，业务相关，对MQ透明。

     结论：幂等性，不仅对MQ有要求，对业务上下游也有要求
</pre>


<pre>
幂等的处理方式

      1）查询与删除是天然幂等的。
      2）唯一索引，防止新增脏数据。
      3）悲观锁for update
      5) 乐观锁 CAS 版本号，判断条件等。
      6）分布式锁
      7）状态机幂等，如果状态机已经处于下一个状态，这时候来了一个上一个状态的变更，理论是通不过的，这样的话，保证了有限状态机
         的幂等。
</pre>

<pre>
消息去重

      去重原则：
              1）幂等性  2）业务去重
      幂等性：
            无论这个业务请求被执行多少次，数据库的结构都是唯一的，不可改变的。
      去重策略：
            1：建立一个消息表，拿到这个消息做数据库的insert操作，给这个消息做一个唯一的主键
               或者唯一约束，那么就算出现重复消费的情况，就会导致主键冲突。
      高并发下去重：
            采用Redis去重，key天然支持原子性并要求不可重复，但是由于不再一个事务，要求有适当
            的补偿策略。

            2：利用Redis事务，主键（必须把全量的操作数据都存放在redis里，然后定时去和数据库）
               数据同步，即消费处理后，该处理本来应该保存在数据库的，先保存在Redis。
            3：利用Redis和关系型数据库一起做去重机制。
            5：拿到这个消息做Redis的set操作，Redis就是天然幂等性
            6：准备一个第三方介质来做消费处理，以Redis为例，给消息分配一个全局ID，只要消费国该消息，
               将<id, message>以K-V形式写入Redis，那消费者开始消费前，先去Redis中查询有没有消费记录即可。
</pre>

![](https://i.imgur.com/zHVZGyH.png)

![](https://i.imgur.com/yHVYwu1.png)

<pre>
Kafka幂等性

      Kafka Producer 在实现时有以下两个重要机制：
            PID（Producer ID），用来标识每个 producer client；
            sequence numbers，client 发送的每条消息都会带相应的 sequence number，Server 端就是
            根据这个值来判断数据是否重复。

      PID：
          每个 Producer 在初始化时都会被分配一个唯一的 PID，这个 PID 对应用是透明的，完全没有暴露给
          用户。对于一个给定的 PID，sequence number 将会从0开始自增，每个 Topic-Partition 都会有一
          个独立的 sequence number。Producer 在发送数据时，将会给每条 msg 标识一个 sequence 
          number，Server 也就是通过这个来验证数据是否重复。这里的 PID 是全局唯一的，Producer 故障后
          重新启动后会被分配一个新的 PID，这也是幂等性无法做到跨会话的一个原因。
</pre>

