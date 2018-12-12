Spring Boot 整合 Elasticsearch，实现 function score query 权重分查询
阅读：15,561 次 Posted on 2017年5月19日
摘要: 原创出处 www.bysocket.com 「泥瓦匠BYSocket 」欢迎转载，保留摘要，谢谢！

『 预见未来最好的方式就是亲手创造未来 - 《史蒂夫·乔布斯传》 』

运行环境：JDK 7 或 8，Maven 3.0+
技术栈：SpringBoot 1.5+，ElasticSearch 2.3.2

本文提纲
一、ES 的使用场景
二、运行 springboot-elasticsearch 工程
三、springboot-elasticsearch 工程代码详解

一、ES 的使用场景
简单说，ElasticSearch（简称 ES）是搜索引擎，是结构化数据的分布式搜索引擎。在《Elasticsearch 和插件 elasticsearch-head 安装详解》  和 《Elasticsearch 默认配置 IK 及 Java AnalyzeRequestBuilder 使用》 我详细的介绍了如何安装，初步使用了 IK 分词器。这里，我主要讲下 SpringBoot 工程中如何使用 ElasticSearch。

ES 的使用场景大致分为两块
1. 全文检索。加上分词（IK 是其中一个）、拼音插件等可以成为强大的全文搜索引擎。
2. 日志统计分析。可以实时动态分析海量日志数据。

二、运行 springboot-elasticsearch 工程
注意的是这里使用的是 ElasticSearch 2.3.2。是因为版本对应关系 ：

Spring Boot Version (x) Spring Data Elasticsearch Version (y) Elasticsearch Version (z)
x <= 1.3.5 y <= 1.3.4 z <= 1.7.2* x >= 1.4.x 2.0.0 <=y < 5.0.0** 2.0.0 <= z < 5.0.0**
* - 只需要你修改下对应的 pom 文件版本号
** - 下一个 ES 的版本会有重大的更新
git clone 下载工程 springboot-elasticsearch ，项目地址见 GitHub - https://github.com/JeffLi1993/springboot-learning-example。

1. 后台起守护线程启动 Elasticsearch

cd elasticsearch-2.3.2/
./bin/elasticsearch -d
下面开始运行工程步骤（Quick Start）：

2. 项目结构介绍

org.spring.springboot.controller - Controller 层
org.spring.springboot.repository - ES 数据操作层
org.spring.springboot.domain - 实体类
org.spring.springboot.service - ES 业务逻辑层
Application - 应用启动类
application.properties - 应用配置文件，应用启动会自动读取配置
本地启动的 ES ，就不需要改配置文件了。如果连测试 ES 服务地址，需要修改相应配置

3.编译工程
在项目根目录 springboot-elasticsearch，运行 maven 指令：

mvn clean install
4.运行工程
右键运行 Application 应用启动类（位置：/springboot-learning-example/springboot-elasticsearch/src/main/java/org/spring/springboot/Application.java）的 main 函数，这样就成功启动了 springboot-elasticsearch 案例。

用 Postman 工具新增两个城市
新增城市信息

POST http://127.0.0.1:8080/api/city
{
	"id":"1",
	"provinceid":"1",
	"cityname":"温岭",
	"description":"温岭是个好城市"
}
 
POST http://127.0.0.1:8080/api/city
{
	"id":"2",
	"provinceid":"2",
	"cityname":"温州",
	"description":"温州是个热城市"
}
可以打开 ES 可视化工具 head 插件：http://localhost:9200/_plugin/head/：
（如果不知道怎么安装，请查阅 《Elasticsearch 和插件 elasticsearch-head 安装详解》 。）
在「数据浏览」tab，可以查阅到 ES 中数据是否被插入，插入后的数据格式如下：

{
    "_id": "1",
    "_index": "cityindex",
    "_score": 1,
    "_source": {
        "cityname": "温岭",
        "description": "温岭是个好城市",
        "id": 1,
        "provinceid": 1
    },
    "_type": "city",
    "_version": 1
}
下面验证下权重分查询搜索接口的实现：
GET http://localhost:8080/api/city/search?pageNumber=0&pageSize=10&searchContent=温岭
数据是会出现

[
    {
        "cityname": "温岭",
        "description": "温岭是个好城市",
        "id": 1,
        "provinceid": 1
    },
    {
        "cityname": "温州",
        "description": "温州是个热城市",
        "id": 2,
        "provinceid": 2
    }
]
从启动后台 Console 可以看出，打印出来对应的 DSL 语句：

{
    "function_score": {
        "functions": [
            {
                "filter": {
                    "bool": {
                        "should": {
                            "match": {
                                "cityname": {
                                    "query": "温岭",
                                    "type": "boolean"
                                }
                            }
                        }
                    }
                },
                "weight": 1000.0
            },
            {
                "filter": {
                    "bool": {
                        "should": {
                            "match": {
                                "description": {
                                    "query": "温岭",
                                    "type": "boolean"
                                }
                            }
                        }
                    }
                },
                "weight": 100.0
            }
        ]
    }
}
为什么会出现 温州 城市呢？因为 function score query 权重分查询，无相关的数据默认分值为 1。如果想除去，设置一个 setMinScore 分值即可。

三、springboot-elasticsearch 工程代码详解
具体代码见 GitHub - https://github.com/JeffLi1993/springboot-learning-example
1.pom.xml 依赖

<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
 
    <groupId>springboot</groupId>
    <artifactId>springboot-elasticsearch</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>springboot-elasticsearch :: 整合 Elasticsearch </name>
 
    <!-- Spring Boot 启动父依赖 -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.5.1.RELEASE</version>
    </parent>
 
    <dependencies>
 
        <!-- Spring Boot Elasticsearch 依赖 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
        </dependency>
 
        <!-- Spring Boot Web 依赖 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
 
        <!-- Junit -->
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
        </dependency>
    </dependencies>
</project>
这里依赖的 spring-boot-starter-data-elasticsearch 版本是 1.5.1.RELEASE，对应的 spring-data-elasticsearch 版本是 2.1.0.RELEASE。后面数据操作层都是通过该 spring-data-elasticsearch 提供的接口实现。

操作对应官方文档：http://docs.spring.io/spring-data/elasticsearch/docs/2.1.0.RELEASE/reference/html/。

2. application.properties 配置 ES 地址

# ES
spring.data.elasticsearch.repositories.enabled = true
spring.data.elasticsearch.cluster-nodes = 127.0.0.1:9300
默认 9300 是 Java 客户端的端口。9200 是支持 Restful HTTP 的接口。
更多配置：

spring.data.elasticsearch.cluster-name Elasticsearch 集群名。(默认值: elasticsearch)
spring.data.elasticsearch.cluster-nodes 集群节点地址列表，用逗号分隔。如果没有指定，就启动一个客户端节点。
spring.data.elasticsearch.propertie 用来配置客户端的额外属性。
spring.data.elasticsearch.repositories.enabled 开启 Elasticsearch 仓库。(默认值:true。)
3. ES 数据操作层
@Repository
public interface CityRepository extends ElasticsearchRepository<City,Long> {
 
 
}
接口只要继承 ElasticsearchRepository 类即可。默认会提供很多实现，比如 CRUD 和搜索相关的实现。

4. 实体类

@Document(indexName = "cityindex", type = "city")
public class City implements Serializable{
 
    private static final long serialVersionUID = -1L;
 
    /**
     * 城市编号
     */
    private Long id;
 
    /**
     * 省份编号
     */
    private Long provinceid;
 
    /**
     * 城市名称
     */
    private String cityname;
 
    /**
     * 描述
     */
    private String description;
}
注意
index 配置必须是全部小写，不然会暴异常。
org.elasticsearch.indices.InvalidIndexNameException: Invalid index name [cityIndex], must be lowercase

5. ES 业务逻辑层
/**
 * 城市 ES 业务逻辑实现类
 *
 * Created by bysocket on 07/02/2017.
 */
@Service
public class CityESServiceImpl implements CityService {
 
    private static final Logger LOGGER = LoggerFactory.getLogger(CityESServiceImpl.class);
 
    @Autowired
    CityRepository cityRepository;
 
    @Override
    public Long saveCity(City city) {
 
        City cityResult = cityRepository.save(city);
        return cityResult.getId();
    }
 
    @Override
    public List<City> searchCity(Integer pageNumber,
                                 Integer pageSize,
                                 String searchContent) {
        // 分页参数
        Pageable pageable = new PageRequest(pageNumber, pageSize);
 
        // Function Score Query
        FunctionScoreQueryBuilder functionScoreQueryBuilder = QueryBuilders.functionScoreQuery()
                .add(QueryBuilders.boolQuery().should(QueryBuilders.matchQuery("cityname", searchContent)),
                    ScoreFunctionBuilders.weightFactorFunction(1000))
                .add(QueryBuilders.boolQuery().should(QueryBuilders.matchQuery("description", searchContent)),
                        ScoreFunctionBuilders.weightFactorFunction(100));
 
        // 创建搜索 DSL 查询
        SearchQuery searchQuery = new NativeSearchQueryBuilder()
                .withPageable(pageable)
                .withQuery(functionScoreQueryBuilder).build();
 
        LOGGER.info("\n searchCity(): searchContent [" + searchContent + "] \n DSL  = \n " + searchQuery.getQuery().toString());
 
        Page<City> searchPageResults = cityRepository.search(searchQuery);
        return searchPageResults.getContent();
    }
 
}
保存逻辑很简单。

分页 function score query 搜索逻辑如下：

先创建分页参数，然后用 FunctionScoreQueryBuilder 定义 Function Score Query，并设置对应字段的权重分值。城市名称 1000 分，description 100 分。
然后创建该搜索的 DSL 查询，并打印出来。

四、小结
实际场景还会很复杂。这里只是点睛之笔，后续大家优化或者更改下 DSL 语句就可以完成自己想要的搜索规则。