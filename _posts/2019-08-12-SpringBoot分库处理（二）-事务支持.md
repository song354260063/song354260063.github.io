---
layout:     post                    # 使用的布局（不需要改）
title:      SpringBoot+MyBatis+Druid 分库处理（二）-事务支持             # 标题
subtitle:   解决多数据源事务问题！ #副标题
date:       2019-08-12              # 时间
author:     songjunhao                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - SpringBoot
    - MyBatis
    - 数据库
    - 分库
---

### 问题描述

当我们使用第一讲的分库方法进行分库后，如果在方法上加上 ***@Transactional***  注解使用 Spring 声明式 事务，会发现无法切换数据库，在当前方法中，只能使用一个数据源，虽然使用 DataSourceContextHolder.setDB("111"); 方法切换，但实际并没有效果。

### 问题原因分析

Spring集成MyBatis 默认事务管理器使用的是 *org.mybatis.spring.transaction.SpringManagedTransaction*，我们分析一下源码。具体见如下源码截图。

![](https://i.loli.net/2019/08/13/LY7fJgs2WSmokwu.jpg)

在默认事务处理器中的 getConnection 方法中，首先会判断 当前事务是否已经缓存了 Connection 连接，如果已经缓存了，那么就不会获取新的连接，所以虽然分库了，但是依然使用的是缓存中第一次获取的连接。导致分库失败。

### 解决方案

既然是默认事务处理器导致的问题，那就换一个事务处理器。自定义一个支持多连接的事务处理器即可。

Mybatis 提供了一个事务接口，类似 SpringManagedTransaction 的实现，我们只需要实现 org.apache.ibatis.transaction.Transaction 接口即可。

Transaction 接口中的方法描述：
+ 连接相关
 + getConnection() 方法，获得连接。
 + close() 方法，关闭连接。

+ 事务相关
 + commit() 方法，事务提交。
 + rollback() 方法，事务回滚。
 + getTimeout() 方法，事务超时时间。实际上，目前这个方法都是空实现。

### 代码实现

需要创建的类
+ MultiDataSourceTransaction 自定义事务管理器，保存多个连接，并对其 提交和回滚 进行处理。
+ MultiDataSourceTransactionFactory Spring-Mybatis 事务工厂，用于设置 Mybatis 所用 事务管理器。返回自定义的 MultiDataSourceTransaction。

```java
import org.apache.ibatis.transaction.Transaction;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.jdbc.CannotGetJdbcConnectionException;
import org.springframework.jdbc.datasource.DataSourceUtils;
import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.SQLException;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;

public class MultiDataSourceTransaction implements Transaction {

    private static final Logger LOGGER = LoggerFactory.getLogger(MultiDataSourceTransaction.class);

    private final DataSource dataSource;

    private Connection mainConnection;

    private String mainDatabaseIdentification;

    private ConcurrentMap<String, Connection> otherConnectionMap;

    private boolean isConnectionTransactional;

    private boolean autoCommit;

    public MultiDataSourceTransaction(DataSource dataSource) {
        this.dataSource = dataSource;
        otherConnectionMap = new ConcurrentHashMap<>();

        mainDatabaseIdentification = DataSourceContextHolder.getDB();
    }

    @Override
    public Connection getConnection() throws SQLException {
        String databaseIdentification = DataSourceContextHolder.getDB();
        if (databaseIdentification.equals(mainDatabaseIdentification)) {
            if (mainConnection != null) return mainConnection;
            else {
                openMainConnection();
                mainDatabaseIdentification =databaseIdentification;
                return mainConnection;
            }
        } else {
            if (!otherConnectionMap.containsKey(databaseIdentification)) {
                try {
                    Connection conn = dataSource.getConnection();
                    otherConnectionMap.put(databaseIdentification, conn);
                } catch (SQLException ex) {
                    throw new CannotGetJdbcConnectionException("Could not get JDBC Connection", ex);
                }
            }
            return otherConnectionMap.get(databaseIdentification);
        }

    }

    private void openMainConnection() throws SQLException {
        this.mainConnection = DataSourceUtils.getConnection(this.dataSource);
        this.autoCommit = this.mainConnection.getAutoCommit();
        LOGGER.info("this.autoCommit : {}",this.autoCommit);
        this.isConnectionTransactional = DataSourceUtils.isConnectionTransactional(this.mainConnection, this.dataSource);
        LOGGER.info(" this.isConnectionTransactional = {}",this.isConnectionTransactional);

        LOGGER.info(
                    "JDBC Connection ["
                            + this.mainConnection
                            + "] will"
                            + (this.isConnectionTransactional ? " " : " not ")
                            + "be managed by Spring");
    }

    @Override
    public void commit() throws SQLException {
        LOGGER.info("this.autoCommit : {}", this.autoCommit);
        LOGGER.info("this.mainConnection != null : {}", this.mainConnection != null);
        LOGGER.info("this.isConnectionTransactional : {}", this.isConnectionTransactional);
        if (this.mainConnection != null && !this.isConnectionTransactional && !this.autoCommit) {
            LOGGER.info("Committing JDBC Connection [" + this.mainConnection + "]");
            this.mainConnection.commit();
            for (Connection connection : otherConnectionMap.values()) {
                connection.commit();
            }
        }
    }

    @Override
    public void rollback() throws SQLException {
        if (this.mainConnection != null && !this.isConnectionTransactional && !this.autoCommit) {
            LOGGER.info("Rolling back JDBC Connection [" + this.mainConnection + "]");
            this.mainConnection.rollback();
            for (Connection connection : otherConnectionMap.values()) {
                connection.rollback();
            }
        }
    }

    @Override
    public void close() throws SQLException {
        DataSourceUtils.releaseConnection(this.mainConnection, this.dataSource);
        for (Connection connection : otherConnectionMap.values()) {
            DataSourceUtils.releaseConnection(connection, this.dataSource);
        }
    }

    @Override
    public Integer getTimeout() throws SQLException {
        return null;
    }

}
```


```java
import org.apache.ibatis.session.TransactionIsolationLevel;
import org.apache.ibatis.transaction.Transaction;
import org.mybatis.spring.transaction.SpringManagedTransactionFactory;

import javax.sql.DataSource;

public class MultiDataSourceTransactionFactory extends SpringManagedTransactionFactory {

    @Override
    public Transaction newTransaction(DataSource dataSource, TransactionIsolationLevel level, boolean autoCommit) {
        return new MultiDataSourceTransaction(dataSource);
    }

}
```

创建这两个类之后，需要设置 MyBatis 的事务处理器为 上述自定义的事务处理器。

```java
    ...省略其他代码
    @Bean
    public SqlSessionFactory sqlSessionFactory() throws Exception {
    	DataSource dataSource = dynamicRoutingConfig.dynamicRoutingDataSource();
        SqlSessionFactoryBean factoryBean = new SqlSessionFactoryBean();
        factoryBean.setDataSource(dataSource);
        // 设置自定义的事务处理器。解决分库切换及事务问题。
        factoryBean.setTransactionFactory(new MultiDataSourceTransactionFactory());
        ResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
        factoryBean.setMapperLocations(resolver.getResources("classpath:mapper/*.xml"));
        return factoryBean.getObject();
    }
    ...省略其他代码
```

>别忘了设置数据源的 defaultAutoCommit = false，否则事务会无效哦。

注意使用此方法不能使用 @Transaction 注解，使用Spring的事务注解，会使用Spring的事务管理器，而非MyBatis的事务管理器。

不使用 @Transaction 注解则，自动使用MyBatis的事务处理器。
