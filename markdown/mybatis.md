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

SqlSessionFactoryBuilder 的作用在于创建SqlSessionFactory ，创建成功后，SqlSessionFactoryBuilder 就失去了作用，所以它只能存在于创建Sq!SessionFactory 的方法中，而不要让其长期存在。

SqlSessionFactory 可以被认为是一个数据库连接池，它的作用是创建SqlSession 接口对象。因为MyBatis 的本质就是Java 对数据库的操作，所以SqlSessionFactory 的生命周期存在于整个MyBatis 的应用之中，所以一旦创建了SqlSessionFactory ， 就要长期保存它，直至不再使用MyBatis 应用，所以可以认为SqlSessionFactory 的生命周期就等同于MyBatis 的应用周期。
由于SqlSe ssionFactory 是一个对数据库的连接池，所以它占据着数据库的连接资源。如果创建多个SqlSessionFactory ，那么就存在多个数据库连接池，这样不利于对数据库资源的控制，也会导致数据库连接资源被消耗光，出现系统密机等情况，所以尽量避免发生这样的情况。因此在一般的应用中我们往往希望SqlSessionFactory 作为一个单例，让它在应用中被共享。

如果说SqlSessionFactory 相当于数据库连接池，那么SqlSession 就相当于一个数据库连接（ Connection 对象） ，你可以在一个事务里面执行多条SQL ，然后通过它的commit、rollback 等方法， 提交或者回滚事务。所以它应该存活在一个业务请求中，处理完整个请求后，应该关闭这条连接，让它归还给SqlSessionFactory ， 否则数据库资源就很快被耗费精光，系统就会瘫痪，所以用try... catch .. . finally ...语句来保证其正确关闭。

Mapper 是一个接口，它由SqlSession 所创建，所以它的最大生命周期至多和SqlSession保持一致，尽管它很好用，但是由于SqlSession 的关闭，它的数据库连接资源也会消失，所以它的生命周期应该小于等于Sq!Session 的生命周期。Mapper 代表的是一个请求中的业务处理，所以它应该在一个请求中，一旦处理完了相关的业务，就应该废弃它。