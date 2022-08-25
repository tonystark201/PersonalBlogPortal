## 📖 使用DockerCompose创建Zookeeper集群

ZooKeeper是一个开源的分布式协调服务，是大数据生态系统使用的有组织服务的标准。因此，本篇博文主要介绍zookeeper的基础知识以及如何使用Docker容器搭建集群。

### 📌 Zookeeper history

关于Zookeeper的历史成因，以下内容都是道听途说，未有考究。

> 众所周知，分布式系统中的很多问题都是由于缺乏协调机制造成的。对于雅虎这样的大公司来说，他们也面临着同样的困境。ZooKeeper 最初起源于雅虎研究院的一个研究小组。当时雅虎内部的很多大型系统基本上都是依赖类似的系统进行分布式协调，但是这些系统往往存在分布式单点问题。
>
> 因此，雅虎开发者尝试开发一个没有单点问题的通用分布式协调框架，让开发者可以专注于处理业务逻辑。在技术领先方面，无论哪个方面都有 Google 的存在，当然在分布式协调技术方面，Google 推出了 Chubby，但 Chubby 是非开源的。
>
> 幸运的是，雅虎将 ZooKeeper 作为开源程序捐赠给了 Apache，为我们日常软件开发带来了便利。由于当时很多自研项目都以动物命名，雅虎研究院首席科学家拉古·拉马克里希南开玩笑说：“如果这样下去，我们的地方就会变成动物园！”。这就是“Zookeeper”这个名字的由来。

### 📌  Zookeeper Mechanism

Zookeeper 是一个基于观察者模式设计的分布式服务管理框架。它负责存储和管理大家关心的数据，然后接受观察者的注册。一旦这些数据的状态发生变化，Zookeeper 将负责通知那些在 Zookeeper 上注册的观察者做出相应的响应。

#### 🐼 基本特性

__*以下特性的介绍均来自于官方文档*__

+ __简单的数据模型__

  > Zookeeper 使分布式程序通过共享的树形结构的命名空间相互协调。Zookeeper 服务器内存中的数据模型由一系列称为“ZNodes”的数据节点组成，Zookeeper 存储了全部的数据量。内存中的数据，以提高服务器吞吐量并减少延迟。

+ __集群化__

  > 一个 Zookeeper 集群通常由一组机器组成，每台机器在内存中维护当前的服务器状态，每台机器之间相互通信。

+ __顺序访问__

  > 对于客户端的每一次更新请求，Zookeeper 都会分配一个全局唯一的增量号，它反映了所有事务操作的顺序。

+ __高性能__

  > Zookeeper 将全量数据存储在内存中，并直接服务于来自客户端的所有非事务性请求。

#### 🐼 选举机制

__Zookeeper 集群是“一主多从”的模式。主人是领导者，奴隶是跟随者。领导者是通过选举获得的。__

Leader选举是保证分布式数据一致性的关键。 Zookeeper进入以下两种状态时，需要进入leader选举。第一个是“服务器初始化”，另一个是leader宕机。

服务器初始化时的选举会经过以下步骤：

1. 集群中的节点相互通信，每台机器都试图找到leader，因此进入选举状态
2. 每个节点都会投票给自己作为领导者，然后将这个投票发送给集群中的其他节点
3. 集群中的每台服务器收到投票后，首先判断投票的有效性。如果通过有效性检查，对于每一个投票，服务器需要将其他人的投票与自己的投票进行比较
4. 每次投票后，节点都会统计投票信息，以确定是否有超过一半的机器收到了相同的投票信息。
5. 一旦确定了Leader，每个服务器都会更新自己的状态。如果是追随者，则改为“FOLLOWING”，如果是领导者，则改为“LEADING”。
6. 有关共识机制的信息，请阅读技术专家 Lamport 的 Paxos 大作。

> Leslie B. Lamport（生于 1941 年 2 月 7 日）是美国计算机科学家。 Lamport 以其在分布式系统方面的开创性工作而闻名。Leslie Lamport 是 2013 年图灵奖的获得者，因为他对分布式计算系统看似混乱的行为施加了清晰、定义明确的一致性。

#### 🐼 观察者

__Zookeeper 是一个成熟的分布式协调框架，“发布/订阅”模式非常重要，这种模式被称为观察者模式。__观察者会订阅一些感兴趣的话题，然后一旦这些话题发生变化，就会自动通知这些观察者。

Zookeeper 的“发布/订阅”模式是观察机制，是一种轻量级的设计。因为它采用了“推拉”的组合方式。服务器一旦感知到主题变化，只会向相关客户端发送事件类型和节点信息，而不会包含变化的具体内容。然后，接收到变更通知的客户端需要自己拉取变更后的数据。

1. Zookeeper 节点可以被监控，包括节点数据修改和子节点变化。
2. Watch 机制是一次性的。一个事件监听完成后，如果需要继续监听，需要重新注册。
3. Watch 通知事件是异步的，但 Zookeeper 保证客户端在看到 Watch 事件之前不会看到节点数据变化。
4. Zookeeper 中的所有读取操作都可以设置为监控。

### 📌 Build the Cluster

#### 🐼 脚本结构

```shell
D:.
│  docker-compose.yml
│  
├─zookeeper1
│  ├─data        
│  └─datalog
│              
├─zookeeper2
│  ├─data      
│  └─datalog
│              
└─zookeeper3
    ├─data         
    └─datalog
```

#### 🐼 启动集群

`docker-compose.yml` 脚本如下所示：

```yaml
version: '3.0'

services:

  zookeeper1:
    image: zookeeper
    restart: unless-stopped
    hostname: zookeeper1
    container_name: zookeeper1
    ports:
      - 2182:2181
    environment:
      ZOO_MY_ID: 1
      ZOO_SERVERS: server.1=0.0.0.0:2888:3888;2181 server.2=zookeeper2:2888:3888;2181 server.3=zookeeper3:2888:3888;2181
    volumes:
      - D:\zookeeper1\data:/data
      - D:\zookeeper1\datalog:/datalog
    networks:
      zookeeper:
        ipv4_address: 172.32.0.3

  zookeeper2:
    image: zookeeper
    restart: unless-stopped
    hostname: zookeeper2
    container_name: zookeeper2
    ports:
      - 2183:2181
    environment:
      ZOO_MY_ID: 2
      ZOO_SERVERS: server.1=zookeeper1:2888:3888;2181 server.2=0.0.0.0:2888:3888;2181 server.3=zookeeper3:2888:3888;2181
    volumes:
      - D:\zookeeper2\data:/data
      - D:\zookeeper2\datalog:/datalog
    networks:
      zookeeper:
        ipv4_address: 172.32.0.4

  zookeeper3:
    image: zookeeper
    restart: unless-stopped
    hostname: zookeeper3
    container_name: zookeeper3
    ports:
      - 2184:2181
    environment:
      ZOO_MY_ID: 3
      ZOO_SERVERS: server.1=zookeeper1:2888:3888;2181 server.2=zookeeper2:2888:3888;2181 server.3=0.0.0.0:2888:3888;2181
    volumes:
      - D:\zookeeper3\data:/data
      - D:\zookeeper3\datalog:/datalog
    networks:
      zookeeper:
        ipv4_address: 172.32.0.5

networks:
   zookeeper:
      ipam:
         config:
         - subnet: 172.32.0.0/16
```

启动容器：

```shell
docker-compose up -d --build
```

查看容器：

```shell
6952sef6rrt1        zookeeper                          "/docker-entrypoin..."   17 hours ago        Up 17 hours         2888/tcp, 3888/tcp, 8080/tcp, 0.0.0.0:2183->2181/tcp       zookeeper2
356fsgh59sh3        zookeeper                          "/docker-entrypoin..."   17 hours ago        Up 17 hours         2888/tcp, 3888/tcp, 8080/tcp, 0.0.0.0:2184->2181/tcp       zookeeper3
bs5d557b6dc5        zookeeper                          "/docker-entrypoin..."   17 hours ago        Up 17 hours         2888/tcp, 3888/tcp, 8080/tcp, 0.0.0.0:2182->2181/tcp       zookeeper1
```

### 📌 Usage scenarios

+ 协调

  分布式系统中机器之间的典型通信方法是心跳。心跳检测是指在分布式环境中，不同的机器需要检测彼此是否正常运行。传统的方法是通过主机之间相互“PING”来实现。 ZooKeeper，根据其临时节点的特性，不同的机器在ZooKeeper的指定节点下创建临时子节点，不同的机器可以根据这个临时节点判断客户端机器是否存活。其优点是通过ZooKeeper节点关联，大大降低了系统的耦合度。

+ 分布式锁

  分布式锁是一种控制对分布式系统之间共享资源的同步访问的方法。

  使用zookeeper分布式锁，一台机器收到请求后，首先获取zookeeper上的分布式锁，即可以创建一个“Znode”然后执行操作，那么另一台机器无法创建那个“Znode”，只能等待执行第一个任务。

+ 集群管理

  集群管理主要指集群监控和集群控制两个方面，实现收集集群的状态以及对集群的操作和控制。面对集群，我们通常想知道集群中有多少台机器在工作，收集集群中每台机器的运行状态数据，对集群中的机器进行上下操作。Zookeeper 可用于许多系统的集群管理。比如Kafka、Storm、Hadoop等很多分布式系统都会使用zookeeper进行配置信息管理、节点监控等。

+ 高可用性

  高可用可以解决分布式单点问题。 Zookeeper基于临时文件和Watch机制，可以帮助很多分布式服务避免单一故障，实现高可用。

+ 配置中心

  基于Zookeeper的数据“发布/订阅”模式，可以实现配置中心的应用。发布者将数据发布到ZooKeeper节点，供订阅者订阅数据，达到动态获取数据的目的。

+ 负载均衡

  您可以自定义负载均衡算法，根据 Zookeeper 的临时文件目录和监听器来实现服务负载均衡。

+ 选举

  使用 ZooKeeper 创建节点 API 接口提供了强一致性，保证在分布式高并发中节点的创建必须是全局唯一的。

### 📌 Summary

因为 ZooKeeper 可以保证分布式环境中数据的强一致性，并且基于这个特性，越来越多的分布式系统使用 ZooKeeper 作为核心组件。

