---
layout:     post                    # 使用的布局（不需要改）
title:      SpringBoot数据库分库处理             # 标题
subtitle:   使用AbstractRoutingDataSource进行分库操作 #副标题
date:       2019-08-06              # 时间
author:     songjunhao                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - SpringBoot
    - 数据库
    - 分库
---

### 基础原理

SpringBoot 针对多数据源提供了 *org.springframework.jdbc.datasource.lookup.AbstractRoutingDataSource* 类进行多数据源切换。

使用时只需 继承 *AbstractRoutingDataSource* 类，实现其 抽象方法 *determineCurrentLookupKey* ，此方法会在获取数据源时调用，返回要使用的数据源。

*determineCurrentLookupKey* 方法跟线程相关，每一个线程获取数据源时都会先调用此方法，所以使用 *ThreadLocal* 来保存各个线程当前 使用数据源，就不会导致数据源改变，影响其他线程。

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

```java
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Bean;
import javax.sql.DataSource;
import org.apache.ibatis.session.SqlSessionFactory;
import org.mybatis.spring.SqlSessionFactoryBean;
import org.mybatis.spring.SqlSessionTemplate;

@Configuration
public class DataSourceConfig {
    /**
     * 此方法配置 DynamicRoutingDataSource
     * @return 配置好 数据源名称 和 实际数据源的 自定义数据源
     */
    @Bean
    public DynamicRoutingDataSource dynamicRoutingDataSource() throws Exception {
        // 初始化两个 数据源的 配置 map 对象，分别为 dataSourceConfigMapOne 和 dataSourceConfigMapTwo
        Map<String,String> dataSourceConfigMapOne = new HashMap<String,String>();
        dataSourceConfigMapOne.put("url", url);
        dataSourceConfigMapOne.put("username", username);
        dataSourceConfigMapOne.put("password", password);

        Map<String,String> dataSourceConfigMapTwo = new HashMap<String,String>();
        dataSourceConfigMapTwo.put("url", url2);
        dataSourceConfigMapTwo.put("username", username2);
        dataSourceConfigMapTwo.put("password", password2);

        // dataSources 用于存储 自定义 dataSource 的 名称 和 实际数据源。
        Map<Object,Object> dataSources = new HashMap<>();
        // 创建数据源
        DataSource dataSourceOne = DruidDataSourceFactory.createDataSource(dataSourceConfigMapOne);
        DataSource dataSourceTwo = DruidDataSourceFactory.createDataSource(dataSourceConfigMapTwo);

        dataSources.put("default", dataSourceOne);
        dataSources.put("second", dataSourceTwo);

        DynamicRoutingDataSource dynamicRoutingDataSource =new DynamicRoutingDataSource();
        // 将实际数据源设置到 自定义的 DynamicRoutingDataSource 中。
        dynamicRoutingDataSource.setTargetDataSources(dataSources);
        // 设置默认数据源。
        dynamicRoutingDataSource.setDefaultTargetDataSource(dataSources.get("default"));
        return dynamicRoutingDataSource;
    }

    @Bean
    public SqlSessionFactory sqlSessionFactory() throws Exception {
    	DataSource dataSource = dynamicRoutingConfig.dynamicRoutingDataSource();
        SqlSessionFactoryBean factoryBean = new SqlSessionFactoryBean();
        factoryBean.setDataSource(dataSource);
        ResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
        factoryBean.setMapperLocations(resolver.getResources("classpath:mapper/*.xml"));
        return factoryBean.getObject();
    }

    @Bean
    public SqlSessionTemplate sqlSessionTemplate() throws Exception {
        return new SqlSessionTemplate(sqlSessionFactory());
    }
}
```

### 使用方法

使用时，只需在切换数据源之前 修改，切换之后再清空即可。

```java
@Service
public class QueryService {

  public void depotsQuery() {
      // 默认查询使用 默认数据源
      depotsMapper.query();

      // 切换第二个数据源
      DataSourceContextHolder.setDB("second");
      depotsMapper.query();
      // 清除数据源，还原为 默认数据源
      DataSourceContextHolder.clearDB();
  }

}
```
