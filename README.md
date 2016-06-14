# Kafka
搭建Kafka运行环境。
接下来一步一步搭建Kafka运行环境。
Step 1: 下载Kafka点击下载最新的版本并解压.
> tar -xzf kafka_2.9.2-0.8.1.1.tgz
> cd kafka_2.9.2-0.8.1.1
Step 2: 启动服务
Kafka用到了Zookeeper，所有首先启动Zookper，下面简单的启用一个单实例的Zookkeeper服务。可以在命令的结尾加个&符号，这样就可以启动后离开控制台。
> bin/zookeeper-server-start.sh config/zookeeper.properties &
[2013-04-22 15:01:37,495] INFO Reading configuration from: config/zookeeper.properties (org.apache.zookeeper.server.quorum.QuorumPeerConfig)
...

现在启动Kafka:

> bin/kafka-server-start.sh config/server.properties &
[2013-04-22 15:01:47,028] INFO Verifying properties (kafka.utils.VerifiableProperties)
[2013-04-22 15:01:47,051] INFO Property socket.send.buffer.bytes is overridden to 1048576 (kafka.utils.VerifiableProperties)
...

Step 3: 创建 topic
创建一个叫做“test”的topic，它只有一个分区，一个副本。

> bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test
可以通过list命令查看创建的topic:
> bin/kafka-topics.sh --list --zookeeper localhost:2181
test

除了手动创建topic，还可以配置broker让它自动创建topic.
Step 4:发送消息.
Kafka 使用一个简单的命令行producer，从文件中或者从标准输入中读取消息并发送到服务端。默认的每条命令将发送一条消息。

运行producer并在控制台中输一些消息，这些消息将被发送到服务端：
> bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test 
This is a messageThis is another message
ctrl+c可以退出发送。

Step 5: 启动consumer
Kafka also has a command line consumer that will dump out messages to standard output.
Kafka也有一个命令行consumer可以读取消息并输出到标准输出：

> bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic mytest --from-beginning
This is a message
This is another message

你在一个终端中运行consumer命令行，另一个终端中运行producer命令行，就可以在一个终端输入消息，另一个终端读取消息。
这两个命令都有自己的可选参数，可以在运行的时候不加任何参数可以看到帮助信息。
 
Step 6: 搭建一个多个broker的集群
刚才只是启动了单个broker，现在启动有3个broker组成的集群，这些broker节点也都是在本机上的：
首先为每个节点编写配置文件：
 
> cp config/server.properties config/server-1.properties
> cp config/server.properties config/server-2.properties
在拷贝出的新文件中添加以下参数：

config/server-1.properties:
    broker.id=1
    port=9093
    log.dir=/tmp/kafka-logs-1
 
config/server-2.properties:
    broker.id=2
    port=9094
    log.dir=/tmp/kafka-logs-2

broker.id在集群中唯一的标注一个节点，因为在同一个机器上，所以必须制定不同的端口和日志文件，避免数据被覆盖。
 
We already have Zookeeper and our single node started, so we just need to start the two new nodes:
刚才已经启动可Zookeeper和一个节点，现在启动另外两个节点：
> bin/kafka-server-start.sh config/server-1.properties & 
...
> bin/kafka-server-start.sh config/server-2.properties &
...

创建一个拥有3个副本的topic:
> bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 3 --partitions 1 --topic my-replicated-topic
现在我们搭建了一个集群，怎么知道每个节点的信息呢？运行“"describe topics”命令就可以了：
> bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic my-replicated-topic 
Topic:my-replicated-topic       PartitionCount:1        ReplicationFactor:3     Configs:
        Topic: my-replicated-topic      Partition: 0    Leader: 1       Replicas: 1,2,0 Isr: 1,2,0

下面解释一下这些输出。第一行是对所有分区的一个描述，然后每个分区都会对应一行，因为我们只有一个分区所以下面就只加了一行。


	* leader：负责处理消息的读和写，leader是从所有节点中随机选择的.
	* replicas：列出了所有的副本节点，不管节点是否在服务中.
	* isr：是正在服务中的节点.

在我们的例子中，节点1是作为leader运行。
向topic发送消息：
> bin/kafka-console-producer.sh --broker-list localhost:9092 --topic my-replicated-topic
...
my test message 1my test message 2^C 

消费这些消息：

> bin/kafka-console-consumer.sh --zookeeper localhost:2181 --from-beginning --topic my-replicated-topic
...
my test message 1
my test message 2
^C
测试一下容错能力.Broker 1作为leader运行，现在我们kill掉它：

> ps | grep server-1.properties7564 ttys002    0:15.91 /System/Library/Frameworks/JavaVM.framework/Versions/1.6/Home/bin/java...
> kill -9 7564
另外一个节点被选做了leader,node 1 不再出现在 in-sync 副本列表中：

> bin/kafka-topics.sh --describe --zookeeper localhost:218192 --topic my-replicated-topic
Topic:my-replicated-topic       PartitionCount:1        ReplicationFactor:3     Configs:
        Topic: my-replicated-topic      Partition: 0    Leader: 2       Replicas: 1,2,0 Isr: 2,0

虽然最初负责续写消息的leader down掉了，但之前的消息还是可以消费的：
> bin/kafka-console-consumer.sh --zookeeper localhost:2181 --from-beginning --topic my-replicated-topic
...
my test message 1
my test message 2
^C
看来Kafka的容错机制还是不错的。



删除

bin/kafka-topics.sh --zookeeper 172.16.66.60:2182 --delete --topic datatest

