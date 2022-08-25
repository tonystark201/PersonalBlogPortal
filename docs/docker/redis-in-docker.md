## 📖 Redis的几种部署测试探究

通过使用Redis缓存，可以很方便地使用多种数据结构来实现复杂的服务。 本文主要展示Redis集群的部署模式，并进行相关代码演示，以更好地理解Redis集群的概念、搭建和使用场景。

### 🧱 Why use Redis

> 在传统的互联网业务中，当流量增加时，DB负载增加，导致响应缓慢变慢，甚至存在宕机风险。 这时候就需要加一层缓存，避免频繁读取数据库。 缓存是 Redis 最受欢迎的用例之一，因为 Redis 完美地满足了应用程序分布式缓存层的要求。

### 🧱 Deploy Strategy

#### Client sharding

+ 优点

  + 所有逻辑可控，不依赖第三方分布式中间件。 
  + 开发人员知道如何实现分片和路由规则。

+ 缺点

  + 这是一种静态分片模式。 
  + 当需要增加或减少 Redis 实例数量时，需要在程序中手动调整分片程序。
  + 任何操作和维护不善都需要运维人员和开发人员的配合。
  + 在不同的客户端程序中，维护相同的数据分区逻辑成本很高。

+ 架构

    <div align=center>
    <img width="500" src="/docker/img/redis-client-sharding.png"/>
    </div>

#### Server sharding

+ 优势

  + twemproxy几乎没有性能损失，并且读写分离可以大大提高集群的响应能力。
  + Twemproxy通过透明连接池、内存零拷贝、epoll模型实现高速轻量化。
  + 连接池维护了前端连接数，减少了后端连接数，大大降低了后端Redis节点的负载。
  + Twemproxy 通过一致的哈希算法拆分数据。
  + Twemproxy 支持 Redis 和 Memcache 集群的构建。
  + Twemproxy 支持多种算法实现哈希分片的一致性，包括 crc32、crc16、MD5 等。
  + 代理逻辑和存储逻辑是隔离的

+ 缺点

  + 存在导致缓存不可用的单点故障风险，需要使用HAProxy等工具来实现高可用
  + 操作维护不友好
  + 中心化
  + 无法平滑扩展/收缩

+ 架构

    <div align=center>
    <img width="600" src="/docker/img/redis-server-sharding.png"/>
    </div>

#### Redis Sentinel

+ 优点

  + Redis Sentinel 集群部署简单，拥有主从模式的所有优点。
  + 可以解决Redis主从模式下的高可用切换问题。
  + 非常方便的实现Redis数据节点的线性扩展，轻松突破Redis本身的单线程瓶颈
  + 可以实现一组 Sentinel 来监控一组 Redis 数据节点或多组数据节点

+ 缺点

  + Redis 很难支持在线扩容，当集群容量达到上限时，在线扩容会变得非常复杂

+ 架构

  Sentinel 的作用类似于 zookeeper ，主要以主从模式监控Redis，实现健康检查、主从切换等。

    <div align=center>
    <img width="600" src="/docker/img/redis-sentinel.png"/>
    </div>

#### Redis Cluster

+ 优点
  + 去中心化
  + 数据按照hash slot存储和分布在多个节点，数据在节点间共享，数据分布可以动态调整。
  + 节点可以线性扩展至1000个节点，节点可以动态添加或删除。
  + 高可用性
  + 节点通过 gossip 协议交换状态信息，利用投票机制完成从 Slave 到 Master 的角色升级
  + 降低运维成本，提高系统可扩展性和可用性

+ 缺点

  + 客户端实现复杂，驱动需要Smart Client的实现，hash slot映射信息及时缓存更新，增加了开发难度
  + 数据是异步复制的，无法保证数据的强一致性
  + 资源隔离差
  + slave完全冷备，不能做读操作
  + MultiOp 和 Pipeline 可能不再有效
  + 数据迁移是基于key而不是hash slot，迁移的过程比较慢

+ 架构

  Redis Cluster的基本原理可以从数据共享、数据迁移、集群通信、故障检测、故障转移等方面来理解。

  + Redis 集群中的节点负责存储数据和记录集群状态。 
  + 集群节点可以自动发现其他节点，检测节点状态，并在需要时移除故障节点，并提升新的主节点。 
  + 客户端可以从任何节点获取数据。

    <div align=center>
    <img width="600" src="/docker/img/redis-cluster.png"/>
    </div>

### 🧱 Deploy practice

#### Twemproxy

+ 项目解构

```shell
D:.
│  docker-compose.yml
│  
├─redis
│      dockerfile
│      redis.conf
│      
└─twemproxy
        dockerfile
        nutcracker.yml
```

+ DockerCompose 脚本

```yaml
version: '3.0'
services:
  redis1:
    container_name: r1
    build: redis
    ports: 
      - "7000:6379"
    networks:
         twem_redis:
            ipv4_address: 172.30.0.3
            
  redis2:
    container_name: r2
    build: redis
    ports: 
      - "7001:6379"
    networks:
         twem_redis:
            ipv4_address: 172.30.0.4
            
  redis3:
    container_name: r3
    build: redis
    ports: 
      - "7002:6379"
    networks:
         twem_redis:
            ipv4_address: 172.30.0.5
  twem:
    container_name: twem
    build: twemproxy
    ports: 
      - "11211:11211"
      - "22121:22121"
      - "22122:22122"
      - "22123:22123"
    volumes:
      - D:\twemproxy\nutcracker.yml:/usr/local/twemproxy/conf/nutcracker.yml
    networks:
         twem_redis:
            ipv4_address: 172.30.0.2
  
networks:
   twem_redis:
      ipam:
         config:
         - subnet: 172.30.0.0/16
```

+ Redis Dockerfile脚本(`dockerfile`)

```dockerfile
FROM redis:5.0.3-alpine3.9
MAINTAINER tsz201 <xxxxxx@xxx.com>
EXPOSE 6379
COPY redis.conf /etc/redis/redis.conf
ENTRYPOINT redis-server /etc/redis/redis.conf
```

+ Redis 配置文件(`redis.conf`)

```shell
bind 0.0.0.0
port 6379
daemonize no
save 900 1
save 300 10
save 60 10000
appendonly yes
```

+ twemproxy Dockerfile文件(`dockerfile`)

```dockerfile
FROM ubuntu:20.04
MAINTAINER tsz201 <xxxxxx@xxxxxx.com>
RUN apt-get update && apt-get install -y libtool make automake wget unzip psmisc lsof

RUN wget https://github.com/twitter/twemproxy/archive/master.zip && \
	unzip master.zip -d /home && \
	rm master.zip && \
	mv home/twemproxy-master/ home/twemproxy
	
WORKDIR /home/twemproxy
RUN autoreconf -fvi && \
	./configure --prefix=/usr/local/twemproxy && \
	make && make install
	
WORKDIR /usr/local/twemproxy/
RUN echo "PATH=$PATH:/usr/local/twemproxy/sbin/" >> /etc/profile && \
	mkdir conf run && \
	cp -r /home/twemproxy/conf/nutcracker.yml /usr/local/twemproxy/conf/ && \
	echo killall nutcracker > stop.sh && chmod 777 stop.sh

RUN /bin/bash -c "source /etc/profile"

RUN apt-get remove -y libtool make automake unzip 
EXPOSE 3000 22121 22122 22123 22124
CMD ["./sbin/nutcracker","-c","/usr/local/twemproxy/conf/nutcracker.yml"]
```

+ twemproxy 配置(`nutcracker.yml`)

```yaml
redis:                                   
  listen: 0.0.0.0:22121
  hash: fnv1a_64
  hash_tag: "{}"
  distribution: ketama
  auto_eject_hosts: false
  timeout: 400
  redis: true
  servers:                           
   - 172.30.0.3:6379:1
   - 172.30.0.4:6379:1
   - 172.30.0.5:6379:1
```

注意事项：

+ 我们定制了一个桥接网络，并指定了每个服务的网络地址和端口映射。（具体可参考Docker官方文档）
+ 在编写twemproxy的dockerfile时，使用的是ubuntu的基础镜像，不需要安装psmisc和lsof工具。 以上脚本安装了这些工具来查看`nutcracker `进程是否启动成功，并使用`kill all`来清除它。
+ twemproxy可参考官方文档进行相关的配置

容器启动后，Redis服务映射到主机的三个端口分别是7000、7001和7002。

可以使用Redis客户端工具插入1000条数据，可以看到数据被分派到不同的 Redis 实例:

```
redis1:0>dbsize
"280"
redis2:0>dbsize
"500"
redis3:0>dbsize
"220"
```

#### Redis Sentinel

+ 项目结构

  ```
  D:.
  │  docker-compose.yml
  │  
  ├─conf
  │      master.conf
  │      sentinel.conf
  │      sentinel1.conf
  │      sentinel2.conf
  │      sentinel3.conf
  │      slave1.conf
  │      slave2.conf
  │      
  ├─redis
  │     dockerfile
  │      
  └─sentinel
        dockerfile
  ```

+ DockerCompose脚本(`docker-compose.yml`)

  ```yaml
  version: '3'
  services:
    master:
      build: redis
      container_name: redis-master
      network_mode: host
      volumes:
        - D:\conf\master.conf:/etc/redis/redis.conf
    
    slave1:
      build: redis
      container_name: redis-slave-1
      network_mode: host
      command: redis-server --slaveof 10.0.75.2 6380 
      volumes:
        - D:\conf\slave1.conf:/etc/redis/redis.conf
     
    slave2:
      build: redis
      container_name: redis-slave-2
      network_mode: host
      command: redis-server --slaveof 10.0.75.2 6380 
      volumes:
        - D:\conf\slave2.conf:/etc/redis/redis.conf
     
    sentinel1:
      build: sentinel
      container_name: redis-sentinel-1
      network_mode: host
      volumes:
        - D:\conf\sentinel1.conf:/etc/redis/sentinel.conf
    
    sentinel2:
      build: sentinel
      container_name: redis-sentinel-2
      network_mode: host
      volumes:
        - D:\conf\sentinel2.conf:/etc/redis/sentinel.conf
   
    sentinel3:
      build: sentinel
      container_name: redis-sentinel-3
      network_mode: host
      volumes:
        - D:\conf\sentinel3.conf:/etc/redis/sentinel.conf
  ```

  + Redis Dockerfile文件(`dockerfile`)

  ```dockerfile
  FROM redis:latest
  MAINTAINER tsz201 <xxxxxx@xxxxxx.com>
  EXPOSE 6379 6380 6381 6382
  ENTRYPOINT redis-server /etc/redis/redis.conf
  ```

  + Sentinel Dockerfie文件(`dockerfile`)

  ```dockerfile
  FROM redis:latest
  MAINTAINER tsz201 <xxxxxx@xxxxxx.com>
  EXPOSE 26379 23680 26381
  ENTRYPOINT redis-sentinel /etc/redis/sentinel.conf
  ```

  + Redis配置文件(`master.conf`,`slave1.conf`,`slave2.conf`)

  ```
  bind 0.0.0.0
  port 6380          // need change the port for each instance
  daemonize no
  protected-mode no
  save 900 1
  save 300 10
  save 60 10000
  appendonly yes
  
  ```

  + sentinel1配置文件（`sentinel1.conf`）

  ```
  port 26379
  sentinel myid 61865092d2406f6be9e37d2dad02ff3daa7bad63
  sentinel deny-scripts-reconfig yes
  sentinel monitor mymaster 10.0.75.2 6382 2
  sentinel failover-timeout mymaster 10000
  sentinel config-epoch mymaster 2
  ```

  + sentinel2配置文件(`sentinel2.conf`)

  ```
  port 26380
  sentinel myid 0ec30b268408df721d6dcad1bb6e95219b8491e2
  sentinel deny-scripts-reconfig yes
  sentinel monitor mymaster 10.0.75.2 6382 2
  sentinel failover-timeout mymaster 10000
  sentinel config-epoch mymaster 2
  ```

  + sentinel3配置文件(`sentinel3.conf`)

  ```
  port 26381
  sentinel myid 85744214ead957df461e4468b9b711112a642664
  sentinel deny-scripts-reconfig yes
  sentinel monitor mymaster 10.0.75.2 6382 2
  sentinel failover-timeout mymaster 10000
  sentinel config-epoch mymaster 2
  ```

__注意事项：__

1. 在制作Redis和Sentinel镜像时，`entrypoint`使用shell命令的形式通过配置文件启动Redis和Sentinel，所以需要将相关的配置文件挂载到容器中。
2. 我们需要了解 docker 网络。 在构建 Redis Sentinel 集群时，不要使用默认的桥接模式，而是使用主机模式。__Redis Sentinel官方说明参考：[Here](https://redis.io/docs/manual/sentinel/)__
3. windows环境下，docker容器使用Linux容器。 当网络使用主机模式时，主机的 IP 地址不再是 `127.0.0.1`。 Linux 主机地址将映射到 `10.0.75.0` 网段（有可能是其它网段，可以通过Docker命令行查看）。 因此，我们需要通过容器中的 ifconfig 命令查看主机地址。
4. 为了保证 Sentinel 和主从 Redis 之间传递消息，我们需要声明每个服务的网络地址或端口。__Redis配置文件参考：[Here](https://raw.githubusercontent.com/redis/redis/5.0/redis.conf)__

__验证：__

如果主节点挂了，Sentinel会感知并从从节点中选出一个主节点。

```
# before master is stop
>>r.discover()
('10.0.75.2', 6380)
[('10.0.75.2', 6382), ('10.0.75.2', 6381)]
# after master is stop
r.discover()
('10.0.75.2', 6382)
[('10.0.75.2', 6380), ('10.0.75.2', 6381)]
# after master restart
r.discover()
('10.0.75.2', 6382)
[('10.0.75.2', 6380), ('10.0.75.2', 6381)]
```

salves 将以异步方式增量复制主节点的数据。

```
master:0>select 1
"OK"
master:1>dbsize
"100"
slave1:0>select 1
"OK"
slave1:1>dbsize
"100"
```

#### Redis Cluster

+ 项目结构

```
D:.
│  docker-compose.yml
│  
├─conf
│      node1.conf
│      node2.conf
│      node3.conf
│      node4.conf
│      node5.conf
│      node6.conf
│      
└─create_cluster
       dockerfile
```

+ DockerCompose文件(`docker-compose.yml`)

```yaml
version: '3'
services:
  redis1:
    image: redis:latest
    network_mode: host
    restart: always
    command: redis-server /etc/redis/redis.conf
    volumes:
      - D:\conf\node1.conf:/etc/redis/redis.conf
 
  redis2:
    image: redis:latest
    network_mode: host
    restart: always
    command: redis-server /etc/redis/redis.conf
    volumes:
      - D:\conf\node2.conf:/etc/redis/redis.conf

  redis3:
    image: redis:latest
    network_mode: host
    restart: always
    command: redis-server /etc/redis/redis.conf
    volumes:
      - D:\conf\node3.conf:/etc/redis/redis.conf

  redis4:
    image: redis:latest
    network_mode: host
    restart: always
    command: redis-server /etc/redis/redis.conf
    volumes:
      - D:\conf\node4.conf:/etc/redis/redis.conf

  redis5:
    image: redis:latest
    network_mode: host
    restart: always
    command: redis-server /etc/redis/redis.conf
    volumes:
      - D:\conf\node5.conf:/etc/redis/redis.conf

  redis6:
    image: redis:latest
    network_mode: host
    restart: always
    command: redis-server /etc/redis/redis.conf
    volumes:
      - D:\conf\node6.conf:/etc/redis/redis.conf
      
  create:
    build: create_cluster
    network_mode: host
    depends_on:
      - redis1
      - redis2
      - redis3
      - redis4
      - redis5
      - redis6
      
```

+ Redis Dockerfile文件(`dockerfile`)

```dockerfile
FROM redis:latest
MAINTAINER tsz201 <xxxxxx@xxxxxx.com>
ENTRYPOINT echo "yes"|redis-cli --cluster create \
10.0.75.2:7001 \
10.0.75.2:7002 \
10.0.75.2:7003 \
10.0.75.2:7004 \
10.0.75.2:7005 \
10.0.75.2:7006 \
--cluster-replicas 1
```

+ Redis配置文件(`node1.conf`,`node2.conf`,`node3.conf`,`node4.conf`,`node5.conf`,`node6.conf`)

```
bind 0.0.0.0
port 7001        // need change eache instance port
daemonize no
save 900 1
save 300 10
save 60 10000
appendonly yes
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 15000
cluster-announce-ip 10.0.75.2
```

__注意事项：__

1. 我们直接使用 官方镜像来编排容器。
2. 其他节点的配置内容与第一个节点类似，只需要更改端口号即可。

执行docker-compose的容器构建命令后，所有容器都启动成功，从桌面管理器可以看到集群已经建立。

 <div align=center>
  <img width="600" src="/docker/img/redis-cluster-show1.png"/>
  </div>

### 🧱 Summary

技术知识：

1. 了解 docker 基本技术细节

   1. Docker Network工作原理
   2. Docker Volume工作原理
   3. Docker操作指令
   4. 制作Docker镜像
   5. Docker Compose编排容器

2. 了解Redis集群部署策略

   1. 客户端切片
   2. 服务端切片（使用代理）
   3. 哨兵模式
   4. 集群模式

   
