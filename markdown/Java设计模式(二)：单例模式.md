# Java设计模式(二)：单例模式

### 1.定义

**单例设计模式**（Singleton Design Pattern）理解起来非常简单。一个类只允许创建一个对象（或者实例），那这个类就是一个单例类，这种设计模式就叫作单例设计模式，简称单例模式。

### 2.使用场景

单例模式应用的场景一般发现在以下条件下：

1）资源共享的情况下，避免由于资源操作时导致的性能或损耗等。如上述中的日志文件，应用配置。

2）控制资源的情况下，方便资源之间的互相通信。如线程池等。

从业务概念上，如果有些数据在系统中只应保存一份，那就比较适合设计为单例类。比如，配置信息类、数据库连接池、日志对象等等。在系统中，我们只有一个配置文件，当配置文件被加载到内存之后，以对象的形式存在，也理所应当只有一份。如果资源对象存在多个可能造成资源冲突、数据不一致等问题。

如下面的唯一递增 ID 号码生成器，如果程序中有两个对象，那就会存在生成重复 ID 的情况，所以，我们应该将 ID 生成器类设计为单例

```java

import java.util.concurrent.atomic.AtomicLong;
public class IdGenerator {
  // AtomicLong是一个Java并发库中提供的一个原子变量类型,
  // 它将一些线程不安全需要加锁的复合操作封装为了线程安全的原子操作，
  // 比如下面会用到的incrementAndGet().
  private AtomicLong id = new AtomicLong(0);
  private static final IdGenerator instance = new IdGenerator();
  private IdGenerator() {}
  public static IdGenerator getInstance() {
    return instance;
  }
  public long getId() { 
    return id.incrementAndGet();
  }
}

// IdGenerator使用举例
long id = IdGenerator.getInstance().getId();
```

### 3.单例模式的实现方式

要实现一个单例，我们需要关注一下几点：

1）构造函数需要是 private 访问权限的，这样才能避免外部通过 new 创建实例；

2）考虑对象创建时的线程安全问题；

3）考虑是否支持延迟加载；

4）考虑 getInstance() 性能是否高（是否加锁）

针对以上几个注意点，实现单例方式大概有以下几种：

**饿汉式**

饿汉式的实现方式比较简单。**在类加载的时候，instance 静态实例就已经创建并初始化好了，所以，instance 实例的创建过程是线程安全的**。不过，这样的实现方式**不支持延迟加载**（在真正用到 IdGenerator 的时候，再创建实例），从名字中我们也可以看出这一点，不关三七二十一先创建单例对象再说。具体的代码实现如下所示：

```java

public class IdGenerator { 
  private AtomicLong id = new AtomicLong(0);
  private static final IdGenerator instance = new IdGenerator();
  
  private IdGenerator() {}
  
  public static IdGenerator getInstance() {
    return instance;
  }
  
  public long getId() { 
    return id.incrementAndGet();
  }
}
```

因为不支持延迟加载，如果实例占用资源多（比如占用内存多）或初始化耗时长（比如需要加载各种配置文件），提前初始化实例是一种浪费资源的行为。最好的方法应该在用到的时候再去初始化，不过对此看法仁者见仁智者见智罢了，因为凡事都是有两面性的，这种饿汉式加载是典型的空间换时间，虽然我们在项目启动时加载实例化单例对象花费了空间和时间，但是相对于用的时候再加载实例化的好处就是用的时候不用花费时间去创建，极大提高了接口响应速度。

**懒汉式**

通过上面知道饿汉式不支持延迟加载，所以懒汉式相对于饿汉式的优势是支持延迟加载，具体实现如下：

```java

public class IdGenerator { 
  private AtomicLong id = new AtomicLong(0);
  private static IdGenerator instance;
  
  private IdGenerator() {}
  
  public static synchronized IdGenerator getInstance() {
    if (instance == null) {
      instance = new IdGenerator();
    }
    return instance;
  }
  
  public long getId() { 
    return id.incrementAndGet();
  }
}
```

不过懒汉式的缺点也很明显，我们给 getInstance() 这个方法加了一把大锁（synchronzed），导致这个函数的并发度很低。量化一下的话，并发度是 1，也就相当于串行操作了。而这个函数是在单例使用期间，一直会被调用。如果这个单例类偶尔会被用到，那这种实现方式还可以接受。但是，如果频繁地用到，那频繁加锁、释放锁及并发度低等问题，会导致性能瓶颈，这种实现方式就不可取了。

**双重检测式**

饿汉式不支持延迟加载，懒汉式有性能问题，不支持高并发。那我们再来看一种既支持延迟加载、又支持高并发的单例实现方式，也就是双重检测实现方式。

在这种实现方式中，只要 instance 被创建之后，即便再调用 getInstance() 函数也不会再进入到加锁逻辑中了。所以，这种实现方式解决了懒汉式并发度低的问题。具体的代码实现如下所示：

```java

public class IdGenerator { 
  private AtomicLong id = new AtomicLong(0);
  private volatile static IdGenerator instance;
  
  private IdGenerator() {}
  
  public static IdGenerator getInstance() {
    if (instance == null) {
      synchronized(IdGenerator.class) { // 此处为类级别的锁
        if (instance == null) {
          instance = new IdGenerator();
        }
      }
    }
    return instance;
  }
  
  public long getId() { 
    return id.incrementAndGet();
  }
}
```

**静态内部类**

再来看一种比双重检测更加简单的实现方法，那就是利用 Java 的静态内部类。它有点类似饿汉式，但又能做到了延迟加载。具体是怎么做到的呢？我们先来看它的代码实现

```java

public class IdGenerator { 
  private AtomicLong id = new AtomicLong(0);
  
  private IdGenerator() {}

  //内部类
  private static class SingletonHolder{
    private static final IdGenerator instance = new IdGenerator();
  }
  
  public static IdGenerator getInstance() {
    return SingletonHolder.instance;
  }
 
  public long getId() { 
    return id.incrementAndGet();
  }
}
```

**SingletonHolder 是一个静态内部类，当外部类 IdGenerator 被加载的时候，并不会创建 SingletonHolder 实例对象。只有当调用 getInstance() 方法时，SingletonHolder 才会被加载，这个时候才会创建 instance。instance 的唯一性、创建过程的线程安全性，都由 JVM 来保证。所以，这种实现方法既保证了线程安全，又能做到延迟加载**。

**枚举实现**

基于枚举类型的单例实现是一种最简单的实现方式。这种实现方式通过 Java 枚举类型本身的特性，保证了实例创建的线程安全性和实例的唯一性。具体的代码如下所示：

```java

public enum IdGenerator {
  INSTANCE;
  private AtomicLong id = new AtomicLong(0);
 
  public long getId() { 
    return id.incrementAndGet();
  }
}
```

调用方法：

```java
public class Main {

    public static void main(String[] args) {
        IdGenerator.INSTANCE.getId();
    }

}
```

### 4.单例模式存在的问题

大部分情况下，我们在项目中使用单例，都是用它来表示一些全局唯一类，比如配置信息类、连接池类、ID 生成器类。单例模式书写简洁、使用方便，在代码中，我们不需要创建对象，直接通过类似 IdGenerator.getInstance().getId() 这样的方法来调用就可以了。但是，这种使用方法有点类似硬编码（hard code），会带来诸多问题。

1）不适用于变化的对象，如果同一类型的对象总是要在不同的用例场景发生变化，单例就会引起数据的错误，不能保存彼此的状态。 
2）由于单利模式中没有抽象层，因此单例类的扩展有很大的困难。 
3）单例类的职责过重，在一定程度上违背了“单一职责原则”。 
4）滥用单例将带来一些负面问题，如为了节省资源将数据库连接池对象设计为的单例类，可能会导致共享连接池对象的程序过多而出现连接池溢出；如果实例化的对象长时间不被利用，系统会认为是垃圾而被回收，这将导致对象状态的丢失。

使用注意事项：
 1）使用时不能用反射模式创建单例，否则会实例化一个新的对象 
 2）使用懒单例模式时注意线程安全问题 
 3）饿单例模式和懒单例模式构造方法都是私有的，因而是不能被继承的，有些单例模式可以被继承。

