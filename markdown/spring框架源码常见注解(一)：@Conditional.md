# spring框架源码常见注解(一)：@Conditional

#### 1.@conditional

@Conditional：该注解是在spring4中新加的，其作用顾名思义就是按照一定的条件进行判断，满足条件才将bean注入到容器中，注解源码如下：

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Conditional {

	/**
	 * All {@link Condition} classes that must {@linkplain Condition#matches match}
	 * in order for the component to be registered.
	 */
	Class<? extends Condition>[] value();

}
```

从代码中可知，该注解可左右用类，方法上，同时只有一个属性value，是一个Class数组，并且需要继承或者实现Condition接口：

```java
@FunctionalInterface
public interface Condition {
    boolean matches(ConditionContext var1, AnnotatedTypeMetadata var2);
}
```

**@FunctionalInterface**：表示该接口是一个函数式接口，既可以使用函数式编程，lambda表达式。

Condition是个接口，需要实现matches方法，**返回true则注入bean，false则不注入**。

总结：@Conditional注解通过传入一个或者多个实现了的condition接口的实现类，重写condition接口的matches方法，其条件逻辑在该方法之后中，作用于创建bean的地方。

示例代码请参考：https://blog.csdn.net/xcy1193068639/article/details/81491071

#### 2.@ConditionalOnBean  

@ConditionalOnBean ：当给定的在bean存在时,则实例化当前Bean，示例如下

```java
    @Bean
    @ConditionalOnBean(name = "address")
    public User (Address address) {
        //这里如果address实体没有成功注入 这里就会报空指针
        address.setCity("hangzhou");
        address.setId(1l)
        return new User("魅影", city);
    }
```

这里加了ConditionalOnBean注解，表示只有address这个bean存在才会实例化user

#### 3.@ConditionalOnMissingBean

@ConditionalOnMissingBean:当给定的在bean不存在时,则实例化当前Bean, 与@ConditionalOnBean相反

```java
@Configuration
public class BeanConfig {
 
    @Bean(name = "notebookPC")
    public Computer computer1(){
        return new Computer("笔记本电脑");
    }
 
    @ConditionalOnMissingBean(Computer.class)
    @Bean("reservePC")
    public Computer computer2(){
        return new Computer("备用电脑");
    }
```

ConditionalOnMissingBean无参的情况，通过源码可知，当这个注解没有参数时，仅当他注解到方法，且方法上也有@Bean，才有意义，否则无意义。那意义在于已被注解方法的返回值类型的名字作为ConditionalOnMissingBean的type属性的值。

注解实现原理请参考：https://blog.csdn.net/xcy1193068639/article/details/81517456

#### 4.@ConditionalOnClass

@ConditionalOnClass：当给定的类名在类路径上存在，则实例化当前Bean

#### 5.@ConditionalOnMissingClass

@ConditionalOnMissingClass：当给定的类名在类路径上不存在，则实例化当前Bean

#### 6.@ConditionalOnProperty

@ConditionalOnProperty：Spring Boot通过@ConditionalOnProperty来控制Configuration是否生效

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ ElementType.TYPE, ElementType.METHOD })
@Documented
@Conditional(OnPropertyCondition.class)
public @interface ConditionalOnProperty {

    // 数组，获取对应property名称的值，与name不可同时使用
    String[] value() default {};

    // 配置属性名称的前缀，比如spring.http.encoding
    String prefix() default "";

    // 数组，配置属性完整名称或部分名称
    // 可与prefix组合使用，组成完整的配置属性名称，与value不可同时使用
    String[] name() default {};

    // 可与name组合使用，比较获取到的属性值与havingValue给定的值是否相同，相同才加载配置
    String havingValue() default "";

    // 缺少该配置属性时是否可以加载。如果为true，没有该配置属性时也会正常加载；反之则不会生效
    boolean matchIfMissing() default false;    // 是否可以松散匹配，至今不知道怎么使用的    boolean relaxedNames() default true;}
```

通过其两个属性**name**以及**havingValue**来实现的，其中**name**用来从**application.properties**中读取某个属性值。
如果该值为空，则返回false;
如果值不为空，则将该值与havingValue指定的值进行比较，如果一样则返回true;否则返回false。
如果返回值为false，则该configuration不生效；为true则生效。

当然**havingValue**也可以不设置，只要配置项的值不是false或“false”，都加载Bean

示例代码：

```yaml
feign:
  hystrix:
    enabled: true
```

fegin开启断路器hystrix：

```java
	@Bean
		@Scope("prototype")
		@ConditionalOnMissingBean
		@ConditionalOnProperty(name = "feign.hystrix.enabled")
		public Feign.Builder feignHystrixBuilder() {
			return HystrixFeign.builder();
		}

```

结论：@Conditional及其衍生注解，是为了方便程序根据当前环境或者容器情况来动态注入bean。