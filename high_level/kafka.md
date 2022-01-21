


### producer、consumer、broker、partition、group

broker：kafka 集群中，一个kafka实例被称为一个代理(Broker)节点。

roducer：是消息生产者，向 kafka broker 发消息。

consumer：是消息消费者，向 kafka broker 取消息。

partition：每一个 topic 可以有一个或者多个分区(partition)。partition 中的每条消息都会被分配一个有序的id（offset），
消费时kafka 只保证按一个 partition 中的消息的顺序，不保证多个 partition 间的顺序。每个partition在同一时间只被一个 consumer 消费。
consumer 可以把 offset 调成一个较老的值，去重新消费老的消息。

group：可扩展且具有容错性的消费者机制。将消费者分组，一个message只会被一个group消费一次，group之间是相互独立的。

### consumer 是推还是拉

拉。推对于对于不同消费速率的 consumer 就不太好处理。

### 好处

 - 解耦：可以独立修改生产者、消费者。
 - 缓冲、峰值处理能力：生产和消费速度不一致、顶住突发流量的压力。
 - 异步通信：生产者、消费者异步。

### Kafka中，ZooKeeper的作用
存放meta数据，成员管理等

### 与传统MQ区别
 - kafka基于持久化日志，可以反复读取。
 - kafka是一个分布式系统：它以集群的方式运行，可以灵活伸缩，在内部通过复制数据。提升容错能力和高可用性。
 - kafka支持实时的流式处理。

### offset提交

消费者可以自动提交偏移量，或者手动提交，将 enable.auto.commit 设为 false。

### kafka吞吐量为何如此高

 - 顺序读写：每个partition是一个文件，收到消息插入到文件末尾。每个partition还有offset表示消费到哪里。
 - 零拷贝：利用sendfile，数据不会经过用户空间，减少了数据复制，还减少了上下文切换的次数。（[参考](https://www.cnblogs.com/zlcxbb/p/6411568.html)）
 - 分区：每个topic分成多个partition，partition内存可以再分段。
