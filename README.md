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

