# 系统高可用番外篇：浅析sentinel源码

### 1.概念

#### sentinel是什么

Sentinel 是面向分布式服务架构的流量控制组件，主要以流量为切入点，从**限流、流量整形、熔断降级、系统负载保护、热点防护**等多个维度来帮助开发者保障微服务的稳定性。

#### 资源

资源是 Sentinel 的关键概念。它可以是 Java 应用程序中的任何内容，例如，由应用程序提供的服务，或由应用程序调用的其它应用提供的服务，甚至可以是一段代码。在接下来的文档中，我们都会用资源来描述代码块。

只要通过 Sentinel API 定义的代码，就是资源，能够被 Sentinel 保护起来。大部分情况下，可以使用方法签名，URL，甚至服务名称作为资源名来标示资源。

#### 规则

围绕资源的实时状态设定的规则，可以包括流量控制规则、熔断降级规则以及系统保护规则。所有规则可以动态实时调整。

我们说的资源，可以是任何东西，服务，服务里的方法，甚至是一段代码。使用 Sentinel 来进行资源保护，主要分为几个步骤:

1）定义资源

2）定义规则

3）检验规则是否生效

#### 定义资源

- **主流框架的默认适配**

为了减少开发的复杂程度，我们对大部分的主流框架，例如 Web Servlet、Dubbo、Spring Cloud、gRPC、Spring WebFlux、Reactor 等都做了适配。您只需要引入对应的依赖即可方便地整合 Sentinel。可以参见: [主流框架的适配](https://github.com/alibaba/Sentinel/wiki/主流框架的适配)。

- **抛出异常的方式定义资源**

`SphU` 包含了 try-catch 风格的 API。用这种方式，当资源发生了限流之后会抛出 `BlockException`。这个时候可以捕捉异常，进行限流之后的逻辑处理。示例代码如下:

```java
// 1.5.0 版本开始可以利用 try-with-resources 特性（使用有限制）
// 资源名可使用任意有业务语义的字符串，比如方法名、接口名或其它可唯一标识的字符串。
try (Entry entry = SphU.entry("resourceName")) {
  // 被保护的业务逻辑
  // do something here...
} catch (BlockException ex) {
  // 资源访问阻止，被限流或被降级
  // 在此处进行相应的处理操作
}
```

**特别地**，若 entry 的时候传入了热点参数，那么 exit 的时候也一定要带上对应的参数（`exit(count, args)`），否则可能会有统计错误。这个时候不能使用 try-with-resources 的方式。另外通过 `Tracer.trace(ex)` 来统计异常信息时，由于 try-with-resources 语法中 catch 调用顺序的问题，会导致无法正确统计异常数，因此统计异常信息时也不能在 try-with-resources 的 catch 块中调用 `Tracer.trace(ex)`。

手动 exit 示例：

```java
Entry entry = null;
// 务必保证 finally 会被执行
try {
  // 资源名可使用任意有业务语义的字符串，注意数目不能太多（超过 1K），超出几千请作为参数传入而不要直接作为资源名
  // EntryType 代表流量类型（inbound/outbound），其中系统规则只对 IN 类型的埋点生效
  entry = SphU.entry("自定义资源名");
  // 被保护的业务逻辑
  // do something...
} catch (BlockException ex) {
  // 资源访问阻止，被限流或被降级
  // 进行相应的处理操作
} catch (Exception ex) {
  // 若需要配置降级规则，需要通过这种方式记录业务异常
  Tracer.traceEntry(ex, entry);
} finally {
  // 务必保证 exit，务必保证每个 entry 与 exit 配对
  if (entry != null) {
    entry.exit();
  }
}
```

热点参数埋点示例：

```java
Entry entry = null;
try {
    // 若需要配置例外项，则传入的参数只支持基本类型。
    // EntryType 代表流量类型，其中系统规则只对 IN 类型的埋点生效
    // count 大多数情况都填 1，代表统计为一次调用。
    entry = SphU.entry(resourceName, EntryType.IN, 1, paramA, paramB);
    // Your logic here.
} catch (BlockException ex) {
    // Handle request rejection.
} finally {
    // 注意：exit 的时候也一定要带上对应的参数，否则可能会有统计错误。
    if (entry != null) {
        entry.exit(1, paramA, paramB);
    }
}
```

`SphU.entry()` 的参数描述：

| 参数名    | 类型        | 解释                                                         | 默认值          |
| --------- | ----------- | ------------------------------------------------------------ | --------------- |
| entryType | `EntryType` | 资源调用的流量类型，是入口流量（`EntryType.IN`）还是出口流量（`EntryType.OUT`），注意系统规则只对 IN 生效 | `EntryType.OUT` |
| count     | `int`       | 本次资源调用请求的 token 数目                                | 1               |
| args      | `Object[]`  | 传入的参数，用于热点参数限流                                 | 无              |

**注意**：`SphU.entry(xxx)` 需要与 `entry.exit()` 方法成对出现，匹配调用，否则会导致调用链记录异常，抛出 `ErrorEntryFreeException` 异常。常见的错误：

自定义埋点只调用 `SphU.entry()`，没有调用 `entry.exit()`

顺序错误，比如：`entry1 -> entry2 -> exit1 -> exit2`，应该为 `entry1 -> entry2 -> exit2 -> exit1`

- **返回布尔值方式定义资源**

`SphO` 提供 if-else 风格的 API。用这种方式，当资源发生了限流之后会返回 `false`，这个时候可以根据返回值，进行限流之后的逻辑处理。示例代码如下:

```java
  // 资源名可使用任意有业务语义的字符串
  if (SphO.entry("自定义资源名")) {
    // 务必保证finally会被执行
    try {
      /**
      * 被保护的业务逻辑
      */
    } finally {
      SphO.exit();
    }
  } else {
    // 资源访问阻止，被限流或被降级
    // 进行相应的处理操作
  }
```

**注意**：`SphO.entry(xxx)` 需要与 SphO.exit()`方法成对出现，匹配调用，位置正确，否则会导致调用链记录异常，抛出`ErrorEntryFreeException` 异常。

- **注解方式定义资源**

Sentinel 支持通过 `@SentinelResource` 注解定义资源并配置 `blockHandler` 和 `fallback` 函数来进行限流之后的处理。示例：

```java
// 原本的业务方法.
@SentinelResource(blockHandler = "blockHandlerForGetUser")
public User getUserById(String id) {
    throw new RuntimeException("getUserById command failed");
}

// blockHandler 函数，原方法调用被限流/降级/系统保护的时候调用
public User blockHandlerForGetUser(String id, BlockException ex) {
    return new User("admin");
}
```

注意 `blockHandler` 函数会在原方法被限流/降级/系统保护的时候调用，而 `fallback` 函数会针对所有类型的异常。请注意 `blockHandler` 和 `fallback` 函数的形式要求，更多指引可以参见 [Sentinel 注解支持文档](https://github.com/alibaba/Sentinel/wiki/注解支持)。

- **异步调用支持**

Sentinel 支持异步调用链路的统计。在异步调用中，需要通过 `SphU.asyncEntry(xxx)` 方法定义资源，并通常需要在异步的回调函数中调用 `exit` 方法。以下是一个简单的示例：

```java
try {
    AsyncEntry entry = SphU.asyncEntry(resourceName);

    // 异步调用.
    doAsync(userId, result -> {
        try {
            // 在此处处理异步调用的结果.
        } finally {
            // 在回调结束后 exit.
            entry.exit();
        }
    });
} catch (BlockException ex) {
    // Request blocked.
    // Handle the exception (e.g. retry or fallback).
}
```

`SphU.asyncEntry(xxx)` 不会影响当前（调用线程）的 Context，因此以下两个 entry 在调用链上是平级关系（处于同一层），而不是嵌套关系：

```java
// 调用链类似于：
// -parent
// ---asyncResource
// ---syncResource
asyncEntry = SphU.asyncEntry(asyncResource);
entry = SphU.entry(normalResource);
```

若在异步回调中需要嵌套其它的资源调用（无论是 `entry` 还是 `asyncEntry`），只需要借助 Sentinel 提供的上下文切换功能，在对应的地方通过 `ContextUtil.runOnContext(context, f)` 进行 Context 变换，将对应资源调用处的 Context 切换为生成的异步 Context，即可维持正确的调用链路关系。示例如下：

```java
public void handleResult(String result) {
    Entry entry = null;
    try {
        entry = SphU.entry("handleResultForAsync");
        // Handle your result here.
    } catch (BlockException ex) {
        // Blocked for the result handler.
    } finally {
        if (entry != null) {
            entry.exit();
        }
    }
}

public void someAsync() {
    try {
        AsyncEntry entry = SphU.asyncEntry(resourceName);

        // Asynchronous invocation.
        doAsync(userId, result -> {
            // 在异步回调中进行上下文变换，通过 AsyncEntry 的 getAsyncContext 方法获取异步 Context
            ContextUtil.runOnContext(entry.getAsyncContext(), () -> {
                try {
                    // 此处嵌套正常的资源调用.
                    handleResult(result);
                } finally {
                    entry.exit();
                }
            });
        });
    } catch (BlockException ex) {
        // Request blocked.
        // Handle the exception (e.g. retry or fallback).
    }
}
```

此时的调用链就类似于：

```
-parent
---asyncInvocation
-----handleResultForAsync
```

更详细的示例可以参考 Demo 中的 [AsyncEntryDemo](https://github.com/alibaba/Sentinel/blob/master/sentinel-demo/sentinel-demo-basic/src/main/java/com/alibaba/csp/sentinel/demo/AsyncEntryDemo.java)，里面包含了普通资源与异步资源之间的各种嵌套示例。

### 2.sentinel工作原理

Sentinel 的核心骨架，将不同的 Slot 按照顺序串在一起（责任链模式），从而将不同的功能（限流、降级、系统保护）组合在一起。slot chain 其实可以分为两部分：统计数据构建部分（statistic）和判断部分（rule checking）。核心结构如下：

![](https://img2.baidu.com/it/u=842767091,1320350273&fm=253&fmt=auto&app=138&f=JPEG?w=707&h=490)

**NodeSelectorSlot**

负责收集资源的路径，并将这些资源的调用路径，以树状结构存储起来，用于根据调用路径来限流降。

**ClusterBuilderSlot**

用于存储资源的统计信息以及调用者信息，例如该资源的 RT, QPS, thread count，Block count，Exception count 等等，这些信息将用作为多维度限流，降级的依据。简单来说，就是用于构建ClusterNode。

**StatisticSlot**

用于记录、统计不同纬度的 runtime 指标监控信息。

**ParamFlowSlot**

对应热点流控。

**FlowSlot**

用于根据预设的限流规则以及前面 slot 统计的状态，来进行流量控制。对应流控规则。

**AuthoritySlot**

根据配置的黑白名单和调用来源信息，来做黑白名单控制。对应授权规则。

**DegradeSlot**

通过统计信息以及预设的规则，来做熔断降级。对应降级规则。

**SystemSlot**

通过系统的状态，例如 load1 等，来控制总的入口流量。对应系统规则。

#### 2.1spi机制

  SPI全称Service Provider Interface，是Java提供的一套用来被第三方实现或者扩展的接口，它可以用来启用框架扩展和替换组件。 SPI的作用就是为这些被扩展的API寻找服务实现。本质是**将接口实现类的全限定名配置在文件中，并由服务加载器读取配置文件，加载实现类**。这样可以在运行时，动态为接口替换实现类。通过在ClassPath路径下的META-INF/services文件夹查找文件，自动加载文件里所定义的类，进而实现可插拔，解耦。

Sentinel槽链中各Slot的执行顺序是固定好的。但并不是绝对不能改变的。Sentinel将ProcessorSlot 作为 SPI 接口进行扩展，使得 SlotChain 具备了扩展能力。用户可以自定义Slot并编排Slot 间的顺序。

![](https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/sentinel-slot-spi.png)



### 2.2 核心源码解析

分析源码之前首先说说几个核心类：

**context**

这是众多框架都有的一个概念，Context 代表调用链路**上下文**，贯穿一次调用链路中的所有 `Entry`。Context 维持着入口节点（`entranceNode`）、本次调用链路的 curNode、调用来源（`origin`）等信息。Context 名称即为调用链路入口名称。

Context 维持的方式：通过 ThreadLocal 传递，只有在入口 `enter` 的时候生效。由于 Context 是通过 ThreadLocal 传递的，因此对于异步调用链路，线程切换的时候会丢掉 Context，因此需要手动通过 `ContextUtil.runOnContext(context, f)` 来变换 context。

**Entry**

每一次资源调用都会创建一个 `Entry`。`Entry` 包含了资源名、curNode（当前统计节点）、originNode（来源统计节点）等信息。

`CtEntry` 为普通的 `Entry`，在调用 `SphU.entry(xxx)` 的时候创建。特性：Linked entry within current context（内部维护着 `parent` 和 `child`）

**需要注意的一点**：CtEntry 构造函数中会做**调用链的变换**，即将当前 Entry 接到传入 Context 的调用链路上（`setUpEntryFor`）。

资源调用结束时需要 `entry.exit()`。exit 操作会过一遍 slot chain exit，恢复调用栈，exit context 然后清空 entry 中的 context 防止重复调用。

**Node**

Sentinel 里面的各种种类的统计节点：

- `StatisticNode`：最为基础的统计节点，包含秒级和分钟级两个滑动窗口结构。
- `DefaultNode`：链路节点，用于统计调用链路上某个资源的数据，维持树状结构。
- `ClusterNode`：簇点，用于统计每个资源全局的数据（不区分调用链路），以及存放该资源的按来源区分的调用数据（类型为 `StatisticNode`）。特别地，`Constants.ENTRY_NODE` 节点用于统计全局的入口资源数据。
- `EntranceNode`：入口节点，特殊的链路节点，对应某个 Context 入口的所有调用数据。`Constants.ROOT` 节点也是入口节点。

构建的时机：

- `EntranceNode` 在 `ContextUtil.enter(xxx)` 的时候就创建了，然后塞到 Context 里面。
- `NodeSelectorSlot`：根据 context 创建 `DefaultNode`，然后 set curNode to context。
- `ClusterBuilderSlot`：首先根据 resourceName 创建 `ClusterNode`，并且 set clusterNode to defaultNode；然后再根据 origin 创建来源节点（类型为 `StatisticNode`），并且 set originNode to curEntry。

几种 Node 的维度（数目）：

- `ClusterNode` 的维度是 resource
- `DefaultNode` 的维度是 resource * context，存在每个 NodeSelectorSlot 的 `map` 里面
- `EntranceNode` 的维度是 context，存在 ContextUtil 类的 `contextNameNodeMap` 里面
- 来源节点（类型为 `StatisticNode`）的维度是 resource * origin，存在每个 ClusterNode 的 `originCountMap` 里面

<img src="https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/sentinel-node.png" style="zoom:67%;" />

下面开始分析重要源码：

从sentinel的核心功能限流来看，我们首先需要定义资源，然后在调用资源的时候做限流规则校验，从上面讲到的我们可以得知sentinel定义资源主流方式有：**1）整合主流框架，例如整合web，这样我们的每个接口都是资源；2）使用注解@SentinelResource 。3）直接使用SphU.entry("resourceName")，然后try-catch捕获异常。**

其实无论那种方式其最终都是通过**SphU.entry()**实现的

上面的方式1，方式2都是通过定义拦截器，然后再拦截器里面调用SphU.entry()实现限流校验的。

**主流框架**

如何对接口资源进行限流的，如下图所示：

![](https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/sentinel-web.png)

可以看出sentinel适配很多框架，对于我们的web接口定义了一个拦截器

**使用注解@SentinelResource**

使用注解定义资源，在sentinel整合springboot项目中时，自动配置类SentinelAutoConfiguration中一个**aop切面拦截器**bean，

```java
    @Bean
    @ConditionalOnMissingBean
    public SentinelResourceAspect sentinelResourceAspect() {
        return new SentinelResourceAspect();
    }
```

切面拦截器类**SentinelResourceAspect**

```java
@Aspect   //  AspectJ切面
public class SentinelResourceAspect extends AbstractSentinelAspectSupport {

    // 指定切入点为@SentinelResource注解
    @Pointcut("@annotation(com.alibaba.csp.sentinel.annotation.SentinelResource)")
    public void sentinelResourceAnnotationPointcut() {
    }

    // 指定此为环绕通知 around advice
    @Around("sentinelResourceAnnotationPointcut()")
    public Object invokeResourceWithSentinel(ProceedingJoinPoint pjp) throws Throwable {
        Method originMethod = resolveMethod(pjp);

        SentinelResource annotation = originMethod.getAnnotation(SentinelResource.class);
        if (annotation == null) {
            // Should not go through here.
            throw new IllegalStateException("Wrong state for SentinelResource annotation");
        }
        String resourceName = getResourceName(annotation.value(), originMethod);
        EntryType entryType = annotation.entryType();
        int resourceType = annotation.resourceType();
        Entry entry = null;
        try {
            // 要织入的、增强的功能，sentinel的核心入口
            entry = SphU.entry(resourceName, resourceType, entryType, pjp.getArgs());
            // 调用目标方法
            Object result = pjp.proceed();
            return result;
        } catch (BlockException ex) {
            return handleBlockException(pjp, annotation, ex);
        } catch (Throwable ex) {
            Class<? extends Throwable>[] exceptionsToIgnore = annotation.exceptionsToIgnore();
            // The ignore list will be checked first.
            if (exceptionsToIgnore.length > 0 && exceptionBelongsTo(ex, exceptionsToIgnore)) {
                throw ex;
            }
            if (exceptionBelongsTo(ex, annotation.exceptionsToTrace())) {
                traceException(ex);
                return handleFallback(pjp, annotation, ex);
            }

            // No fallback function can handle the exception, so throw it out.
            throw ex;
        } finally {
            if (entry != null) {
                entry.exit(1, pjp.getArgs());
            }
        }
    }
}

```

接下来，我们看 `SphU#entry`。自己跟进去，我们会来到 `CtSph#entryWithPriority` 方法，这个方法是 Sentinel 的骨架，非常重要

```java
 /**
     *
     * @param resourceWrapper  资源实例
     * @param count  默认值为1
     * @param prioritized  默认值为false
     * @param args
     * @return
     * @throws BlockException
     */
    private Entry entryWithPriority(ResourceWrapper resourceWrapper, int count, boolean prioritized, Object... args)
        throws BlockException {
        // 从ThreadLocal中获取context
        // 即一个请求会占用一个线程，一个线程会绑定一个context
        Context context = ContextUtil.getContext();
        // 若context是NullContext类型，则表示当前系统中的context数量已经超出的阈值
        // 即访问请求的数量已经超出了阈值。此时直接返回一个无需做规则检测的资源操作对象
        if (context instanceof NullContext) {
            // The {@link NullContext} indicates that the amount of context has exceeded the threshold,
            // so here init the entry only. No rule checking will be done.
            return new CtEntry(resourceWrapper, null, context);
        }

        // 若当前线程中没有绑定context，则创建一个context并将其放入到ThreadLocal
        if (context == null) {
            // Using default context.
            context = InternalContextUtil.internalEnter(Constants.CONTEXT_DEFAULT_NAME);
        }

        // 若全局开关是关闭的，则直接返回一个无需做规则检测的资源操作对象
        // Global switch is close, no rule checking will do.
        if (!Constants.ON) {
            return new CtEntry(resourceWrapper, null, context);
        }

        // 查找SlotChain
        ProcessorSlot<Object> chain = lookProcessChain(resourceWrapper);

        /*
         * Means amount of resources (slot chain) exceeds {@link Constants.MAX_SLOT_CHAIN_SIZE},
         * so no rule checking will be done.
         */
        // 若没有找到chain，则意味着chain数量超出了阈值，则直接返回一个无需做规则检测的资源操作对象
        if (chain == null) {
            return new CtEntry(resourceWrapper, null, context);
        }

        // 创建一个资源操作对象
        Entry e = new CtEntry(resourceWrapper, chain, context);
        try {
            // 对资源进行操作
            chain.entry(context, resourceWrapper, null, count, prioritized, args);
        } catch (BlockException e1) {
            e.exit(count, args);
            throw e1;
        } catch (Throwable e1) {
            // This should not happen, unless there are errors existing in Sentinel internal.
            RecordLog.info("Sentinel unexpected exception", e1);
        }
        return e;
    }
```

`lookProcessChain(resourceWrapper)` 这个方法。Sentinel 的处理核心都在这个**责任链**中，链中每一个节点是一个 `Slot` 实例，这个链通过 BlockException 异常来告知调用入口最终的执行情况

```java
 ProcessorSlot<Object> lookProcessChain(ResourceWrapper resourceWrapper) {
        // 从缓存map中获取当前资源的SlotChain
        // 缓存map的key为资源，value为其相关的SlotChain
        ProcessorSlotChain chain = chainMap.get(resourceWrapper);
        // DCL
        // 若缓存中没有相关的SlotChain，则创建一个并放入到缓存
        if (chain == null) {
            synchronized (LOCK) {
                chain = chainMap.get(resourceWrapper);
                if (chain == null) {
                    // Entry size limit.
                    // 缓存map的size >= chain数量最大阈值，则直接返回null，不再创建新的chain
                    if (chainMap.size() >= Constants.MAX_SLOT_CHAIN_SIZE) {
                        return null;
                    }

                    // 创建新的chain
                    chain = SlotChainProvider.newSlotChain();

                    // 防止迭代稳定性问题
                    Map<ResourceWrapper, ProcessorSlotChain> newMap = new HashMap<ResourceWrapper, ProcessorSlotChain>(
                        chainMap.size() + 1);
                    newMap.putAll(chainMap);
                    newMap.put(resourceWrapper, chain);
                    chainMap = newMap;
                }
            }
        }
        return chain;
    }
```

这个责任链由 `SlotChainProvider#newSlotChain` 生产，Sentinel 提供了 SPI 端点，让我们可以自己定制 Builder，如添加一个 Slot 进去。由于 `SlotChainBuilder` 接口设计的问题，我们只能全局所有的 resource 使用相同的责任链配置。

```java
 public static ProcessorSlotChain newSlotChain() {
        // 若builder不为null，则直接使用builder构建一个chain，否则先创建一个builder
        if (slotChainBuilder != null) {
            return slotChainBuilder.build();
        }

        // Resolve the slot chain builder SPI.
        // 通过SPI方式创建一个builder
        slotChainBuilder = SpiLoader.of(SlotChainBuilder.class).loadFirstInstanceOrDefault();

        // 若通过SPI方式未能创建builder，则手工new一个DefaultSlotChainBuilder
        if (slotChainBuilder == null) {
            // Should not go through here.
            RecordLog.warn("[SlotChainProvider] Wrong state when resolving slot chain builder, using default");
            slotChainBuilder = new DefaultSlotChainBuilder();
        } else {
            RecordLog.info("[SlotChainProvider] Global slot chain builder resolved: {}",
                slotChainBuilder.getClass().getCanonicalName());
        }
        // 构建一个chain
        return slotChainBuilder.build();
    }
```

在`slotChainBuilder.build()`中使用spi机制构建责任链chain，并且我们自定义slot加入该责任链扩展功能

```java
public class DefaultSlotChainBuilder implements SlotChainBuilder {

    @Override
    public ProcessorSlotChain build() {
        ProcessorSlotChain chain = new DefaultProcessorSlotChain();

        // 通过SPI方式构建Slot
        List<ProcessorSlot> sortedSlotList = SpiLoader.of(ProcessorSlot.class).loadInstanceListSorted();
        for (ProcessorSlot slot : sortedSlotList) {
            if (!(slot instanceof AbstractLinkedProcessorSlot)) {
                RecordLog.warn("The ProcessorSlot(" + slot.getClass().getCanonicalName() + ") is not an instance of AbstractLinkedProcessorSlot, can't be added into ProcessorSlotChain");
                continue;
            }

            chain.addLast((AbstractLinkedProcessorSlot<?>) slot);
        }

        return chain;
    }
}
```

接下来会进入`chain.entry(context, resourceWrapper, null, count, prioritized, args)`，执行责任链中的每个slot逻辑，这里就不一一详细叙述了，着重讲一个**限流FlowSlot、降级DegradeSlot**

![](https://mmbiz.qpic.cn/mmbiz_png/rtJ5LhxxzwmkzmkRURGVp3FT5Cxqicy7e2kQyc5ZsWYvLU31xVHe8RPAse2PRPqDbZXGXC5d1SgNRuTUGw1dz0w/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**FlowSlot**

这里的sp注解指定order值，在使用spi机制构建责任链chain时通过order指定每个slot的顺序

```java
@Spi(order = Constants.ORDER_FLOW_SLOT)
public class FlowSlot extends AbstractLinkedProcessorSlot<DefaultNode> {

    private final FlowRuleChecker checker;

    public FlowSlot() {
        this(new FlowRuleChecker());
    }

    /**
     * Package-private for test.
     *
     * @param checker flow rule checker
     * @since 1.6.1
     */
    FlowSlot(FlowRuleChecker checker) {
        AssertUtil.notNull(checker, "flow checker should not be null");
        this.checker = checker;
    }

    @Override
    public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count,
                      boolean prioritized, Object... args) throws Throwable {
        // 检测并应用流控规则
        checkFlow(resourceWrapper, context, node, count, prioritized);
        // 触发下一个Slot
        fireEntry(context, resourceWrapper, node, count, prioritized, args);
    }

    void checkFlow(ResourceWrapper resource, Context context, DefaultNode node, int count, boolean prioritized)
        throws BlockException {
        checker.checkFlow(ruleProvider, resource, context, node, count, prioritized);
    }

    @Override
    public void exit(Context context, ResourceWrapper resourceWrapper, int count, Object... args) {
        fireExit(context, resourceWrapper, count, args);
    }

    private final Function<String, Collection<FlowRule>> ruleProvider = new Function<String, Collection<FlowRule>>() {
        @Override
        public Collection<FlowRule> apply(String resource) {
            // Flow rule map should not be null.
            // 获取到所有资源的流控规则
            // map的key为资源名称，value为该资源上加载的所有流控规则
            Map<String, List<FlowRule>> flowRules = FlowRuleManager.getFlowRuleMap();
            // 获取指定资源的所有流控规则
            return flowRules.get(resource);
          
        }
    };
}
```

规则校验：FlowRuleChecker

```java
public class FlowRuleChecker {

    public void checkFlow(Function<String, Collection<FlowRule>> ruleProvider, ResourceWrapper resource,
                          Context context, DefaultNode node, int count, boolean prioritized) throws BlockException {
        if (ruleProvider == null || resource == null) {
            return;
        }
        // 获取到指定资源的所有流控规则
        Collection<FlowRule> rules = ruleProvider.apply(resource.getName());
        if (rules != null) {
            // 逐个应用流控规则。若无法通过则抛出异常，后续规则不再应用
            for (FlowRule rule : rules) {
                if (!canPassCheck(rule, context, node, count, prioritized)) {
                    throw new FlowException(rule.getLimitApp(), rule);
                }
            }
        }
    }

    public boolean canPassCheck(/*@NonNull*/ FlowRule rule, Context context, DefaultNode node,
                                                    int acquireCount) {
        return canPassCheck(rule, context, node, acquireCount, false);
    }

    public boolean canPassCheck(/*@NonNull*/ FlowRule rule, Context context, DefaultNode node, int acquireCount,
                                                    boolean prioritized) {
        // 从规则中获取要限定的来源
        String limitApp = rule.getLimitApp();
        // 若限流的来源为null，则请求直接通过
        if (limitApp == null) {
            return true;
        }

        // 使用规则处理集群流控
        if (rule.isClusterMode()) {
            return passClusterCheck(rule, context, node, acquireCount, prioritized);
        }

        // 使用规则处理单机流控
        return passLocalCheck(rule, context, node, acquireCount, prioritized);
    }

    private static boolean passLocalCheck(FlowRule rule, Context context, DefaultNode node, int acquireCount,
                                          boolean prioritized) {
        // 通过规则形成选择出的规则node
        Node selectedNode = selectNodeByRequesterAndStrategy(rule, context, node);
        // 若没有选择出node，说明没有规则，则直接返回true，表示通过检测
        if (selectedNode == null) {
            return true;
        }

        // 使用规则进行逐项检测
        return rule.getRater().canPass(selectedNode, acquireCount, prioritized);
    }

    static Node selectReferenceNode(FlowRule rule, Context context, DefaultNode node) {
        String refResource = rule.getRefResource();
        int strategy = rule.getStrategy();

        if (StringUtil.isEmpty(refResource)) {
            return null;
        }

        if (strategy == RuleConstant.STRATEGY_RELATE) {
            return ClusterBuilderSlot.getClusterNode(refResource);
        }

        if (strategy == RuleConstant.STRATEGY_CHAIN) {
            if (!refResource.equals(context.getName())) {
                return null;
            }
            return node;
        }
        // No node.
        return null;
    }

    private static boolean filterOrigin(String origin) {
        // Origin cannot be `default` or `other`.
        return !RuleConstant.LIMIT_APP_DEFAULT.equals(origin) && !RuleConstant.LIMIT_APP_OTHER.equals(origin);
    }

    static Node selectNodeByRequesterAndStrategy(/*@NonNull*/ FlowRule rule, Context context, DefaultNode node) {
        // The limit app should not be empty.
        String limitApp = rule.getLimitApp();
        int strategy = rule.getStrategy();
        String origin = context.getOrigin();

        if (limitApp.equals(origin) && filterOrigin(origin)) {
            if (strategy == RuleConstant.STRATEGY_DIRECT) {
                // Matches limit origin, return origin statistic node.
                return context.getOriginNode();
            }

            return selectReferenceNode(rule, context, node);
        } else if (RuleConstant.LIMIT_APP_DEFAULT.equals(limitApp)) {
            if (strategy == RuleConstant.STRATEGY_DIRECT) {
                // Return the cluster node.
                return node.getClusterNode();
            }

            return selectReferenceNode(rule, context, node);
        } else if (RuleConstant.LIMIT_APP_OTHER.equals(limitApp)
            && FlowRuleManager.isOtherOrigin(origin, rule.getResource())) {
            if (strategy == RuleConstant.STRATEGY_DIRECT) {
                return context.getOriginNode();
            }

            return selectReferenceNode(rule, context, node);
        }

        return null;
    }

    private static boolean passClusterCheck(FlowRule rule, Context context, DefaultNode node, int acquireCount,
                                            boolean prioritized) {
        try {
            TokenService clusterService = pickClusterService();
            if (clusterService == null) {
                return fallbackToLocalOrPass(rule, context, node, acquireCount, prioritized);
            }
            long flowId = rule.getClusterConfig().getFlowId();
            TokenResult result = clusterService.requestToken(flowId, acquireCount, prioritized);
            return applyTokenResult(result, rule, context, node, acquireCount, prioritized);
            // If client is absent, then fallback to local mode.
        } catch (Throwable ex) {
            RecordLog.warn("[FlowRuleChecker] Request cluster token unexpected failed", ex);
        }
        // Fallback to local flow control when token client or server for this rule is not available.
        // If fallback is not enabled, then directly pass.
        return fallbackToLocalOrPass(rule, context, node, acquireCount, prioritized);
    }

    private static boolean fallbackToLocalOrPass(FlowRule rule, Context context, DefaultNode node, int acquireCount,
                                                 boolean prioritized) {
        if (rule.getClusterConfig().isFallbackToLocalWhenFail()) {
            return passLocalCheck(rule, context, node, acquireCount, prioritized);
        } else {
            // The rule won't be activated, just pass.
            return true;
        }
    }

    private static TokenService pickClusterService() {
        if (ClusterStateManager.isClient()) {
            return TokenClientProvider.getClient();
        }
        if (ClusterStateManager.isServer()) {
            return EmbeddedClusterTokenServerProvider.getServer();
        }
        return null;
    }

    private static boolean applyTokenResult(/*@NonNull*/ TokenResult result, FlowRule rule, Context context,
                                                         DefaultNode node,
                                                         int acquireCount, boolean prioritized) {
        switch (result.getStatus()) {
            case TokenResultStatus.OK:
                return true;
            case TokenResultStatus.SHOULD_WAIT:
                // Wait for next tick.
                try {
                    Thread.sleep(result.getWaitInMs());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                return true;
            case TokenResultStatus.NO_RULE_EXISTS:
            case TokenResultStatus.BAD_REQUEST:
            case TokenResultStatus.FAIL:
            case TokenResultStatus.TOO_MANY_REQUEST:
                return fallbackToLocalOrPass(rule, context, node, acquireCount, prioritized);
            case TokenResultStatus.BLOCKED:
            default:
                return false;
        }
    }

```

**DegradeSlot**

```java
@Spi(order = Constants.ORDER_DEGRADE_SLOT)
public class DegradeSlot extends AbstractLinkedProcessorSlot<DefaultNode> {

    @Override
    public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count,
                      boolean prioritized, Object... args) throws Throwable {
        // 完成熔断降级检测
        performChecking(context, resourceWrapper);
        // 触发下一个节点
        fireEntry(context, resourceWrapper, node, count, prioritized, args);
    }

    void performChecking(Context context, ResourceWrapper r) throws BlockException {
        // 获取到当前资源的所有熔断器
        List<CircuitBreaker> circuitBreakers = DegradeRuleManager.getCircuitBreakers(r.getName());
        // 若熔断器为空，则直结束
        if (circuitBreakers == null || circuitBreakers.isEmpty()) {
            return;
        }
        // 逐个尝试所有熔断器
        for (CircuitBreaker cb : circuitBreakers) {
            // 若没有通过当前熔断器，则直接抛出异常
            if (!cb.tryPass(context)) {
                throw new DegradeException(cb.getRule().getLimitApp(), cb.getRule());
            }
        }
    }

    @Override
    public void exit(Context context, ResourceWrapper r, int count, Object... args) {
        Entry curEntry = context.getCurEntry();
        if (curEntry.getBlockError() != null) {
            fireExit(context, r, count, args);
            return;
        }
        List<CircuitBreaker> circuitBreakers = DegradeRuleManager.getCircuitBreakers(r.getName());
        if (circuitBreakers == null || circuitBreakers.isEmpty()) {
            fireExit(context, r, count, args);
            return;
        }

        if (curEntry.getBlockError() == null) {
            // passed request
            for (CircuitBreaker circuitBreaker : circuitBreakers) {
                circuitBreaker.onRequestComplete(context);
            }
        }

        fireExit(context, r, count, args);
    }
}

```

### 3.总结

sentinel主要是基于7种不同的Slot形成了一个链表，每个Slot都各司其职，自己做完分内的事之后，会把请求传递给下一个Slot，直到在某一个Slot中命中规则后抛出BlockException而终止。

前三个Slot负责做统计，后面的Slot负责根据统计的结果结合配置的规则进行具体的控制，是Block该请求还是放行。

