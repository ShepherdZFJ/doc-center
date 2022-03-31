# mybatis

#### 1.mybatis概述

mybatis是一个优秀的基于java的持久层框架，它内部封装了jdbc，使开发者只需要关注sql语句本身，而不需要花费精力去处理加载驱动、创建连接、创建statement等繁杂的过程。 mybatis通过xml或注解的方式将要执行的各种statement配置起来，并通过java对象和statement中sql的动态参数进行映射生成最终执行的sql语句，最后由mybatis框架执行sql并将结果映射为java对象并返回。 采用ORM思想解决了实体和数据库映射的问题，对jdbc进行了封装，屏蔽了jdbc api底层访问细节，使我们不用与jdbc api打交道，就可以完成对数据库的持久化操作。

#### 2.Hibernate 和MyBatis 的区别

Hibernate 和My Batis 的增、删、查、改，对于业务逻辑层来说大同小异，对于映射层而言Hibernate 的配置不需要接口和SQL ，相反MyBatis 是需要的。对于Hibernate 而言，不需要编写大量的SQL ，就可以完全映射，同时提供了日志、缓存、级联（级联比MyBatis强大）等特性， 此外还提供HQL (Hibernate Query Language ）对POJO 进行操作，使用十分方便，但是它也有致命的缺陷。由于无须SQL ，当多表关联超过3 个的时候，通过Hibernate 的级联会造成太多性能的丢失，又或者我现在访问一个财务的表，然后它会关联财产信息表，财产又分为机械、原料等，显然机械和原料的字段是不一样的，这样关联宇段只能根据特定的条件变化而变化，而H ibernate 无法支持这样的变化。遇到存储过程， Hibernate 只能作罢。更为关键的是性能，在管理系统的时代，对于性能的要求不是那么苛刻，但是在互联网时代性能就是系统的根本，响应过慢就会丧失客户，试想一下谁会去用一个经常需要等待超过10 秒以上的应用呢？

以上的问题MyBatis 都可以解决， MyBatis 可以自由书写SQL 、支持动态SQL 、处理列表、动态生成表名、支持存储过程。这样就可以灵活地定义查询语句，满足各类需求和性能优化的需要，这些在互联网系统中是十分重要的。但MyBatis 也有缺陷。首先，它要编写SQL 和映射规则，其工作量稍微大于Hibernate 。其次，它支持的工具也很有限，不能像Hibernate 那样有许多的插件可以帮助生成映射代码和关联关系，而即使使用生成工具，往往也需要开发者进一步简化， MyBatis 通过手工编码，工作量相对大些。所以对于性能要求不太苛刻的系统，比如管理系统、ERP 等推荐使用Hibernate ；而对于性能要求高、响应快、灵活的系统则推荐使用MyBatis 。

#### 3.mybatis相关案例

mybatis入门crud案例mybatis_example：https://github.com/ShepherdZFJ/mybatis_code_learn

自己手写实现一个mybatis案例mybatis_design:https://github.com/ShepherdZFJ/mybatis_code_learn

#### 4.mybatis核心组件

- SqlSessionFactoryBuilder (构造器)：它会根据配置或者代码来生成SqISessionFactory ，采用的是分步构建的Builder 模式。
- SqlSessionFactory (工厂接口)：依靠它来生成SqlSession ，使用的是工厂模式。
- SqlSession (会话)： 一个既可以发送SQL 执行返回结果，也可以获取Mapper 的接口。在现有的技术中， 一般我们会让其在业务逻辑代码中“消失”，而使用的是MyBatis 提供的SQLMapper 接口编程技术，它能提高代码的可读性和可维护性
-  SQL Mapper(映射器)： MyBatis 新设计存在的组件，它由一个Java 接口和XML文件(或注解)构成，需要给出对应的SQL 和映射规则。它负责发送SQL 去执行，并返回结果。

使用案例如下图所示：

![](https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/%E5%85%A5%E9%97%A8%E6%A1%88%E4%BE%8B%E7%9A%84%E5%88%86%E6%9E%90.png)

组件功能调用流程图如下图所示：

![](https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/04mybatis%E7%9A%84%E5%88%86%E6%9E%90.png)

##### 4.1 SqlSessionFactoryBuilder (构造器)

##### 4.2 SqlSessionFactory(工厂接口)

使用MyBatis 首先是使用配置或者代码去生产SqlSessionFactory ，而MyBatis 提供了构造器SqlSessionFactoryBuiIder 。 它提供了一个类org.apache.ibatis.session.Configuration 作为引导，采用的是Builder 模式。具体的分步则是在Configuration 类里面完成的。

当配置了XML 或者提供代码后， MyBatis 会读取配置文件，通过Configuration 类对象构建整个MyBatis 的上下文。注意， SqlSessionFactory是一个接口，在MyBatis 中它存在两个实现类： SqlSessionManager 和DefaultSqlSessionFactory 。一般而言，具体是由DefaultSqlSessionFactory 去实现的，而SqlSessionManager 使用在多线程的环境中，它的具体实现依靠DefaultSqlSessionFactory，它们之间的关系图如下所示：

![](https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/sqlSessionFactory.png)

每个基于MyBatis 的应用都是以一个SqlSessionFactory 的实例为中心的， 而SqlSessionFactory 唯一的作用就是生产MyBatis 的核心接口对象SqlSession ，所以它的责任是唯一的。我们往往会采用单例模式处理它。

##### 4.3 SqlSession

在MyBatis 中， SqlSession 是其核心接口。在MyBatis 中有两个实现类， DefaultSqlSession和SqlSessionManager 。DefaultSqlSession 是单线程使用的，而SqlSessionManager 在多线程环境下使用。SqlSession 的作用类似于一个JDBC 中的Connection 对象，代表着一个连接资源的启用具体而言，它的作用有3个：

- 获取Mapper 接口。
- 发送SQL 给数据库。
- 控制数据库事务。

##### 4.4 映射器

映射器是MyBatis 中最重要、最复杂的组件，它由一个接口和对应的XML 文件(或注解)组成。它可以配置以下内容：

- 描述映射规则。
-  提供SQL 语旬， 并可以配置SQL 参数类型、返回类型、缓存刷新等信息。
-  配置缓存。
-  提供动态SQL 。

##### 4.5 生命周期

SqlSessionFactoryBuilder 的作用在于创建SqlSessionFactory ，创建成功后，SqlSessionFactoryBuilder 就失去了作用，所以它只能存在于创建SqlSessionFactory 的方法中，而不要让其长期存在。

SqlSessionFactory 可以被认为是一个数据库连接池，它的作用是创建SqlSession 接口对象。因为MyBatis 的本质就是Java 对数据库的操作，所以SqlSessionFactory 的生命周期存在于整个MyBatis 的应用之中，所以一旦创建了SqlSessionFactory ， 就要长期保存它，直至不再使用MyBatis 应用，所以可以认为SqlSessionFactory 的生命周期就等同于MyBatis 的应用周期。
由于SqlSe ssionFactory 是一个对数据库的连接池，所以它占据着数据库的连接资源。如果创建多个SqlSessionFactory ，那么就存在多个数据库连接池，这样不利于对数据库资源的控制，也会导致数据库连接资源被消耗光，出现系统宕机等情况，所以尽量避免发生这样的情况。因此在一般的应用中我们往往希望SqlSessionFactory 作为一个单例，让它在应用中被共享。

如果说SqlSessionFactory 相当于数据库连接池，那么SqlSession 就相当于一个数据库连接（ Connection 对象） ，你可以在一个事务里面执行多条SQL ，然后通过它的commit、rollback 等方法， 提交或者回滚事务。所以它应该存活在一个业务请求中，处理完整个请求后，应该关闭这条连接，让它归还给SqlSessionFactory ， 否则数据库资源就很快被耗费精光，系统就会瘫痪，所以用try... catch .. . finally ...语句来保证其正确关闭。

Mapper 是一个接口，它由SqlSession 所创建，所以它的最大生命周期至多和SqlSession保持一致，尽管它很好用，但是由于SqlSession 的关闭，它的数据库连接资源也会消失，所以它的生命周期应该小于等于SqlSession 的生命周期。Mapper 代表的是一个请求中的业务处理，所以它应该在一个请求中，一旦处理完了相关的业务，就应该废弃它。

#### 5.mybatis运行原理

My Batis 的运行过程分为两大步：(1).读取配置文件缓存到Configuration 对象，用以创建SqlSessionFactory 。(2) SqISession 的执行过程

##### 5.1构建SqlSessionFactory 过程

第1 步： 通过org.apache.ibatis.builder.xml .XMLConfigBuilder 解析配置的XML 文件，读出所配置的参数，并将读取的内容存入org.apache.ibatis.session.Configuration 类对象中。而Configuration 采用的是单例模式，几乎所有的MyBatis 配置内容都会存放在这个单例对象中，以便后续将这些内容读出。
第2 步：使用Confinguration 对象去创建SqlSessionFactory 。MyBatis 中的SqlSessionFactory是一个接口，而不是一个实现类，为此MyBatis 提供了一个默认的实现类org.apache.ibatis.session.defaults.DefaultSqlSessionFactory 

##### 5.2sqlSession运行过程

如下图所示：展示了sqlsession执行过程中涉及的核心类：

<img src="https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/mybatis%E5%B1%82%E6%AC%A1%E7%BB%93%E6%9E%84%E5%9B%BE.jpg" style="zoom:67%;" />

1）SqlSession 作为MyBatis工作的主要顶层API，表示和数据库交互的会话，完成必要数据库增删改查功能

2）Executor MyBatis执行器，是MyBatis 调度的核心，负责SQL语句的生成和查询缓存的维护

3）StatementHandler 封装了JDBC Statement操作，负责对JDBCstatement的操作，如设置参数、将Statement结果集转换成List集合。

4）ParameterHandler 负责对用户传递的参数转换成JDBC Statement 所需要的参数

5）ResultSetHandler 负责将JDBC返回的ResultSet结果集对象转换成List类型的集合；

6）TypeHandler 负责java数据类型和jdbc数据类型之间的映射和转换

7）MappedStatement MappedStatement维护了一条<select|update|delete|insert>节点的封装，其包含很多属性信息

8）SqlSource 负责根据用户传递的parameterObject，动态地生成SQL语句，将信息封装到BoundSql对象中，并返回

9）BoundSql 表示动态生成的SQL语句以及相应的参数信息

10）Configuration MyBatis所有的配置信息都维持在Configuration对象之中

整体执行流程图如下所示：

![](https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/%E9%9D%9E%E5%B8%B8%E9%87%8D%E8%A6%81%E7%9A%84%E4%B8%80%E5%BC%A0%E5%9B%BE-%E5%88%86%E6%9E%90%E4%BB%A3%E7%90%86dao%E7%9A%84%E6%89%A7%E8%A1%8C%E8%BF%87%E7%A8%8B.png)

详细源码实现流程可以参考：https://mp.weixin.qq.com/s/jBUV9J0OEgAX8QEI31ZVdw

https://mp.weixin.qq.com/s/r3rRB6JbGzUH0pMc4MjKyg

 这里不做细节阐述。

#### 7.springboot整合mybatis原理

##### 7.1初始化数据源

打开`MybatisAutoConfiguration`自动配置类，可以看到该自动配置类被`@AutoConfigureAfter(DataSourceAutoConfiguration.class)`注解标注，表示当前自动配置类在`DataSourceAutoConfiguration`配置类之后解析。

打开`DruidDataSourceAutoConfigure`自动配置类，可以看到该配置类被`@AutoConfigureBefore(DataSourceAutoConfiguration.class)`注解标注，表示当前自动配置类在`DataSourceAutoConfiguration`配置类之前解析。

`DruidDataSourceAutoConfigure`自动配置类定义如下：

```
@Bean(initMethod = "init")
@ConditionalOnMissingBean
public DataSource dataSource() {
    LOGGER.info("Init DruidDataSource");
    return new DruidDataSourceWrapper();
}
```

创建`DruidDataSourceWrapper`对象，并在Bean创建之后调用`init()`方法，`DruidDataSourceWrapper`又实现了`InitializingBean`接口，`InitializingBean`接口实现的回调优于`init()`方法

##### 7.2创建SqlSessionFactory

使用`MybatisProperties`配置`SqlSessionFactoryBean`，通过`SqlSessionFactoryBean`创建`DefaultSqlSessionFactory`，`DefaultSqlSessionFactory`持有`Configuration`,`Configuration`持有`Environment`,`Environment`持有`SpringManagedTransactionFactory`和`DataSource`

```
@Bean
@ConditionalOnMissingBean
public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
    SqlSessionFactoryBean factory = new SqlSessionFactoryBean();
    // ......省略部分代码，可自行查看源代码
    return factory.getObject();
}
```

##### 7.3创建SqlSessionTemplate

```
SqlSessionTemplate`的创建比较简单，直接注入`DefaultSqlSessionFactory
public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory, ExecutorType executorType,
      PersistenceExceptionTranslator exceptionTranslator) {

    notNull(sqlSessionFactory, "Property 'sqlSessionFactory' is required");
    notNull(executorType, "Property 'executorType' is required");

    this.sqlSessionFactory = sqlSessionFactory;
    this.executorType = executorType;
    this.exceptionTranslator = exceptionTranslator;
    this.sqlSessionProxy = (SqlSession) newProxyInstance(
        SqlSessionFactory.class.getClassLoader(),
        new Class[] { SqlSession.class },
        new SqlSessionInterceptor());
}
```

##### 7.4扫描含有@Mapper注解的接口

```
public static class AutoConfiguredMapperScannerRegistrar
      implements BeanFactoryAware, ImportBeanDefinitionRegistrar, ResourceLoaderAware {
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        // 定义扫描器
        ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);

        try {
            if (this.resourceLoader != null) {
                scanner.setResourceLoader(this.resourceLoader);
            }
   // 自动配置路径
            List<String> packages = AutoConfigurationPackages.get(this.beanFactory);
            if (logger.isDebugEnabled()) {
                for (String pkg : packages) {
                    logger.debug("Using auto-configuration base package '{}'", pkg);
                }
            }
   // 指定扫描注解
            scanner.setAnnotationClass(Mapper.class);
            scanner.registerFilters();
            scanner.doScan(StringUtils.toStringArray(packages));
        } catch (IllegalStateException ex) {
            logger.debug("Could not determine auto-configuration package, automatic mapper scanning disabled.", ex);
        }
    }
}
```

该部分自定义了扫描器，扫描自动配置包路径下含有`@Mapper`注解的接口，注解为`BeanDefinition`并指定类型为`definition.setBeanClass(this.mapperFactoryBean.getClass());`，`MapperFactoryBean`实现了`FactoryBean`接口。因此我们可以得出一个结论：针对每个`Mapper`接口生成一个`MapperFactoryBean`这样一个`Bean`,在注入的时候会调用`FactoryBean`接口`getObject()`的实现。

##### 7.5 MapperFactoryBean

`MapperFactoryBean`不仅实现了`FactoryBean`接口还实现了`InitializingBean`接口，来看看`checkDaoConfig()`实现

```
@Override
protected void checkDaoConfig() {
    Configuration configuration = getSqlSession().getConfiguration();
    if (this.addToConfig && !configuration.hasMapper(this.mapperInterface)) {
        configuration.addMapper(this.mapperInterface);
    }
}
```

可以看到一个逻辑，就是有元素不添加，没有就添加。

##### 7.6 注入Mapper

经过上面的分析我们得知，每个`Mapper`都对应一个`MapperFactoryBean`，`MapperFactoryBean`又实现了`FactoryBean`接口，在注入的时候会调用`getObject()`实现

```
@Override
public T getObject() throws Exception {
    return getSqlSession().getMapper(this.mapperInterface);
}
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    return mapperProxyFactory.newInstance(sqlSession);
}
```

##### 7.7 生成代理

```
protected T newInstance(MapperProxy<T> mapperProxy) {
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
}

public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
}
```

也就是我们在`UserService`中注入的`UserMapper`实际上是一个代理对象，当调用`UserMapper`的目标方法的时候会调用`MapperProxy`的`invoke()`

##### 7.8 调用目标方法

当调用`UserMapper`目标方法的时候会调用`MapperProxy`的`invoke()`

```
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
        if (Object.class.equals(method.getDeclaringClass())) {
            return method.invoke(this, args);
        } else if (isDefaultMethod(method)) {
            return invokeDefaultMethod(proxy, method, args);
        }
    } catch (Throwable t) {
        throw ExceptionUtil.unwrapThrowable(t);
    }
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    return mapperMethod.execute(sqlSession, args);
}
```

##### 7.9 获取连接

在`7.1`章节介绍了初始化连接，那么在哪里获取的连接呢？

打开`org.apache.ibatis.executor.SimpleExecutor#doQuery`，进入`prepareStatement()`

```
private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    Connection connection = getConnection(statementLog);
    stmt = handler.prepare(connection, transaction.getTimeout());
    handler.parameterize(stmt);
    return stmt;
}
```

可以很明显看到获取`Connection`的逻辑

```
protected Connection getConnection(Log statementLog) throws SQLException {
    Connection connection = transaction.getConnection();
    if (statementLog.isDebugEnabled()) {
        return ConnectionLogger.newInstance(connection, statementLog, queryStack);
    } else {
        return connection;
    }
}
```

在`7.2`介绍了`Environment`持有`SpringManagedTransactionFactory`和`DataSource`，`SpringManagedTransactionFactory`创建出来的就是`SpringManagedTransaction`，在`SpringManagedTransaction`的`openConnection()`就可以看到从数据源连接池中获取连接的逻辑了

```
private void openConnection() throws SQLException {
    this.connection = DataSourceUtils.getConnection(this.dataSource);
    this.autoCommit = this.connection.getAutoCommit();
    this.isConnectionTransactional = DataSourceUtils.isConnectionTransactional(this.connection, this.dataSource);

    if (LOGGER.isDebugEnabled()) {
        LOGGER.debug(
            "JDBC Connection ["
            + this.connection
            + "] will"
            + (this.isConnectionTransactional ? " " : " not ")
            + "be managed by Spring");
    }
}
```

**总结**

- 配置数据源，初始化数据库连接池
- 创建SqlSessionFactory
- 创建SqlSessionTemplate
- 扫描含有@Mapper注解的接口注入到IOC容器中
- 注入Mapper接口，调用getObject()获取接口对应MapperProxyFactory生成的代理
- 调用Mapper接口目标方法的时候调用MapperProxy的invoke()
- 从数据源数据库连接池中获取连接执行sql