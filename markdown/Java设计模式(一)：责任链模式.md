# Java设计模式(一)：责任链模式

研究sentinel源码时候说到了责任链这种设计模式，所以近期打算开启java设计模式分析。

### 1.设计模式概念

**概念**

Java包含23种设计模式，是一套对代码设计经验的总结，被人们反复利用，多人熟知的代码设计方式。在众多热门框架源码中用到许多种设计模式，这也是为什么框架可适配、可扩展，经典源码被大家反复阅读学习的原因吧。

**设计模式的好处**

设计模式作为一个程序员进阶的必修之路，合理运用设计模式能写出**可扩展、可读、可维护的高质量代码**。个人总结了一下几个学设计模式带来的直观好处：

1）应对面试中的设计模式相关问题；

2）告别写被人吐槽的烂代码；

3）提高复杂代码的设计和开发能力；

4）让读源码、学框架事半功倍；

5）为你的职场发展做铺垫，将来也写一个大家追捧的框架。

### 2.设计模式六大原则(面试高频考点)

**开闭原则**

 对扩展开放，对修改关闭（尽可能对代码少修改）

**里氏替换原则**

 它是面向对象基本原则之一，任何父类（基类）出现的地方，子类都可以出现，也就是子类可以替换父类的任何功能（体现了父类的可扩展性）

**依赖倒转原则**

 尽可能面向接口编程，依赖接口而不依赖类

**接口隔离原则**

 一个类如果能实现多个接口，尽可能实现多个，为了降低依赖，降低耦合

**最少知道原则**

 一个实体尽可能少的与其他实体产生相互关联关系，将实体的功能独立

**合成复用原则**

 尽量使用合成，聚合的方式，而不使用继承

### 3.设计模式分类

Java设计模式分为三大类

创建型模式：对象实例化的模式，创建型模式用于解耦对象的实例化过程。

结构型模式：把类或对象结合在一起形成一个更大的结构。

行为型模式：类和对象如何交互，及划分责任和算法

![](https://images2017.cnblogs.com/blog/401339/201709/401339-20170928225241215-295252070.png)

接下来步入今天的主角：**责任链模式**

### 4.责任链模式定义

将请求的发送和接收解耦，让多个接收对象都有机会处理这个请求。将这些接收对象串成一条链，并沿着这条链传递这个请求，直到链上的某个接收对象能够处理它为止。实际上，责任链模式还有一种变体，那就是请求会被所有接收对象都能处理一遍，直到某个接收对象不能正常处理再退出，或者全部执行一遍，不存在中途终止的情况。

在责任链模式中，一个产品在流水线经过多个员工（也就是刚刚定义中说的“接收对象”）依次处理加工。一个产品先经过 A 处理加工，然后再把请求传递给 B 处理加工，B 处理完后再传递给 C 处理加工，以此类推，形成一个链条。链条上的每个处理器各自承担各自的处理职责，所以叫责任链模式。

**示例代码**

首先定义一个所有处理器类的抽象父类Handler ，doHandle() 是抽象方法，需要具体处理类来重写实现该方法

```java
/**
 * @author fjzheng
 * @version 1.0
 * @date 2021/11/9 15:15
 */

public abstract class Handler {
    //successor代表责任链中下一环节，eg：这里代表流水线上下一加工人员
    protected Handler next = null;

    public void setNext(Handler next) {
        this.next = next;
    }

    //final修饰不能重写覆盖此方法
    public final void handle() {
        //执行责任链的具体处理类方法
        boolean handled = doHandle();
        //判断chain中当前环节是否执行成功和chain是有下一环节
        if (next != null && handled) {
            next.handle();
        }
    }

    protected abstract Boolean doHandle();
}
```

接下来是三个具体的处理加工类，这里即代表加工人员，通过实现抽象类handler的doHandle()，表明自己在加工责任链中任务，当然这里可以不只是三个，可以是n个，只要实现抽象类handler，然后再责任链中加入自己即可。

```java
/**
 * @author fjzheng
 * @version 1.0
 * @date 2021/11/9 15:16
 */

/**
 * 这里可以写handlerA具体处理逻辑，最终返回处理是否成功
 */
public class HandlerA extends Handler {
    @Override
    public Boolean doHandle() {
        System.out.println("handlerA execute.....");
        return true;

    }
}

/**
 * 这里可以写handlerB具体处理逻辑，最终返回处理是否成功
 */
public class HandlerB extends Handler {
    @Override
    public Boolean doHandle() {
        System.out.println("handlerB execute.....");
        return true;
    }
}

/**
 * 这里可以写handlerC具体处理逻辑，最终返回处理是否成功
 */
public class HandlerC extends Handler{
    @Override
    protected Boolean doHandle() {
        System.out.println("handlerC execute.....");
        return true;
    }
}

```

构建加工处理责任链HandlerChain

```java
/**
 * @author fjzheng
 * @version 1.0
 * @date 2021/11/9 15:17
 */

/**
 * HandlerChain 是加工人员处理链，从数据结构的角度来看，它就是一个记录了链头、链尾的链表。其中，记录链尾是为了方便添加加工人员
 */
public class HandlerChain {
    private Handler head = null;
    private Handler tail = null;

    public void addHandler(Handler handler) {
        handler.setNext(null);
        //第一次添加具体处理类时，头、尾都等于该类
        if (head == null) {
            head = handler;
            tail = handler;
            return;
        }
        //不是第一添加，该类设置tail的下一环节，将尾部设置该类，最终形成一个链表结构
        tail.setNext(handler);
        tail = handler;
    }

    public void handle() {
        //从第一个开始执行
        if (head != null) {
            head.handle();
        }
    }
}
```

测试用例

```java
/**
 * @author fjzheng
 * @version 1.0
 * @date 2021/11/9 15:18
 */

/**
 * 这里模拟一个产品在流水线上加工流程，如果在某个加工人员出错，就不再流向下一个加工人员
 * 往责任链chain中添加加工人员handler
 */
public class TestDemo {
    public static void main(String[] args) {
        HandlerChain chain = new HandlerChain();
        chain.addHandler(new HandlerA());
        chain.addHandler(new HandlerB());
        chain.addHandler(new HandlerC());
        chain.handle();
    }
}
```

结果示例如下：

```
handlerA execute.....
handlerB execute.....
handlerC execute.....
```

上面是使用链表来构造责任链的，接下来我们通过数组实现，代码如下：

```java

public interface IHandler {
  boolean handle();
}

public class HandlerA implements IHandler {
  @Override
  public boolean handle() {
    boolean handled = false;
    //...
    return handled;
  }
}

public class HandlerB implements IHandler {
  @Override
  public boolean handle() {
    boolean handled = false;
    //...
    return handled;
  }
}

public class HandlerChain {
  private List<IHandler> handlers = new ArrayList<>();

  public void addHandler(IHandler handler) {
    this.handlers.add(handler);
  }

  public void handle() {
    for (IHandler handler : handlers) {
      boolean handled = handler.handle();
      if (handled) {
        break;
      }
    }
  }
}

// 使用举例
public class Application {
  public static void main(String[] args) {
    HandlerChain chain = new HandlerChain();
    chain.addHandler(new HandlerA());
    chain.addHandler(new HandlerB());
    chain.handle();
  }
}
```

### 5.责任链模式的应用场景

敏感词检测在当今互联网是一个必不可少的功能要求，防止发布一些包含敏感词（比如涉黄、广告、反动等词汇）不良信息。针对这个应用场景，我们就可以利用职责链模式来过滤这些敏感词

```java

public interface SensitiveWordFilter {
  boolean doFilter(Content content);
}

public class SexyWordFilter implements SensitiveWordFilter {
  @Override
  public boolean doFilter(Content content) {
    boolean legal = true;
    //...
    return legal;
  }
}

// PoliticalWordFilter、AdsWordFilter类代码结构与SexyWordFilter类似

public class SensitiveWordFilterChain {
  private List<SensitiveWordFilter> filters = new ArrayList<>();

  public void addFilter(SensitiveWordFilter filter) {
    this.filters.add(filter);
  }

  // return true if content doesn't contain sensitive words.
  public boolean filter(Content content) {
    for (SensitiveWordFilter filter : filters) {
      if (!filter.doFilter(content)) {
        return false;
      }
    }
    return true;
  }
}

public class ApplicationDemo {
  public static void main(String[] args) {
    SensitiveWordFilterChain filterChain = new SensitiveWordFilterChain();
    filterChain.addFilter(new AdsWordFilter());
    filterChain.addFilter(new SexyWordFilter());
    filterChain.addFilter(new PoliticalWordFilter());

    boolean legal = filterChain.filter(new Content());
    if (!legal) {
      // 不发表
    } else {
      // 发表
    }
  }
}
```

具体过滤算法这里不做阐述，毕竟不是本文的主要内容。

接下来展示一下不用责任链设计模式实现敏感词检测功能：

```java

public class SensitiveWordFilter {
  // return true if content doesn't contain sensitive words.
  public boolean filter(Content content) {
    if (!filterSexyWord(content)) {
      return false;
    }

    if (!filterAdsWord(content)) {
      return false;
    }

    if (!filterPoliticalWord(content)) {
      return false;
    }

    return true;
  }

  private boolean filterSexyWord(Content content) {
    //....
  }

  private boolean filterAdsWord(Content content) {
    //...
  }

  private boolean filterPoliticalWord(Content content) {
    //...
  }
}
```

你可能会说，我像上面这样也可以实现敏感词过滤功能，而且代码更加简单，为什么非要使用职责链模式呢？这是不是过度设计呢？

首先，我们来看，**职责链模式如何应对代码的复杂性**。

将大块代码逻辑拆分成函数，将大类拆分成小类，是应对代码复杂性的常用方法。应用职责链模式，我们把各个敏感词过滤函数继续拆分出来，设计成独立的类，进一步简化了 SensitiveWordFilter 类，让 SensitiveWordFilter 类的代码不会过多，过复杂。

其次，我们再来看，**职责链模式如何让代码满足开闭原则，提高代码的扩展性。**

当我们要扩展新的过滤算法的时候，比如，我们还需要过滤特殊符号，按照非职责链模式的代码实现方式，我们需要修改 SensitiveWordFilter 的代码，违反开闭原则。不过，这样的修改还算比较集中，也是可以接受的。而职责链模式的实现方式更加优雅，只需要新添加一个 Filter 类，并且通过 addFilter() 函数将它添加到 FilterChain 中即可，其他代码完全不需要修改

假设敏感词过滤框架并不是我们开发维护的，而是我们引入的一个第三方框架，我们要扩展一个新的过滤算法，不可能直接去修改框架的源码。这个时候，利用职责链模式就能达到开篇所说的，在不修改框架源码的情况下，基于职责链模式提供的扩展点，来扩展新的功能。换句话说，我们在框架这个代码范围内实现了开闭原则。

除此之外，利用职责链模式相对于不用职责链的实现方式，还有一个好处，那就是配置过滤算法更加灵活，可以只选择使用某几个过滤算法。

### 6.框架中的常用过滤器、拦截器中责任链模式的应用

**过滤器Filter**

Servlet Filter 是 Java Servlet 规范中定义的组件，翻译成中文就是过滤器，它可以实现对 HTTP 请求的过滤功能，比如鉴权、限流、记录日志、验证参数等等。因为它是 Servlet 规范的一部分，所以，只要是支持 Servlet 的 Web 容器（比如，Tomcat、Jetty 等），都支持过滤器功能。为了帮助你理解，我画了一张示意图阐述它的工作原理，如下所示。

![](https://static001.geekbang.org/resource/image/32/21/3296abd63a61ebdf4eff3a6530979e21.jpg)

使用示例代码：

```java
@WebFilter
public class LogFilter implements Filter {
  @Override
  public void init(FilterConfig filterConfig) throws ServletException {
    // 在创建Filter时自动调用，
    // 其中filterConfig包含这个Filter的配置参数，比如name之类的（从配置文件中读取的）
  }

  @Override
  public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
    System.out.println("拦截客户端发送来的请求.");
    chain.doFilter(request, response);
    System.out.println("拦截发送给客户端的响应.");
  }

  @Override
  public void destroy() {
    // 在销毁Filter时自动调用
  }
}


```

我们可以看到doFilter()方法中的FilterChain参数，这就是责任链模式的身影，接下来解析一下FilterChain。

不过，我们前面也讲过，Servlet 只是一个规范，并不包含具体的实现，所以，Servlet 中的 FilterChain 只是一个接口定义。具体的实现类由遵从 Servlet 规范的 Web 容器来提供，比如，ApplicationFilterChain 类就是 Tomcat 提供的 FilterChain 的实现类，源码如下所示

```java

public final class ApplicationFilterChain implements FilterChain {
  private int pos = 0; //当前执行到了哪个filter
  private int n; //filter的个数
  private ApplicationFilterConfig[] filters;
  private Servlet servlet;
  
  @Override
  public void doFilter(ServletRequest request, ServletResponse response) {
    if (pos < n) {
      ApplicationFilterConfig filterConfig = filters[pos++];
      Filter filter = filterConfig.getFilter();
      filter.doFilter(request, response, this);
    } else {
      // filter都处理完毕后，执行servlet
      servlet.service(request, response);
    }
  }
  
  public void addFilter(ApplicationFilterConfig filterConfig) {
    for (ApplicationFilterConfig filter:filters)
      if (filter==filterConfig)
         return;

    if (n == filters.length) {//扩容
      ApplicationFilterConfig[] newFilters = new ApplicationFilterConfig[n + INCREMENT];
      System.arraycopy(filters, 0, newFilters, 0, n);
      filters = newFilters;
    }
    filters[n++] = filterConfig;
  }
}
```

ApplicationFilterChain 中的 doFilter() 函数的代码实现比较有技巧，实际上是一个递归调用。你可以用每个 Filter（比如 LogFilter）的 doFilter() 的代码实现，直接替换 ApplicationFilterChain 的第 12 行代码，一眼就能看出是递归调用了。我替换了一下，如下所示：

```java

  @Override
  public void doFilter(ServletRequest request, ServletResponse response) {
    if (pos < n) {
      ApplicationFilterConfig filterConfig = filters[pos++];
      Filter filter = filterConfig.getFilter();
      //filter.doFilter(request, response, this);
      //把filter.doFilter的代码实现展开替换到这里
      System.out.println("拦截客户端发送来的请求.");
      chain.doFilter(request, response); // chain就是this
      System.out.println("拦截发送给客户端的响应.")
    } else {
      // filter都处理完毕后，执行servlet
      servlet.service(request, response);
    }
  }
```

这样实现主要是为了在一个 doFilter() 方法中，支持双向拦截，既能拦截客户端发送来的请求，也能拦截发送给客户端的响应，你可以结合着 LogFilter 那个例子，以及对比待会要讲到的 Spring Interceptor，来自己理解一下。而我们上面给出的两种(基于链表、数组)实现方式，都没法做到在业务逻辑执行的前后，同时添加处理代码。

**拦截器Interceptor**

刚刚讲了 Servlet Filter，现在我们来讲一个功能上跟它非常类似的东西，Spring Interceptor，翻译成中文就是拦截器。尽管英文单词和中文翻译都不同，但这两者基本上可以看作一个概念，都用来实现对 HTTP 请求进行拦截处理。它们不同之处在于，Servlet Filter 是 Servlet 规范的一部分，实现依赖于 Web 容器。Spring Interceptor 是 Spring MVC 框架的一部分，由 Spring MVC 框架来提供实现。客户端发送的请求，会先经过 Servlet Filter，然后再经过 Spring Interceptor，最后到达具体的业务代码中。我画了一张图来阐述一个请求的处理流程，具体如下所示：

![](https://static001.geekbang.org/resource/image/fe/68/febaa9220cb9ad2f0aafd4e5c3c19868.jpg)

使用示例：

```java

public class LogInterceptor implements HandlerInterceptor {

  @Override
  public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
    System.out.println("拦截客户端发送来的请求.");
    return true; // 继续后续的处理
  }

  @Override
  public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
    System.out.println("拦截发送给客户端的响应.");
  }

  @Override
  public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
    System.out.println("这里总是被执行.");
  }
}


@Configuration
public class OrderWebConfig implements WebMvcConfigurer {
    @Resource
    private AuthInterceptor authInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(LogInterceptor).excludePathPatterns("/api/mall/order/orderNo/**");
    }
}
```

拦截器也是基于职责链模式实现的。其中，HandlerExecutionChain 类是职责链模式中的处理器链。它的实现相较于 Tomcat 中的 ApplicationFilterChain 来说，逻辑更加清晰，不需要使用递归来实现，主要是因为它将请求和响应的拦截工作，拆分到了两个函数中实现。HandlerExecutionChain 的源码如下所示，同样，我对代码也进行了一些简化，只保留了关键代码

```java

public class HandlerExecutionChain {
 private final Object handler;
 private HandlerInterceptor[] interceptors;
 
 public void addInterceptor(HandlerInterceptor interceptor) {
  initInterceptorList().add(interceptor);
 }

 boolean applyPreHandle(HttpServletRequest request, HttpServletResponse response) throws Exception {
  HandlerInterceptor[] interceptors = getInterceptors();
  if (!ObjectUtils.isEmpty(interceptors)) {
   for (int i = 0; i < interceptors.length; i++) {
    HandlerInterceptor interceptor = interceptors[i];
    if (!interceptor.preHandle(request, response, this.handler)) {
     triggerAfterCompletion(request, response, null);
     return false;
    }
   }
  }
  return true;
 }

 void applyPostHandle(HttpServletRequest request, HttpServletResponse response, ModelAndView mv) throws Exception {
  HandlerInterceptor[] interceptors = getInterceptors();
  if (!ObjectUtils.isEmpty(interceptors)) {
   for (int i = interceptors.length - 1; i >= 0; i--) {
    HandlerInterceptor interceptor = interceptors[i];
    interceptor.postHandle(request, response, this.handler, mv);
   }
  }
 }

 void triggerAfterCompletion(HttpServletRequest request, HttpServletResponse response, Exception ex)
   throws Exception {
  HandlerInterceptor[] interceptors = getInterceptors();
  if (!ObjectUtils.isEmpty(interceptors)) {
   for (int i = this.interceptorIndex; i >= 0; i--) {
    HandlerInterceptor interceptor = interceptors[i];
    try {
     interceptor.afterCompletion(request, response, this.handler, ex);
    } catch (Throwable ex2) {
     logger.error("HandlerInterceptor.afterCompletion threw exception", ex2);
    }
   }
  }
 }
}
```

在 SpringMVC 框架中，DispatcherServlet 的 doDispatch() 方法来分发请求，它在真正的业务逻辑执行前后，执行 HandlerExecutionChain 中的 applyPreHandle() 和 applyPostHandle() 函数，用来实现拦截的功能。

