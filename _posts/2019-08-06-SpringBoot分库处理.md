---
layout:     post                    # 使用的布局（不需要改）
title:      SpringBoot+MyBatis+Druid 分库处理             # 标题
subtitle:   使用AbstractRoutingDataSource进行分库操作 #副标题
date:       2019-08-06              # 时间
author:     songjunhao                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - SpringBoot
    - MyBatis
    - 数据库
    - 分库
---

### 基础原理

SpringBoot 针对多数据源提供了 *org.springframework.jdbc.datasource.lookup.AbstractRoutingDataSource* 类进行多数据源切换。

使用时只需 继承 *AbstractRoutingDataSource* 类，实现其 抽象方法 *determineCurrentLookupKey* ，此方法会在获取数据源时调用，返回要使用的数据源。

*determineCurrentLookupKey* 方法跟线程相关，每一个线程获取数据源时都会先调用此方法，所以使用 *ThreadLocal* 来保存各个线程当前 使用数据源，就不会导致数据源改变，影响其他线程。

### Maven 依赖

pom 依赖

```xml
<dependencies>
      <!-- MySQL 连接驱动依赖 -->
      <dependency>
          <groupId>mysql</groupId>
          <artifactId>mysql-connector-java</artifactId>
      </dependency>
      <!-- Druid 数据连接池依赖 -->
      <dependency>
          <groupId>com.alibaba</groupId>
          <artifactId>druid</artifactId>
          <version>1.1.5</version>
      </dependency>
      <dependency>
          <groupId>org.springframework.boot</groupId>
          <artifactId>spring-boot-configuration-processor</artifactId>
          <optional>true</optional>
      </dependency>
</dependencies>
```

### 代码实现
>ThreadLocal 可以保存当前线程的数据，线程与线程之间彼此独立。

首先创建一个 用于保存 线程当前 **数据源名字** 的 静态公共 ThreadLocal 处理类。

切换数据源时，只需要 调用 DataSourceContextHolder.setDB('a数据源') 即可切换。调用完毕后，再调用 DataSourceContextHolder.clearDB() 切换回默认数据源。

```java
public class DataSourceContextHolder {

    private static final ThreadLocal<String> contextHolder = new ThreadLocal<String>();

    // 设置数据源名
    public static void setDB(String dbType) {
        contextHolder.set(dbType);
    }

    // 获取数据源名
    public static String getDB() {
        return (contextHolder.get());
    }

    // 清除数据源名
    public static void clearDB() {
        contextHolder.remove();
    }
}
```

然后创建自定义路由动态数据源

继承 *AbstractRoutingDataSource* ，实现 *determineCurrentLookupKey* 方法。

```java
import org.springframework.jdbc.datasource.lookup.AbstractRoutingDataSource;

public class DynamicRoutingDataSource extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        return DataSourceContextHolder.getDB();
    }

}
```

细心地你肯定会发现，这个类并没有 @Bean 类型注解，不会注册到Spring容器中，那肯定还要继续配置。

既然是动态数据源，这个数据源里并没有保存任何数据源，所以我们需要用一个配置类，将数据源配置到 我们自定义的 ***DynamicRoutingDataSource*** 中。

数据源的配置，最好还是放到 配置文件中，我们先创建一下数据源 Properties类。

```java
public class DataSourceObject {

	private String name ;

	private String url ;

	private String username ;

	private String password ;

	private String driver;

	private String maxActive;

	private String minIdle;

	private String maxWait;

  ... 省略 get set

}
```

使用 SpringBoot 的 @ConfigurationProperties 加载 配置文件。
```java
import org.springframework.boot.context.properties.ConfigurationProperties;
import java.util.ArrayList;
import java.util.List;

@ConfigurationProperties(prefix = "dynamic-routing")
public class DataSourcesPropertiesConfig {

	private List<DataSourceObject>  connections = new ArrayList<>();

	public List<DataSourceObject> getConnections() {
		return connections;
	}

	public void setConnections(List<DataSourceObject> connections) {
		this.connections = connections;
	}

}
```

最后配置 DynamicRoutingDataSource 并设置 MyBatis 的 SqlSessionFactory 数据源为 DynamicRoutingDataSource。

```java
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Bean;
import javax.sql.DataSource;
import org.apache.ibatis.session.SqlSessionFactory;
import org.mybatis.spring.SqlSessionFactoryBean;
import org.mybatis.spring.SqlSessionTemplate;

@Configuration
@EnableConfigurationProperties(DataSourcesPropertiesConfig.class)
public class DataSourceConfig {

    @Autowired
    private DataSourcesPropertiesConfig connectionsConfig;

    @Bean
    public Map<Object, Object> createDataSources() {
        Map<Object,Object> dataSources = new HashMap<>();
        for(DataSourceObject connection:connectionsConfig.getConnections()) {
            Map<String,String> map = new HashMap<String,String>();
            map.put("url", connection.getUrl());
            map.put("username", connection.getUsername());
            map.put("password", connection.getPassword());
            map.put("maxActive", connection.getMaxActive());
            map.put("minIdle", connection.getMinIdle());
            map.put("maxWait", connection.getMaxWait());
            // Druid 监控配置 不使用监控 暂时注释掉
            //map.put("filters","stat,wall");
            //map.put("connectionProperties","druid.stat.mergeSql=true;druid.stat.slowSqlMillis=2000");
            //map.put("useGlobalDataSourceStat","true");
            DataSource dataSource;
            try {
                dataSource = DruidDataSourceFactory.createDataSource(map);
                dataSources.put(connection.getName(),dataSource);
            } catch (Exception e) {

            }
        }
        return dataSources;
    }

    @Bean
    public DynamicRoutingDataSource dynamicRoutingDataSource() {
        Map<Object,Object> map = createDataSources();
        DynamicRoutingDataSource dynamicRoutingDataSource =new DynamicRoutingDataSource();
        dynamicRoutingDataSource.setTargetDataSources(map);
        dynamicRoutingDataSource.setDefaultTargetDataSource(map.get("default"));
        return dynamicRoutingDataSource;
    }

    @Bean
    public SqlSessionFactory sqlSessionFactory() throws Exception {
    	DataSource dataSource = dynamicRoutingDataSource();
        SqlSessionFactoryBean factoryBean = new SqlSessionFactoryBean();
        factoryBean.setDataSource(dataSource);
        ResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
        factoryBean.setMapperLocations(resolver.getResources("classpath:mapper/*.xml"));
        return factoryBean.getObject();
    }

    @Bean
    public SqlSessionTemplate sqlSessionTemplate() throws Exception {
        SqlSessionTemplate template = new SqlSessionTemplate(sqlSessionFactory());
        return template;
    }

}
```

### 使用方法

在配置文件中添加 多数据源。
```yml
dynamic-routing:
  connections:
    - name: default
      url: jdbc:mysql://127.0.0.1:3306/picc_db?useSSL=false&&characterEncoding=utf8&&zeroDateTimeBehavior=convertToNull
      driver: com.mysql.jdbc.Driver
      username: 111
      password: 111
      maxActive: 500
      minIdle: 5
      maxWait: 60000
    - name: dataSource02
      url: jdbc:oracle:thin:@127.0.0.1:1521:uatnew
      driver: oracle.jdbc.driver.OracleDriver
      username: smisrelease
      password: Picc,Life#123
      maxActive: 50
      minIdle: 5
      maxWait: 60000
    - name: dataSource03
      url: jdbc:oracle:thin:@127.0.0.1:1521:smistest
      driver: oracle.jdbc.driver.OracleDriver
      username: piccsmis
      password: Passw0rd,123
      maxActive: 100
      minIdle: 5
      maxWait: 60000
```
使用时，只需在切换数据源之前 修改，切换之后再清空即可。
```java
@Service
public class QueryService {

  public void depotsQuery() {
      // 默认查询使用 默认数据源
      depotsMapper.query();

      // 切换第二个数据源
      DataSourceContextHolder.setDB("dataSource02");
      depotsMapper.query();
      // 清除数据源，还原为 默认数据源
      DataSourceContextHolder.clearDB();
  }

}
```
