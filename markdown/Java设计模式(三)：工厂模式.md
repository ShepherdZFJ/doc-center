# Java设计模式(三)：工厂模式

### 1.定义

工厂模式（Factory Pattern）是 Java 中最常用的设计模式之一。这种类型的设计模式属于创建型模式，它提供了一种创建对象的最佳方式。在工厂模式中，我们在创建对象时不会对客户端暴露创建逻辑，并且是通过使用一个共同的接口来指向新创建的对象。

工厂模式分为三种更加细分的类型：简单工厂、工厂方法和抽象工厂。

接下来我将创建一个 *Shape* 接口和实现 *Shape* 接口的实体类。下一步是定义工厂类 *ShapeFactory*。

*FactoryPatternDemo* 类使用 *ShapeFactory* 来获取 *Shape* 对象。它将向 *ShapeFactory* 传递信息（*CIRCLE / RECTANGLE / SQUARE*），以便获取它所需对象的类型

![](https://www.runoob.com/wp-content/uploads/2014/08/AB6B814A-0B09-4863-93D6-1E22D6B07FF8.jpg)

### 2.简单工厂

简单工厂模式的工厂类一般是使用静态方法，通过接收的参数的不同来返回不同的对象实例。**不修改简单工厂类代码的话，是无法扩展的**。类图如下：

![](https://p-blog.csdn.net/images/p_blog_csdn_net/superbeck/EntryImages/20090814/SimpleFactory.jpg)

首先创建一个共同接口让子类实现：

```java
/**
 * @author fjzheng
 * @version 1.0
 * @date 2021/12/3 16:39
 */
public interface Shape {
    void draw();
}
```

接下来创建接口的各种实现类：

```java
public class Circle implements Shape{
    @Override
    public void draw() {
        System.out.println(" I can draw circle....");
    }
}
```

```java
public class Square implements Shape{
    @Override
    public void draw() {
        System.out.println("I can draw square....");
    }
}

```

```java
public class Rectangle implements Shape{
    @Override
    public void draw() {
        System.out.println("I can draw rectangle....");
    }
}
```

以上我们创建了画圆圈、正方形、长方形等形状的实现类，下面我们要创建简单工厂类来创建对象了

```java
/**
 * @author fjzheng
 * @version 1.0
 * @date 2021/12/3 16:52
 */
public class ShapeFactory {

    public static Shape createShape(String type) {
        if (Objects.equals("circle", type)) {
            return new Circle();
        } else if (Objects.equals("square", type)) {
            return new Square();
        } else if (Objects.equals("rectangle", type)) {
            return new Rectangle();
        }
        return null;

    }
}
```

测试demo

```java
public class ShapeFactoryDemo {
    public static void main(String[] args) {
        Shape circle = ShapeFactory.createShape("circle");
        circle.draw();
    }
}
```

我们根据告诉工厂我们要画什么，工厂就能创建出对应的实现类画画了。

在上面的代码实现中，我们每次调用 `ShapeFactory` 的 `createShape` 的时候，都要创建一个新的 shape。实际上，如果 shape 可以复用，为了节省内存和对象创建的时间，我们可以将 shape 事先创建好缓存起来。当调用 createShape() 函数的时候，我们从缓存中取出 shape 对象直接使用。这有点类似单例模式和简单工厂模式的结合，具体的代码实现如下所示。在接下来的讲解中，我们把上一种实现方法叫作简单工厂模式的第一种实现方法，把下面这种实现方法叫作简单工厂模式的第二种实现方法。改造如下：

```java
/**
 * @author fjzheng
 * @version 1.0
 * @date 2021/12/3 17:09
 */
public class ShapeFactory {
    private static final Map<String, Shape> SHAPE_MAP = new HashMap<>();

    static {
        SHAPE_MAP.put("circle", new Circle());
        SHAPE_MAP.put("square", new Square());
        SHAPE_MAP.put("rectangle", new Rectangle());
    }
    
    public static Shape createShape(String type) {
        return SHAPE_MAP.get(type);
    }
}

```

对于上面两种简单工厂模式的实现方法，如果我们要添加新的 shape，那势必要改动到 ShapeFactory 的代码，比如现在我们现在需要画三角形`triangle`，这时候我们需要新增一个`triangle`实现shape接口，同时修改工厂类加上创建`triangle`的逻辑。那这是不是违反开闭原则呢？实际上，如果不是需要频繁地添加新的 shape，只是偶尔修改一下 ShapeFactory 代码，稍微不符合开闭原则，也是完全可以接受的。

除此之外，在 shapeFactory 的第一种代码实现中，有一组 if 分支判断逻辑，是不是应该用多态或其他设计模式来替代呢？实际上，如果 if 分支并不是很多，代码中有 if 分支也是完全可以接受的。应用多态或设计模式来替代 if 分支判断逻辑，也并不是没有任何缺点的，它虽然提高了代码的扩展性，更加符合开闭原则，但也增加了类的个数，牺牲了代码的可读性。关于这一点，我们在后面章节中会详细讲到。总结一下，尽管简单工厂模式的代码实现中，有多处 if 分支判断逻辑，违背开闭原则，但权衡扩展性和可读性，这样的代码实现在大多数情况下（比如，不需要频繁地添加 shape，也没有太多的 shape）是没有问题的。

如果我们非得要将 if 分支逻辑去掉，那该怎么办呢？比较经典处理方法就是利用多态，也就是接下来要讲的工厂方法模式

### 3.工厂方法

工厂方法是针对每一种产品提供一个工厂类。通过不同的工厂实例来创建不同的产品实例。在同一等级结构中，支持增加任意产品。

![](https://p-blog.csdn.net/images/p_blog_csdn_net/superbeck/EntryImages/20090814/FactoryMethod.jpg)

首先创建一个工厂接口，有一个`createShape()`方法供每个实现类实现，每一个工厂实现类创建想要的具体对象。

```java
/**
 * @author fjzheng
 * @version 1.0
 * @date 2021/12/3 17:26
 */
public interface IShapeFactory {
    Shape createShape();
}

```

接下来创建工厂的实现类，用于创建具体的对象：

```java
public class CircleShapeFactory implements IShapeFactory{
    @Override
    public Shape createShape() {
        return new Circle();
    }
}


public class SquareShapeFactory implements IShapeFactory{
    @Override
    public Shape createShape() {
        return new Square();
    }
}

public class RectangleShapeFactory implements IShapeFactory{
    @Override
    public Shape createShape() {
        return new Rectangle();
    }
}
```

实际上，这就是工厂方法模式的典型代码实现。这样当我们新增一种 shape 的时候，只需要新增一个实现了 IShapeFactory 接口的 Factory 类即可。所以，**工厂方法模式比起简单工厂模式更加符合开闭原则**，这时候我们就可以根据不同的工厂接口类直接生产我们想要的具体对象了。

**那什么时候该用工厂方法模式，而非简单工厂模式呢？**

之所以将某个代码块剥离出来，独立为函数或者类，原因是这个代码块的逻辑过于复杂，剥离之后能让代码更加清晰，更加可读、可维护。但是，如果代码块本身并不复杂，就几行代码而已，我们完全没必要将它拆分成单独的函数或者类。基于这个设计思想，当对象的创建逻辑比较复杂，不只是简单的 new 一下就可以，而是要组合其他类对象，做各种初始化操作的时候，我们推荐使用工厂方法模式，将复杂的创建逻辑拆分到多个工厂类中，让每个工厂类都不至于过于复杂。而使用简单工厂模式，将所有的创建逻辑都放到一个工厂类中，会导致这个工厂类变得很复杂。除此之外，在某些场景下，如果对象不可复用，那工厂类每次都要返回不同的对象。如果我们使用简单工厂模式来实现，就只能选择第一种包含 if 分支逻辑的实现方式。如果我们还想避免烦人的 if-else 分支逻辑，这个时候，我们就推荐使用工厂方法模式。

### 4.抽象工厂

在简单工厂和工厂方法中，类只有一种分类方式，如上面只能生产shape形状，如果这时候要同时生产不同颜色的颜料，上面模式就比较难以搞定了。抽象工厂就是针对这种非常特殊的场景而诞生的。我们可以让一个工厂负责创建多个不同类型的对象（shape、pigment 等），而不是只创建一种 shape 对象。抽象工厂在实际场景不怎么使用，所以这里简单介绍一下就行了。其对应的接口工厂类如下：

```java
public interface ShapeFactory {
  Shape createShape();
  Pigment createPigment();
}
```

### 5.工厂模式在spring中的应用

在 Spring 中，工厂模式最经典的应用莫过于实现 IOC 容器，对应的 Spring 源码主要是 BeanFactory 类和 ApplicationContext 相关类（AbstractApplicationContext、ClassPathXmlApplicationContext、FileSystemXmlApplicationContext…）

BeanFactory的类图如下：

![](https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/spring_factory)

ApplicationContext的类图：

![](https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/spring_applicationcontext)

从上面可以看出，ApplicationContext 是BeanFactory 的子接口之一，换句话说BeanFactory 是Spring IOC 容器所定义的最底层接口，而ApplicationContext 是其高级接口之一，并且对BeanFactory 功能做了许多有用的扩展，所以在绝大部分的工作场景下， 都会使用ApplicationContext 作为Spring IoC 容器。

```java
* ApplicationContext的三个常用实现类：
*      ClassPathXmlApplicationContext：它可以加载类路径下的配置文件，要求配置文件必须在类路径下。不在的话，加载不了。(更常用)
*      FileSystemXmlApplicationContext：它可以加载磁盘任意路径下的配置文件(必须有访问权限）
*
*      AnnotationConfigApplicationContext：它是用于读取注解创建容器的，是明天的内容。
*
* 核心容器的两个接口引发出的问题：
*  ApplicationContext:     单例对象适用              采用此接口
*      它在构建核心容器时，创建对象采取的策略是采用立即加载的方式。也就是说，只要一读取完配置文件马上就创建配置文件中配置的对象。
*
*  BeanFactory:            多例对象使用
*      它在构建核心容器时，创建对象采取的策略是采用延迟加载的方式。也就是说，什么时候根据id获取对象了，什么时候才真正的创建对象。
```

无论是BeanFctory还是appliationContext，都是采用了工厂模式实现对象bean的配置解析、对象创建和生命周期的管理。spring ioc的依赖注入DI就是一个根据描述创建对象bean的''大工程''，这里的创建对象不是简单new，需要进行复杂的操作和流程处理，所以用工厂模式再合适不过了

