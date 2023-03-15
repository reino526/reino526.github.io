---
sort: 12
---



# Redis

​	单点Redis的问题及解决方法：

- 数据丢失问题：通过实现Redis数据持久化
- 并发能力问题：搭建主从集群，实现读写分离
- 故障恢复问题：利用Redis哨兵，实现健康检测和自动恢复
- 存储能力问题：搭建分片集群，利用插槽机制实现动态扩容



## Redis 持久化

​	

### RDB

​	![image-20220727150058147](C:/Users/H/Desktop/Java/图片/image-20220727150058147.png)

![image-20220727150253904](C:/Users/H/Desktop/Java/图片/image-20220727150253904.png)

![image-20220727150745371](C:/Users/H/Desktop/Java/图片/image-20220727150745371.png)

RDB的缺点：

- RDB执行间隔时间长，两次RDB之间写入数据有丢失的风险
- fork子进程、压缩、写出RDB文件都比较耗时



### AOF

​	AOF全称为Append Only File(追加文件），Redis处理的每一个写命令都会记录在AOF文件，可以看做是命令日志文件

![image-20220727151346684](C:/Users/H/Desktop/Java/图片/image-20220727151346684.png)

​	因为是记录命令，AOF文件会比RDB文件大的多。而且AOF会记录对同一个key的多次写操作，但只有最后一次写操作才有意义，通过执行bgrewriteaof命令，可以让AOF文件执行重写功能，用最少的命令达到相同效果

![image-20220727151742187](C:/Users/H/Desktop/Java/图片/image-20220727151742187.png)



### 对比

​	RDB和AOF各有自己的优缺点，如果对数据安全性要求较高，在实际开发中往往会结合两者来使用

|                |                     RDB                      |                           AOF                            |
| :------------: | :------------------------------------------: | :------------------------------------------------------: |
|   持久化方式   |             定时对整个内存做快照             |                   记录每一次执行的命令                   |
|   数据完整性   |          不完整，两次备份之间会丢失          |                 相对完整，取决于刷盘策略                 |
|    文件大小    |             会有压缩，文件体积小             |                  记录命令，文件体积很大                  |
|  宕机恢复速度  |                     很快                     |                            慢                            |
| 数据恢复优先级 |          低，因为数据完整性不如AOF           |                  高，因为数据完整性更高                  |
|  系统资源占用  |            高，大量CPU和内存消耗             | 低，主要是磁盘IO资源，但AOF重写时会占用大量CPU和内存资源 |
|    使用场景    | 可以容忍数分钟的数据丢失，追求更快的启动速度 |                   对数据安全性要求较高                   |



## Redis 主从

​	单节点Redis的并发能力是有上限的，要进一步提高Redis的并发能力，就需要搭建主从集群，实现读写分离

​	主节点可写，从节点只读，有两种配置方法：

- 修改配置文件（永久生效）
  - 在redis.conf中添加一行配置：slaveof \<masterip\> \<masterport\>
- 使用redis-cli客户端连接到redis服务，执行slaveof命令（重启后失效）：

```bash
slaveof <masterip> <masterport>
```

​	在5.0以后新增命令replicaof，与salveof效果一致



#### 全量同步

![image-20220727153758715](C:/Users/H/Desktop/Java/图片/image-20220727153758715.png)

​	master判断slave是不是第一次来同步数据会用到两个很重要的概念:

- Replication ld：简称replid，是数据集的标记，id一致则说明是同一数据集。每一个master都有唯一的replid,slave则会继承master节点的replid
- offset：偏移量，随着记录在repl_baklog中的数据增多而逐渐增大。slave完成同步时也会记录当前同步的offset，如果slave的offset小于master的offset，说明slave数据落后于master，需要更新

​	因此slave做数据同步，必须向master声明自己的replication id和offset，master才可以判断到底需要同步哪些数据

​	所以全量同步的流程：

- slave节点请求增量同步
- master节点判断replid，发现不一致，拒绝增量同步master将完整内存数据生成RDB，发送RDB到slave
- slave清空本地数据，加载master的RDB
- master将RDB期间的命令记录在repl_baklog，并持续将log中的命令发送给slave
- slave执行接收到的命令，保持与master之间的同步



#### 增量同步

![image-20220727154832404](C:/Users/H/Desktop/Java/图片/image-20220727154832404.png)

​	

#### 优化

​	可以从以下几个方面来优化Redis主从集群：

- 在master中配置repl-diskless-sync yes启用无磁盘复制（直接发网络里，需要网络好），避免全量同步时的磁盘IO
- Redis单节点上的内存占用不要太大，减少RDB导致的过多磁盘IO
- 适当提高repl_baklog的大小，发现slave宕机时尽快实现故障恢复，尽可能避免全量同步
- 限制一个master上的slave节点数量，如果实在太多slave，则可以采用主-从-从链式结构，减少master压力（树状图）



## Redis 哨兵

​	Redis提供了哨兵（Sentinel）机制来实现主从集群的自动故障恢复，哨兵的结构和作用如下：

- 监控：Sentinel会不断检查master和slave是否按预期工作
- 自动故障恢复：如果master故障，Sentinel会将一个slave提升为master，当故障实例恢复后也以新的master为主
- 通知：Sentinel充当Redis客户端的服务发现来源，当集群发生故障转移时，会将最新信息推送给Redis的客户端

​	Sentinel基于心跳机制监测服务状态，每隔1秒向集群的每个实例发送ping命令：

- 主观下线：如果某sentinel节点发现某实例未在规定时间响应，则认为该实例主观下线
- 客观下线：若超过指定数量（quorum)的sentinel都认为该实例主观下线，则该实例客观下线，quorum值最好超过Sentinel实例数量的一半

​	一旦发现master故障，sentinel需要在slave中选择一个作为新的master，选择依据是这样的：

- 首先会判断slave节点与master节点断开时间长短，如果超过指定值(down-after-milliseconds *10）则会排除该slave节点
- 然后判断slave节点的slave-priority值，越小优先级越高，如果是0则永不参与选举

- 如果slave-prority一样，则判断slave节点的offset值，越大说明数据越新，优先级越高

- 最后是判断slave节点的运行id大小，越小优先级越高

​	选中新的master后故障转移的步骤：

- sentinel给选中的slave节点发送slaveof no one命令，让该节点成为master
- sentinel给所有其它slave发送slaveof 192.168.150.101 7002命令，让这些slave成为新master的从节点，开始从新的master上同步数据
- 最后，sentinel将故障节点标记为slave，当故障节点恢复后会自动成为新的master的slave节点

​	搭建方法就是给每个redis都写一个sentinel.conf的配置文件，然后用命令 redis-sentinel \<sentinel.conf的路径\>来启动每一个redis节点的sentinel



#### RedisTemplate 的哨兵模式

​	在Sentinel集群监管下的Redis主从集群，其节点会因为自动故障转移而发生变化Redis的客户端必须感知这种变化，及时更新连接信息。Spring的RedisTemplate底层利用lettuce实现了节点的感知和自动切换

- 在pom文件中引入redis的starter依赖：

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

- 在配置文件application.yml中指定sentinel相关信息

```yaml
spring:
	redis:
		sentinel:
			master: mymaster  # 指定master名称，是在sentinel.conf里设置的
			nodes:  # 是redis-sentinel的集群地址
				- 192.168.150.101:27001
				- 192.168.150.101:27002
				- 192.168.150.101:27003
				# ...
```

- 配置主从读写分离

```java
@Bean
public LettuceClientConfigurationBuilderCustomizer configurationBuilderCustomizer() {
	return configBuilder -> configBuilder.readFrom(ReadFrom.REPLICA_PREFERRED);
}
```

这里的ReadFrom是配置Redis的读取策略，是一个枚举，包括下面选择: 

- MASTER：从主节点读取
- MASTER PREFERRED：优先从master节点读取，master不可用才读取replica 
- REPLICA:：从slave (replica）节点读取
- REPLICA_PREFERRED：优先从slave ( replica）节点读取，所有的slave都不可用才读取master



## Redis 分片集群

​	主从和哨兵可以解决高可用、高并发读的问题，但是依然有两个问题没有解决：

- 海量数据存储问题

- 高并发写的问题

​	使用分片集群可以解决上述问题，分片集群特征：

- 集群中有多个master，每个master保存不同数据

- 每个master都可以有多个slave节点 

- master之间通过ping监测彼此健康状态
- 客户端请求可以访问集群任意节点，最终都会被转发到正确节点



### 集群命令

1. 在Redis5.0之前集群命令都是用redis安装包下的src/redis-trib.rb来实现的，因为redis-trib.rb是有ruby语言编写的所以需要安装ruby环境
2. 在Redis5.0之后集群管理集成到了redis-cli中，就是redis-cli --cluster或者./redis-trib.rb命令



### 散列插槽

​	Redis会把每一个master节点映射到0~16383共16384个插槽（hash slot）上

​	数据key不是与节点绑定，而是与插槽绑定，redis会根据key的有效部分计算插槽值，分两种情况：

- key中包含"{}"，且"{}"中至少包含1个字符，"{}"中的部分是有效部分
- key中不包含"{}"，整个key都是有效部分

​	例如：key是num，那么就根据num计算，如果是{itcast}num，则根据itcast计算，计算方式是利用CRC16算法得到一
个hash值，然后对16384取余，得到的结果就是slot值

​	{}的存在意义是为了可以让同一类的数据保存在同一个实例里，{}里的值相同就能获得相同的hash值，对应存放在相同的节点



### 数据迁移

​	分片集群不需要哨兵也能自动实现数据迁移，而且绝大部分情况都是人为的去切换（更换新的性能好的节点）

![image-20220727224231031](C:/Users/H/Desktop/Java/图片/image-20220727224231031.png)



### RedisTemplate 访问分片集群

​	RedisTemplate底层同样基于lettuce实现了分片集群的支持，而使用的步骤与哨兵模式基本一致：

- 引入redis的starter依赖
- 配置分片集群地址

```yaml
spring:
	redis:
		cluster:
			nodes:  # 指定分片集群的每一个节点信息
```

- 配置读写分离
