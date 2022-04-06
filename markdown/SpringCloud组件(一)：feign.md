# SpringCloud组件(一)：feign

#### 1.Feign是什么

Feign是一款Java语言编写的HttpClient绑定器，在Spring Cloud微服务中用于实现微服务之间的声明式调用。Feign 可以定义请求到其他服务的接口，用于微服务间的调用，**不用自己再写http请求（eg：使用spring自带的restTemplate或者httpClinents工具构建http请求调用第三方服务接口**，在客户端实现，调用此接口就像远程调用其他服务一样，当请求出错时可以调用接口的实现类来返回

Feign是一个声明式的web service客户端，它使得编写web service客户端更为容易。创建接口，为接口添加注解，即可使用Feign。Feign可以使用Feign注解或者JAX-RS注解，还支持热插拔的编码器和解码器。Spring Cloud为Feign添加了Spring MVC的注解支持，并整合了Ribbon和Eureka来为使用Feign时提供负载均衡。

feign源码的github地址：https://github.com/OpenFeign/feign

#### 2.feign的使用

1.引入依赖，feign作为springcloud五大组件之一，只需要简单引入下面依赖即可：

```xml
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
  </dependency>

```

2.定义一个feign接口，通过@FeignClient来实现你要调用微服务中的哪个服务，哪个接口，代码示例如下：

```java
/**
 * @author fjzheng
 * @version 1.0
 * @date 2021/7/5 15:04
 */

@FeignClient(name = "${micro-server.mall-product}", path = "/api/mall/product")
public interface ProductService {

    @GetMapping("/sku/{skuId}")
    ResponseVO<ProductSku> getSku(@PathVariable("skuId") Long skuId);

    @PostMapping("/sku/price")
    ResponseVO<List<ProductSku>> getSkuPrice(@RequestBody SkuQuery query);

}
```

如上订单服务需要调商品服务的两个接口，就按照上面写，@FeignClient注解的具体属性后面详细解析

3.开启feign功能，需要在服务的启动类上添加注解EnableFeignClients，如下所示：

```java
@SpringBootApplication
@EnableDiscoveryClient
@MapperScan(basePackages = "com.shepherd.mallorder.dao")
@ComponentScan(basePackages = {"com.shepherd"})
@EnableFeignClients(basePackages = "com.shepherd.mallorder")
@EnableRabbit    //要想监听队列接收消息必须开启此注解
public class MallOrderServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(MallOrderServiceApplication.class, args);
    }

}
```

完成以上步骤，只要商品服务注册到了微服务的服务中心了，那么订单服务就可以调用商品服务的相关接口了。

#### 3.@FeignClient注解

注解@FeignClient的源码如下：

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

- FeignClient注解被@Target(ElementType.TYPE)修饰，表示FeignClient注解的作用目标在接口上@Retention(RetentionPolicy.RUNTIME)，注解会在class字节码文件中存在，在运行时可以通过反射获取到；@Documented表示该注解将被包含在javadoc中。

- feign 用于声明具有该接口的REST客户端的接口的注释应该是创建（例如用于自动连接到另一个组件。 如果功能区可用，那将是
  用于负载平衡后端请求，并且可以配置负载平衡器使用与伪装客户端相同名称（即值）@RibbonClient 。

- 其中value()和name()一样，是被调用的 service的名称。url(),直接填写硬编码的url,decode404()即404是否被解码，还是抛异常；configuration()，标明FeignClient的配置类，默认的配置类为FeignClientsConfiguration类，可以覆盖Decoder、Encoder和Contract等信息，进行自定义配置。fallback(),填写熔断器的信息类。

  从源码可以得知，name是value的别名，value也是name的别名。两者的作用是一致的，name指定FeignClient的名称，如果项目使用了Ribbon，name属性会作为微服务的名称，用于服务发现。其中，serviceId和value的作用一样，用于指定服务ID，已经废弃。

  qualifier：该属性用来指定@Qualifier注解的值，该值是该FeignClient的限定词，可以使用改值进行引用

  url：该属性一般用于调试程序，允许我们手动指定@FeignClient调用的地址

  decode404：当发生http 404错误时，如果该字段位true，会调用decoder进行解码，否则抛出FeignException

  configuration：Feign配置类，可以自定义Feign的Encoder、Decoder、LogLevel、Contract。

  参考：https://blog.csdn.net/weixin_38912281/article/details/104538676

从注解源码中可以得知，feign客户端默认的配置类为FeignClientsConfiguration，这个配置类注入了很多的相关配置的bean，包括feignRetryer、FeignLoggerFactory、FormattingConversionService等,其中还包括了Decoder、Encoder、Contract，如果这三个bean在没有注入的情况下，会自动注入默认的配置。

```java
@Configuration(proxyBeanMethods = false)
public class FeignClientsConfiguration {

	@Autowired
	private ObjectFactory<HttpMessageConverters> messageConverters;

	@Autowired(required = false)
	private List<AnnotatedParameterProcessor> parameterProcessors = new ArrayList<>();

	@Autowired(required = false)
	private List<FeignFormatterRegistrar> feignFormatterRegistrars = new ArrayList<>();

	@Autowired(required = false)
	private Logger logger;

	@Autowired(required = false)
	private SpringDataWebProperties springDataWebProperties;

	@Bean
	@ConditionalOnMissingBean
	public Decoder feignDecoder() {
		return new OptionalDecoder(
				new ResponseEntityDecoder(new SpringDecoder(this.messageConverters)));
	}

	@Bean
	@ConditionalOnMissingBean
	@ConditionalOnMissingClass("org.springframework.data.domain.Pageable")
	public Encoder feignEncoder() {
		return new SpringEncoder(this.messageConverters);
	}

	@Bean
	@ConditionalOnClass(name = "org.springframework.data.domain.Pageable")
	@ConditionalOnMissingBean
	public Encoder feignEncoderPageable() {
		PageableSpringEncoder encoder = new PageableSpringEncoder(
				new SpringEncoder(this.messageConverters));
		if (springDataWebProperties != null) {
			encoder.setPageParameter(
					springDataWebProperties.getPageable().getPageParameter());
			encoder.setSizeParameter(
					springDataWebProperties.getPageable().getSizeParameter());
			encoder.setSortParameter(
					springDataWebProperties.getSort().getSortParameter());
		}
		return encoder;
	}

	@Bean
	@ConditionalOnMissingBean
	public Contract feignContract(ConversionService feignConversionService) {
		return new SpringMvcContract(this.parameterProcessors, feignConversionService);
	}

	@Bean
	public FormattingConversionService feignConversionService() {
		FormattingConversionService conversionService = new DefaultFormattingConversionService();
		for (FeignFormatterRegistrar feignFormatterRegistrar : this.feignFormatterRegistrars) {
			feignFormatterRegistrar.registerFormatters(conversionService);
		}
		return conversionService;
	}

	@Bean
	@ConditionalOnMissingBean
	public Retryer feignRetryer() {
		return Retryer.NEVER_RETRY;
	}

	@Bean
	@Scope("prototype")
	@ConditionalOnMissingBean
	public Feign.Builder feignBuilder(Retryer retryer) {
		return Feign.builder().retryer(retryer);
	}

	@Bean
	@ConditionalOnMissingBean(FeignLoggerFactory.class)
	public FeignLoggerFactory feignLoggerFactory() {
		return new DefaultFeignLoggerFactory(this.logger);
	}

	@Bean
	@ConditionalOnClass(name = "org.springframework.data.domain.Page")
	public Module pageJacksonModule() {
		return new PageJacksonModule();
	}

	@Configuration(proxyBeanMethods = false)
	@ConditionalOnClass({ HystrixCommand.class, HystrixFeign.class })
	protected static class HystrixFeignConfiguration {

		@Bean
		@Scope("prototype")
		@ConditionalOnMissingBean
		@ConditionalOnProperty(name = "feign.hystrix.enabled")
		public Feign.Builder feignHystrixBuilder() {
			return HystrixFeign.builder();
		}

	}

}

```

从配置类源码中可以知道，我们自定义注入该配置类中bean，如果我们注入自定义的bean，那么此配置类的默认bean将不注入。比如FeignClientsConfiguration的默认重试次数为Retryer.NEVER_RETRY，即不重试，那么如果希望重试，那么就可以重写这个bean，即自定义，注入feignRetryer的bean,代码如下：

```java
@Configuration
public class FeignConfig {

    @Bean
    public Retryer feignRetryer() {
        return new Retryer.Default(100, SECONDS.toMillis(1), 5);
    }

}

```



#### 4.feign的工作原理

一切冲项目启动类加上注解@EnableFeignClients开始，该组件开启feign客户端扫描，同时已代表开启feign组件功能：

<img src="https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/feign-1" style="zoom:50%;" />

通过包扫描注入FeignClient的bean，该源码在FeignClientsRegistrar类：首先在启动配置上检查是否有@EnableFeignClients注解，如果有该注解，则开启包扫描，扫描被@FeignClient注解接口，描被@FeignClient注解接口。扫描出该注解后，通过beanDefinition注入到IOC容器中，方便后续被调用使用

<img src="https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/feign-2" style="zoom:50%;" />

可以发现这个FeignClientsRegistrar类实现了 ImportBeanDefinitionRegistrar，阅读过Spring源码的同学们应该很清楚，这个类是Spring提供的一个扩展点，提供给外界去动态扩展自己的需要的Bean，这里面有一个关键的方法#registerBeanDefinitions，这个方法会在Spring初始化上下文 refresh方法进行调用。

接下来是开启包扫描和注入bean的核心源码：大致的流程是初始化一个扫描器Scanner 完成包名下面扫描所有的带有FeignClent注解的类，最后调用registerFeignClient方法

1.方法registerFeignClients

```java
	public void registerFeignClients(AnnotationMetadata metadata,
			BeanDefinitionRegistry registry) {
		ClassPathScanningCandidateComponentProvider scanner = getScanner();
		scanner.setResourceLoader(this.resourceLoader);

		Set<String> basePackages;

		Map<String, Object> attrs = metadata
				.getAnnotationAttributes(EnableFeignClients.class.getName());
		AnnotationTypeFilter annotationTypeFilter = new AnnotationTypeFilter(
				FeignClient.class);
		final Class<?>[] clients = attrs == null ? null
				: (Class<?>[]) attrs.get("clients");
		if (clients == null || clients.length == 0) {
			scanner.addIncludeFilter(annotationTypeFilter);
			basePackages = getBasePackages(metadata);
		}
		else {
			final Set<String> clientClasses = new HashSet<>();
			basePackages = new HashSet<>();
			for (Class<?> clazz : clients) {
				basePackages.add(ClassUtils.getPackageName(clazz));
				clientClasses.add(clazz.getCanonicalName());
			}
			AbstractClassTestingTypeFilter filter = new AbstractClassTestingTypeFilter() {
				@Override
				protected boolean match(ClassMetadata metadata) {
					String cleaned = metadata.getClassName().replaceAll("\\$", ".");
					return clientClasses.contains(cleaned);
				}
			};
			scanner.addIncludeFilter(
					new AllTypeFilter(Arrays.asList(filter, annotationTypeFilter)));
		}

		for (String basePackage : basePackages) {
			Set<BeanDefinition> candidateComponents = scanner
					.findCandidateComponents(basePackage);
			for (BeanDefinition candidateComponent : candidateComponents) {
				if (candidateComponent instanceof AnnotatedBeanDefinition) {
					// verify annotated class is an interface
					AnnotatedBeanDefinition beanDefinition = (AnnotatedBeanDefinition) candidateComponent;
					AnnotationMetadata annotationMetadata = beanDefinition.getMetadata();
					Assert.isTrue(annotationMetadata.isInterface(),
							"@FeignClient can only be specified on an interface");

					Map<String, Object> attributes = annotationMetadata
							.getAnnotationAttributes(
									FeignClient.class.getCanonicalName());

					String name = getClientName(attributes);
					registerClientConfiguration(registry, name,
							attributes.get("configuration"));

					registerFeignClient(registry, annotationMetadata, attributes);
				}
			}
		}
	}
```

2.方法：registerFeignClient

```java
	private void registerFeignClient(BeanDefinitionRegistry registry,
			AnnotationMetadata annotationMetadata, Map<String, Object> attributes) {
		String className = annotationMetadata.getClassName();
		BeanDefinitionBuilder definition = BeanDefinitionBuilder
				.genericBeanDefinition(FeignClientFactoryBean.class);
		validate(attributes);
		definition.addPropertyValue("url", getUrl(attributes));
		definition.addPropertyValue("path", getPath(attributes));
		String name = getName(attributes);
		definition.addPropertyValue("name", name);
		String contextId = getContextId(attributes);
		definition.addPropertyValue("contextId", contextId);
		definition.addPropertyValue("type", className);
		definition.addPropertyValue("decode404", attributes.get("decode404"));
		definition.addPropertyValue("fallback", attributes.get("fallback"));
		definition.addPropertyValue("fallbackFactory", attributes.get("fallbackFactory"));
		definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);

		String alias = contextId + "FeignClient";
		AbstractBeanDefinition beanDefinition = definition.getBeanDefinition();

		boolean primary = (Boolean) attributes.get("primary"); // has a default, won't be
																// null

		beanDefinition.setPrimary(primary);

		String qualifier = getQualifier(attributes);
		if (StringUtils.hasText(qualifier)) {
			alias = qualifier;
		}

		BeanDefinitionHolder holder = new BeanDefinitionHolder(beanDefinition, className,
				new String[] { alias });
		BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry);
	}
```

启动后通过包扫描，当类有@FeignClient注解，将注解的信息取出，连同类名一起取出，赋给BeanDefinitionBuilder，然后根据BeanDefinitionBuilder得到beanDefinition，最后beanDefinition式注入到ioc容器中。在registerFeignClient方法中首先封装了一个FeignClientFactoryBean的，然后调用BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry)将FeignClientFactoryBean注册到Spring IOC容器里面，这里需要注意的是 FeignClientFactoryBean实现了FactoryBean，所以在程序真正调用这个FeignClient注解对应的类时候，实际上是会调用FeignClientFactoryBean里面的 getObject方法返回的对象。

```java
	@Override
	public Object getObject() throws Exception {
		return getTarget();
	}

	/**
	 * @param <T> the target type of the Feign client
	 * @return a {@link Feign} client created with the specified data and the context
	 * information
	 */
	<T> T getTarget() {
		FeignContext context = this.applicationContext.getBean(FeignContext.class);
		Feign.Builder builder = feign(context);

		if (!StringUtils.hasText(this.url)) {
			if (!this.name.startsWith("http")) {
				this.url = "http://" + this.name;
			}
			else {
				this.url = this.name;
			}
			this.url += cleanPath();
			return (T) loadBalance(builder, context,
					new HardCodedTarget<>(this.type, this.name, this.url));
		}
		if (StringUtils.hasText(this.url) && !this.url.startsWith("http")) {
			this.url = "http://" + this.url;
		}
		String url = this.url + cleanPath();
		Client client = getOptional(context, Client.class);
		if (client != null) {
			if (client instanceof LoadBalancerFeignClient) {
				// not load balancing because we have a url,
				// but ribbon is on the classpath, so unwrap
				client = ((LoadBalancerFeignClient) client).getDelegate();
			}
			if (client instanceof FeignBlockingLoadBalancerClient) {
				// not load balancing because we have a url,
				// but Spring Cloud LoadBalancer is on the classpath, so unwrap
				client = ((FeignBlockingLoadBalancerClient) client).getDelegate();
			}
			builder.client(client);
		}
		Targeter targeter = get(context, Targeter.class);
		return (T) targeter.target(this, builder, context,
				new HardCodedTarget<>(this.type, this.name, url));
	}
```

至此，feign客户端接口的代理类已经生成，可供后续@Autowired 或者@Resource自动装配使用，使用时的对象示例如下：

<img src="https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/feign-3" style="zoom:50%;" />

这里假如引入了hystrix熔断机制，feign会通过代理模式， 自动将所有的方法用 hystrix 进行包装。其代理类为：HystrixInvocationHandler。开启配置如下：

```yaml
feign:

  hystrix:
    enabled: true
  #  httpclient:
  #    connection-timeout: 10000
  client:
    config:
      default: # 指定feignclients对应的名称 如果指定的是default 表示全局所有的client 的超时时间设置
        connectTimeout: 10000
        readTimeout: 10000
```

当请求Feign Client的方法时会被拦截，如果没有开启hystrix，那么其代理类为：ReflectiveFeign

```java

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      if ("equals".equals(method.getName())) {
        try {
          Object otherHandler =
              args.length > 0 && args[0] != null ? Proxy.getInvocationHandler(args[0]) : null;
          return equals(otherHandler);
        } catch (IllegalArgumentException e) {
          return false;
        }
      } else if ("hashCode".equals(method.getName())) {
        return hashCode();
      } else if ("toString".equals(method.getName())) {
        return toString();
      }

      return dispatch.get(method).invoke(args);
    }
```

在SynchronousMethodHandler类进行拦截处理，当被FeignClient的方法被拦截会根据参数生成RequestTemplate对象，该对象就是http请求的模板，代码如下：

```java

  @Override
  public Object invoke(Object[] argv) throws Throwable {
    RequestTemplate template = buildTemplateFromArgs.create(argv);
    Options options = findOptions(argv);
    Retryer retryer = this.retryer.clone();
    while (true) {
      try {
        return executeAndDecode(template, options);
      } catch (RetryableException e) {
        try {
          retryer.continueOrPropagate(e);
        } catch (RetryableException th) {
          Throwable cause = th.getCause();
          if (propagationPolicy == UNWRAP && cause != null) {
            throw cause;
          } else {
            throw th;
          }
        }
        if (logLevel != Logger.Level.NONE) {
          logger.logRetry(metadata.configKey(), logLevel);
        }
        continue;
      }
    }
  }
```



#### 5.微服务直接feign调用相关问题

1.**Feign远程调用丢失请求头问题**：即当我们登录之后，登录信息是放在请求头里面，但是当我们从订单服务使用feign调用用户服务获取收货地址啥的这时候登录信息丢失了，因为此时请求头为空，最根本的原因是使用feign调用第三方服务时，feign组件是根据相关信息生成一个**全新请求**去调用第三方服务接口，自然没有之前携带登录信息的请求头了，解决方案如下：实现RequestInterceptor拦截器接口

```java
@Component
@Slf4j
public class FeignInterceptor implements RequestInterceptor {
    @Override
    public void apply(RequestTemplate requestTemplate) {
       //spring的上下文对象
        ServletRequestAttributes requestAttributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        if (requestAttributes != null) {
            HttpServletRequest request = requestAttributes.getRequest();
            Enumeration<String> headerNames = request.getHeaderNames();
            if (headerNames != null) {
                while (headerNames.hasMoreElements()) {
                    String name = headerNames.nextElement();
                    // 跳过 content-length
                    if (Objects.equals("content-length", name)){
                        continue;
                    }
                    String value = request.getHeader(name);
                    requestTemplate.header(name, value);
                }
            }
        }

    }
}

```



2.**Feign异步情况丢失上下文问题**，根本原因是上下文对象的属性是使用threallocal存储的

<img src="https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/feign-4" style="zoom:50%;" />

解决办法：1）使用feign不使用多线程

​                   2）开启新线程之前先获取上下文信息，开启线程之后把上下文信息放入新的新线程中：

```java
        RequestAttributes requestAttributes = RequestContextHolder.getRequestAttributes();
        CompletableFuture<Void> itemAndStockFuture = CompletableFuture.supplyAsync(() -> {
            RequestContextHolder.setRequestAttributes(requestAttributes);
            //1. 查出所有选中购物项
            List<OrderItemVo> checkedItems = cartFeignService.getCheckedItems();
            confirmVo.setItems(checkedItems);
            return checkedItems;
        }, executor).thenAcceptAsync((items) -> {
            //4. 库存
            List<Long> skuIds = items.stream().map(OrderItemVo::getSkuId).collect(Collectors.toList());
            Map<Long, Boolean> hasStockMap = wareFeignService.getSkuHasStocks(skuIds).stream().collect(Collectors.toMap(SkuHasStockVo::getSkuId, SkuHasStockVo::getHasStock));
            confirmVo.setStocks(hasStockMap);
        }, executor);
```



3.在feign接口中封装调用get接口方法时，不能用一个对象当做参数，否则在feign客户端组装请求时最后变成post请求，这时候即时多个参数我们也应该用@RequestParam注解去一一映射对象里面的属性参数。