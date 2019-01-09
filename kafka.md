# 什么是kafka
Kafka是一个分布式发布 - 订阅消息系统和一个强大的队列，可以处理大量的数据，并使您能够将消息从一个端点传递到另一个端点。 Kafka适合离线和在线消息消费。 Kafka消息保留在磁盘上，并在群集内复制以防止数据丢失。
优势：高吞吐率，高性能，多重订阅，消息持久性，良好的伸缩性，高可用。<br>
- [kafka官网](http://kafka.apache.org/documentation/)

# kafka监控
### Kafka Manager or Kafka Eagle(国人开源) 两种监控工具
- [Eagle](http://www.cnblogs.com/smartloli/p/5829395.html)

# 快速上手kafka
### 创建一个名为“test”的topic(主题)，它只包含一个分区，只有一个副本
bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test<br>

### 查看topic
bin/kafka-topics.sh --list --zookeeper localhost:2181<br>

### 生产者，它将从文件或标准输入中获取输入，并将其作为消息发送到Kafka集群。默认情况下，每行将作为单独的消息发送
bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test<br>

### 消费者，它会将消息转储到标准输出
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning<br>

### 修改topic,修改分区数
bin/kafka-topics.sh --zookeeper localhost:2181 --alter --topic test --parti-tions 2<br>

### 删除topic
bin/kafka-topics.sh --zookeeper localhost:2181 --delete --topic test<br>

# 集群
复制 server.properties 文件重命名(server-1.properties,server-2.properties,server-3.properties)<br>

zookeeper.connect=hostname1:port,hostname2:port,hostname3:port<br>
log.dir=/tmp/kafka-1.logs<br>
port=9093<br>
集群文件里不同的配置<br>
broker.id=1  #id是群集中每个节点的唯一且永久的名称<br>
listeners=127.0.0.1:9093  #list<br>

### 启动集群的单节点
bin/kafka-server-start.sh config/server-1.properties &<br>
bin/kafka-server-start.sh config/server-2.properties &<br>
bin/kafka-server-start.sh config/server-3.properties &<br>

### 创建集群的topic(主题)，因为有3个代理，所以分配3个副本值
bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 3 --partitions 1 --topic clust_test<br>

### 现在我们有了一个集群，我们怎么知道哪个节点(经纪人)在做什么呢?
bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic my-replicated-topic<br>
输出:<br>
Topic:my-replicated-topic   PartitionCount:1    ReplicationFactor:3 Configs:<br>
Topic:my-replicated-topic   Partition: 0        Leader: 1   Replicas: 1,2,0 Isr: 1,2,0<br>

"leader"是负责给定分区的所有读写的节点。每个节点将成为随机选择的分区部分的领导者。<br>
"Replicas"是复制此分区日志的节点列表，无论它们是否为领导者，或者即使它们当前处于活动状态。<br>
"Isr"是同步复制品的集合。这是副本列表的子集，该列表当前处于活跃状态并且已经被领导者捕获。<br>

# topic(主题)配置:[官网](http://kafka.apache.org/documentation/#topicconfigs)
### 创建topic时配置属性
bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 3 --partitions 1 --topic test --config max.message.bytes=64000 #设置最大信息数<br>
### 修改，覆盖
bin/kafka-configs.sh --zookeeper localhost:2181 --entity-type topics --entity-name test --alter --add-config max.message.bytes=128000<br>
### 删除覆盖
bin/kafka-configs.sh --zookeeper localhost:2181 --entity-type topics --entity-name test --alter --delete-config max.message.bytes<br>
