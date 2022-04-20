# 分布式锁实现方案：Redisson

### 1.Redisson是什么

Redisson是一个在Redis的基础上实现的Java驻内存数据网格（In-Memory Data Grid）。它不仅提供了一系列的分布式的Java常用对象，还提供了许多分布式服务。其中包括(`BitSet`, `Set`, `Multimap`, `SortedSet`, `Map`, `List`, `Queue`, `BlockingQueue`, `Deque`, `BlockingDeque`, `Semaphore`, `Lock`, `AtomicLong`, `CountDownLatch`, `Publish / Subscribe`, `Bloom filter`, `Remote service`, `Spring cache`, `Executor service`, `Live Object service`, `Scheduler service`) Redisson提供了使用Redis的最简单和最便捷的方法。Redisson的宗旨是促进使用者对Redis的关注分离（Separation of Concern），从而让使用者能够将精力更集中地放在处理业务逻辑上。

![](https://camo.githubusercontent.com/266bd96dac2f7104c719ff2c7c451ca36edbad5e38e266789140e8ebb7a828f8/68747470733a2f2f7265646973736f6e2e6f72672f6172636869746563747572652e706e67)

- **Netty 框架**：Redisson采用了基于NIO的Netty框架，不仅能作为Redis底层驱动客户端，具备提供对Redis各种组态形式的连接功能，对Redis命令能以同步发送、异步形式发送、异步流形式发送或管道形式发送的功能，LUA脚本执行处理，以及处理返回结果的功能
- **基础数据结构**：将原生的Redis `Hash`，`List`，`Set`，`String`，`Geo`，`HyperLogLog`等数据结构封装为Java里大家最熟悉的`映射（Map）`，`列表（List）`，`集（Set）`，`通用对象桶（Object Bucket）`，`地理空间对象桶（Geospatial Bucket）`，`基数估计算法（HyperLogLog）`等结构，
- **分布式数据结构**：这基础上还提供了分布式的多值映射（Multimap），本地缓存映射（LocalCachedMap），有序集（SortedSet），计分排序集（ScoredSortedSet），字典排序集（LexSortedSet），列队（Queue），阻塞队列（Blocking Queue），有界阻塞列队（Bounded Blocking Queue），双端队列（Deque），阻塞双端列队（Blocking Deque），阻塞公平列队（Blocking Fair Queue），延迟列队（Delayed Queue），布隆过滤器（Bloom Filter），原子整长形（AtomicLong），原子双精度浮点数（AtomicDouble），BitSet等Redis原本没有的分布式数据结构。
- **分布式锁**：Redisson还实现了Redis文档中提到像分布式锁`Lock`这样的更高阶应用场景。事实上Redisson并没有不止步于此，在分布式锁的基础上还提供了`联锁（MultiLock）`，`读写锁（ReadWriteLock）`，`公平锁（Fair Lock）`，`红锁（RedLock）`，`信号量（Semaphore）`，`可过期性信号量（PermitExpirableSemaphore）`和`闭锁（CountDownLatch）`这些实际当中对多线程高并发应用至关重要的基本部件。正是通过实现基于Redis的高阶应用方案，使Redisson成为构建分布式系统的重要工具。
- **节点**：Redisson作为独立节点可以用于独立执行其他节点发布到`分布式执行服务`和`分布式调度服务`里的远程任务。

Redisson基于redis进行了封装和加强，提供了很多功能，具体详情可以查阅官方文档：https://github.com/redisson/redisson

接下来我们进入今天主题：使用Redisson实现分布式锁。

### 2.springboot项目整合Redisson

#### 2.1引入依赖

```xml
   <dependency>
     <groupId>org.redisson</groupId>
     <artifactId>redisson</artifactId>
     <version>3.11.1</version>
   </dependency>
```

#### 2.2 配置Redisson

```java
@Configuration
public class RedissonConfig {

    private static final String REDISSON_PREFIX = "redis://";
    @Value("${spring.redis.host}")
    private String redisHost;
    @Value("${spring.redis.port}")
    private String redisPort;



    @Bean(destroyMethod = "shutdown")
    public RedissonClient redisson() {
        // 1、创建配置
        Config config = new Config();
        // Redis url should start with redis:// or rediss://
        config.useSingleServer().setAddress(REDISSON_PREFIX+redisHost+":"+redisPort);

        // 2、根据 Config 创建出 RedissonClient 实例
        return Redisson.create(config);
    }
}
```

上面是redis单节点模式配置，下面是redis集群模式的配置：

```java
Config config = new Config();
config.useClusterServers()
    .setScanInterval(2000) // 集群状态扫描间隔时间，单位是毫秒
    //可以用"rediss://"来启用SSL连接
    .addNodeAddress("redis://127.0.0.1:7000", "redis://127.0.0.1:7001")
    .addNodeAddress("redis://127.0.0.1:7002");

RedissonClient redisson = Redisson.create(config);
```

基于以上操作，我们就可以在项目中使用redisson啦。

### 3.Redisson分布式锁和同步器

#### 3.1 可重入锁

基于Redis的Redisson分布式可重入锁[`RLock`](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RLock.html) Java对象实现了`java.util.concurrent.locks.Lock`接口。同时还提供了异步（Async）、反射式（Reactive）和RxJava2标准的接口。

在介绍redisson的可重入锁之前，我先讲讲基于redis基于setnx命令实现的分布式锁，下面是我一个mall项目查询商品分类树的方法，不需要过多关注该方法业务，我们只需要关心分布锁的实现即可。

```java
  List<CategoryDTO> getCategoryTreeWithRedisLock() {

        //1、占分布式锁。去redis占坑 设置过期时间必须和加锁是同步的，保证原子性（避免死锁）
        String uuid = UUID.randomUUID().toString();
        Boolean lock = stringRedisTemplate.opsForValue().setIfAbsent("lock", uuid, 300, TimeUnit.SECONDS);
        if (lock) {
            log.info("获取分布式锁成功...");
            List<CategoryDTO> categoryDTOList = null;
            try {
                String categoryJson = stringRedisTemplate.opsForValue().get(CATEGORY_CACHE);
                if (StringUtils.isBlank(categoryJson)) {
                    //加锁成功...，并且redis还没有数据库，执行业务
                    categoryDTOList = getCategoryTree();
                    stringRedisTemplate.opsForValue().set(CATEGORY_CACHE, JSON.toJSONString(categoryDTOList), 5, TimeUnit.MINUTES);
                } else {
                    categoryDTOList = JSON.parseArray(categoryJson, CategoryDTO.class);
                }
            } finally {
                // lua 脚本解锁
                String script = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
                // 删除锁
                stringRedisTemplate.execute(new DefaultRedisScript<>(script, Long.class), Collections.singletonList("lock"), uuid);
            }
            //先去redis查询下保证当前的锁是自己的
            //获取值对比，对比成功删除=原子性 lua脚本解锁
            // String lockValue = stringRedisTemplate.opsForValue().get("lock");
            // if (uuid.equals(lockValue)) {
            //     //删除我自己的锁
            //     stringRedisTemplate.delete("lock");
            // }
            return categoryDTOList;
        } else {
            log.info("获取分布式锁失败...等待重试...");
            //加锁失败...重试机制
            try {
                TimeUnit.MILLISECONDS.sleep(10);
            } catch (InterruptedException e) {
                log.error("redis分布式锁发生错误", e);
            }
            return tryAgainWithTime();
        }
    }
```

redis实现分布式锁，使用命令set key value [EX seconds] [NX|XX]，但有如下问题：
1、**加锁和设置过期时间必须是原子性的**，不然有可能加锁之后还没有执行到设置过期时间代码时服务不可用，锁一直不释放，造成死锁
2、**主动删除key，即解锁需要注意：如果在key设置的过期时间之前删除key那么没问题，测试业务执行完正常解锁，但是如果删除key在过期之后就有问题了**，此时当前线程的锁已经因为过期自动解锁，另外的请求线程拿到锁，所以这时候删除的不是当前线程的锁，
解决方案：利用CAS原理，删除之前先比较value值是不是之前存放进去的，这里为了保证每次的value都不一样，使用uuid生成value，但是这时候有一种情况就是你去拿value的时候锁没有过期，此时拿到value和传入的一样，但是当你刚刚获取完value之后锁就过期了，其他请求线程拿到锁，然后你再根据传入的值和获取的value值去删除锁，这时候删除的是其他请求线程的锁，造成问题，所以利用CAS原理值比较和删除锁必须是原子性操作。

当然上面代码出现并不是一定就会出现上述问题的，一般情况下是可以正常使用的，但是在某些情况下会出现，并入并发量大的时候。

下面我们着手讲述Redisson实现的可重入锁，测试可重入锁demo代码如下：这也是分布式锁实现的简单案例

```java
    public void testLock() {

        //1、获取一把锁，只要锁的名字一样，就是同一把锁
        RLock rLock = redissonClient.getLock("my-lock");

        //2、加锁
        rLock.lock();      //阻塞式等待。默认加的锁都是30s
        //1）、锁的自动续期，如果业务超长，运行期间自动锁上新的30s。不用担心业务时间长，锁自动过期被删掉
        //2）、加锁的业务只要运行完成，就不会给当前锁续期，即使不手动解锁，锁默认会在30s内自动过期，不会产生死锁问题
        // myLock.lock(10,TimeUnit.SECONDS);   //10秒钟自动解锁,自动解锁时间一定要大于业务执行时间
        //问题：在锁时间到了以后，不会自动续期
        //1、如果我们传递了锁的超时时间，就发送给redis执行脚本，进行占锁，默认超时就是 我们制定的时间
        //2、如果我们指定锁的超时时间，就使用 lockWatchdogTimeout = 30 * 1000 【看门狗默认时间】
        //只要占锁成功，就会启动一个定时任务【重新给锁设置过期时间，新的过期时间就是看门狗的默认时间】,每隔10秒都会自动的再次续期，续成30秒
        // internalLockLeaseTime 【看门狗时间】 / 3， 10s
        try {
            System.out.println("加锁成功，执行业务..." + Thread.currentThread().getId());
            try {
                TimeUnit.SECONDS.sleep(20);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        } catch (Exception ex) {
            ex.printStackTrace();
        } finally {
            //3、解锁  假设解锁代码没有运行，Redisson会不会出现死锁
            System.out.println("释放锁..." + Thread.currentThread().getId());
            rLock.unlock();
        }
    }
```

#### 3.2 读写锁（ReadWriteLock）

基于Redis的Redisson分布式可重入读写锁[`RReadWriteLock`](http://static.javadoc.io/org.redisson/redisson/3.4.3/org/redisson/api/RReadWriteLock.html) Java对象实现了`java.util.concurrent.locks.ReadWriteLock`接口。其中读锁和写锁都继承了[RLock](https://github.com/redisson/redisson/wiki/8.-分布式锁和同步器#81-可重入锁reentrant-lock)接口。

分布式可重入读写锁允许同时有多个读锁和一个写锁处于加锁状态.

**写锁**

```java
public String writeValue() {
        String s = "";
        RReadWriteLock readWriteLock = redissonClient.getReadWriteLock("rw-lock");
        RLock rLock = readWriteLock.writeLock();
        try {
            //1、改数据加写锁，读数据加读锁
            rLock.lock();
            s = UUID.randomUUID().toString();
            ValueOperations<String, String> ops = stringRedisTemplate.opsForValue();
            ops.set("writeValue",s);
            TimeUnit.SECONDS.sleep(10);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            rLock.unlock();
        }
        return s;
}
```

**读锁**

```java
public String writeValue() {
        String s = "";
        RReadWriteLock readWriteLock = redissonClient.getReadWriteLock("rw-lock");
        RLock rLock = readWriteLock.writeLock();
        try {
            //1、改数据加写锁，读数据加读锁
            rLock.lock();
            s = UUID.randomUUID().toString();
            ValueOperations<String, String> ops = stringRedisTemplate.opsForValue();
            ops.set("writeValue",s);
            TimeUnit.SECONDS.sleep(10);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            rLock.unlock();
        }
        return s;
}
```

保证一定能读到最新数据，修改期间，写锁是一个排它锁（互斥锁、独享锁），读锁是一个共享锁
写锁没释放读锁必须等待
读 + 读 ：相当于无锁，并发读，只会在Redis中记录好，所有当前的读锁。他们都会同时加锁成功
写 + 读 ：必须等待写锁释放
写 + 写 ：阻塞方式
读 + 写 ：有读锁。写也需要等待
只要有读或者写的存都必须等待

#### 3.3 信号量（Semaphore）

基于Redis的Redisson的分布式信号量（[Semaphore](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RSemaphore.html)）Java对象`RSemaphore`采用了与`java.util.concurrent.Semaphore`相似的接口和用法。同时还提供了[异步（Async）](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RSemaphoreAsync.html)、[反射式（Reactive）](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RSemaphoreReactive.html)和[RxJava2标准](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RSemaphoreRx.html)的接口。

```java
    /**
     * 车库停车
     * 3个车位
     * 信号量也可以做分布式限流
     */
    public String park() throws InterruptedException {
        RSemaphore park = redissonClient.getSemaphore("park");
        park.acquire();     //获取一个信号、获取一个值,占一个车位
        boolean flag = park.tryAcquire();
        if (flag) {
            //执行业务
        } else {
            return "error";
        }

        return "ok=>" + flag;
    }

    public String go() {
        RSemaphore park = redissonClient.getSemaphore("park");
        park.release();     //释放一个车位
        return "ok";
    }
```

#### 3.4 闭锁（CountDownLatch）

基于Redisson的Redisson分布式闭锁（[CountDownLatch](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RCountDownLatch.html)）Java对象`RCountDownLatch`采用了与`java.util.concurrent.CountDownLatch`相似的接口和用法

```java
   /**
     * 放假、锁门
     * 1班没人了
     * 5个班，全部走完，我们才可以锁大门
     * 分布式闭锁
     */

    public String lockDoor() throws InterruptedException {

        RCountDownLatch door = redissonClient.getCountDownLatch("door");
        door.trySetCount(5);
        door.await();       //等待闭锁完成

        return "放假了...";
    }

    public String gogogo(@PathVariable("id") Long id) {
        RCountDownLatch door = redissonClient.getCountDownLatch("door");
        door.countDown();       //计数-1

        return id + "班的人都走了...";
    }

```

### 4.Redisson实现分布式锁原理

回到我们上面交的基于redis实现的分布式锁和基于Redisson实现的可重入锁，接下来我们讲述一下为什么使用Redisson能解决基于redis实现分布式锁的问题。

1.加锁和设置过期时间是原子性的，所以不能存在死锁，因为即时服务宕机了，该锁到达过期时间也会自动删除。

2.着重讲述一下Redisson怎么解决删除lock的问题。

**Redisson在业务逻辑执行完成之前不会删除lock，会自动续期，基于看门狗机制**。

Redisson内部提供了一个监控锁的看门狗，它的作用是在Redisson实例被关闭前，不断的延长锁的有效期。默认情况下，看门狗的检查锁的超时时间是30秒钟，也可以通过修改[Config.lockWatchdogTimeout](https://github.com/redisson/redisson/wiki/2.-配置方法#lockwatchdogtimeout监控锁的看门狗超时单位毫秒)来另行指定。

如果我们未制定 lock 的超时时间，就使用 30 秒作为看门狗的默认时间。只要占锁成功，就会启动一个`定时任务`：每隔 10 秒重新给锁设置过期的时间，过期时间为 30 秒。

![](https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/watchdog)



源码解读：Redisson实现的可重入锁，也可以选择公平锁的

```java
RLock rLock = redissonClient.getLock("my-lock");
// 公平锁
RLock fairLock = redisson.getFairLock("my-lock");
```

RLock提供常用获取锁方法：

```java
void lock();
// //指定时间自动解锁，不会自动续期，自动解锁时间一定要大于业务执行时间
void lock(long leaseTime, TimeUnit unit);
```

锁实现类`RedissonLock`: 对应上面的两个方法

```java
    @Override
    public void lock() {
        try {
            lock(-1, null, false);
        } catch (InterruptedException e) {
            throw new IllegalStateException();
        }
    }

    @Override
    public void lock(long leaseTime, TimeUnit unit) {
        try {
            lock(leaseTime, unit, false);
        } catch (InterruptedException e) {
            throw new IllegalStateException();
        }
    }
```

获取锁真正入口：

```java
 private void lock(long leaseTime, TimeUnit unit, boolean interruptibly) throws InterruptedException {
        // 获取当前线程id
        long threadId = Thread.currentThread().getId();
        Long ttl = tryAcquire(leaseTime, unit, threadId);
        // lock acquired
        if (ttl == null) {
            return;
        }

        RFuture<RedissonLockEntry> future = subscribe(threadId);
        commandExecutor.syncSubscription(future);

        try {
            while (true) {
                ttl = tryAcquire(leaseTime, unit, threadId);
                // lock acquired
                if (ttl == null) {
                    break;
                }

                // waiting for message
                if (ttl >= 0) {
                    try {
                        getEntry(threadId).getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
                    } catch (InterruptedException e) {
                        if (interruptibly) {
                            throw e;
                        }
                        getEntry(threadId).getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
                    }
                } else {
                    if (interruptibly) {
                        getEntry(threadId).getLatch().acquire();
                    } else {
                        getEntry(threadId).getLatch().acquireUninterruptibly();
                    }
                }
            }
        } finally {
            unsubscribe(future, threadId);
        }
//        get(lockAsync(leaseTime, unit));
    }
```

```java
    private Long tryAcquire(long leaseTime, TimeUnit unit, long threadId) {
        return get(tryAcquireAsync(leaseTime, unit, threadId));
    }
    
    
   private <T> RFuture<Long> tryAcquireAsync(long leaseTime, TimeUnit unit, long threadId) {
        if (leaseTime != -1) {
            // 设置了自动解锁时间
            return tryLockInnerAsync(leaseTime, unit, threadId, RedisCommands.EVAL_LONG);
        }
        // 没有设置自动解锁时间，接下来会添加看门狗机制
        RFuture<Long> ttlRemainingFuture = tryLockInnerAsync(commandExecutor.getConnectionManager().getCfg().getLockWatchdogTimeout(), TimeUnit.MILLISECONDS, threadId, RedisCommands.EVAL_LONG);
        ttlRemainingFuture.onComplete((ttlRemaining, e) -> {
            if (e != null) {
                return;
            }

            // lock acquired
            if (ttlRemaining == null) {
                scheduleExpirationRenewal(threadId);
            }
        });
        return ttlRemainingFuture;
    }

    // 执行lua脚本命令，往redis添加lock
    <T> RFuture<T> tryLockInnerAsync(long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {
        internalLockLeaseTime = unit.toMillis(leaseTime);

        return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, command,
                  "if (redis.call('exists', KEYS[1]) == 0) then " +
                      "redis.call('hset', KEYS[1], ARGV[2], 1); " +
                      "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                      "return nil; " +
                  "end; " +
                  "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
                      "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
                      "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                      "return nil; " +
                  "end; " +
                  "return redis.call('pttl', KEYS[1]);",
                    Collections.<Object>singletonList(getName()), internalLockLeaseTime, getLockName(threadId));
    }


    // 定时任务调度，看门狗机制
    private void scheduleExpirationRenewal(long threadId) {
        ExpirationEntry entry = new ExpirationEntry();
        ExpirationEntry oldEntry = EXPIRATION_RENEWAL_MAP.putIfAbsent(getEntryName(), entry);
        if (oldEntry != null) {
            oldEntry.addThreadId(threadId);
        } else {
            entry.addThreadId(threadId);
            renewExpiration();
        }
    }

    private void renewExpiration() {
        ExpirationEntry ee = EXPIRATION_RENEWAL_MAP.get(getEntryName());
        if (ee == null) {
            return;
        }
        
        Timeout task = commandExecutor.getConnectionManager().newTimeout(new TimerTask() {
            @Override
            public void run(Timeout timeout) throws Exception {
                ExpirationEntry ent = EXPIRATION_RENEWAL_MAP.get(getEntryName());
                if (ent == null) {
                    return;
                }
                Long threadId = ent.getFirstThreadId();
                if (threadId == null) {
                    return;
                }
                
                RFuture<Boolean> future = renewExpirationAsync(threadId);
                future.onComplete((res, e) -> {
                    if (e != null) {
                        log.error("Can't update lock " + getName() + " expiration", e);
                        return;
                    }
                    
                    if (res) {
                        // reschedule itself
                        renewExpiration();
                    }
                });
            }
        }, internalLockLeaseTime / 3, TimeUnit.MILLISECONDS);
        // 上面解释了自动续期周期为过期时间的1/3
        ee.setTimeout(task);
    }

```

从上面可知Redisson底层大量使用lua脚本来保证操作的原子性，同时使用看门狗机制实现自动续期，基于以上两点解决了基于redis实现分布式可能出现的问题。