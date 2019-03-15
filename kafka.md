kafka
* [什么是kafka](##什么是kafka)
* [topic主题配置](##topic(主题)配置)

## 什么是kafka
Kafka是一个分布式发布 - 订阅消息系统和一个强大的队列，可以处理大量的数据，并使您能够将消息从一个端点传递到另一个端点。 Kafka适合离线和在线消息消费。 Kafka消息保留在磁盘上，并在群集内复制以防止数据丢失。<br>
优势：高吞吐率，高性能，多重订阅，消息持久性，良好的伸缩性(动态扩展)，高可用。<br>
Kafka设计之初是为日志处理而生，给人们留下了数据可靠性要求不高的不良印象，但是随着版本的升级优化，其可靠性得到极大的增强。<br>
- [kafka官网](http://kafka.apache.org/documentation/)

## kafka应用场景
- 1.消息服务器
- 2.网站活动跟踪
- 3.实时数据流聚合
- 4.日志聚合

# 消息队列中间件的性能、功能选择与对比
- 金融支付领域使用 RabbitMQ 居多，而在日志处理、大数据等方面 Kafka 使用居多。
- 如果要在功能和性能方面做一个抉择的话，那么首选性能，因为总体上来说性能优化的空间没有功能扩展的空间大。然而看长期发展，生态又比性能以及功能都要重要。
- 消息堆积分内存式堆积和磁盘式堆积<br>
RabbitMQ 是典型的内存式堆积，但这并非绝对，在某些条件触发后会有换页动作来将内存中的消息换页到磁盘(换页动作会影响吞吐)，或者直接使用惰性队列来将消息直接持久化至磁盘中。<br>
Kafka 是一种典型的磁盘式堆积，所有的消息都存储在磁盘中。一般来说，磁盘的容量会比内存的容量要大得多，对于磁盘式的堆积其堆积能力就是整个磁盘的大小。<br>
- 消息幂等性<br>
确保消息在生产者和消费者之间进行传输，一般有三种传输保障(Delivery Guarantee)：<br>
At most once，至多一次，消息可能丢失，但绝不会重复传输。<br>
At least once，至少一次，消息绝不会丢，但是可能会重复。<br>
Exactly once，精确一次，每条消息肯定会被传输一次且仅一次。<br>
<br>
>对于大多数消息中间件而言，一般只提供 At most once 和 At least once 两种传输保障，对于第三种一般很难做到，由此消息幂等性也很难保证。
Kafka 自 0.11 版本开始引入了幂等性和事务，Kafka 的幂等性是指单个生产者对于单分区单会话的幂等。
>而事务可以保证原子性地写入到多个分区，即写入到多个分区的消息要么全部成功，要么全部回滚，这两个功能加起来可以让 Kafka 具备EOS(ExactlyOnceSemantic)的能力。

## kafka监控
### Kafka Manager、Kafka Eagle(国人开源)、Kafka Monitor、Kafka Offset Monitor、Burrow、Chaperone、Confluent Control Center 等产品，尤其是 Cruise 还可以提供自动化运维的功能。
- [Eagle](http://www.cnblogs.com/smartloli/p/5829395.html)

## Kafka Python client
- [kafka-python](https://github.com/dpkp/kafka-python)

## 快速上手kafka
### 创建一个名为“test”的topic(主题)，它只包含一个分区，只有一个副本
    bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test

### 查看topic
    bin/kafka-topics.sh --list --zookeeper localhost:2181

### 生产者，它将从文件或标准输入中获取输入，并将其作为消息发送到Kafka集群。默认情况下，每行将作为单独的消息发送
    bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test

### 消费者，它会将消息转储到标准输出
    bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning

### 修改topic,修改分区数
    bin/kafka-topics.sh --zookeeper localhost:2181 --alter --topic test --parti-tions 2

### 删除topic
    bin/kafka-topics.sh --zookeeper localhost:2181 --delete --topic test

## 集群
复制 server.properties 文件重命名(server-1.properties,server-2.properties,server-3.properties)<br>

zookeeper.connect=hostname1:port,hostname2:port,hostname3:port<br>
log.dir=/tmp/kafka-1.logs<br>
port=9093<br>
集群文件里不同的配置<br>
broker.id=1  #id是群集中每个节点的唯一且永久的名称<br>
listeners=127.0.0.1:9093  #list<br>

### 启动集群的单节点
    bin/kafka-server-start.sh config/server-1.properties &
    bin/kafka-server-start.sh config/server-2.properties &
    bin/kafka-server-start.sh config/server-3.properties &

### 创建集群的topic(主题)，因为有3个代理，所以分配3个副本值
    bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 3 --partitions 1 --topic clust_test

### 现在我们有了一个集群，我们怎么知道哪个节点(经纪人)在做什么呢?
    bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic my-replicated-topic
    输出:
    Topic:my-replicated-topic   PartitionCount:1    ReplicationFactor:3 Configs:
    Topic:my-replicated-topic   Partition: 0        Leader: 1   Replicas: 1,2,0 Isr: 1,2,0

"leader"是负责给定分区的所有读写的节点。每个节点将成为随机选择的分区部分的领导者。<br>
"Replicas"是复制此分区日志的节点列表，无论它们是否为领导者，或者即使它们当前处于活动状态。<br>
"Isr"是同步复制品的集合。这是副本列表的子集，该列表当前处于活跃状态并且已经被领导者捕获。<br>

## topic(主题)配置
- [主题配置官网](http://kafka.apache.org/documentation/#topicconfigs)
### 创建topic时配置属性
    bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 3 --partitions 1 --topic test --config max.message.bytes=64000 #设置最大信息数
### 修改，覆盖
    bin/kafka-configs.sh --zookeeper localhost:2181 --entity-type topics --entity-name test --alter --add-config max.message.bytes=128000
### 删除覆盖
    bin/kafka-configs.sh --zookeeper localhost:2181 --entity-type topics --entity-name test --alter --delete-config max.message.bytes

## 配置
num.io.threads=8                        #线程数<br>
num.partitions=1                        #每个topic分配个数<br>
num.recovery.threads.per.data.dir=1<br>
log.dirs=/tmp/kafka=logs                #多个日志用逗号分开<br>
log.retention.hours=168                 #kafka中消息的存储时间，这里是168小时<br>
log.segment.bytes=1073741824            #segment大小<br>
#log.retention.check.interval.ms=300000 #默认没有配置，如果配置则根据设置的大小，会根据写入时间删除至设置大小的要求<br>
num.network.threads=3                   #broker网络请求的最大线程数，设置cpu核数<br>
socket.send.buffer.bytes=102400         #设置buffer缓冲区大小，设置为-1则为系统默认数<br>
socket.receive.buffer.bytes=102400      #socket接收缓冲区大小，设置为-1则为系统默认数<br>
socket.request.max.bytes=104857600      #设置socket请求的最大字节数<br>
