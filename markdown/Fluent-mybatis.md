# Fluent mybatis

#### 1.背景

众多框架都是从无到有，从有到简的一个过程，核心理念都是为简化开发为生。

像Spring这样延续至今的佼佼者，也逐渐摒弃XML，向代码化配置的方式发展。在这方面，iBatis一直是个保守派，即使在MyBatis接过iBatis的衣钵之后，也只是”重磅“推出了支持代码执行SQL的@Select/@Insert/@Update/@Delete注解（以及相应的4种Provider注解），用来抵挡开发者们对XML泛滥的吐槽，但是使用注解也存在一些问题，比如说代码可读性插，可复用性差等等。这时候孕育出一大批基于mybatis加强的orm框架，例如mybatis-plus，tk-mybatis等等。

mybatis-plus的人气比tk-mybatis高很多，原因我个人觉得有两个：

- mybatis-plus的功能更强，封装的更好。
- mybatis-plus文档比较全，有官网https://mp.baomidou.com/，tk-mybatis至今我没找到。。。

但是以上两个都不是我今天讲的主角，今天主角是 **Fluent-Mybatis: 基于mybatis但青出于蓝而胜于蓝**

- **No XML, No Mapper, No If else.....**
- **只需Entity就实现强大的FluentAPI: 支持分页, 嵌套查询, AND OR组合, 聚合函数...**

<img src="https://images.gitee.com/uploads/images/2021/0707/221513_90936288_1767937.png" alt="img" style="zoom: 50%;" />

#### 2.实现原理

使用fluent-mybatis非常简单：

```xml
<dependencies>
    <!-- 引入fluent-mybatis 运行依赖包, scope为compile -->
    <dependency>
        <groupId>com.github.atool</groupId>
        <artifactId>fluent-mybatis</artifactId>
        <version>1.6.8</version>
    </dependency>
    <!-- 引入fluent-mybatis-processor, scope设置为provider 编译需要，运行时不需要 -->
    <dependency>
        <groupId>com.github.atool</groupId>
        <artifactId>fluent-mybatis-processor</artifactId>
        <version>1.6.8</version>
    </dependency>
</dependencies>
```

引入依赖之后我们只需要编写entity实现类信息即可。然后通过编译会生成对应的mapper接口类供我们使用，其实现原理如下图所示：

<img src="https://images.gitee.com/uploads/images/2021/0707/221539_123cd4fa_1767937.png" alt="FluentMybatis原理" style="zoom: 50%;" />

- 核心接口类, 使用时需要了解

1. mapper/*Mapper: mybatis的Mapper定义接口, 定义了一系列通用的数据操作接口方法。
2. dao/*BaseDao: Dao实现基类, 所有的DaoImpl都继承各自基类 根据分层编码的原则，我们不会在Service类中直接使用Mapper类，而是引用Dao类。我们在Dao实现类中根据条件实现具体的数据操作方法。
3. wrapper/*Query: **fluent mybatis核心类**, 用来进行动态sql的构造, 进行条件查询。
4. wrapper/*Updater: fluent mybatis核心类, 用来动态构造update语句。
5. entity/*EntityHelper: Entity帮助类, 实现了Entity和Map的转换方法

- 辅助实现时, 实现fluent mybatis动态sql拼装和fluent api时内部用到的类，使用时无需了解 在使用上，我们主要会接触到上述5个生成的java类。Fluent Mybatis为了实现动态拼接和Fluent API功能，还生成了一系列辅助类。

1. helper/*Mapping: 表字段和Entity属性映射定义类
2. helper/*SqlProviderP: **Mapper接口动态sql提供者**
3. helper/*WrapperHelper: Query和Updater具体功能实现, 包含几个实现:select, where, group by, having by, order by, limit



#### 3.动态sql的实现

我们通过一个比较典型的业务需求来具体实现和对比下，假如有学生成绩表结构如下:

```sql
create table `student_score`
(
    id           bigint auto_increment comment '主键ID' primary key,
    student_id   bigint            not null comment '学号',
    gender_man   tinyint default 0 not null comment '性别, 0:女; 1:男',
    school_term  int               null comment '学期',
    subject      varchar(30)       null comment '学科',
    score        int               null comment '成绩',
    gmt_create   datetime          not null comment '记录创建时间',
    gmt_modified datetime          not null comment '记录最后修改时间',
    is_deleted   tinyint default 0 not null comment '逻辑删除标识'
) engine = InnoDB default charset=utf8;
```

现在有需求:

**「统计 2000 年三门学科('英语', '数学', '语文')及格分数按学期,学科统计最低分，最高分和平均分, 且样本数需要大于 1 条,统计结果按学期和学科排序」**

我们可以写 SQL 语句如下

```sql
select school_term,
       subject,
       count(score) as count,
       min(score)   as min_score,
       max(score)   as max_score,
       avg(score)   as avg_score
from student_score
where school_term >= 2000
  and subject in ('英语', '数学', '语文')
  and score >= 60
  and is_deleted = 0
group by school_term, subject
having count(score) > 1
order by school_term, subject;
```

那上面的需求，分别用`fluent mybatis`, 原生`mybatis`和`Mybatis plus`来实现一番。

**使用fluent mybatis 来实现上面的功能**

<img src="https://mmbiz.qpic.cn/mmbiz_png/cWeX2iblLviaEUiaql8H5zUgFYsD5a6ATiarpBoyUn2C37NSpKGY8SvINia3GRM6AasCCTGCDNgAqUNCib41KxnfzPHg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1" alt="图片" style="zoom:50%;" />

需要本文具体演示代码可加我微信：codedq，免费获取！

我们可以看到`fluent api`的能力，以及 IDE 对代码的渲染效果。

**换成mybatis原生实现效果**

定义Mapper接口

```java
public interface MyStudentScoreMapper {
    List<Map<String, Object>> summaryScore(SummaryQuery paras);
}
```

定义接口需要用到的参数实体 SummaryQuery

```java
@Data
@Accessors(chain = true)
public class SummaryQuery {
    private Integer schoolTerm;

    private List<String> subjects;

    private Integer score;

    private Integer minCount;
}
```

定义实现业务逻辑的`mapper xml`文件

```
<select id="summaryScore" resultType="map" parameterType="cn.org.fluent.mybatis.springboot.demo.mapper.SummaryQuery">
    select school_term,
    subject,
    count(score) as count,
    min(score) as min_score,
    max(score) as max_score,
    avg(score) as max_score
    from student_score
    where school_term >= #{schoolTerm}
    and subject in
    <foreach collection="subjects" item="item" open="(" close=")" separator=",">
        #{item}
    </foreach>
    and score >= #{score}
    and is_deleted = 0
    group by school_term, subject
    having count(score) > #{minCount}
    order by school_term, subject
</select>
```

实现业务接口(这里是测试类，实际应用中应该对应 Dao 类)

```java
@RunWith(SpringRunner.class)
@SpringBootTest(classes = QuickStartApplication.class)
public class MybatisDemo {
    @Autowired
    private MyStudentScoreMapper mapper;

    @Test
    public void mybatis_demo() {
        // 构造查询参数
        SummaryQuery paras = new SummaryQuery()
            .setSchoolTerm(2000)
            .setSubjects(Arrays.asList("英语", "数学", "语文"))
            .setScore(60)
            .setMinCount(1);

        List<Map<String, Object>> summary = mapper.summaryScore(paras);
        System.out.println(summary);
    }
}
```

总之，直接使用 mybatis，实现步骤还是相当的繁琐，效率太低。那换成`mybatis plus`的效果怎样呢？

**换成mybatis plus实现效果**

`mybatis plus`的实现比`mybatis`会简单比较多，实现效果如下

<img src="https://mmbiz.qpic.cn/mmbiz_png/cWeX2iblLviaEUiaql8H5zUgFYsD5a6ATiarSWGyxbcibArMrnEFQic9fWvOtY16PkM6icQBJeUU9VSHqoeWxNAF3jY7g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1" alt="图片" style="zoom:50%;" />

如红框圈出的，写`mybatis plus`实现用到了比较多字符串的硬编码（可以用 Entity 的 get lambda 方法部分代替字符串编码）。字符串的硬编码，会给开发同学造成不小的使用门槛，个人觉得主要有 2 点：

1. 字段名称的记忆和敲码困难
2. Entity 属性跟随数据库字段发生变更后的运行时错误

**fluent-mybatis实现动态sql原理**

Fluent Mybatis构造动态SQL语句的方式是直接使用mybatis3中的SQLProvider功能。

@SelectProvider, @UpdateProvider, @InsertProvider, @DeleteProvider 这些Provider是声明在Mapper接口类上的, 比如你定义了一个YourEntity实体类, fluent mybatis在编译时，会生成对应的Mapper文件和SqlProvider文件

总结一下FluentMybatis和mybatis plus的区别

| -                            | Mybatis Plus                                                 | Fluent Mybatis                                               |
| ---------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 代码生成                     | 生成 Entity, Mapper, Wrapper等文件                           | 只生成Entity, 再通过编译生成 Mapper, Query, Update 和 SqlProvider |
| 和Mybatis的共生关系          | 需要替换原有的SqlSessionFactoryBean                          | 对Mybatis没有任何修改,原来怎么用还是怎么用                   |
| 动态SQL构造方式              | 应用启动时, 根据Entity注解信息构造动态xml片段，注入到Mybatis解析器 | 应用编译时，根据Entity注解，编译生成对应方法的SqlProvider，利用mybatis的Mapper上@InsertProvider @SelectProvider @UpdateProvider注解关联 |
| 动态SQL结果是否容易DEBUG跟踪 | 不容易debug                                                  | 容易，直接定位到SQLProvider方法上，设置断点即可              |
| 动态SQL构造                  | 通过硬编码字段名称, 或者利用Entity的get方法的lambda表达式    | 通过编译手段生成对应的方法名，直接调用方法即可               |
| 字段变更后的错误发现         | 通过get方法的lambda表达的可以编译发现，通过字段编码的无法编译发现 | 编译时便可发现                                               |
| 不同字段动态SQL构造方法      | 通过接口参数方式                                             | 通过接口名称方式, FluentAPI的编码效率更高                    |
| 语法渲染特点                 | 无                                                           | 通过关键变量select, update, set, and, or可以利用IDE语法渲染, 可读性更高 |

框架文档地址：https://gitee.com/fluent-mybatis/fluent-mybatis-docs