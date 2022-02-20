# java线程池笔记

####  1.池化背景

​	 在面向对象编程中，创建和销毁对象是很费时间的，因为创建一个对象要获取内存资源或者其它更多资源。在Java中更是如此，虚拟机将试图跟踪每一个对象，以便能够在对象销毁后进行垃圾回收。所以提高服务程序效率的一个手段就是尽可能减少创建和销毁对象的次数，特别是一些很耗资源的对象创建和销毁。如何利用已有对象来服务就是一个需要解决的关键问题，其实这就是一些"池化资源"技术产生的原因 。

#### 2.java线程池的优势

（1）：降低资源消耗。通过重复利用已创建的线程降低线程创建和销毁造成的消耗。

（2）：提高响应速度。当任务到达时，任务可以不需要等到线程创建就能立即执行。

（3）：提高线程的可管理性。线程是稀缺资源，如果无限制地创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一分配、调优和监控。但是，要做到合理利用线程池，必须对其实现原理了如指掌

#### 3.线程池实现原理

当向线程池提交一个任务之后，线程池是如何处理这个任务的呢？本节来看一下线程池的主要处理流程，处理流程图如图3-1所示：

![](https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/%E7%BA%BF%E7%A8%8B%E6%B1%A0-1)

从图中可以看出，当提交一个新任务到线程池时，线程池的处理流程如下。
1）线程池判断核心线程池里的线程是否都在执行任务。如果不是，则创建一个新的工作线程来执行任务。如果核心线程池里的线程都在执行任务，则进入下个流程。
2）线程池判断工作队列是否已经满。如果工作队列没有满，则将新提交的任务存储在这个工作队列里。如果工作队列满了，则进入下个流程。
3）线程池判断线程池的线程是否都处于工作状态。如果没有，则创建一个新的工作线程来执行任务。如果已经满了，则交给饱和策略来处理这个任务。
ThreadPoolExecutor执行execute()方法的示意图，如图3-2所示：

<img src="https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/%E7%BA%BF%E7%A8%8B%E6%B1%A0-2" style="zoom:50%;" />

   ThreadPoolExecutor执行execute方法分下面4种情况。
1）如果当前运行的线程少于corePoolSize，则创建新线程来执行任务（注意，执行这一步骤需要获取全局锁）。
2）如果运行的线程等于或多于corePoolSize，则将任务加入BlockingQueue。
3）如果无法将任务加入BlockingQueue（队列已满），则创建新的线程来处理任务（注意，执行这一步骤需要获取全局锁）。
4）如果创建新线程将使当前运行的线程超出maximumPoolSize，任务将被拒绝，并调用RejectedExecutionHandler.rejectedExecution()方法。
ThreadPoolExecutor采取上述步骤的总体设计思路，是为了在执行execute()方法时，尽可能地避免获取全局锁（那将会是一个严重的可伸缩瓶颈）。在ThreadPoolExecutor完成预热之后（当前运行的线程数大于等于corePoolSize），几乎所有的execute()方法调用都是执行步骤2，而步骤2不需要获取全局锁。
源码分析：上面的流程分析让我们很直观地了解了线程池的工作原理，让我们再通过源代码来看看是如何实现的，线程池执行任务的方法如下。

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    //获取clt，clt记录着线程池状态和运行线程数。
    int c = ctl.get();
    //运行线程数小于核心线程数时，创建线程放入线程池中，并且运行当前任务。
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        //创建线程失败，重新获取clt。
        c = ctl.get();
    }
    //线程池是运行状态并且运行线程大于核心线程数时，把任务放入队列中。
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        //重新检查线程池不是运行状态时，
        //把任务移除队列，并通过拒绝策略对该任务进行处理。
        if (! isRunning(recheck) && remove(command))
            reject(command);
        //当前运行线程数为0时，创建线程加入线程池中。
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    //运行线程大于核心线程数时并且队列已满时，
    //创建线程放入线程池中，并且运行当前任务。
    else if (!addWorker(command, false))
        //运行线程大于最大线程数时，失败则拒绝该任务
        reject(command);
}

```

在execute()方法中多次调用addWorker方法。其源码如下

```java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        //获取clt，clt记录着线程池状态和运行线程数。
        int c = ctl.get();
        //获取线程池的运行状态。
        int rs = runStateOf(c);

        //线程池处于关闭状态，或者当前任务为null
        //或者队列不为空，则直接返回失败。
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        for (;;) {
            //获取线程池中的线程数
            int wc = workerCountOf(c);
            //线程数超过CAPACITY，则返回false；
            //这里的core是addWorker方法的第二个参数，
            //如果为true则根据核心线程数进行比较，
            //如果为false则根据最大线程数进行比较。
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            //尝试增加线程数，如果成功，则跳出第一个for循环
            if (compareAndIncrementWorkerCount(c))
                break retry;
            //如果增加线程数失败，则重新获取ctl
            c = ctl.get();
            //如果当前的运行状态不等于rs，说明状态已被改变，
            //返回第一个for循环继续执行
            if (runStateOf(c) != rs)
                continue retry;
        }
    }

    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        //根据当前任务来创建Worker对象
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                //获得锁以后，重新检查线程池状态
                int rs = runStateOf(ctl.get());

                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive())
                        throw new IllegalThreadStateException();
                    //把刚刚创建的线程加入到线程池中
                    workers.add(w);
                    int s = workers.size();
                    //记录线程池中出现过的最大线程数量
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            if (workerAdded) {
                //启动线程，开始运行任务
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}

```

4.线程池的创建

```java
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.acc = System.getSecurityManager() == null ?
                null :
                AccessController.getContext();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```

创建一个线程池时需要输入几个参数，如下。
1）**corePoolSize（线程池的基本大小）(必需)**：当提交一个任务到线程池时，线程池会创建一个线程来执行任务，即使其他空闲的基本线程能够执行新任务也会创建线程，等到需要执行的任务数大于线程池基本大小时就不再创建。如果调用了线程池的prestartAllCoreThreads()方法，线程池会提前创建并启动所有基本线程。

2）**maximumPoolSize（线程池最大数量）(必需)**：线程池允许创建的最大线程数。如果队列满了，并
且已创建的线程数小于最大线程数，则线程池会再创建新的线程执行任务。值得注意的是，如
果使用了无界的任务队列这个参数就没什么效果。

3）**keepAliveTime（线程活动保持时间）(必需）**：线程池的工作线程空闲后，保持存活的时间。所以，
如果任务很多，并且每个任务执行的时间比较短，可以调大时间，提高线程的利用率。

4）**TimeUnit（线程活动保持时间的单位）(必需）**：可选的单位有天（DAYS）、小时（HOURS）、分钟
（MINUTES）、毫秒（MILLISECONDS）、微秒（MICROSECONDS，千分之一毫秒）和纳秒
（NANOSECONDS，千分之一微秒）。

5）**workQueue（任务队列）(必需)**：用于保存等待执行的任务的阻塞队列。可以选择以下几个阻塞队列。

- ArrayBlockingQueue：是一个基于数组结构的有界阻塞队列，此队列按FIFO（先进先出）原
  则对元素进行排序。
- LinkedBlockingQueue：一个基于链表结构的阻塞队列，此队列按FIFO排序元素，吞吐量通常要高于ArrayBlockingQueue。静态工厂方法Executors.newFixedThreadPool()使用了这个队
- SynchronousQueue：一个不存储元素的阻塞队列。每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于阻塞状态，吞吐量通常要高于Linked-BlockingQueue，静态工厂方法Executors.newCachedThreadPool使用了这个队列。
- PriorityBlockingQueue：一个具有优先级的无限阻塞队列。

6）**ThreadFactory (可选)**：用于设置创建线程的工厂，可以通过线程工厂给每个创建出来的线程设置更有意义的名字。使用开源框架guava提供的ThreadFactoryBuilder可以快速给线程池里的线程设置有意义的名字，代码如下：
new ThreadFactoryBuilder().setNameFormat("XX-task-%d").build();

7) **RejectedExecutionHandler（饱和策略**）：当队列和线程池都满了，说明线程池处于饱和状态，那么必须采取一种策略处理提交的新任务。这个策略默认情况下是AbortPolicy，表示无法处理新任务时抛出异常。在JDK 1.5中Java线程池框架提供了以下4种策略。

- AbortPolicy：直接抛出异常。

- CallerRunsPolicy：只用调用者所在线程来运行任务。

- DiscardOldestPolicy：丢弃队列里最近的一个任务，并执行当前任务。

- DiscardPolicy：不处理，丢弃掉。

当然，也可以根据应用场景需要来实现RejectedExecutionHandler接口自定义策略。如记录日志或持久化存储不能处理的任

示例代码：

```java
private ThreadFactory namedThreadFactory = new ThreadFactoryBuilder()
            .setNameFormat("atlas-pool-%d").build();

private ExecutorService fixedThreadPool = new ThreadPoolExecutor(Runtime.getRuntime().availableProcessors() + 1,                                          Runtime.getRuntime().availableProcessors() * 40,
                   0L, 
                   TimeUnit.MILLISECONDS,
            new LinkedBlockingQueue<Runnable>(Runtime.getRuntime().availableProcessors() * 20), 
                   namedThreadFactory);


//多检查使用
public List<AtlasElementDataDTO> getData(@RequestBody AtlasElementDataVO atlasElementDataVO, HttpServletRequest request) throws InterruptedException {
        CountDownLatch countDownLatch = new CountDownLatch(atlasElementDataVO.getAtlasElementDataParamList().size());
        UserForm currentUser = RequestUserHolder.getCurrentUser();
        String url = IasStringUtils.getFileNameByUrl(request.getHeader("Referer"));
        List<AtlasElementDataDTO> list = new ArrayList<>();
        log.info("data query start total start" + IasDateUtils.dateTimeToStringTime(new Date()));
        atlasElementDataVO.getAtlasElementDataParamList().forEach(atlasElementDataParamDTO -> {
            fixedThreadPool.submit(() -> {
                try {
                    log.info("data query start" + IasDateUtils.dateTimeToStringTime(new Date()));
                    AtlasElementDataDTO atlasElementDataDTO = new AtlasElementDataDTO();
                    atlasElementDataDTO.setId(atlasElementDataParamDTO.getId());
                    atlasElementDataDTO.setRecord(atlasService.getData(atlasElementDataParamDTO.getId(), atlasElementDataParamDTO.getParam(), currentUser != null ? currentUser.getUserId() : null, url));
                    atlasElementDataDTO.set_xAxis(atlasElementDataParamDTO.get_xAxis());
                    list.add(atlasElementDataDTO);
                    log.info("data query end" + IasDateUtils.dateTimeToStringTime(new Date()));
                } catch (Exception e) {
                    throw new BusinessException(ErrorCodeEnum.QUERY_ERROR.getCode(), ErrorCodeEnum.QUERY_ERROR.getMessage());
                }finally {
                    countDownLatch.countDown();
                }
            });
        });
        countDownLatch.await();
        log.info("data query start total  end ：" + IasDateUtils.dateTimeToStringTime(new Date()));
        return list;
    }
```



#### 5.线程池执行任务

可以使用两个方法向线程池提交任务，分别为execute()和submit()方法。execute()方法用于提交不需要返回值的任务，所以无法判断任务是否被线程池执行成功。通过以下代码可知execute()方法输入的任务是一个Runnable类的实例。

```java
threadsPool.execute(new Runnable() {
                    @Override
                    public void run() {
                    // TODO Auto-generated method stub
                    }
});
```

submit()方法用于提交需要返回值的任务。线程池会返回一个future类型的对象，通过这个future对象可以判断任务是否执行成功，并且可以通过future的get()方法来获取返回值，get()方法会阻塞当前线程直到任务完成，而使用get（long timeout，TimeUnit unit）方法则会阻塞当前线程一段时间后立即返回，这时候有可能任务没有执行完。

```java
Future<Object> future = executor.submit(harReturnValuetask);
try {
Object s = future.get();
} catch (InterruptedException e) {
// 处理中断异常
} catch (ExecutionException e) {
// 处理无法执行任务异常
} finally {
// 关闭线程池
executor.shutdown();
}
```

#### 6.关闭线程池

可以通过调用线程池的shutdown或shutdownNow方法来关闭线程池。它们的原理是遍历线程池中的工作线程，然后逐个调用线程的interrupt方法来中断线程，所以无法响应中断的任务可能永远无法终止。但是它们存在一定的区别，shutdownNow首先将线程池的状态设置成STOP，然后尝试停止所有的正在执行或暂停任务的线程，并返回等待执行任务的列表，而shutdown只是将线程池的状态设置成SHUTDOWN状态，然后中断所有没有正在执行任务的线程。只要调用了这两个关闭方法中的任意一个，isShutdown方法就会返回true。当所有的任务都已关闭后，才表示线程池关闭成功，这时调用isTerminaed方法会返回true。至于应该调用哪一种方法来关闭线程池，应该由提交到线程池的任务特性决定，通常调用shutdown方法来关闭线程池，如果任务不一定要执行完，则可以调用shutdownNow方法。

#### 7.合理配置线程池

要想合理地配置线程池，就必须首先分析任务特性，可以从以下几个角度来分析。
任务的性质：CPU密集型任务、IO密集型任务和混合型任务。

- 任务的优先级：高、中和低。

- 任务的执行时间：长、中和短。
- 任务的依赖性：是否依赖其他系统资源，如数据库连接。

性质不同的任务可以用不同规模的线程池分开处理。CPU密集型任务应配置尽可能小的线程，如配置Ncpu+1个线程的线程池。由于IO密集型任务线程并不是一直在执行任务，则应配置尽可能多的线程，如2*Ncpu。混合型的任务，如果可以拆分，将其拆分成一个CPU密集型任务和一个IO密集型任务，只要这两个任务执行的时间相差不是太大，那么分解后执行的吞吐量将高于串行执行的吞吐量。如果这两个任务执行时间相差太大，则没必要进行分解。可以通过Runtime.getRuntime().availableProcessors()方法获得当前设备的CPU个数。优先级不同的任务可以使用优先级队列PriorityBlockingQueue来处理。它可以让优先级高的任务先执行。
**注意　如果一直有优先级高的任务提交到队列里，那么优先级低的任务可能永远不能执行。**
执行时间不同的任务可以交给不同规模的线程池来处理，或者可以使用优先级队列，让执行时间短的任务先执行。
依赖数据库连接池的任务，因为线程提交SQL后需要等待数据库返回结果，等待的时间越长，则CPU空闲时间就越长，那么线程数应该设置得越大，这样才能更好地利用CPU。
**建议使用有界队列**。有界队列能增加系统的稳定性和预警能力，可以根据需要设大一点儿，比如几千。有一次，我们系统里后台任务线程池的队列和线程池全满了，不断抛出抛弃任务的异常，通过排查发现是数据库出现了问题，导致执行SQL变得非常缓慢，因为后台任务线程池里的任务全是需要向数据库查询和插入数据的，所以导致线程池里的工作线程全部阻塞，任务积压在线程池里。如果当时我们设置成无界队列，那么线程池的队列就会越来越多，
有可能会撑满内存，导致整个系统不可用，而不只是后台任务出现问题。当然，我们的系统所有的任务是用单独的服务器部署的，我们使用不同规模的线程池完成不同类型的任务，但是出现这样问题时也会影响到其他任务。

 一般来说池中总线程数是核心池线程数量两倍，只要确保当核心池有线程停止时，核心池外能有线程进入核心池即可。 线程中的任务最终是交给CPU的线程去处理的，而CPU可同时处理线程数量大部分是CPU核数的两倍，运行环境中CPU的核数我们可以通过Runtime.getRuntime().availableProcessors()这个方法而获取。理论上来说核心池线程数量应该为Runtime.getRuntime().availableProcessors()*2，那么结果是否符合我们的预期呢，事实上大部分的任务都是I/O密集型的，即大部分任务消耗集中在的输入输出。而CPU密集型任务主要消耗CPU资源进行计算，**当任务为CPU密集型时，核心池线程数设置为CPU核数+1**即可）

#### 8.线程池监控

如果在系统中大量使用线程池，则有必要对线程池进行监控，方便在出现问题时，可以根据线程池的使用状况快速定位问题。可以通过线程池提供的参数进行监控，在监控线程池的时候可以使用以下属性。
1）**taskCount**：线程池需要执行的任务数量。
2）**completedTaskCount**：线程池在运行过程中已完成的任务数量，小于或等于taskCount。
3）**largestPoolSize**：线程池里曾经创建过的最大线程数量。通过这个数据可以知道线程池是否曾经满过。如该数值等于线程池的最大大小，则表示线程池曾经满过。
4）**getPoolSize**：线程池的线程数量。如果线程池不销毁的话，线程池里的线程不会自动销毁，所以这个大小只增不减。
5）**getActiveCount**：获取活动的线程数。
通过扩展线程池进行监控。可以通过继承线程池来自定义线程池，重写线程池的beforeExecute、afterExecute和terminated方法，也可以在任务执行前、执行后和线程池关闭前执行一些代码来进行监控。例如，监控任务的平均执行时间、最大执行时间和最小执行时间等。这几个方法在线程池里是空方法。