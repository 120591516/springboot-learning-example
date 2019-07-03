摘要: 原创出处 https://www.bysocket.com 「公众号：泥瓦匠BYSocket 」欢迎关注和转载，保留摘要，谢谢！
摘要: 原创出处 www.bysocket.com 「泥瓦匠BYSocket 」欢迎转载，保留摘要，谢谢！
『 产品没有价值，开发团队再优秀也无济于事 – 《启示录》 』
本文提纲
一、缓存的应用场景
二、更新缓存的策略
三、运行 springboot-mybatis-redis 工程案例
四、springboot-mybatis-redis 工程代码配置详解
运行环境：
Mac OS 10.12.x
JDK 8 +
Redis 3.2.8
Spring Boot 1.5.1.RELEASE
一、缓存的应用场景
什么是缓存？
在互联网场景下，尤其 2C 端大流量场景下，需要将一些经常展现和不会频繁变更的数据，存放在存取速率更快的地方。缓存就是一个存储器，在技术选型中，常用 Redis 作为缓存数据库。缓存主要是在获取资源方便性能优化的关键方面。
Redis 是一个高性能的 key-value 数据库。GitHub 地址：https://github.com/antirez/redis 。Github 是这么描述的：
Redis is an in-memory database that persists on disk. The data model is key-value, but many different kind of values are supported: Strings, Lists, Sets, Sorted Sets, Hashes, HyperLogLogs, Bitmaps.
缓存的应用场景有哪些呢？
比如常见的电商场景，根据商品 ID 获取商品信息时，店铺信息和商品详情信息就可以缓存在 Redis，直接从 Redis 获取。减少了去数据库查询的次数。但会出现新的问题，就是如何对缓存进行更新？这就是下面要讲的。
二、更新缓存的策略
参考《缓存更新的套路》http://coolshell.cn/articles/17416.html，缓存更新的模式有四种：Cache aside, Read through, Write through, Write behind caching。
这里我们使用的是 Cache Aside 策略，从三个维度：（摘自 耗子叔叔博客）
失效：应用程序先从cache取数据，没有得到，则从数据库中取数据，成功后，放到缓存中。
命中：应用程序从cache中取数据，取到后返回。
更新：先把数据存到数据库中，成功后，再让缓存失效。
大致流程如下：
获取商品详情举例
a. 从商品 Cache 中获取商品详情，如果存在，则返回获取 Cache 数据返回。
b. 如果不存在，则从商品 DB 中获取。获取成功后，将数据存到 Cache 中。则下次获取商品详情，就可以从 Cache 就可以得到商品详情数据。
c. 从商品 DB 中更新或者删除商品详情成功后，则从缓存中删除对应商品的详情缓存

三、运行 springboot-mybatis-redis 工程案例
git clone 下载工程 springboot-learning-example ，项目地址见 GitHub – https://github.com/JeffLi1993/springboot-learning-example。
下面开始运行工程步骤（Quick Start）：
1.数据库和 Redis 准备
a.创建数据库 springbootdb：
CREATE DATABASE springbootdb;
b.创建表 city ：(因为我喜欢徒步)
DROP TABLE IF EXISTS  `city`;
CREATE TABLE `city` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT COMMENT '城市编号',
  `province_id` int(10) unsigned  NOT NULL COMMENT '省份编号',
  `city_name` varchar(25) DEFAULT NULL COMMENT '城市名称',
  `description` varchar(25) DEFAULT NULL COMMENT '描述',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;
c.插入数据
INSERT city VALUES (1 ,1,'温岭市','BYSocket 的家在温岭。');
d.本地安装 Redis
详见写过的文章《 Redis 安装 》http://www.bysocket.com/?p=917
2. springboot-mybatis-redis 工程项目结构介绍
springboot-mybatis-redis 工程项目结构如下图所示：
org.spring.springboot.controller - Controller 层
org.spring.springboot.dao - 数据操作层 DAO
org.spring.springboot.domain - 实体类
org.spring.springboot.service - 业务逻辑层
Application - 应用启动类
application.properties - 应用配置文件，应用启动会自动读取配置
3.改数据库配置
打开 application.properties 文件， 修改相应的数据源配置，比如数据源地址、账号、密码等。
（如果不是用 MySQL，自行添加连接驱动 pom，然后修改驱动名配置。）
4.编译工程
在项目根目录 springboot-learning-example，运行 maven 指令：
mvn clean install
5.运行工程
右键运行 springboot-mybatis-redis 工程 Application 应用启动类的 main 函数。
项目运行成功后，这是个 HTTP OVER JSON 服务项目。所以用 postman 工具可以如下操作
根据 ID，获取城市信息
GET http://127.0.0.1:8080/api/city/1
再请求一次，获取城市信息会发现数据获取的耗时快了很多。服务端 Console 输出的日志：
2017-04-13 18:29:00.273  INFO 13038 --- [nio-8080-exec-1] o.s.s.service.impl.CityServiceImpl       : CityServiceImpl.findCityById() : 城市插入缓存 >> City{id=12, provinceId=3, cityName='三亚', description='水好,天蓝'}
2017-04-13 18:29:03.145  INFO 13038 --- [nio-8080-exec-2] o.s.s.service.impl.CityServiceImpl       : CityServiceImpl.findCityById() : 从缓存中获取了城市 >> City{id=12, provinceId=3, cityName='三亚', description='水好,天蓝'}
可见，第一次是从数据库 DB 获取数据，并插入缓存，第二次直接从缓存中取。
更新城市信息
PUT http://127.0.0.1:8080/api/city
删除城市信息
DELETE http://127.0.0.1:8080/api/city/2
这两种操作中，如果缓存有对应的数据，则删除缓存。服务端 Console 输出的日志：
2017-04-13 18:29:52.248  INFO 13038 --- [nio-8080-exec-9] o.s.s.service.impl.CityServiceImpl       : CityServiceImpl.deleteCity() : 从缓存中删除城市 ID >> 12

四、springboot-mybatis-redis 工程代码配置详解
这里，我强烈推荐 注解 的方式实现对象的缓存。但是这里为了更好说明缓存更新策略。下面讲讲工程代码的实现。
pom.xml 依赖配置:
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>springboot</groupId>
    <artifactId>springboot-mybatis-redis</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>springboot-mybatis-redis :: 整合 Mybatis 并使用 Redis 作为缓存</name>

    <!-- Spring Boot 启动父依赖 -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.5.1.RELEASE</version>
    </parent>

    <properties>
        <mybatis-spring-boot>1.2.0</mybatis-spring-boot>
        <mysql-connector>5.1.39</mysql-connector>
        <spring-boot-starter-redis-version>1.3.2.RELEASE</spring-boot-starter-redis-version>
    </properties>

    <dependencies>

        <!-- Spring Boot Reids 依赖 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-redis</artifactId>
            <version>${spring-boot-starter-redis-version}</version>
        </dependency>

        <!-- Spring Boot Web 依赖 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- Spring Boot Test 依赖 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

        <!-- Spring Boot Mybatis 依赖 -->
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>${mybatis-spring-boot}</version>
        </dependency>

        <!-- MySQL 连接驱动依赖 -->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>${mysql-connector}</version>
        </dependency>

        <!-- Junit -->
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
        </dependency>
    </dependencies>
</project>

包括了 Spring Boot Reids 依赖、 MySQL 依赖和 Mybatis 依赖。
在 application.properties 应用配置文件，增加 Redis 相关配置
## 数据源配置
spring.datasource.url=jdbc:mysql://localhost:3306/springbootdb?useUnicode=true&characterEncoding=utf8
spring.datasource.username=root
spring.datasource.password=123456
spring.datasource.driver-class-name=com.mysql.jdbc.Driver

## Mybatis 配置
mybatis.typeAliasesPackage=org.spring.springboot.domain
mybatis.mapperLocations=classpath:mapper/*.xml

## Redis 配置
## Redis数据库索引（默认为0）
spring.redis.database=0
## Redis服务器地址
spring.redis.host=127.0.0.1
## Redis服务器连接端口
spring.redis.port=6379
## Redis服务器连接密码（默认为空）
spring.redis.password=
## 连接池最大连接数（使用负值表示没有限制）
spring.redis.pool.max-active=8
## 连接池最大阻塞等待时间（使用负值表示没有限制）
spring.redis.pool.max-wait=-1
## 连接池中的最大空闲连接
spring.redis.pool.max-idle=8
## 连接池中的最小空闲连接
spring.redis.pool.min-idle=0
## 连接超时时间（毫秒）
spring.redis.timeout=0
详细解释可以参考注释。对应的配置类：org.springframework.boot.autoconfigure.data.redis.RedisProperties
CityRestController 控制层依旧是 Restful 风格的，详情可以参考《Springboot 实现 Restful 服务，基于 HTTP / JSON 传输》。 http://www.bysocket.com/?p=1627 domain 对象 City 必须实现序列化，因为需要将对象序列化后存储到 Redis。如果没实现 Serializable ，控制台会爆出以下异常：
Serializable
java.lang.IllegalArgumentException: DefaultSerializer requires a Serializable payload but received an object of type
City.java 城市对象：
package org.spring.springboot.domain;

import java.io.Serializable;

/**
 * 城市实体类
 *
 * Created by bysocket on 07/02/2017.
 */
public class City implements Serializable {

    private static final long serialVersionUID = -1L;

    /**
     * 城市编号
     */
    private Long id;

    /**
     * 省份编号
     */
    private Long provinceId;

    /**
     * 城市名称
     */
    private String cityName;

    /**
     * 描述
     */
    private String description;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public Long getProvinceId() {
        return provinceId;
    }

    public void setProvinceId(Long provinceId) {
        this.provinceId = provinceId;
    }

    public String getCityName() {
        return cityName;
    }

    public void setCityName(String cityName) {
        this.cityName = cityName;
    }

    public String getDescription() {
        return description;
    }

    public void setDescription(String description) {
        this.description = description;
    }

    @Override
    public String toString() {
        return "City{" +
                "id=" + id +
                ", provinceId=" + provinceId +
                ", cityName='" + cityName + '\'' +
                ", description='" + description + '\'' +
                '}';
    }
}

如果需要自定义序列化实现，只要实现 RedisSerializer 接口去实现即可，然后在使用 RedisTemplate.setValueSerializer 方法去设置你实现的序列化实现。
主要还是城市业务逻辑实现类 CityServiceImpl.java：
package org.spring.springboot.service.impl;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.spring.springboot.dao.CityDao;
import org.spring.springboot.domain.City;
import org.spring.springboot.service.CityService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.core.ValueOperations;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.concurrent.TimeUnit;

/**
 * 城市业务逻辑实现类
 * <p>
 * Created by bysocket on 07/02/2017.
 */
@Service
public class CityServiceImpl implements CityService {

    private static final Logger LOGGER = LoggerFactory.getLogger(CityServiceImpl.class);

    @Autowired
    private CityDao cityDao;

    @Autowired
    private RedisTemplate redisTemplate;

    /**
     * 获取城市逻辑：
     * 如果缓存存在，从缓存中获取城市信息
     * 如果缓存不存在，从 DB 中获取城市信息，然后插入缓存
     */
    public City findCityById(Long id) {
        // 从缓存中获取城市信息
        String key = "city_" + id;
        ValueOperations<String, City> operations = redisTemplate.opsForValue();

        // 缓存存在
        boolean hasKey = redisTemplate.hasKey(key);
        if (hasKey) {
            City city = operations.get(key);

            LOGGER.info("CityServiceImpl.findCityById() : 从缓存中获取了城市 >> " + city.toString());
            return city;
        }

        // 从 DB 中获取城市信息
        City city = cityDao.findById(id);

        // 插入缓存
        operations.set(key, city, 10, TimeUnit.SECONDS);
        LOGGER.info("CityServiceImpl.findCityById() : 城市插入缓存 >> " + city.toString());

        return city;
    }

    @Override
    public Long saveCity(City city) {
        return cityDao.saveCity(city);
    }

    /**
     * 更新城市逻辑：
     * 如果缓存存在，删除
     * 如果缓存不存在，不操作
     */
    @Override
    public Long updateCity(City city) {
        Long ret = cityDao.updateCity(city);

        // 缓存存在，删除缓存
        String key = "city_" + city.getId();
        boolean hasKey = redisTemplate.hasKey(key);
        if (hasKey) {
            redisTemplate.delete(key);

            LOGGER.info("CityServiceImpl.updateCity() : 从缓存中删除城市 >> " + city.toString());
        }

        return ret;
    }

    @Override
    public Long deleteCity(Long id) {

        Long ret = cityDao.deleteCity(id);

        // 缓存存在，删除缓存
        String key = "city_" + id;
        boolean hasKey = redisTemplate.hasKey(key);
        if (hasKey) {
            redisTemplate.delete(key);

            LOGGER.info("CityServiceImpl.deleteCity() : 从缓存中删除城市 ID >> " + id);
        }
        return ret;
    }

}

首先这里注入了 RedisTemplate 对象。联想到 Spring 的 JdbcTemplate ，RedisTemplate 封装了 RedisConnection，具有连接管理，序列化和 Redis 操作等功能。还有针对 String 的支持对象 StringRedisTemplate。
Redis 操作视图接口类用的是 ValueOperations，对应的是 Redis String/Value 操作。还有其他的操作视图，ListOperations、SetOperations、ZSetOperations 和 HashOperations 。ValueOperations 插入缓存是可以设置失效时间，这里设置的失效时间是 10 s。
回到更新缓存的逻辑
a. findCityById 获取城市逻辑：
如果缓存存在，从缓存中获取城市信息
如果缓存不存在，从 DB 中获取城市信息，然后插入缓存
b. deleteCity 删除 / updateCity 更新城市逻辑：
如果缓存存在，删除
如果缓存不存在，不操作
其他不明白的，可以 git clone 下载工程 springboot-learning-example ，工程代码注解很详细。 https://github.com/JeffLi1993/springboot-learning-example。
五、小结
本文涉及到 Spring Boot 在使用 Redis 缓存时，一个是缓存对象需要序列化，二个是缓存更新策略是如何的。
欢迎扫一扫我的公众号关注 — 及时得到博客订阅哦！
— http://www.bysocket.com/ —
— https://github.com/JeffLi1993 —

代码示例
本文示例读者可以通过查看下面仓库的中代码 ：
GitHub（springboot-learning-example）
Gitee（springboot-learning-example）
如果您对这些感兴趣，欢迎 star、follow、收藏、转发给予支持！
以下专题教程也许您会有兴趣
《Spring Boot 2.x 系列教程》
《Java 核心系列教程》

 
（关注微信公众号，领取 Java 精选干货学习资料） 
（添加我微信：bysocket01。加入纯技术交流群，成长技术）