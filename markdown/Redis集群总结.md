# Redis集群总结

### 1.主从复制

#### 1.1 概念

主从复制是指将一台Redis服务器的数据复制到其他的Redis服务器。前者称为主节点(master/leader) 后者称为从节点(slave/follower) 数据的复制是单向的。只能由主节点到从节点。Master以写为主，Slave 以读为主。

#### 1.2 为什么需要主从复制

- 数据备份：主从复制实现数据的热备份，当主节点发生故障时，可以切换到从节点运行redis实例，保证服务可用。
- 负载均衡：主从架构配合读写分离，可以当主节点写数据，从节点读数据。可以大大提Redis服务器的并发，提高服务器的吞吐量。
- 高可用(集群)基石： 除了上述作用以外，主从复制是哨兵和集群 能够实施的基础。 因此主从复制是Redis高可用的基础。

#### 1.3 主从复制配置

默认情况下，每台redis服务器启动只是都是master节点，我们可以使用`info replication`查看主从复制的相关信息。这里我的redis实例是通过docker部署的。

```shell
[root@shepherd-node1 ~]# docker exec -it redis bash
root@2cd78d800978:/data# redis-cli
127.0.0.1:6379> info replication
# Replication
role:master
connected_slaves:1
slave0:ip=10.10.0.18,port=6379,state=online,offset=285810,lag=1
master_failover_state:no-failover
master_replid:798f5715a81dd8694c56bc8c4eb54dbb30d24b5b
master_replid2:e22795444702ae55481268fa64db8b2440ab825a
master_repl_offset:285824
second_repl_offset:110019
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:85803
repl_backlog_histlen:200022
```

配置主从直接在从节点的redis-cli执行命令`slaveof <ip><port>`即可，但是这种方式在我们重启redis实例之后该节点又变成了默认的master节点了，所以我们建议吧这个主从配置放到redis的配置文件redis.conf中，如下：

```shell
slaveof 10.10.0.34 6379
```

在主节点上写，在从节点可以读取数据，但是不能写数据

```shell
127.0.0.1:6379> set a 123
(error) READONLY You can't write against a read only replica.
```

通过分析主从库间第一次数据同步的过程，你可以看到，一次全量复制中，对于主库来说，需要完成两个耗时的操作：生成 RDB 文件和传输 RDB 文件。如果从库数量很多，而且都要和主库进行全量复制的话，就会导致主库忙于 fork 子进程生成 RDB 文件，进行数据全量同步。fork 这个操作会阻塞主线程处理正常请求，从而导致主库响应应用程序的请求速度变慢。此外，传输 RDB 文件也会占用主库的网络带宽，同样会给主库的资源使用带来压力。

redis支持主从同步，也支持**从从同步**，上一个Slave可以是下一个slave的Master，Slave同样可以接收其他 slaves的连接和同步请求，那么该slave作为了链条中下一个的master, 可以有效减轻master的写压力,去中心化降低风险。用 `slaveof <ip><port>`这里的ip和port设置成某个从节点的即可，但是这种同步配置需要注意：中途变更转向:会清除之前的数据，重新建立拷贝最新的。风险是一旦某个slave宕机，后面的slave都没法备份主机挂了，从节点还是从节点，无法写数据了。

**当主节点服务不可用时，我们在从节客户端执行`slave no one`是当前节点变为主节点**

#### 1.4 主从复制实现原理

Slave启动成功连接到master后会发送一个sync命令

Master接到命令启动后台的存盘进程，同时收集所有接收到的用于修改数据集命令， 在后台进程执行完毕之后，master将传送整个数据文件到slave,以完成一次完全同步

增量复制：Master继续将新的所有收集到的修改命令依次传给slave,完成同步。Redis 同步的是指令流，主节点会将那些对自己的状态产生修改性影响的指令记录在本地的内存 buffer 中，然后异步将 buffer 中的指令同步到从节点，从节点一边执行同步的指令流来达到和主节点一样的状态，一遍向主节点反馈自己同步到哪里了 (偏移量)。 

因为内存的 buffer 是有限的，所以 Redis 主库不能将所有的指令都记录在内存 buffer 中。**Redis 的复制内存 buffer 是一个定长的环形数组，如果数组内容满了，就会从头开始覆盖前面的内容**。**如果因为网络状况不好，从节点在短时间内无法和主节点进行同步，那么当网络状况恢复时，Redis 的主节点中那些没有同步的指令在 buffer 中有可能已经被后续的指令覆盖掉了，从节点将无法直接通过指令流来进行同步，这个时候就需要用到更加复杂的同步机制 —— 快照同步。**

全量复制：slave服务在接收到数据库文件数据后，将其存盘并加载到内存中。

#### 1.5 主从库断连怎么办？

在 Redis 2.8 之前，如果主从库在命令传播时出现了网络闪断，那么，从库就会和主库重新进行一次全量复制，开销非常大。从 Redis 2.8 开始，网络断了之后，主从库会采用增量复制的方式继续同步。听名字大概就可以猜到它和全量复制的不同：全量复制是同步所有数据，而增量复制只会把主从库网络断连期间主库收到的命令，同步给从库。

当主从库断连后，主库会把断连期间收到的写操作命令，会把这些操作命令也写入 repl_backlog_buffer 这个缓冲区。repl_backlog_buffer 是一个环形缓冲区，主库会记录自己写到的位置，从库则会记录自己已经读到的位置。

主从库的连接恢复之后，从库首先会给主库发送 psync 命令，并把自己当前的 slave_repl_offset(从库同步读到的位置偏移量) 发给主库，主库会判断自己的 master_repl_offset(主库写到的位置偏移量) 和 slave_repl_offset 之间的差距。在网络断连阶段，主库可能会收到新的写操作命令，所以，一般来说，master_repl_offset 会大于 slave_repl_offset。此时，主库只用把 master_repl_offset 和 slave_repl_offset 之间的命令操作同步给从库就行。

不过，因为 repl_backlog_buffer 是一个环形缓冲区，所以在缓冲区写满后，主库会继续写入，此时，就会覆盖掉之前写入的操作。如果从库的读取速度比较慢，就有可能导致从库还未读取的操作被主库新写的操作覆盖了，这会导致主从库间的数据不一致，这时候就需要全量复制了。

所以主从库重新连接后是选择全量复制还是增量复制，主要看下面两点：

1. 一个从库如果和主库断连时间过长，造成它在主库repl_backlog_buffer的slave_repl_offset位置上的数据已经被覆盖掉了，此时从库和主库间将进行全量复制。

2. 每个从库会记录自己的slave_repl_offset，每个从库的复制进度也不一定相同。在和主库重连进行恢复时，从库会通过psync命令把自己记录的slave_repl_offset发给主库，主库会根据从库各自的复制进度，来决定这个从库可以进行增量复制，还是全量复制。

### 2.哨兵模式(sentinel)

#### 2.1 redis-sentinel是什么

主从复制架构一大缺点就是当我们的主库挂掉之后，从库不会自动切换为主库，导致服务不可用。只能人为手动修改配置，显然这种解决办法是低效的。所以我们必须有一个高可用方案来抵抗节点故障，当故障发生时可以自动进行从主切换，程序可以不用重启，运维可以继续睡大觉，仿佛什么事也没发生一样。Redis 官方提供了这样一种方案 —— Redis Sentinel(哨兵)。

哨兵模式是一种特殊的模式，首先Redis提供了哨兵的命令，哨兵是一个独立的进程，作为进程程，它会独立运行。其原理是哨兵通过发送命令，等待Redis服务器响应， 从而监控运行的多个Redis实例。如下图所示：

![](https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/redis-sentinel-1)

我们可以将 Redis Sentinel 集群看成是一个 ZooKeeper 集群，它是集群高可用的心脏，它一般是由 3～5 个节点组成，这样挂了个别节点集群还可以正常运转。 

它负责持续监控主从节点的健康，当主节点挂掉时，自动选择一个最优的从节点切换为主节点。客户端来连接集群时，会首先连接 sentinel，通过 sentinel 来查询主节点的地址，然后再去连接主节点进行数据交互。当主节点发生故障时，客户端会重新向 sentinel 要地址，sentinel 会将最新的主节点地址告诉客户端。如此应用程序将无需重启即可自动完成节点切换。

Sentinel 会持续监控已经挂掉了主节点，待它恢复后，此时原先挂掉的主节点现在变成了从节点，从新的主节点那里建立复制关系。

#### 2.2 消息丢失

Redis 主从采用异步复制，意味着当主节点挂掉时，从节点可能没有收到全部的同步消息，这部分未同步的消息就丢失了。如果主从延迟特别大，那么丢失的数据就可能会特别多。Sentinel 无法保证消息完全不丢失，但是也尽可能保证消息少丢失。它有两个选项可以限制主从延迟过大。

min-slaves-to-write 1 

min-slaves-max-lag 10 

第一个参数表示主节点必须至少有一个从节点在进行正常复制，否则就停止对外写服务，丧失可用性。 

何为正常复制，何为异常复制？这个就是由第二个参数控制的，它的单位是秒，表示如果 10s 没有收到从节点的反馈，就意味着从节点同步不正常，要么网络断开了，要么一直没有给反馈。 

#### 2.3 sentinel搭建和使用

使用docker部署redis-sentinel比较简单，用如下命令即可：

```shell
docker run -d --name redis-sentinel -p 26379:26379 -v /mydata/redis/conf/sentinel.conf:/conf/sentinel.conf  redis redis-sentinel /conf/sentinel.conf --sentinel
```

注意这里用了redis和redis-sentinel两个镜像。

sentinel.conf配置文件如下：

```shell
sentinel monitor mymaster 10.10.0.18 6379 1
# 其中mymaster为监控对象起的服务器名称， 1 为至少有多少个哨兵同意迁移的数量
```

启动之后使用`docker logs -tf redis-sentinel`查看日志，会打印监控所有主从节点和切换过程。

```shell
[root@shepherd-node1 ~]# docker logs -tf redis-sentinel
2022-02-21T10:00:58.205269112Z 1:X 21 Feb 2022 10:00:58.205 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
2022-02-21T10:00:58.205284362Z 1:X 21 Feb 2022 10:00:58.205 # Redis version=6.2.2, bits=64, commit=00000000, modified=0, pid=1, just started
2022-02-21T10:00:58.205286611Z 1:X 21 Feb 2022 10:00:58.205 # Configuration loaded
2022-02-21T10:00:58.205593984Z 1:X 21 Feb 2022 10:00:58.205 * monotonic clock: POSIX clock_gettime
2022-02-21T10:00:58.205820953Z 1:X 21 Feb 2022 10:00:58.205 * Running mode=sentinel, port=26379.
2022-02-21T10:00:58.205826658Z 1:X 21 Feb 2022 10:00:58.205 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
2022-02-21T10:00:58.208554638Z 1:X 21 Feb 2022 10:00:58.208 # Could not rename tmp config file (Device or resource busy)
2022-02-21T10:00:58.208629083Z 1:X 21 Feb 2022 10:00:58.208 # WARNING: Sentinel was not able to save the new configuration on disk!!!: Device or resource busy
2022-02-21T10:00:58.208634843Z 1:X 21 Feb 2022 10:00:58.208 # Sentinel ID is a71ceda8647db3402f39ef27f0d69193d189c9c8
2022-02-21T10:00:58.208636575Z 1:X 21 Feb 2022 10:00:58.208 # +monitor master mymaster 10.10.0.18 6379 quorum 1
2022-02-21T10:01:48.388323006Z 1:X 21 Feb 2022 10:01:48.388 # +sdown master mymaster 10.10.0.18 6379
2022-02-21T10:01:48.388343937Z 1:X 21 Feb 2022 10:01:48.388 # +odown master mymaster 10.10.0.18 6379 #quorum 1/1
2022-02-21T10:01:48.388346128Z 1:X 21 Feb 2022 10:01:48.388 # +new-epoch 1
2022-02-21T10:01:48.388347611Z 1:X 21 Feb 2022 10:01:48.388 # +try-failover master mymaster 10.10.0.18 6379
2022-02-21T10:01:48.389039845Z 1:X 21 Feb 2022 10:01:48.388 # Could not rename tmp config file (Device or resource busy)
2022-02-21T10:01:48.389059208Z 1:X 21 Feb 2022 10:01:48.389 # WARNING: Sentinel was not able to save the new configuration on disk!!!: Device or resource busy
2022-02-21T10:01:48.389062730Z 1:X 21 Feb 2022 10:01:48.389 # +vote-for-leader a71ceda8647db3402f39ef27f0d69193d189c9c8 1
2022-02-21T10:01:48.389064363Z 1:X 21 Feb 2022 10:01:48.389 # +elected-leader master mymaster 10.10.0.18 6379
2022-02-21T10:01:48.389065815Z 1:X 21 Feb 2022 10:01:48.389 # +failover-state-select-slave master mymaster 10.10.0.18 6379
2022-02-21T10:01:48.461008816Z 1:X 21 Feb 2022 10:01:48.460 # -failover-abort-no-good-slave master mymaster 10.10.0.18 6379
2022-02-21T10:01:48.538026620Z 1:X 21 Feb 2022 10:01:48.537 # Next failover delay: I will not start a failover before Mon Feb 21 10:07:49 2022
2022-02-21T10:05:41.944568702Z 1:X 21 Feb 2022 10:05:41.944 # -sdown master mymaster 10.10.0.18 6379
2022-02-21T10:05:41.944590202Z 1:X 21 Feb 2022 10:05:41.944 # -odown master mymaster 10.10.0.18 6379
2022-02-21T10:07:23.140444490Z 1:X 21 Feb 2022 10:07:23.140 * +slave slave 10.10.0.34:6379 10.10.0.34 6379 @ mymaster 10.10.0.18 6379
2022-02-21T10:07:23.141401682Z 1:X 21 Feb 2022 10:07:23.141 # Could not rename tmp config file (Device or resource busy)
2022-02-21T10:07:23.141446344Z 1:X 21 Feb 2022 10:07:23.141 # WARNING: Sentinel was not able to save the new configuration on disk!!!: Device or resource busy
2022-02-21T10:12:16.444520188Z 1:X 21 Feb 2022 10:12:16.444 # +sdown master mymaster 10.10.0.18 6379
2022-02-21T10:12:16.444541780Z 1:X 21 Feb 2022 10:12:16.444 # +odown master mymaster 10.10.0.18 6379 #quorum 1/1
2022-02-21T10:12:16.444544345Z 1:X 21 Feb 2022 10:12:16.444 # +new-epoch 2
2022-02-21T10:12:16.444545991Z 1:X 21 Feb 2022 10:12:16.444 # +try-failover master mymaster 10.10.0.18 6379
2022-02-21T10:12:16.445343675Z 1:X 21 Feb 2022 10:12:16.445 # Could not rename tmp config file (Device or resource busy)
2022-02-21T10:12:16.445440207Z 1:X 21 Feb 2022 10:12:16.445 # WARNING: Sentinel was not able to save the new configuration on disk!!!: Device or resource busy
2022-02-21T10:12:16.445455587Z 1:X 21 Feb 2022 10:12:16.445 # +vote-for-leader a71ceda8647db3402f39ef27f0d69193d189c9c8 2
2022-02-21T10:12:16.445458737Z 1:X 21 Feb 2022 10:12:16.445 # +elected-leader master mymaster 10.10.0.18 6379
2022-02-21T10:12:16.445460277Z 1:X 21 Feb 2022 10:12:16.445 # +failover-state-select-slave master mymaster 10.10.0.18 6379
2022-02-21T10:12:16.507644620Z 1:X 21 Feb 2022 10:12:16.507 # +selected-slave slave 10.10.0.34:6379 10.10.0.34 6379 @ mymaster 10.10.0.18 6379
2022-02-21T10:12:16.507664283Z 1:X 21 Feb 2022 10:12:16.507 * +failover-state-send-slaveof-noone slave 10.10.0.34:6379 10.10.0.34 6379 @ mymaster 10.10.0.18 6379
2022-02-21T10:12:16.566786703Z 1:X 21 Feb 2022 10:12:16.566 * +failover-state-wait-promotion slave 10.10.0.34:6379 10.10.0.34 6379 @ mymaster 10.10.0.18 6379
2022-02-21T10:12:17.207994922Z 1:X 21 Feb 2022 10:12:17.207 # Could not rename tmp config file (Device or resource busy)
2022-02-21T10:12:17.208020685Z 1:X 21 Feb 2022 10:12:17.207 # WARNING: Sentinel was not able to save the new configuration on disk!!!: Device or resource busy
2022-02-21T10:12:17.208025670Z 1:X 21 Feb 2022 10:12:17.207 # +promoted-slave slave 10.10.0.34:6379 10.10.0.34 6379 @ mymaster 10.10.0.18 6379
2022-02-21T10:12:17.208027447Z 1:X 21 Feb 2022 10:12:17.207 # +failover-state-reconf-slaves master mymaster 10.10.0.18 6379
2022-02-21T10:12:17.273563209Z 1:X 21 Feb 2022 10:12:17.273 # +failover-end master mymaster 10.10.0.18 6379
2022-02-21T10:12:17.273584926Z 1:X 21 Feb 2022 10:12:17.273 # +switch-master mymaster 10.10.0.18 6379 10.10.0.34 6379
2022-02-21T10:12:17.273728698Z 1:X 21 Feb 2022 10:12:17.273 * +slave slave 10.10.0.18:6379 10.10.0.18 6379 @ mymaster 10.10.0.34 6379
2022-02-21T10:12:17.274502585Z 1:X 21 Feb 2022 10:12:17.274 # Could not rename tmp config file (Device or resource busy)
2022-02-21T10:12:17.274513416Z 1:X 21 Feb 2022 10:12:17.274 # WARNING: Sentinel was not able to save the new configuration on disk!!!: Device or resource busy
2022-02-21T10:12:47.283203367Z 1:X 21 Feb 2022 10:12:47.283 # +sdown slave 10.10.0.18:6379 10.10.0.18 6379 @ mymaster 10.10.0.34 6379
2022-02-21T10:22:36.009530344Z 1:X 21 Feb 2022 10:22:36.009 # -sdown slave 10.10.0.18:6379 10.10.0.18 6379 @ mymaster 10.10.0.34 6379
2022-02-21T10:22:46.160892449Z 1:X 21 Feb 2022 10:22:46.160 * +convert-to-slave slave 10.10.0.18:6379 10.10.0.18 6379 @ mymaster 10.10.0.34 6379
2022-02-21T14:33:08.454413104Z 1:X 21 Feb 2022 14:33:08.454 # +sdown slave 10.10.0.18:6379 10.10.0.18 6379 @ mymaster 10.10.0.34 6379
2022-02-21T14:46:21.371179327Z 1:X 21 Feb 2022 14:46:21.370 * +reboot slave 10.10.0.18:6379 10.10.0.18 6379 @ mymaster 10.10.0.34 6379
2022-02-21T14:46:21.424697555Z 1:X 21 Feb 2022 14:46:21.424 # -sdown slave 10.10.0.18:6379 10.10.0.18 6379 @ mymaster 10.10.0.34 6379
2022-02-21T14:46:32.483330106Z 1:X 21 Feb 2022 14:46:32.483 * +convert-to-slave slave 10.10.0.18:6379 10.10.0.18 6379 @ mymaster 10.10.0.34 6379
2022-02-21T15:05:29.997191181Z 1:X 21 Feb 2022 15:05:29.996 # +sdown slave 10.10.0.18:6379 10.10.0.18 6379 @ mymaster 10.10.0.34 6379
2022-02-21T15:11:32.806547732Z 1:X 21 Feb 2022 15:11:32.806 * +reboot slave 10.10.0.18:6379 10.10.0.18 6379 @ mymaster 10.10.0.34 6379
2022-02-21T15:11:32.884624737Z 1:X 21 Feb 2022 15:11:32.884 # -sdown slave 10.10.0.18:6379 10.10.0.18 6379 @ mymaster 10.10.0.34 6379
2022-02-21T15:11:42.912484754Z 1:X 21 Feb 2022 15:11:42.912 * +convert-to-slave slave 10.10.0.18:6379 10.10.0.18 6379 @ mymaster 10.10.0.34 6379



```

接下来我们就可以在springboot中整合redis的sentinel模式了

```yaml
spring:
#  使用哨兵模式不能加以下两行配置，其他配置可以加
#  redis:
#    host: 10.10.0.18
#    port: 6379
  redis:
    sentinel:
      master: mymaster
      nodes: 10.10.0.34:26379
```

测试demo

```java
/**
 * @author fjzheng
 * @version 1.0
 * @date 2022/2/21 17:40
 */
@SpringBootTest
@RunWith(SpringRunner.class)
public class RedisSentinelTest {


    @Resource
    private RedisTemplate redisTemplate;

    @Test
    public void test001() throws Exception {
        while (true) {
            String key = "time:" + new Date().getTime();
            redisTemplate.opsForValue().set(key, new Date().getTime(),5, TimeUnit.MINUTES);
            TimeUnit.SECONDS.sleep(5);
            System.out.println(redisTemplate.opsForValue().get(key));
        }
    }
}
```

这是使用睡眠循环操作redis，方便我们测试主节点挂掉，主从自动切换的过程：

```java
1645513152596
2022-02-22 14:59:17.841 ShepherddeMacBook-Pro.local [mall-user-service] [] INFO io.lettuce.core.protocol.ConnectionWatchdog log [lettuce-eventExecutorLoop-1-7@67950] : Reconnecting, last destination was /10.10.0.34:6379
2022-02-22 14:59:18.032 ShepherddeMacBook-Pro.local [mall-user-service] [] WARN io.lettuce.core.protocol.ConnectionWatchdog log [lettuce-nioEventLoop-4-4@67950] : Cannot reconnect to [10.10.0.34:6379]: Connection refused: /10.10.0.34:6379
2022-02-22 14:59:24.131 ShepherddeMacBook-Pro.local [mall-user-service] [] INFO io.lettuce.core.protocol.ConnectionWatchdog log [lettuce-eventExecutorLoop-1-5@67950] : Reconnecting, last destination was 10.10.0.34:6379
2022-02-22 14:59:24.249 ShepherddeMacBook-Pro.local [mall-user-service] [] WARN io.lettuce.core.protocol.ConnectionWatchdog log [lettuce-nioEventLoop-4-2@67950] : Cannot reconnect to [10.10.0.34:6379]: Connection refused: /10.10.0.34:6379
2022-02-22 14:59:36.827 ShepherddeMacBook-Pro.local [mall-user-service] [] INFO io.lettuce.core.protocol.ConnectionWatchdog log [lettuce-eventExecutorLoop-1-1@67950] : Reconnecting, last destination was 10.10.0.34:6379
2022-02-22 14:59:36.964 ShepherddeMacBook-Pro.local [mall-user-service] [] WARN io.lettuce.core.protocol.ConnectionWatchdog log [lettuce-nioEventLoop-4-6@67950] : Cannot reconnect to [10.10.0.34:6379]: Connection refused: /10.10.0.34:6379
2022-02-22 14:59:43.154 ShepherddeMacBook-Pro.local [mall-user-service] [] INFO com.alibaba.nacos.client.config.impl.ClientWorker run [com.alibaba.nacos.client.Worker.longPolling.fixed-10.10.0.14_8848@67950] : get changedGroupKeys:[]
2022-02-22 14:59:53.428 ShepherddeMacBook-Pro.local [mall-user-service] [] INFO io.lettuce.core.protocol.ConnectionWatchdog log [lettuce-eventExecutorLoop-1-3@67950] : Reconnecting, last destination was 10.10.0.34:6379
2022-02-22 14:59:53.560 ShepherddeMacBook-Pro.local [mall-user-service] [] INFO io.lettuce.core.protocol.ReconnectionHandler lambda$null$4 [lettuce-nioEventLoop-4-8@67950] : Reconnected to 10.10.0.18:6379
1645513157684
1645513198630
1645513204721
```

从上面日志可以看到从`14:59:17`到`14:59:53`，经过36秒sentinel完成了主从切换，服务再次正常执行。



### 3.Codis

#### 3.1 Codis是什么

Codis是redis集群方案之一，Codis 使用 Go 语言开发，它是一个代理中间件，它和 Redis 一样也使用 Redis 协议对外提供服务，当客户端向 Codis 发送指令时，Codis 负责将指令转发到后面的 Redis 实例来执行，并将返回结果再转回给客户端。 

Codis 上挂接的所有 Redis 实例构成一个 Redis 集群，当集群空间不足时，可以通过动态增加 Redis 实例来实现扩容需求。 

客户端操纵 Codis 同操纵 Redis 几乎没有区别，还是可以使用相同的客户端 SDK，不需要任何变化。 

因为 Codis 是无状态的，它只是一个转发代理中间件，这意味着我们可以启动多个 Codis 实例，供客户端使用，每个 Codis 节点都是对等的。因为单个 Codis 代理能支撑的 QPS 比较有限，通过启动多个 Codis 代理可以显著增加整体的 QPS 需求，还能起到容灾功能，挂掉一个 Codis 代理没关系，还有很多 Codis 代理可以继续服务。

![](https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/codis)

#### 3.2 Codis分片原理

Codis 要负责将特定的 key 转发到特定的 Redis 实例，那么这种对应关系 Codis 是如何管理的呢？ 

Codis 将所有的 key 默认划分为 1024 个槽位(slot)，它首先对客户端传过来的 key 进行 crc32 运算计算哈希值，再将 hash 后的整数值对 1024 这个整数进行取模得到一个余数，这个余数就是对应 key 的槽位。

**每个槽位都会唯一映射到后面的多个 Redis 实例之一，Codis 会在内存维护槽位和 Redis 实例的映射关系**。这样有了上面 key 对应的槽位，那么它应该转发到哪个 Redis 实例就很明确了。 

hash = crc32(command.key) 

slot_index = hash % 1024 

redis = slots[slot_index].redis 

redis.do(command) 

槽位数量默认是1024，它是可以配置的，如果集群节点比较多，建议将这个数值配置大一些，比如2048、4096。 

如果 Codis 的槽位映射关系只存储在内存里，那么不同的 Codis 实例之间的槽位关系就无法得到同步。所以 Codis 还需要一个分布式配置存储数据库专门用来持久化槽位关系。Codis 开始使用 ZooKeeper。Codis 将槽位关系存储在 zk 中，并且提供了一个 Dashboard 可以用来观察和修改槽位关系，当槽位关系变化时，Codis Proxy 会监听到变化并重新同步槽位关系，从而实现多个 Codis Proxy 之间共享相同的槽位关系配置。 

#### 3.3 扩容和数据迁移

刚开始 Codis 后端只有一个 Redis 实例，1024 个槽位全部指向同一个 Redis。然后一个 Redis 实例内存不够了，所以又加了一个 Redis 实例。这时候需要对槽位关系进行调整，将一半的槽位划分到新的节点。这意味着需要对这一半的槽位对应的所有 key 进行迁移，迁移到新的 Redis 实例。 

那 **Codis** 如果找到槽位对应的所有 **key** 呢？ 

Codis 对 Redis 进行了改造，增加了 SLOTSSCAN 指令，可以遍历指定 slot 下所有的 key。Codis 通过 SLOTSSCAN 扫描出待迁移槽位的所有的 key，然后挨个迁移每个 key 到新的 Redis 节点。 

在迁移过程中，Codis 还是会接收到新的请求打在当前正在迁移的槽位上，因为当前槽位的数据同时存在于新旧两个槽位中，Codis 如何判断该将请求转发到后面的哪个具体实例呢？ 

Codis 无法判定迁移过程中的 key 究竟在哪个实例中，所以它采用了另一种完全不同的思路。当 Codis 接收到位于正在迁移槽位中的 key 后，会立即强制对当前的单个 key 进行迁移，迁移完成后，再将请求转发到新的 Redis 实例。 

**自动均衡**：Redis 新增实例，手工均衡slots太繁琐，所以 Codis 提供了自动均衡功能。自动均衡会在系统比较空闲的时候观察每个 Redis 实例对应的 Slots 数量，如果不平衡，就会自动进行迁移。

#### 3.4 Codis不足

Codis 给 Redis 带来了扩容的同时，也损失了其它一些特性。因为 Codis 中所有的 key 分散在不同的 Redis 实例中，所以**事务**就不能再支持了，事务只能在单个 Redis 实例中完成。同样` rename` 操作也很危险，它的参数是两个 key，如果这两个 key 在不同的 Redis 实例中，rename 操作是无法正确完成的。

同样为了支持扩容，单个 key 对应的 value 不宜过大，因为集群的迁移的最小单位是 key，对于一个 hash 结构，它会一次性使用 hgetall 拉取所有的内容，然后使用 hmset 放置到另一个节点。如果 hash 内部的 kv 太多，可能会带来迁移卡顿。官方建议单个集合结构的总字节容量不要超过 1M。如果我们要放置社交关系数据，例如粉丝列表这种，就需要注意了，可以考虑分桶存储，在业务上作折中。 

Codis 因为增加了 Proxy 作为中转层，所有在网络开销上要比单个 Redis 大，毕竟数据包多走了一个网络节点，整体在性能上要比单个 Redis 的性能有所下降。但是这部分性能损耗不是太明显，可以通过增加 Proxy 的数量来弥补性能上的不足。 

Codis 的集群配置中心使用 zk 来实现，意味着在部署上增加了 zk 运维的代价，不过大部分互联网企业内部都有 zk 集群，可以使用现有的 zk 集群使用即可。 

### 4.redis-cluster

RedisCluster 是 Redis 的亲儿子，它是 Redis 作者自己提供的 Redis 集群化方案。 

相对于 Codis 的不同，它是去中心化的，如图所示，该集群有三个 Redis 节点组成，每个节点负责整个集群的一部分数据，每个节点负责的数据多少可能不一样。这三个节点相互连接组成一个对等的集群，它们之间通过一种特殊的二进制协议相互交互集群信。

![](https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/redis-cluster-1)

Cluster 默认会对 key 值使用 crc16 算法进行 hash 得到一个整数值，然后用这个整数值对 16384 进行取模来得到具体槽位。 

Cluster 还允许用户强制某个 key 挂在特定槽位上，通过在 key 字符串里面嵌入 tag 标记，这就可以强制 key 所挂在的槽位等于 tag 所在的槽位。 

**跳转** 

当客户端向一个错误的节点发出了指令，该节点会发现指令的 key 所在的槽位并不归自己管理，这时它会向客户端发送一个特殊的跳转指令携带目标操作的节点地址，告诉客户端去连这个节点去获取数据。 

GET x 

-MOVED 3999 127.0.0.1:6381 

MOVED 指令的第一个参数 3999 是 key 对应的槽位编号，后面是目标节点地址。MOVED 指令前面有一个减号，表示该指令是一个错误消息。 

客户端收到 MOVED 指令后，要立即纠正本地的槽位映射表。后续所有 key 将使用新的槽位映射表。 

在迁移过程中，客户端访问的流程会有很大的变化。 

首先新旧两个节点对应的槽位都存在部分 key 数据。客户端先尝试访问旧节点，如果对应的数据还在旧节点里面，那么旧节点正常处理。如果对应的数据不在旧节点里面，那么有两种可能，要么该数据在新节点里，要么根本就不存在。旧节点不知道是哪种情况，所以它会向客户端返回一个-ASK targetNodeAddr的重定向指令。客户端收到这个重定向指令后，先去目标节点执行一个不带任何参数的asking指令，然后在目标节点再重新执行原先的操作指令。 

为什么需要执行一个不带参数的asking指令呢？ 

因为在迁移没有完成之前，按理说这个槽位还是不归新节点管理的，如果这个时候向目标节点发送该槽位的指令，节点是不认的，它会向客户端返回一个-MOVED重定向指令告诉它去源节点去执行。如此就会形成 重定向循环。asking指令的目标就是打开目标节点的选项，告诉它下一条指令不能不理，而要当成自己的槽位来处理。

Redis Cluster 可以为每个主节点设置若干个从节点，单主节点故障时，集群会自动将其中某个从节点提升为主节点。如果某个主节点没有从节点，那么当它发生故障时，集群将完全处于不可用状态。不过 Redis 也提供了一个参数cluster-require-full-coverage可以允许部分节点故障，其它节点还可以继续提供对外访问。

