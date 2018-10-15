# kafka消息队列集群

kafka集群连接信息: 
10.19.53.222:9092,10.19.91.16:9092,10.19.36.184:9092

kafka需要连接到Zookpper服务: 
10.19.53.222:2181,10.19.91.16:2181,10.19.36.184:2181

## kafka消息队列集群环境部署

### 1. 服务器环境

|IP            |Broker id |Version           |`KFk_HOME`
|:-------------|:---------|:-----------------|:---------
|10.19.53.222  |1         |kafka 2.11-2.0.0  |`/opt/app/kafka/kafka_2.11-2.0.0`
|10.19.91.16   |2         |kafka 2.11-2.0.0  |`/opt/app/kafka/kafka_2.11-2.0.0`
|10.19.36.184  |3         |kafka 2.11-2.0.0  |`/opt/app/kafka/kafka_2.11-2.0.0`

### 2. kafka基本配置

将kafka2.0.0源码包解压到`/opt/app/kafka/`目录下，并创建`logs`日志目录

```bash
$ mkdir /data/app
$ ln -s /data/app /opt/app

$ tar -zxf kafka_2.11-2.0.0.tg -C /opt/app/kafka/
$ mkdir /opt/app/kafka/kafka_2.11-2.0.0/logs
```

修改config/server.properties配置文件，使用vi或vim打开server.properties，增加或修改如下配置

```
broker.id=1
listeners=PLAINTEXT://10.19.53.222:9092
log.dirs=/data/app/kafka/kafka_2.11-2.0.0/logs
log.retention.hours=168
message.max.bytes=5242880
default.replication.factor=2
replica.fetch.max.bytes=5242880
zookeeper.connect=10.19.53.222:2181,10.19.91.16:2181,10.19.36.184:2181
zookeeper.connection.timeout.ms=6000
```

需要注意的是，在另外两个节点仅需要修改`broker.id`和`listeners`，分别对应节点ID和节点的监听信息

#### In 节点2
```
broker.id=2
listeners=PLAINTEXT://10.19.91.16:9092
```

#### In 节点3
```
broker.id=3
listeners=PLAINTEXT://10.19.36.184:9092
```

### 3. 以后台运行方式启动kafka

在所有kafka节点启动kafka服务
```bash
$ cd /opt/app/kafka/kafka_2.11-2.0.0/
$ ./bin/kafka-server-start.sh -daemon config/server.properties
```

查看jps进程信息
```bash
$ jps
12657 Jps
12565 Kafka
12072 QuorumPeerMain
```

## 创建topic消息队列测试

### 创建消息队列
在任意节点使用以下命令创建topic消息队列

```bash
$ cd /opt/app/kafka/kafka_2.11-2.0.0/bin/
$ ./kafka-topics.sh --create --replication-factor 2 --partitions 3 --topic my-topic --zookeeper 10.19.53.222:2181,10.19.91.16:2181,10.19.36.184:2181
```

```
--replication-factor 2 // 复制两份
--partitions 3         // 创建3个分区
--topic                // 主题为my-topic
--zookeeper            // 此处为zookeeper监听地址
```

### 创建生产者

```bash
$ ./kafka-console-producer.sh --topic my-topic --broker-list 10.19.53.222:9092,10.19.91.16:9092,10.19.36.184:9092
```

### 创建消费者

```bash
$ ./kafka-console-consumer.sh --topic my-topic --bootstrap-server 10.19.53.222:9092,10.19.91.16:9092,10.19.36.184:9092 --from-beginning
```

### 查看消息队列状态

```bash
$ ./kafka-topics.sh --list --zookeeper 10.19.53.222:2181,10.19.91.16:2181,10.19.36.184:2181
$ ./kafka-topics.sh --describe --zookeeper 10.19.53.222:2181 --topic my-topic
```

### 示例

```
[root@10-19-53-222 bin]# ./kafka-console-producer.sh --topic my-topic --broker-list 10.19.53.222:9092,10.19.91.16:9092,10.19.36.184:9092
>hello
>

[root@10-19-91-16 bin]# ./kafka-console-consumer.sh --topic my-topic --bootstrap-server 10.19.53.222:9092,10.19.91.16:9092,10.19.36.184:9092
hello

[root@10-19-91-16 bin]# ./kafka-topics.sh --describe --zookeeper 10.19.53.222:2181 --topic my-topic
Topic:my-topic	PartitionCount:3	ReplicationFactor:2	Configs:
	Topic: my-topic	Partition: 0	Leader: 1	Replicas: 1,2	Isr: 1,2
	Topic: my-topic	Partition: 1	Leader: 2	Replicas: 2,3	Isr: 2,3
	Topic: my-topic	Partition: 2	Leader: 3	Replicas: 3,1	Isr: 3,1

[root@10-19-91-16 bin]# ./kafka-topics.sh --list --zookeeper 10.19.53.222:2181,10.19.91.16:2181,10.19.36.184:2181
__consumer_offsets
my-topic
```
