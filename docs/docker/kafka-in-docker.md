## 📖 使用DockerCompose创建Kafka集群

Apache Kafka 是一个快速、可扩展、高吞吐量、容错的分布式消息传递系统。 名为 Producer 的一方将消息流发送到另一方名为 Consumer 的 Kafka 消费。 具有高吞吐、内置分区、支持数据复制和容错等特点，适用于大规模消息处理场景。

这里我们介绍一下Kafka的基础知识以及如何使用Docker搭建集群。

### 📌 Kafka History

关于Kafka的历史成因，以下内容都是道听途说，未有考究。

> Kafka是一种分布式消息发布订阅系统，具有高性能、高吞吐的特点，广泛应用于大数据传输场景。 它由 LinkedIn 开发，用 Scala 语言编写，后来成为 Apache 基金会的顶级项目。Kafka是LinkedIn的中枢神经系统，管理着各种应用的聚合，数据经过处理后才分发到其他地方。 Kafka不同于传统的企业信息排队系统。 它以近乎实时的方式处理流经公司的所有数据。 目前服务于LinkedIn、Netflix、Uber、Verizon，并为此建立了实时信息处理平台。
>

### 📌  Kafka Concepts

+ Topic

  + Kafka 中的消息订阅和发送是基于某个主题的。

  + 主题就像一个特定主题的收件箱，生产者把它扔进去，消费者把它拿走。

+ Partition

  + 分区是主题的物理分组。一个topic可以分为多个partition，每个partition就是一个有序队列。

  + 通过分区提供了吞吐量，也实现了高可用。

+ Broker
  + Broker 是 Kafka 集群或服务单元中的一个实例，连接到同一个 zookeeper 的多个 broker 实例形成一个 Kafka 集群
  + 在几个broker中，一个broker是leader，其余的都是follower
  + 集群启动时选举出leader，负责与外界的通信。当leader死亡时，followers会再次通过选举，选出新的leader，保证集群的正常运行。

+ replica
  + 每个分区会有多个数据副本，保证Kafka的高可用。 
  + 每个partition都有自己的replica，其中只有一个是leader副本，其余都是follower副本。
  + 当消息进来时，会先存储在leader副本中，然后再从leader副本复制到follower副本
  + 消费者只有在所有复制完成后才能消费该消息。这是为了确保在发生事故时可以恢复数据
  + 消费者也是从Leader副本中读取的

+ Zookeeper
  + Kafka 使用 Zookeeper 来管理集群

+ Producers and Consumers
  + 生产者和消费者完全解耦并且彼此不可知，这是实现 Kafka 众所周知的高可扩展性的关键设计元素
  + Kafka Consumer 订阅 Kafka 主题，并主动从 Kafka Broker 拉取消息进行处理
  + 每个 Kafka Consumer 都维护着上次读取消息的偏移量，下次从这个偏移量开始请求消息。这种基于拉取的机制大大降低了 Broker 的压力，使得 Kafka Broker 的吞吐率很高。

+ Consumer Group
  + Kafka支持Consumer Group，多个Consummer共同读取同一个topic中的数据，提高数据读取效率。
  + 同一个Consumer Group下的不同Consumer对同一个topic的消息消费不会重复。
  + Kafka可以自动为同组的Consumer分配负载，实现消息的并发读取，当一个Consumer发生故障时，它会自动将其处理的分区转移给同组的其他Consumer进行处理。

+ Offset
  + 每个分区由一系列有序且不可变的消息组成，这些消息依次附加到分区上
  + 分区中的每条消息都有一个连续的序列号，称为“偏移量”，分区用它来唯一标识一条消息。
  + 每个分区中的消息从0开始记录消息。

### 📌 Kafka Design features

#### 🦊 Persistence

__关于持久化，请参考官方文档：[Here](https://kafka.apache.org/documentation/#persistence) __

> **Don’t fear the filesystem!**
>
> There is a general perception that “disks are slow” which makes people skeptical that a persistent structure can offer competitive performance. In fact disks are both much slower and much faster than people expect depending on how they are used; and a properly designed disk structure can often be as fast as the network.

Kafka线性访问磁盘，在很多情况下比随机内存访问快很多，有利于持久化。传统的形式是使用内存作为磁盘缓存。 Kafka 直接将数据写入日志文件，并以 append 的形式写入。

因此，基于文件持久化形式，Kafka具有以下优势：

+ 读操作不会阻塞写操作和其他操作。因为读写都是追加的形式，所以是顺序的，不会造成乱序和阻塞。数据大小不影响性能
+ 无容量 受限于建立在硬盘上的消息系统。
+ 对磁盘的线性访问和快速读写，并且可以存储任意时间段。

#### 🦊 Efficiency

__关于高效率的介绍，请参考官方文档：[Here](https://kafka.apache.org/documentation/#maximizingefficiency) __

+ 顺序读写磁盘
  + Kafka 的生产者生产数据需要写入日志文件。
  + 写入过程是追加到文件末尾进行顺序写入。
  + 由于省内头部寻址次数多，顺序写入速度很快。

+ 零拷贝
  + 零拷贝不是不必要的拷贝，而是减少不必要的拷贝数量。
  + 零拷贝是指无需应用程序直接将数据从磁盘文件拷贝到网卡设备，减少内核和用户态的上下文切换
  + Producer产生的数据持久化到broker，通过MMAP文件映射实现顺序快速写入。
  + Consumer从broker读取数据，使用“send file” 将磁盘文件读入OS内核缓冲区，然后直接传输到socket缓冲区进行网络传输。

+ 批量压缩
  + Kafka 使用批量压缩来提高压缩效率。
  +  Kafka 允许使用递归消息集合，批量消息可以以压缩形式传输，也可以在日志中保持压缩状态，直到被消费者解压。
  +  Kafka支持多种压缩协议，包括Gzip和Snappy压缩协议

Kafka高效的秘诀在于它将所有消息都变成了一个批处理文件，并进行合理的批处理压缩以减少网络IO损失，并通过MMAP提高I/O速度。

### 📌 Build the Cluster

#### 🦊 脚本结构

```shell
D:.
│  docker-compose.yml
│  
├─kafka1
│  └─data
│                  
├─kafka2
│  └─data
│                  
└─kafka3
    └─data
```

#### 🦊 启动集群

`docker-compose.yml` 脚本如下所示：

```yaml
version: '3.0'

# The network is created by the zookeeper cluster.
networks:
  zookeeperdemo_zookeeper:
    external: true

services:

  kafka1:
    image: wurstmeister/kafka
    restart: unless-stopped
    container_name: kafka1
    ports:
      - "9093:9092"
    external_links:
      - zookeeper1
      - zookeeper2
      - zookeeper3
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ADVERTISED_HOST_NAME: 10.0.75.2                   
      KAFKA_ADVERTISED_PORT: 9093                                 
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://10.0.75.2:9093 
      KAFKA_ZOOKEEPER_CONNECT: "zookeeper1:2181,zookeeper2:2181,zookeeper3:2181"
    volumes:
      - D:\kafka1\data:/kafka
    networks:
      - zookeeperdemo_zookeeper

  kafka2:
    image: wurstmeister/kafka
    restart: unless-stopped
    container_name: kafka2
    ports:
      - "9094:9092"
    external_links:
      - zookeeper1
      - zookeeper2
      - zookeeper3
    environment:
      KAFKA_BROKER_ID: 2
      KAFKA_ADVERTISED_HOST_NAME: 10.0.75.2                 
      KAFKA_ADVERTISED_PORT: 9094                              
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://10.0.75.2:9094  
      KAFKA_ZOOKEEPER_CONNECT: "zookeeper1:2181,zookeeper2:2181,zookeeper3:2181"
    volumes:
      - D:\kafka2\data:/kafka
    networks:
      - zookeeperdemo_zookeeper

  kafka3:
    image: wurstmeister/kafka
    restart: unless-stopped
    container_name: kafka3
    ports:
      - "9095:9092"
    external_links:
      - zookeeper1
      - zookeeper2
      - zookeeper3
    environment:
      KAFKA_BROKER_ID: 3
      KAFKA_ADVERTISED_HOST_NAME: 10.0.75.2                 
      KAFKA_ADVERTISED_PORT: 9095                             
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://10.0.75.2:9095  
      KAFKA_ZOOKEEPER_CONNECT: "zookeeper1:2181,zookeeper2:2181,zookeeper3:2181"
    volumes:
      - D:\kafka3\data:/kafka
    networks:
      - zookeeperdemo_zookeeper

  kafka-manager:
    image: sheepkiller/kafka-manager:latest
    restart: unless-stopped
    container_name: kafka-manager
    hostname: kafka-manager
    ports:
      - "9000:9000"
    links:          
      - kafka1
      - kafka2
      - kafka3
    external_links:   
      - zookeeper1
      - zookeeper2
      - zookeeper3
    environment:
      ZK_HOSTS: zookeeper1:2181,zookeeper2:2181,zookeeper3:2181 
      LOGGER_STARTUP_TIMEOUT: 60s               
      TZ: CST-8
    networks:
      - zookeeperdemo_zookeeper
```

启动容器：

```shell
docker-compose up -d --build
```

查看容器：

```shell
CONTAINER ID        IMAGE                              COMMAND                  CREATED             STATUS              PORTS                                                      NAMES
05sd43yjn9sa        sheepkiller/kafka-manager:latest   "./start-kafka-man..."   9 minutes ago       Up 9 minutes        0.0.0.0:9000->9000/tcp                                     kafka-manager
68bsd791dsd7        wurstmeister/kafka                 "start-kafka.sh"         9 minutes ago       Up 9 minutes        0.0.0.0:9094->9092/tcp                                     kafka2
3f7ghji8680f        wurstmeister/kafka                 "start-kafka.sh"         9 minutes ago       Up 9 minutes        0.0.0.0:9093->9092/tcp                                     kafka1
8t7fg236hjk73        wurstmeister/kafka                 "start-kafka.sh"         9 minutes ago       Up 9 minutes        0.0.0.0:9095->9092/tcp                                     kafka3
```

### 📌 UI visualization

`docker-compose.yml`文件有一个Kafka-manager模块，就是Kafka的可视化管理模块。 因为Kafka的元数据和配置信息是由Zookeeper管理的，现在我们可以在UI管理器中进行配置。

打开浏览器，输入`http://127.0.0.1:9000`，打开管理页面，通过输入集群名称和 zookeeper 主机来添加集群。

+ Show1

  配置后可以看到默认有一个topic。 topic分为50个partition，用于记录Kafka的消费offset。

  <div align=center>
  <img width="800" src="/docker/img/kafkamanager1.png"/>
  </div>

+ Show2

  搭建了的集群有三个实例，所以broker的数量是3。

  <div align=center>
  <img width="800" src="/docker/img/kafkamanager2.png"/>
  </div>

### 📌 Summary

Kafka是一款经典的消息中间件，可以用于日志系统解耦、异步消息处理、日志聚合等应用场景。目前支持多种语言的SDK，使用较为广泛。通过此文字记录，可以学到如下技术要点：

1. Kafka的基本概念
2. Kafka的设计特征
3. Kafka的集群搭建
4. Docker和DockerCompose相关技术知识
5. Kafka集群的管理和可视化

