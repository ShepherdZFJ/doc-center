# 浅析动态代理原理与实现

##### 1.代理模式

 代理是一种软件设计模式，目的地希望能做到代码重用。具体上讲，代理这种设计模式是通过不直接访问被代理对象的方式，而访问被代理对象的方法。这个就好比 商户---->明星经纪人(代理)---->明星这种模式。我们可以不通过直接与明星对话的情况下，而通过明星经纪人(代理)与其产生间接对话 

> **项目推荐**：基于SpringBoot2.x、SpringCloud和SpringCloudAlibaba企业级系统架构底层框架封装，解决业务开发时常见的非功能性需求，防止重复造轮子，方便业务快速开发和企业技术栈框架统一管理。引入组件化的思想实现高内聚低耦合并且高度可配置化，做到可插拔。严格控制包依赖和统一版本管理，做到最少化依赖。注重代码规范和注释，非常适合个人学习和企业使用
>
> **Github地址**：https://github.com/plasticene/plasticene-boot-starter-parent
>
> **Gitee地址**：https://gitee.com/plasticene3/plasticene-boot-starter-parent

##### 2.代理模式的使用场景

- 设计模式中有一个设计原则是**开闭原则**，是说对修改关闭对扩展开放，我们在工作中有时会接手很多前人的代码，里面代码逻辑让人摸不着头脑，这时就很难去下手修改代码，那么这时我们就可以通过代理对类进行增强。

- 我们在使用RPC框架的时候，框架本身并不能提前知道各个业务方要调用哪些接口的哪些方法 。那么这个时候，就可用通过动态代理的方式来建立一个中间人给客户端使用，也方便框架进行搭建逻辑，某种程度上也是客户端代码和框架松耦合的一种表现。

- **Spring的AOP机制就是采用动态代理的机制来实现切面编程**。

##### 3.代理模式类别

 我们根据加载被代理类的时机不同，将代理分为**静态代理和动态代理**。如果我们在代码编译时就确定了被代理的类是哪一个，那么就可以直接使用**静态代理**；如果不能确定，那么可以使用类的动态加载机制，在代码运行期间加载被代理的类这就是**动态代理**，比如RPC框架和Spring AOP机制。

代理模式的UML类图：

 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190208215538825.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3JpZW1hbm5f,size_16,color_FFFFFF,t_70) 

图解如下：

- 用户只关心接口功能，而不在乎谁提供了功能。上图中接口是 Subject。
- 接口真正实现者是上图的 RealSubject，但是它不与用户直接接触，而是通过代理。
- 代理就是上图中的 Proxy，由于它实现了 Subject 接口，所以它能够直接与用户接触。
- 用户调用 Proxy 的时候，Proxy 内部调用了 RealSubject。所以，Proxy 是中介者，它可以增强 RealSubject 操作。

##### 4.静态代理

 **静态代理**：由程序员创建或特定工具自动生成源代码，也就是在编译时就已经将接口，被代理类，代理类等确定下来。在程序运行之前，代理类的.class文件就已经生成。 

根据上面代理模式的类图，来写一个简单的静态代理的例子。我这儿举一个比较粗糙的例子，假如一个班的同学要向老师交班费，但是都是通过班长把自己的钱转交给老师。这里，班长就是代理学生上交班费，班长就是学生的代理。

首先，我们创建一个studentService接口。这个接口就是学生（被代理类），和班长（代理类）的公共接口，他们都有上交班费的行为。这样，学生上交班费就可以让班长来代理执行

```java
/**
 * @author fjZheng
 * @version 1.0
 * @date 2021/1/26 18:00
 */
public interface studentService {
    //上交班费
    void giveMoney();
}
```

 Student类实现studentService接口。Student可以具体实施上交班费的动作。 

```java
/**
 * @author fjZheng
 * @version 1.0
 * @date 2021/1/26 18:00
 */
public class Student implements studentService {
    private String name;
    public Student(String name) {
        this.name = name;
    }
    
    @Override
    public void giveMoney() {
       System.out.println(name + "上交班费50元");
    }
}
```

  Monitor 类，这个类也实现了studentService接口，但是还另外持有一个学生类对象，由于实现了studentService接口，同时持有一个学生对象，那么他可以代理学生类对象执行上交班费（执行giveMoney()方法）行为。

```JAVA
/**
 * 学生代理类(班长），也实现了studentService接口，保存一个学生实体，这样既可以代理学生产生行为
 * @author fjzheng
 *
 */
public class Monitor implements studentService{
    //被代理的学生
    Student stu;
    
    public Monitor(studentService stu) {
        // 只代理学生对象
        if(stu.getClass() == Student.class) {
            this.stu = (Student)stu;
        }
    }
    
    //代理上交班费，调用被代理学生的上交班费行为
    public void giveMoney() {
        stu.giveMoney();
    }
}
```

 测试如下：

```java
public class StaticProxyTest {
    public static void main(String[] args) {
        //被代理的学生张三，他的班费上交有代理对象monitor（班长）完成
        studentSerivce zhangsan = new Student("张三");
        
        //生成代理对象(班长)，并将张三传给代理对象
        studentSerivce monitor = new Monitor(zhangsan);
        
        //班长代理上交班费
        monitor.giveMoney();
    }
}
```

##### 5.动态代理

 代理类在程序运行时创建的代理方式被成为动态代理。 我们上面静态代理的例子中，代理类(Montior)是自己定义好的，在程序运行之前就已经编译完成。然而动态代理，代理类并不是在Java代码中定义的，而是在运行时根据我们在Java代码中的“指示”动态生成的。相比于静态代理， 动态代理的优势在于可以很方便的对代理类的函数进行统一的处理，而不用修改每个代理类中的方法。

 spring AOP的拦截功能是由java中的动态代理来实现的。说白了，就是在目标类的基础上增加切面逻辑，生成增强的目标类（该切面逻辑或者在目标类函数执行之前，或者目标类函数执行之后，或者在目标类函数抛出异常时候执行。不同的切入时机对应不同的Interceptor的种类，如BeforeAdviseInterceptor，AfterAdviseInterceptor以及ThrowsAdviseInterceptor等）。

那么动态代理是如何实现将切面逻辑（advise）织入到目标类方法中去的呢？下面我们就来详细介绍并实现AOP中用到的两种动态代理。AOP的源码中用到了两种动态代理来实现拦截切入功能：**jdk动态代理和cglib动态代理**。两种方法同时存在，各有优劣。

jdk动态代理是由java内部的反射机制来实现的，cglib动态代理底层则是借助asm来实现的。总的来说，反射机制在生成类的过程中比较高效，而asm在生成类之后的相关执行过程中比较高效（可以通过将asm生成的类进行缓存，这样解决asm生成类过程低效问题）。还有一点必须注意：jdk动态代理的应用前提，必须是目标类基于统一的接口。如果没有上述前提，jdk动态代理不能应用。由此可以看出，jdk动态代理有一定的局限性，cglib这种第三方类库实现的动态代理应用更加广泛，且在效率上更有优势。 

- **5.1 JDK实现动态代理**

   jdk动态代理是jdk原生就支持的一种代理方式，它的实现原理，就是通过让target类和代理类实现同一接口，代理类持有target对象，来达到方法拦截的作用，这样通过接口的方式有两个弊端，一个是必须保证target类有接口，第二个是如果想要对target类的方法进行代理拦截，那么就要保证这些方法都要在接口中声明，实现上略微有点限制 

  示例代码如下：

  ```java
  package com.shepherd.proxy.jdk;
  
  import java.lang.reflect.InvocationHandler;
  import java.lang.reflect.Method;
  import java.lang.reflect.Proxy;
  
  /**
   * @author fjZheng
   * @version 1.0
   * @date 2021/1/26 17:41
   */
  public class JDKProxyMain {
    
      public static void main(String[] args) {
            /**
           * 要求：被代理类至少实现一个接口
           * newProxyInstance(ClassLoader loader, Class<?>[] interfaces,                                             InvocationHandler h)
           * 参数含义：
           *         ClassLoader：和被代理对象使用相同的类加载器。 
           *         Interfaces：和被代理对象具有相同的行为。实现相同的接口。
           *         InvocationHandler：如何代理，使用反射执行接口原方法，同时可扩展增强方法，如下                                       面的示例JDKProxyTestInvocationHandler。
           */
          JDKProxyTestInterface target = new JDKProxyTestInterfaceImpl();
          // 根据目标对象创建代理对象
          JDKProxyTestInterface proxy = (JDKProxyTestInterface) Proxy
                          .newProxyInstance(target.getClass().getClassLoader(),
                                            target.getClass().getInterfaces(),
                                            new JDKProxyTestInvocationHandler(target));
          // 调用代理对象方法
          proxy.testProxy();
      }
  
      //接口
      interface JDKProxyTestInterface {
          void testProxy();
      }
  
      //接口实现类
      static class JDKProxyTestInterfaceImpl
              implements JDKProxyTestInterface {
          @Override
          public void testProxy() {
              System.out.println("testProxy");
          }
      }
  
      //InvocationHandler实现类，使用反射增强方法
      static class JDKProxyTestInvocationHandler implements InvocationHandler {
          private Object target;
          public JDKProxyTestInvocationHandler(Object target) {
              this.target = target;
          }
  
          @Override
          public Object invoke(Object proxy, Method method,
                               Object[] args) throws Throwable {
              System.out.println("执行前：method before");
              //执行方法
              Object result = method.invoke(this.target, args);
              System.out.println("执行后：method after");
              return result;
          }
      }
  }
  ```

- 5.2 cglib实现动态代理

  Cglib是一个优秀的动态代理框架，它的底层使用ASM在内存中动态的生成被代理类的子类，使用CGLIB即使代理类没有实现任何接口也可以实现动态代理功能。CGLIB具有简单易用，它的运行速度要远远快于JDK的Proxy动态代理。**cglib有两种可选方式：继承和引用**。**第一种是基于继承实现的动态代理，所以可以直接通过super调用target方法，但是这种方式在spring中是不支持的，因为这样的话，这个target对象就不能被spring所管理，所以cglib还是才用类似jdk的方式，通过持有target对象来达到拦截方法的效果**。
  CGLIB的核心类：
    net.sf.cglib.proxy.Enhancer – 主要的增强类
    net.sf.cglib.proxy.MethodInterceptor – 主要的方法拦截类，它是Callback接口的子接口，需要用户实现
    net.sf.cglib.proxy.MethodProxy – JDK的java.lang.reflect.Method类的代理类，可以方便的实现对源对象方法的调用,如下使用：

  Object o = methodProxy.invokeSuper(proxy, args); //虽然第一个参数是被代理对象，也不会出现死循环的问题。

  net.sf.cglib.proxy.MethodInterceptor接口是最通用的回调（callback）类型，它经常被基于代理的AOP用来实现拦截（intercept）方法的调用。这个接口只定义了一个方法：
  public Object intercept(Object object, java.lang.reflect.Method method, Object[] args, MethodProxy proxy) throws Throwable;

  第一个参数是代理对像，第二和第三个参数分别是拦截的方法和方法的参数。原来的方法可能通过使用java.lang.reflect.Method对象的一般反射调用，或者使用 net.sf.cglib.proxy.MethodProxy对象调用。net.sf.cglib.proxy.MethodProxy通常被首选使用，因为它更快。

  使用cglib需要引入依赖：

  ```xml
  <dependency>
      <groupId>cglib</groupId>
      <artifactId>cglib</artifactId>
      <version>2.2.2</version>
  </dependency>
  ```

  示例代码如下：

  ```java
  package com.shepherd.proxy.cglib;
  
  /**
   * @author fjZheng
   * @version 1.0
   * @date 2021/1/26 18:00
   */
  import net.sf.cglib.proxy.Enhancer;
  import net.sf.cglib.proxy.MethodInterceptor;
  import net.sf.cglib.proxy.MethodProxy;
  
  import java.lang.reflect.Method;
  public class CglibProxyTest {
  
      static class CglibProxyService {
          public  CglibProxyService(){
          }
          void sayHello(){
              System.out.println(" hello !");
          }
      }
  
      static class CglibProxyInterceptor implements MethodInterceptor {
          @Override
          public Object intercept(Object sub, Method method,
                                  Object[] objects, MethodProxy methodProxy)
                  throws Throwable {
              System.out.println("before hello");
              Object object = methodProxy.invokeSuper(sub, objects);
              System.out.println("after hello");
              return object;
          }
      }
  
      public static void main(String[] args) {
          // 通过CGLIB动态代理获取代理对象的过程
          Enhancer enhancer = new Enhancer();
          // 设置enhancer对象的父类
          enhancer.setSuperclass(CglibProxyService.class);
          // 设置enhancer的回调对象
          enhancer.setCallback(new CglibProxyInterceptor());
          // 创建代理对象
          CglibProxyService proxy= (CglibProxyService)enhancer.create();
          System.out.println(CglibProxyService.class);
          System.out.println(proxy.getClass());
          // 通过代理对象调用目标方法
          proxy.sayHello();
      }
  }
  ```

- 5.3 jdk实现和cglib实现的区别

  | 类型   | jdk实现                                                      | cglib实现原理                                                |
  | ------ | :----------------------------------------------------------- | ------------------------------------------------------------ |
  | 原理   | java动态代理是利用反射机制生成一个实现代理接口的匿名类，在调用具体方法之前调用invokeHandler | cglib动态代理是利用asm开源包，对代理对象类的class文件加载进来，通过修改其字节码生成子类来处理 |
  | 核心类 | Proxy创建代理利用反射机制生成一个实现代理接口的匿名类。InvokeHandler方法拦截器接口需要实现invoke方法 | net.sf.cglib.proxy.Enhancer：主要增强类，通字节码技术动态创建委托类的子类实例。net.sf.cglib.proxy.MethodInterceptor：方法拦截器接口，需要实现intercept方法 |
  | 局限性 | 只能代理实现了接口的类                                       | 不能代理final修饰的类，也不能处理final修饰的方法             |

  

