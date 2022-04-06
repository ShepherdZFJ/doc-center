# Java独有特性：注解(annotation)

#### 1.什么是注解

注解（Annotation），也叫元数据。一种代码级别的说明。它是JDK1.5及以后版本引入的一个特性，与类、接口、枚举是在同一个层次。它可以声明在包、类、字段、方法、局部变量、方法参数等的前面，用来对这些元素进行说明，注释。它本身并不起任何作用，可以说有它没它都不影响程序的正常运行，注解的作用在于**「注解的处理程序」**，注解处理程序通过捕获被注解标记的代码然后进行一些处理，这就是注解工作的方式。

在java中，自定义一个注解非常简单，通过`@interface`就能定义一个注解，实现如下

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface FeignClient {

	/**
	 * The name of the service with optional protocol prefix. Synonym for {@link #name()
	 * name}. A name must be specified for all clients, whether or not a url is provided.
	 * Can be specified as property key, eg: ${propertyKey}.
	 * @return the name of the service with optional protocol prefix
	 */
	@AliasFor("name")
	String value() default "";

	/**
	 * The service id with optional protocol prefix. Synonym for {@link #value() value}.
	 * @deprecated use {@link #name() name} instead
	 * @return the service id with optional protocol prefix
	 */
	@Deprecated
	String serviceId() default "";

	/**
	 * This will be used as the bean name instead of name if present, but will not be used
	 * as a service id.
	 * @return bean name instead of name if present
	 */
	String contextId() default "";

	/**
	 * @return The service id with optional protocol prefix. Synonym for {@link #value()
	 * value}.
	 */
	@AliasFor("value")
	String name() default "";

	/**
	 * @return the <code>@Qualifier</code> value for the feign client.
	 */
	String qualifier() default "";

	/**
	 * @return an absolute URL or resolvable hostname (the protocol is optional).
	 */
	String url() default "";

	/**
	 * @return whether 404s should be decoded instead of throwing FeignExceptions
	 */
	boolean decode404() default false;

	/**
	 * A custom configuration class for the feign client. Can contain override
	 * <code>@Bean</code> definition for the pieces that make up the client, for instance
	 * {@link feign.codec.Decoder}, {@link feign.codec.Encoder}, {@link feign.Contract}.
	 *
	 * @see FeignClientsConfiguration for the defaults
	 * @return list of configurations for feign client
	 */
	Class<?>[] configuration() default {};

	/**
	 * Fallback class for the specified Feign client interface. The fallback class must
	 * implement the interface annotated by this annotation and be a valid spring bean.
	 * @return fallback class for the specified Feign client interface
	 */
	Class<?> fallback() default void.class;

	/**
	 * Define a fallback factory for the specified Feign client interface. The fallback
	 * factory must produce instances of fallback classes that implement the interface
	 * annotated by {@link FeignClient}. The fallback factory must be a valid spring bean.
	 *
	 * @see feign.hystrix.FallbackFactory for details.
	 * @return fallback factory for the specified Feign client interface
	 */
	Class<?> fallbackFactory() default void.class;

	/**
	 * @return path prefix to be used by all method-level mappings. Can be used with or
	 * without <code>@RibbonClient</code>.
	 */
	String path() default "";

	/**
	 * @return whether to mark the feign proxy as a primary bean. Defaults to true.
	 */
	boolean primary() default true;

}

```

写个测试类给他加上我们写的这个注解吧

```java
@FeignClient
public class AnnotationTest {
    public static void main(String[] args) {
        System.out.println("annotation test OK!");
    }
}
```

我们发现写与不写这个注解的效果是相同的，这也印证了我们说的注解只是一种**「标记」**，有它没它并不影响程序的运行。Java注解是在JDK1.5被引入的技术，配合反射可以在运行期间处理注解。注解在大量java生态框架中用到，简化代码配置，例如spring框架中的IOC和AOP实现都有基于xml配置文件和基于注解两套实现逻辑。

**作用分类：**

①编写文档：通过代码里标识的注解生成文档【生成文档doc文档】

②代码分析：通过代码里标识的注解对代码进行分析【使用反射】

③编译检查：通过代码里标识的注解让编译器能够实现基本的编译检查【Override】

#### 2.元注解

元注解用于注解其他注解的。Java 5.0定义了4个标准的元注解，如下：

**@Target**

@Target注解用于声明注解的作用范围，例如作用范围为类、接口、方法等。它的取值以及值所对应的范围如下：

- CONSTRUCTOR:用于描述构造器
- FIELD:用于描述域
- LOCAL_VARIABLE:用于描述局部变量
- METHOD:用于描述方法
- PACKAGE:用于描述包
- PARAMETER:用于描述参数
- TYPE:用于描述类、接口(包括注解类型) 或enum声明

**@Retention**

该注解声明了注解的生命周期，即注解在什么范围内有效。

- SOURCE:在源文件中有效。注解只在源码阶段保留，在编译器进行编译的时候这类注解被抹除，常见的@Override就属于这种注解
- CLASS:在class文件中有效。注解在编译期保留，但是当Java虚拟机加载class文件时会被丢弃，这个也是@Retention的**默认值**。@Deprecated和@NonNull就属于这样的注解
- RUNTIME:在运行时有效（即运行时保留）。注解在运行期间仍然保留，在程序中可以通过反射获取，Spring中常见的@Controller、@Service等都属于这一类

**大多数注解都为RUNTIME**

**@Documented**

是一个标记注解，有该注解的注解会在生成 java 文档中保留。

**@Inherited**

该注解表明子类是有继承了父类的注解。比如一个注解被该元注解修饰，并且该注解的作用在父类上，那么子类有持有该注解。如果注解没有被该元注解修饰，则子类不持有父类的注解。

#### 3.自定义注解

在使用了自定义注解修饰了类、方法、成员变量等之后这些注解不会自己生效，需要程序员自己提供相应的注解处理工具来处理对应的注解信息Java使用了Annotation接口来代表程序元素前面的注解，该接口是所有注解的父接口，同时Java1.5在java.lang.reflect包下新增了AnnotatedElement接口用于表示程序中可以接受注解的程序元素.我们只需要通过反射获取到对象的Class 便可以通过Class获取对象上的注解信息了，而且通过Class也能获取到Method和Constructor这样也能获取到对应的注解:

| 方法名                                                       | 作用说明                                                     |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| getAnnotation(Class<A>annotationClass)                       | 获取该程序元素上指定类型的注解，如果不存在返回null           |
| getDeclaredAnnotation(Class<A>annotationClass)               | Java8新增方法，尝试获取直接修饰该程序元素指定类型的注解(不包括继承的)，如果不存在返回null |
| getAnnotations()                                             | 返回该程序元素上的所有注解                                   |
| getDeclaredAnnotations()                                     | 返回直接修饰该程序元素的所有注解(不包括继承的)               |
| isAnnotationPresent(Class<?extends Annotation>annotationClass) | 判断该程序元素上是否存在指定类型的注解                       |
| getAnnotationsByType(Class<A> annotationClass)               | 与上面的getAnnotation()方法类似，只是针对java8的重复注解功能，所以需要使用这个方法获取修饰该程序元素指定类型的多个注解 |
| getDeclaredAnnotationsByType(Class<A> annotationClass)       | 与上面的getDeclaredAnnotation方法类似，只是针对java8的重复注解功能，所以需要使用这个方法获取修饰该程序元素指定类型的多个注解 |

下面是我在项目集成quartz分布式定时任务调度框架所使用的的自定义注解：

```java
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface JobAnnotation {

    /**
     * 作业名称
     */
    String jobName() default "";

    /**
     * 分组名称
     */
    String groupName() default "";

    /**
     * 定时任务cron表达式
     */
    String cron() default "";

    /**
     * 作业回调参数
     */
    String parameter() default "";

    /**
     * 作业描述
     */
    String content() default "";

}

```

通过这个注解我们可以在代码中添加定时任务调度，其实现逻辑如下，通过反射拿到该注解信息，然后进行定时任务创建和信息绑定

```java
@Slf4j
@Component
public class JobAnnotationBeanPostProcessor implements BeanPostProcessor {

    @Autowired
    private Scheduler scheduler;

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName)  throws BeansException {
        if(!(bean instanceof Job)){
            return bean;
        }
        try{
            Method[] methods = bean.getClass().getDeclaredMethods();
            for(Method method : methods){
                JobAnnotation t3JobAnnotation = method.getAnnotation(JobAnnotation.class);
                if(null == t3JobAnnotation){
                    continue;
                }
                //定义一个TriggerKey
                TriggerKey triggerKey = TriggerKey.triggerKey(t3JobAnnotation.jobName(), t3JobAnnotation.groupName());
                CronTrigger trigger = (CronTrigger) scheduler.getTrigger(triggerKey);
                if (null == trigger) {
                    Class<? extends Job> clazz = (Class<? extends Job>)bean.getClass();
                    //执行的job任务
                    //定义一个JobDetail,其中的定义Job类，是真正的执行逻辑所在
                    JobDetail jobDetail = JobBuilder.newJob(clazz).
                            withIdentity(t3JobAnnotation.jobName(), t3JobAnnotation.groupName()).build();
                    scheduler.scheduleJob(jobDetail, createTrigger(t3JobAnnotation));
                    log.info("Quartz 创建了job:...:{}",jobDetail.getKey());
                } else {
                    if( trigger.getCronExpression().equals(t3JobAnnotation.cron())){
                        log.info("job已存在:{}",trigger.getKey());
                    } else {
                        //更新trigger 的 cron 表达式
                        scheduler.rescheduleJob(triggerKey, createTrigger(t3JobAnnotation));
                    }
                }
            }
            scheduler.start();
        }catch (Exception e){
            e.printStackTrace();
            System.out.println(e.getMessage());
        }
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object o, String s) throws BeansException {
        return o; //这里要返回o，不然启动时会报错
    }

    private Trigger createTrigger(JobAnnotation t3JobAnnotation){
        CronScheduleBuilder scheduleBuilder = CronScheduleBuilder.cronSchedule(t3JobAnnotation.cron());
        Trigger trigger = TriggerBuilder.newTrigger().withIdentity(t3JobAnnotation.jobName(), t3JobAnnotation.groupName())
                .startNow()
                .withSchedule(scheduleBuilder).build();
        return trigger;
    }
}

```



**案例实战**

基于orm框架mybatis的注解方式，下面简单实现了相同死的案例。

首先定义个一个TableName注解，它的作用范围为类，代码如下：

```java
/**
 * @author fjzheng
 * @version 1.0
 * @date 2021/8/24 11:51
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface TableName {
    String name();
}
```

定义一个TableField注解，作用范围为字段，代码如下：

```java
/**
 * @author fjzheng
 * @version 1.0
 * @date 2021/8/24 11:53
 */
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface TableField {
    String value();
}

```

定义一个Select注解，作用于方法上，代码如下：

```java
/**
 * @author fjzheng
 * @version 1.0
 * @date 2021/8/24 10:46
 */
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface Select {
    String value() default "select %s from";
}

```

定义一个Account类，代码如下：

```java
/**
 * @author fjzheng
 * @version 1.0
 * @date 2021/8/24 10:44
 */
@Data
@TableName(name = "account_info")
public class Account {
    @TableField("_id")
    private Integer id;
    private String name;
    private Date birthday;
    private String sex;
    private String address;
    @TableField("total_amount")
    private BigDecimal amount;

}

```

声明一个获取sql的方法，加上@Select注解：

```java
/**
 * @author fjzheng
 * @version 1.0
 * @date 2021/8/24 11:56
 */
public class AccountMapper {
    
    @Select
    public void getSql(Account account) {

    }
}

```

测试类逻辑如下，它是通过反射来获取表名、字段名，加上判断实体对象的字段值来生成 查询的 sql 语句的。代码如下：

```java
/**
 * @author fjzheng
 * @version 1.0
 * @date 2021/8/24 11:58
 */
public class TestAnnotationDemo {
    public static void main(String[] args) {
        AccountMapper accountMapper = new AccountMapper();
        Class c = accountMapper.getClass();
        StringBuilder sql = new StringBuilder();
        StringBuilder column = new StringBuilder();
        Method[] declaredMethods = c.getDeclaredMethods();
        for(Method method : declaredMethods) {
            Select select = method.getAnnotation(Select.class);
            if (select != null) {
                sql.append(select.value());
                Class<?>[] parameterTypes = method.getParameterTypes();
                for(Class pt : parameterTypes) {
                    if (pt == Account.class) {
                        TableName table = (TableName) pt.getAnnotation(TableName.class);
                        sql.append(" ").append(table.name());
                        Field[] fields = pt.getDeclaredFields();
                        for (Field field : fields) {
                            column.append(" ");
                            TableField tableField = field.getAnnotation(TableField.class);
                            if (tableField == null) {
                                column.append(field.getName());
                            } else {
                                column.append(tableField.value());
                            }
                            column.append(",");
                        }
                    }
                }

            }
        }
        String s = String.format(sql.toString(), column.toString().substring(1, column.toString().length() - 1));
        System.out.println(s);
    }
}

```

运行程序，控制台打印如下：

> select _id, name, birthday, sex, address, total_amount from account_info

