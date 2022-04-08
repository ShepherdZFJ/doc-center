# elasticsearch基础篇：概述、操作和集成

### 1.概述

#### 1.1 什么是elasticsearch

![](https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/es_1)

`elaticsearch`简称为es， es是一个开源的高扩展的分布式全文检索引擎，它可以近乎实时的存储、检索数据；本身扩展性很好，可以扩展到上百台服务器，处理PB级别的数据。es使用Java开发并使用Lucene作为其核心来实现所有索引和搜索的功能，但是它的目的是通过简单的RESTful API来隐藏Lucene的复杂性，从而让全文搜索变得简单。

#### 1.2 全文检索引擎

Google、百度类的网站搜索，它们都是根据网页中的关键字生成索引，我们在搜索的时候输入关键字，它们会将该关键字即索引匹配到的所有网页返回；还有常见的项目中应用日志的搜索等等。对于这些非结构化的数据文本，关系型数据库搜索不是能很好的支持。

一般传统数据库，全文检索都实现的很鸡肋，因为一般也没人用数据库存文本字段。进行全文检索需要扫描整个表，如果数据量大的话即使对SQL 的语法优化，也收效甚微。建立了索引，但是维护起来也很麻烦，对于 insert 和 update 操作都会重新构建索引。

为了解决结构化数据搜索和非结构化数据搜索性能问题，我们就需要专业，健壮，强大的全文搜索引擎。

这里说到的**全文搜索引擎**指的是目前广泛应用的主流搜索引擎。它的工作原理是计算机索引程序通过扫描文章中的每一个词，对每一个词建立一个索引，指明该词在文章中出现的次数和位置，当用户查询时，检索程序就根据事先建立的索引进行查找，并将查找的结果反馈给用户的检索方式。这个过程类似于通过字典中的检索字表查字的过程。

### 1.3 elasticsearch 和 solr

`Lucene` 是Apache 软件基金会Jakarta 项目组的一个子项目，提供了一个简单却强大的应用程式接口，能够做全文索引和搜寻。在Java 开发环境里Lucene 是一个成熟的免费开源工具。就其本身而言，Lucene 是当前以及最近几年最受欢迎的免费Java 信息检索程序库。但Lucene 只是一个提供全文搜索功能类库的核心工具包，而真正使用它还需要一个完善的服务框架搭建起来进行应用。

目前市面上流行的搜索引擎软件，主流的就两款：**Elasticsearch 和Solr**,这两款都是基于Lucene 搭建的，可以独立部署启动的搜索引擎服务软件。由于内核相同，所以两者除了服务器安装、部署、管理、集群以外，对于数据的操作 修改、添加、保存、查询等等都十分类似。在使用过程中，一般都会将Elasticsearch 和Solr 这两个软件对比，然后进行选型。这两个搜索引擎都是流行的，先进的的开源搜索引擎。它们都是围绕核心底层搜索库 - Lucene构建的 - 但它们又是不同的。像所有东西一样，每个都有其优点和缺点，对比如下所示：

![](https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/es_2)

Google 搜索趋势结果表明：

1）与 Solr 相比，Elasticsearch 具有很大的吸引力，但这并不意味着Apache Solr 已经死亡。虽然有些人可能不这么认为，但Solr 仍然是最受欢迎的搜索引擎之一，拥有强大的社区和开源支持。

2）与Solr 相比，Elasticsearch 易于安装且非常轻巧。此外，你可以在几分钟内安装并运行Elasticsearch。但是，如果Elasticsearch 管理不当，这种易于部署和使用可能会成为一个问题。基于JSON 的配置很简单，但如果要为文件中的每个配置指定注释，那么它不适合您。总的来说，如果你的应用使用的是JSON，那么Elasticsearch 是一个更好的选择。否则，请使用Solr，因为它的schema.xml 和solrconfig.xml 都有很好的文档记录。

3）Solr 拥有更大，更成熟的用户，开发者和贡献者社区。ES 虽拥有的规模较小但活跃的用户社区以及不断增长的贡献者社区。Solr 贡献者和提交者来自许多不同的组织，而Elasticsearch 提交者来自单个公司。

4）Solr 更成熟，但ES 增长迅速，更稳定。

5）Solr 是一个非常有据可查的产品，具有清晰的示例和API 用例场景。 Elasticsearch 的文档组织良好，但它缺乏好的示例和清晰的配置说明。

**那么如何选择呢？**

1）由于易于使用，Elasticsearch 在新开发者中更受欢迎。一个下载和一个命令就可以启动一切。

2） 如果除了搜索文本之外还需要它来处理分析查询，Elasticsearch 是更好的选择

3） 如果需要分布式索引，则需要选择Elasticsearch。对于需要良好可伸缩性和以及性能分布式环境，Elasticsearch 是更好的选择。

4） Elasticsearch 在开源日志管理用例中占据主导地位，许多组织在Elasticsearch 中索引它们的日志以使其可搜索。

5） 如果你喜欢监控和指标，那么请使用Elasticsearch，因为相对于Solr，Elasticsearch 暴露了更多的关键指标

#### 1.4 elasticsearch的应用

1）GitHub: 2013 年初，抛弃了Solr，采取Elasticsearch 来做PB 级的搜索。“GitHub 使用Elasticsearch 搜索20TB 的数据，包括13 亿文件和1300 亿行代码”。

2） 维基百科：启动以Elasticsearch 为基础的核心搜索架构

3） SoundCloud：“SoundCloud 使用Elasticsearch 为1.8 亿用户提供即时而精准的音乐搜索服务”。

4） 百度：目前广泛使用Elasticsearch 作为文本数据分析，采集百度所有服务器上的各类指标数据及用户自定义数据，通过对各种数据进行多维分析展示，辅助定位分析实例异常或业务层面异常。目前覆盖百度内部20 多个业务线（包括云分析、网盟、预测、文库、直达号、钱包、风控等），单集群最大100 台机器，200 个ES 节点，每天导入30TB+数据。

5） 新浪：使用Elasticsearch 分析处理32 亿条实时日志。

6） 阿里：使用Elasticsearch 构建日志采集和分析体系。

7） Stack Overflow：解决Bug 问题的网站，全英文，编程人员交流的网站

### 2.elasticsearch入门

#### 2.1 elasticsearch安装

##### 2.1.1 本地安装

本地安装elasticsearch非常简单，只有需要去elasticsearch官网下载对应的系统软件包，然后执行bin文件下的elasticsearch执行文件即可，因为我的电脑是mac，mac软件包如下所示：

![](https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/es_3)

启动之后，我们在浏览器输入http://127.0.0.1:9200

```json
{
  "name" : "node-1",
  "cluster_name" : "es-cluster",
  "cluster_uuid" : "p12KHyIqSpeZkbojXwAh-g",
  "version" : {
    "number" : "7.15.2",
    "build_flavor" : "default",
    "build_type" : "tar",
    "build_hash" : "93d5a7f6192e8a1a12e154a2b81bf6fa7309da0c",
    "build_date" : "2021-11-04T14:04:42.515624022Z",
    "build_snapshot" : false,
    "lucene_version" : "8.9.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

如果我们想本地部署集群：

```shell
bin/elasticsearch -E node.name=node-1 -E cluster.name=es-cluster -E path.data=node1_data
bin/elasticsearch -E node.name=node-2 -E cluster.name=es-cluster -E path.data=node2_data
bin/elasticsearch -E node.name=node-3 -E cluster.name=es-cluster -E path.data=node3_data
```

这里在启动命令后面bin/elasticsearch，需要指定集群名称，节点名称，不同节点的节点名称必须唯一，集群名称必须唯一，path.data指定数据存放的路径，可以不指定，这里是由于在本地一个机器一个安装包起三个节点模拟集群，所以需要指定一下，不同节点的存放数据路径不一样。部署之后，查看集群信息，在浏览器输入http://127.0.0.1:9200/_cat/nodes?v

```
ip        heap.percent ram.percent cpu load_1m load_5m load_15m node.role   master name
127.0.0.1           45          98  18    2.05                  cdfhilmrstw *      node-2
127.0.0.1            9          98  18    2.05                  cdfhilmrstw -      node-3
127.0.0.1           16          98  18    2.05                  cdfhilmrstw -      node-1
```

注意：9300 端口为Elasticsearch 集群间组件的通信端口，9200 端口为浏览器访问的http

Elasticsearch 是使用java 开发的，且7.8 版本的ES 需要JDK 版本1.8 以上，默认安装包带有jdk 环境，如果系统配置JAVA_HOME，那么使用系统默认的JDK，如果没有配置使用自带的JDK，一般建议使用系统配置的JDK，换句话说，希望在安装elasticsearch先安装配置好java jdk环境。

同时，如果我们想要修改elasticsearch的内存大小：请修改config/jvm.options 配置文件

```shell
# 设置JVM 初始内存为1G。此值可以设置与-Xmx 相同，以避免每次垃圾回收完成后JVM 重新分配内存
# Xms represents the initial size of total heap space
# 设置JVM 最大可用内存为1G
# Xmx represents the maximum size of total heap space
-Xms1g
-Xmx1g
```

#### 2.1.2 使用docker安装elasticsearch

使用docker安装elasticsearch非常简单，elasticsearch官方提供了镜像，我们只需要按照如下步骤安装即可：

1）拉取elasticsearch和kibana镜像，这里拉取7.4.2的版本

```shell
# 存储和检索数据
docker pull elasticsearch:7.4.2

```

2）配置挂载数据的文件夹

```shell
# 创建配置文件目录
mkdir -p /mydata/elasticsearch/config

# 创建数据目录
mkdir -p /mydata/elasticsearch/data

# 将/mydata/elasticsearch/文件夹中文件都可读可写
chmod -R 777 /mydata/elasticsearch/

# 配置任意机器可以访问 elasticsearch
echo "http.host: 0.0.0.0" >/mydata/elasticsearch/config/elasticsearch.yml
```

3）运行启动elasticsearch服务

```shell
docker run --name elasticsearch -p 9200:9200 -p 9300:9300 \
-e  "discovery.type=single-node" \
-e ES_JAVA_OPTS="-Xms64m -Xmx512m" \
-v /mydata/elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml \
-v /mydata/elasticsearch/data:/usr/share/elasticsearch/data \
-v  /mydata/elasticsearch/plugins:/usr/share/elasticsearch/plugins \
-d elasticsearch:7.4.2 
```

- `-p 9200:9200 -p 9300:9300`：向外暴露两个端口，9200用于HTTP REST API请求，9300 ES 在分布式集群状态下 ES 之间的通信端口；
- `-e  "discovery.type=single-node"`：es 以单节点运行

- `-e ES_JAVA_OPTS="-Xms64m -Xmx512m"`：设置启动占用内存，不设置可能会占用当前系统所有内存
- -v：挂载容器中的配置文件、数据文件、插件数据到本机的文件夹；

- `-d elasticsearch:7.4.2`：指定要启动的镜像

4）设置elasticsearch随docker重启

```shell
# 当前 Docker 开机自启，所以 ES 现在也是开机自启
docker update elasticsearch --restart=always
```

#### 2.1.3 使用k8s安装elasticsearch

由于使用k8s安装elasticsearch比较吃配置，并且既然都用k8s部署了，肯定是集群部署，有点难度，待后续研究补上..............

#### 2.1.4 安装ES的图形化界面插件

1）安装elasticsearch-head-master插件

这个插件直接安装在浏览器中，比如说google中，然后我们就可以连接elasticsearch，查看相关信息了，如下所示：

![](https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/es_4)

2）安装kibana

Kibana 是一个免费且开放的用户界面，能够让你对 Elasticsearch 数据进行可视化，并让你在 Elastic Stack 中进行导航。你可以进行各种操作，从跟踪查询负载，到理解请求如何流经你的整个应用，都能轻松完成。

本地化安装和安装elasticsearch去elastic公司官网下载安装软件包，执行bin下面kibana执行文件即可，但是安装之前，需要修改配置config/kibana.yml 文件：

```yaml
# 默认端口
server.port: 5601
# ES 服务器的地址
elasticsearch.hosts: ["http://localhost:9200"]
# 索引名
kibana.index: ".kibana"
# 支持中文
i18n.locale: "zh-CN"
```

启动之后在浏览器输入http://127.0.0.1:5601就可以看到如下：

![](https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/es_5)

在kibana中有一个dev-tools工具是我们经常使用的，我们可以在这里输入DSL查询语句，查询es中的文档数据。

当然我们也可以使用docker部署，非常简单。

```shell
docker run --name kibana \
-e ELASTICSEARCH_HOSTS=http://192.168.163.131:9200 \
-p 5601:5601 \
-d kibana:7.4.2
```

`-e ELASTICSEARCH_HOSTS=``http://192.168.163.131:9200`: **这里要设置成自己的虚拟机IP地址**

### 2.2 elasticsearch中的数据格式

Elasticsearch 是面向文档型数据库，一条数据在这里就是一个文档。为了方便大家理解，我们将Elasticsearch 里存储文档数据和关系型数据库MySQL 存储数据的概念进行一个类比

![](https://markdown-file-zfj.oss-cn-hangzhou.aliyuncs.com/es_6)

ES 里的Index 可以看做一个库，而Types 相当于表，Documents 则相当于表的行。这里Types 的概念已经被逐渐弱化，Elasticsearch 6.X 中，一个index 下已经只能包含一个type，Elasticsearch 7.X 中, Type 的概念已经被删除了

### 2.3 elasticsearch的增删改查操作

我们对elasticsearch数据进行操作可以使用postman发送http请求，因为elasticsearch提供了rest API支持；也可以通过可视化插件、kibana书写DSL语句操作es数据，推荐使用后者，因为这种方式我们平常使用Navicat操作关系型数据库数据一样。

这里列举几个rest api操作，其实rest api和kinaba的DSL语句大同小异：

1）新增索引

使用rest api的PUT http:127.0.0.1:9200/product 就可以创建一个名为product索引了

使用DSL语句也非常简单，使用如下语句即可：创建一个product索引，并设置索引分片数为3，分片副本数为2

```shell
PUT product
{
  "settings": {
    "number_of_shards": 3
    , "number_of_replicas": 2
  }
}
```

2）删除索引

```shell
DELETE product
```

3）_cat 查看es相关信息

```shell
#查看所有索引    ?v 查询的信息更多
GET _cat/indices?v

#查看所有节点    ?v 查询的信息更多
GET _cat/nodes?v
```

4）新增文档数据

我们可以使用`PUT`和`POST`新增文档数据， POST新增，如果不指定id，会自动生成id。指定id如果该id存在就会修改这个数据，并新增版本号；PUT可以新增也可以修改。PUT必须指定id；由于PUT需要指定id，我们一般用来做修改操作，不指定id会报错。

```shell
POST product/_doc/1
{
  "name":"苹果手机13pro",
  "category":"手机",
  "price":6599,
  "stock":100,
  "brand":"apple",
  "desc":"苹果的手机真的很好用啊"
}

PUT product/_doc/2
{
  "name":"华为p50 pro",
  "category":"手机",
  "price":4999,
  "stock":38,
  "brand":"huawei",
  "desc":"拍照最佳选择，国产手机的骄傲",
  "createTime":"2021-10-11 07:10:00"
}

```

注意这里无论是PUT还是POST，当id存在都是修改数据，但是他们的修改方式都是删除老文档数据，然后再插入。如果要想基于原文档修改某个字段属性值，要使用如下：

```shell
POST product/_doc/1/_update
{
  "doc":{
    "name":"红富士苹果"
  }
}
```

批量插入使用bulk

```shell
POST product/_bulk
{ "index": { "_id": 4 }}
{ "name":"小米11","category":"电子产品", "price":2999, "stock":10000 }
{ "index": { "_id": 5 }}
{ "name":"oppo findx","category":"电子产品", "price":5000, "stock":6666 }
{ "index": { "_id": 6 }}
{ "name":"vivo v20","category":"女性电子产品", "price":3999, "stock":8888 }

```

5）查询数据

```shell
#查询所有
GET product/_search
{
  "query": {
    "match_all": {}
  }
}

# 匹配查询，注意match只能放一个字段条件
GET product/_search
{
  "query": {
    "match": {
      "name": "小米苹果"
    }
  }
}

# 匹配查询  多个条件组合
GET product/_search
{
  "query": {
    "bool": {
      "must": [
        {"match": {
          "category": "手机"
        }},
        {
          "match": {
            "name": "苹果"
          }
        }
      ]
    }
  }
}

# 多字段匹配查询
GET product/_search
{
  "query": {
    "multi_match": {
      "query": "苹果",
      "fields": ["name", "desc"]
    }
  }
}

#精确匹配term
GET product/_search
{
  "query": {
    "term": {
      "stock": {
        "value":38
      }
    }
  }
}

# 范围查询
GET product/_search
{
  "query": {
    "range": {
      "stock": {
        "gte": 10,
        "lte": 100
      }
    }
  }
}

# 排序
GET student/_search
{
  "query": {
    "match_all": {}
  },
  "sort": [
    {
      "rank": {
        "order": "desc"
      },
      "age":
      {
        "order":"desc"
      }
      
    }
  ]
}

# 分页查询
GET product/_search
{
  "query": {
    "match_all": {}
  },
  "sort": [
    {
      "_id": {
        "order": "desc"
      }
    }
  ], 
  "from": 0,
  "size": 2
}

# 索引mapping添加字段
PUT /product/_mapping
{
  "properties": {
    "title": {
      "type": "text"
    }
  }
}
  

# 分词器测试
GET product/_analyze
{
  "analyzer": "ik_smart",
  "text": "我是一个中国人，测试中文分词器"
}

```

更多高阶、详细查询语句请自行查询相关文档。

### 3.springboot集成elasticsearch

springboot集成es现在主流的方式大概有两种：集成`elasticsearch rest client`和使用`spring data`。

今天讲一下集成rest client，在pom.xml加上下面依赖即可：

```xml
     <dependency>
            <groupId>org.elasticsearch</groupId>
            <artifactId>elasticsearch</artifactId>
        </dependency>
        <!-- elasticsearch的客户端 -->
        <dependency>
            <groupId>org.elasticsearch.client</groupId>
            <artifactId>elasticsearch-rest-high-level-client</artifactId>
        </dependency>
        <!-- elasticsearch依赖2.x的log4j -->
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-api</artifactId>
        </dependency>
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-core</artifactId>
        </dependency>

        <dependency>
            <groupId>commons-logging</groupId>
            <artifactId>commons-logging</artifactId>
            <version>1.2</version>
        </dependency>

        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>2.9.9</version>
        </dependency>
        <!-- junit单元测试 -->
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
        </dependency>
```

接下来主要讲述一下es的文档doc的增删改查，这是我们在平时开发项目中主要用到的

1）新增文档

```java
public class InsertDoc {
    public static void main(String[] args) throws Exception {

        RestHighLevelClient esClient = new RestHighLevelClient(
                RestClient.builder(new HttpHost("127.0.0.1", 9200, "http"))
        );

        // 插入数据
        IndexRequest request = new IndexRequest();
        request.index("user").id("1001");

        User user = new User();
        user.setName("zhangsan");
        user.setAge(30);
        user.setSex("男");

        // 向ES插入数据，必须将数据转换位JSON格式
        ObjectMapper mapper = new ObjectMapper();
        String userJson = mapper.writeValueAsString(user);

        request.source(userJson, XContentType.JSON);

        IndexResponse response = esClient.index(request, RequestOptions.DEFAULT);

        System.out.println(response.getResult());

        esClient.close();
    }
}
```

批量新增文档

```java
/**
 * @author fjzheng
 * @version 1.0
 * @date 2021/12/7 17:23
 */
public class InsertDocBatch {
    public static void main(String[] args) throws Exception {

        RestHighLevelClient esClient = new RestHighLevelClient(
                RestClient.builder(new HttpHost("127.0.0.1", 9200, "http"))
        );

        // 批量插入数据
        BulkRequest request = new BulkRequest();

        request.add(new IndexRequest().index("user").id("1001").source(XContentType.JSON, "name", "zhangsan", "age",30,"sex","男"));
        request.add(new IndexRequest().index("user").id("1002").source(XContentType.JSON, "name", "lisi", "age",30,"sex","女"));
        request.add(new IndexRequest().index("user").id("1003").source(XContentType.JSON, "name", "wangwu", "age",40,"sex","男"));
        request.add(new IndexRequest().index("user").id("1004").source(XContentType.JSON, "name", "wangwu1", "age",40,"sex","女"));
        request.add(new IndexRequest().index("user").id("1005").source(XContentType.JSON, "name", "wangwu2", "age",50,"sex","男"));
        request.add(new IndexRequest().index("user").id("1006").source(XContentType.JSON, "name", "wangwu3", "age",50,"sex","男"));
        request.add(new IndexRequest().index("user").id("1007").source(XContentType.JSON, "name", "wangwu44", "age",60,"sex","男"));
        request.add(new IndexRequest().index("user").id("1008").source(XContentType.JSON, "name", "wangwu555", "age",60,"sex","男"));
        request.add(new IndexRequest().index("user").id("1009").source(XContentType.JSON, "name", "wangwu66666", "age",60,"sex","男"));

        BulkResponse response = esClient.bulk(request, RequestOptions.DEFAULT);
        System.out.println(response.getTook());
        System.out.println(response.getItems());

        esClient.close();
    }
}
```

2）更新文档

```java
public class UpdateDoc {
    public static void main(String[] args) throws Exception {

        RestHighLevelClient esClient = new RestHighLevelClient(
                RestClient.builder(new HttpHost("localhost", 9200, "http"))
        );

        // 修改数据
        UpdateRequest request = new UpdateRequest();
        request.index("user").id("1001");
        request.doc(XContentType.JSON, "sex", "女");

        UpdateResponse response = esClient.update(request, RequestOptions.DEFAULT);

        System.out.println(response.getResult());

        esClient.close();
    }
}
```

3）删除文档

```java
/**
 * @author fjzheng
 * @version 1.0
 * @date 2021/12/7 18:08
 */
public class DeleteDoc {
    public static void main(String[] args) throws Exception {

        RestHighLevelClient esClient = new RestHighLevelClient(
                RestClient.builder(new HttpHost("localhost", 9200, "http"))
        );


        DeleteRequest request = new DeleteRequest();
        request.index("user").id("1001");

        DeleteResponse response = esClient.delete(request, RequestOptions.DEFAULT);
        System.out.println(response.toString());

        esClient.close();
    }
}

```

4）根据id查询文档

```java
public class GetDoc {
    public static void main(String[] args) throws Exception {

        RestHighLevelClient esClient = new RestHighLevelClient(
                RestClient.builder(new HttpHost("localhost", 9200, "http"))
        );

        // 查询数据
        GetRequest request = new GetRequest();
        request.index("user").id("1001");
        GetResponse response = esClient.get(request, RequestOptions.DEFAULT);

        System.out.println(response.getSourceAsString());

        esClient.close();
    }

}
```

5）多条件复杂查询

```java
/**
 * @author fjzheng
 * @version 1.0
 * @date 2021/12/7 17:36
 */
public class SearchDoc {
    public static void main(String[] args) throws Exception {

        RestHighLevelClient esClient = new RestHighLevelClient(
                RestClient.builder(new HttpHost("localhost", 9200, "http"))
        );

        // 1. 查询索引中全部的数据
//        SearchRequest request = new SearchRequest();
//        request.indices("user");
//
//        request.source(new SearchSourceBuilder().query(QueryBuilders.matchAllQuery()));
//
//        SearchResponse response = esClient.search(request, RequestOptions.DEFAULT);
//
//        SearchHits hits = response.getHits();
//
//        System.out.println(hits.getTotalHits());
//        System.out.println(response.getTook());
//
//        for ( SearchHit hit : hits ) {
//            System.out.println(hit.getSourceAsString());
//        }

        // 2. 条件查询 : termQuery
//        SearchRequest request = new SearchRequest();
//        request.indices("user");
//
//        request.source(new SearchSourceBuilder().query(QueryBuilders.termQuery("age", 30)));
//        SearchResponse response = esClient.search(request, RequestOptions.DEFAULT);
//
//        SearchHits hits = response.getHits();
//
//        System.out.println(hits.getTotalHits());
//        System.out.println(response.getTook());
//
//        for ( SearchHit hit : hits ) {
//            System.out.println(hit.getSourceAsString());
//        }

        // 3. 分页查询
//        SearchRequest request = new SearchRequest();
//        request.indices("user");
//
//        SearchSourceBuilder builder = new SearchSourceBuilder().query(QueryBuilders.matchAllQuery());
//        // (当前页码-1)*每页显示数据条数
//        builder.from(2);
//        builder.size(2);
//        request.source(builder);
//        SearchResponse response = esClient.search(request, RequestOptions.DEFAULT);
//
//        SearchHits hits = response.getHits();
//
//        System.out.println(hits.getTotalHits());
//        System.out.println(response.getTook());
//
//        for ( SearchHit hit : hits ) {
//            System.out.println(hit.getSourceAsString());
//        }

//        // 4. 查询排序
//        SearchRequest request = new SearchRequest();
//        request.indices("user");
//
//        SearchSourceBuilder builder = new SearchSourceBuilder().query(QueryBuilders.matchAllQuery());
//        //
//        builder.sort("age", SortOrder.DESC);
//
//        request.source(builder);
//        SearchResponse response = esClient.search(request, RequestOptions.DEFAULT);
//
//        SearchHits hits = response.getHits();
//
//        System.out.println(hits.getTotalHits());
//        System.out.println(response.getTook());
//
//        for ( SearchHit hit : hits ) {
//            System.out.println(hit.getSourceAsString());
//        }

//        // 5. 过滤字段
//        SearchRequest request = new SearchRequest();
//        request.indices("user");
//
//        SearchSourceBuilder builder = new SearchSourceBuilder().query(QueryBuilders.matchAllQuery());
//        //
//        String[] excludes = {"age"};
//        String[] includes = {};
//        builder.fetchSource(includes, excludes);
//
//        request.source(builder);
//        SearchResponse response = esClient.search(request, RequestOptions.DEFAULT);
//
//        SearchHits hits = response.getHits();
//
//        System.out.println(hits.getTotalHits());
//        System.out.println(response.getTook());
//
//        for ( SearchHit hit : hits ) {
//            System.out.println(hit.getSourceAsString());
//        }

//        // 6. 组合查询
//        SearchRequest request = new SearchRequest();
//        request.indices("user");
//
//        SearchSourceBuilder builder = new SearchSourceBuilder();
//        BoolQueryBuilder boolQueryBuilder = QueryBuilders.boolQuery();
//
//        //boolQueryBuilder.must(QueryBuilders.matchQuery("age", 30));
//        //boolQueryBuilder.must(QueryBuilders.matchQuery("sex", "男"));
//        //boolQueryBuilder.mustNot(QueryBuilders.matchQuery("sex", "男"));
//        boolQueryBuilder.should(QueryBuilders.matchQuery("age", 30));
//        boolQueryBuilder.should(QueryBuilders.matchQuery("age", 40));
//
//        builder.query(boolQueryBuilder);
//
//        request.source(builder);
//        SearchResponse response = esClient.search(request, RequestOptions.DEFAULT);
//
//        SearchHits hits = response.getHits();
//
//        System.out.println(hits.getTotalHits());
//        System.out.println(response.getTook());
//
//        for ( SearchHit hit : hits ) {
//            System.out.println(hit.getSourceAsString());
//        }

//        // 7. 范围查询
//        SearchRequest request = new SearchRequest();
//        request.indices("user");
//
//        SearchSourceBuilder builder = new SearchSourceBuilder();
//        RangeQueryBuilder rangeQuery = QueryBuilders.rangeQuery("age");
//
//        rangeQuery.gte(30);
//        rangeQuery.lt(50);
//
//        builder.query(rangeQuery);
//
//        request.source(builder);
//        SearchResponse response = esClient.search(request, RequestOptions.DEFAULT);
//
//        SearchHits hits = response.getHits();
//
//        System.out.println(hits.getTotalHits());
//        System.out.println(response.getTook());
//
//        for ( SearchHit hit : hits ) {
//            System.out.println(hit.getSourceAsString());
//        }

        // 8. 模糊查询
//        SearchRequest request = new SearchRequest();
//        request.indices("user");
//
//        SearchSourceBuilder builder = new SearchSourceBuilder();
//        builder.query(QueryBuilders.fuzzyQuery("name", "wangwu").fuzziness(Fuzziness.TWO));
//
//        request.source(builder);
//        SearchResponse response = esClient.search(request, RequestOptions.DEFAULT);
//
//        SearchHits hits = response.getHits();
//
//        System.out.println(hits.getTotalHits());
//        System.out.println(response.getTook());
//
//        for ( SearchHit hit : hits ) {
//            System.out.println(hit.getSourceAsString());
//        }

//        // 9. 高亮查询
//        SearchRequest request = new SearchRequest();
//        request.indices("user");
//
//        SearchSourceBuilder builder = new SearchSourceBuilder();
//        TermsQueryBuilder termsQueryBuilder = QueryBuilders.termsQuery("name", "zhangsan");
//
//        HighlightBuilder highlightBuilder = new HighlightBuilder();
//        highlightBuilder.preTags("<font color='red'>");
//        highlightBuilder.postTags("</font>");
//        highlightBuilder.field("name");
//
//        builder.highlighter(highlightBuilder);
//        builder.query(termsQueryBuilder);
//
//        request.source(builder);
//        SearchResponse response = esClient.search(request, RequestOptions.DEFAULT);
//
//        SearchHits hits = response.getHits();
//
//        System.out.println(hits.getTotalHits());
//        System.out.println(response.getTook());
//
//        for ( SearchHit hit : hits ) {
//            System.out.println(hit.getSourceAsString());
//        }

//        // 10. 聚合查询
//        SearchRequest request = new SearchRequest();
//        request.indices("user");
//
//        SearchSourceBuilder builder = new SearchSourceBuilder();
//
//        AggregationBuilder aggregationBuilder = AggregationBuilders.max("maxAge").field("age");
//        builder.aggregation(aggregationBuilder);
//
//        request.source(builder);
//        SearchResponse response = esClient.search(request, RequestOptions.DEFAULT);
//
//        SearchHits hits = response.getHits();
//
//        System.out.println(hits.getTotalHits());
//        System.out.println(response.getTook());
//
//        for ( SearchHit hit : hits ) {
//            System.out.println(hit.getSourceAsString());
//        }

        // 11. 分组查询
        SearchRequest request = new SearchRequest();
        request.indices("user");

        SearchSourceBuilder builder = new SearchSourceBuilder();

        AggregationBuilder aggregationBuilder = AggregationBuilders.terms("ageGroup").field("age");
        builder.aggregation(aggregationBuilder);

        request.source(builder);
        SearchResponse response = esClient.search(request, RequestOptions.DEFAULT);

        SearchHits hits = response.getHits();

        System.out.println(hits.getTotalHits());
        System.out.println(response.getTook());

        for ( SearchHit hit : hits ) {
            System.out.println(hit.getSourceAsString());
        }



        esClient.close();
    }
}
```

