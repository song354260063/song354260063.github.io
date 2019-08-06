---
layout:     post                    # 使用的布局（不需要改）
title:      SpringBoot日志优化             # 标题
subtitle:   按天按模块分割日志  #副标题
date:       2019-8-5              # 时间
author:     songjunhao                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - 日志
    - SpringBoot
    - logback
---


> SpringBoot 默认使用 slf4j 作为日志门面，logback 作为实际日志工具。

### 自定义配置文件

> 约定大于配置

**约定方式：**

在 resources 目录下 创建 logback.xml 或 logback-spring.xml，后者可使用Spring提供的一些功能，例如根据环境修改存储文件路径名称。

**配置方式：**

配置文件中指定

本文按照 logback-spring.xml 方式配置。

```
logging:
     config: classpath:my-logback-config.xml
```

### 自定义日志级别颜色配置

首先创建一个类继承 ForegroundCompositeConverterBase<ILoggingEvent> 代码如下：

```java
import ch.qos.logback.classic.Level;
import ch.qos.logback.classic.spi.ILoggingEvent;
import ch.qos.logback.core.pattern.color.ANSIConstants;
import ch.qos.logback.core.pattern.color.ForegroundCompositeConverterBase;

public class LogbackColorful extends ForegroundCompositeConverterBase<ILoggingEvent> {

    @Override
    protected String getForegroundColorCode(ILoggingEvent event) {
        Level level = event.getLevel();
        switch (level.toInt()) {
            //ERROR等级为红色
            case Level.ERROR_INT:
                return ANSIConstants.RED_FG;
            //WARN等级为黄色
            case Level.WARN_INT:
                return ANSIConstants.YELLOW_FG;
            //INFO等级为蓝色
            case Level.INFO_INT:
                return ANSIConstants.BLUE_FG;
            //DEBUG等级为绿色
            case Level.DEBUG_INT:
                return ANSIConstants.GREEN_FG;
            //其他为默认颜色
            default:
                return ANSIConstants.DEFAULT_FG;
        }
    }

}
```

然后在 logback-spring.xml 中 添加这个自定义 日志级别的颜色类。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <!--自定义颜色配置-->
   <conversionRule conversionWord="customcolor" converterClass="com.logback.config.LogbackColorful"/>
</configuration>
```

下面配置 控制台日志输出格式 并使用刚才定义的 颜色配置。

>使用自定义的配置文件，如果不配置，则使用IDE开发时，将会看不到日志。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <!--自定义颜色配置-->
   <conversionRule conversionWord="customcolor" converterClass="com.logback.config.LogbackColorful"/>

   <!-- %m输出的信息,%p日志级别,%t线程名,%d日期,%c类的全名,%i索引【从数字0开始递增】 -->
   <!-- appender是configuration的子节点，是负责写日志的组件。 -->
   <!-- ConsoleAppender：把日志输出到控制台 -->
   <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
       <encoder>
           <pattern>
               %magenta(%d{yyyy-MM-dd HH:mm:ss.SSS}) %customcolor(%-5level) %boldWhite([%-15.15thread]) %cyan(%-40.40logger{50}) : %msg%n
           </pattern>
           <!-- 控制台也要使用UTF-8，不要使用GBK，否则会中文乱码 -->
           <charset>UTF-8</charset>
       </encoder>
   </appender>

</configuration>
```

效果如图:

![](https://i.loli.net/2019/08/05/vnFBxtp97RUwhaq.jpg)

### 按天分割日志

按天分割日志需要再添加一个 专门用于 分割日志 的 appender 日志输出控制组件。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <!--自定义颜色配置-->
   <conversionRule conversionWord="customcolor" converterClass="com.logback.config.LogbackColorful"/>

   <!-- %m输出的信息,%p日志级别,%t线程名,%d日期,%c类的全名,%i索引【从数字0开始递增】 -->
   <!-- appender是configuration的子节点，是负责写日志的组件。 -->
   <!-- ConsoleAppender：把日志输出到控制台 -->
   <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
       <encoder>
           <pattern>
               %magenta(%d{yyyy-MM-dd HH:mm:ss.SSS}) %customcolor(%-5level) %boldWhite([%-15.15thread]) %cyan(%-40.40logger{50}) : %msg%n
           </pattern>
           <!-- 控制台也要使用UTF-8，不要使用GBK，否则会中文乱码 -->
           <charset>UTF-8</charset>
       </encoder>
   </appender>
   <!-- RollingFileAppender：滚动记录文件，先将日志记录到指定文件，当符合某个条件时，将日志记录到其他文件 -->
    <!-- 以下的大概意思是：1.先按日期存日志，日期变了，将前一天的日志文件名重命名为XXX%日期%索引，新的日志仍然是sys.log -->
    <!--             2.如果日期没有发生变化，但是当前日志的文件大小超过1KB时，对当前日志进行分割 重命名-->
    <appender name="syslog"
              class="ch.qos.logback.core.rolling.RollingFileAppender">
        <File>/logs/current/sys.log</File>
        <!-- rollingPolicy:当发生滚动时，决定 RollingFileAppender 的行为，涉及文件移动和重命名。 -->
        <!-- TimeBasedRollingPolicy： 最常用的滚动策略，它根据时间来制定滚动策略，既负责滚动也负责出发滚动 -->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!-- 活动文件的名字会根据fileNamePattern的值，每隔一段时间改变一次 -->
            <!-- 文件名：log/sys.2017-12-05.0.log -->
            <fileNamePattern>/logs/day/%d/sys.%d.%i.log</fileNamePattern>
            <!-- 每产生一个日志文件，该日志文件的保存期限为30天 -->
            <!--<maxHistory>360</maxHistory>-->
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <!-- maxFileSize:这是活动文件的大小，默认值是10MB,本篇设置为1KB，只是为了演示 -->
                <maxFileSize>200MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
        </rollingPolicy>
        <encoder>
            <!-- pattern节点，用来设置日志的输入格式 -->
            <pattern>
                %d{yyyy-MM-dd HH:mm:ss.SSS} %-5level [%-15.15thread] %-40.40logger{50} : %msg%n
            </pattern>
            <!-- 记录日志的编码 -->
            <charset>UTF-8</charset> <!-- 此处设置字符集 -->
        </encoder>
    </appender>

    <!-- 控制台输出日志级别 -->
    <root level="info">
        <appender-ref ref="STDOUT"/>
        <!-- 指定默认所有都走自定义的 -->
        <appender-ref ref="syslog"/>
    </root>

</configuration>
```

日志存储结果如图

![](https://i.loli.net/2019/08/05/h3Zo4FX9UP2CEIg.jpg)

![](https://i.loli.net/2019/08/05/an4tjHEDeABr7iy.jpg)

### 按模块分割日志

按模块分割日志，只需要和刚才类似，新添加一个 appender，name 重新定义一个，然后指定某些包或者某个包下的 类的日志 会使用此 appender 进行打印。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <!--自定义颜色配置-->
   <conversionRule conversionWord="customcolor" converterClass="com.logback.config.LogbackColorful"/>

   <!-- %m输出的信息,%p日志级别,%t线程名,%d日期,%c类的全名,%i索引【从数字0开始递增】 -->
   <!-- appender是configuration的子节点，是负责写日志的组件。 -->
   <!-- ConsoleAppender：把日志输出到控制台 -->
   <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
       <encoder>
           <pattern>
               %magenta(%d{yyyy-MM-dd HH:mm:ss.SSS}) %customcolor(%-5level) %boldWhite([%-15.15thread]) %cyan(%-40.40logger{50}) : %msg%n
           </pattern>
           <!-- 控制台也要使用UTF-8，不要使用GBK，否则会中文乱码 -->
           <charset>UTF-8</charset>
       </encoder>
   </appender>
   <!-- RollingFileAppender：滚动记录文件，先将日志记录到指定文件，当符合某个条件时，将日志记录到其他文件 -->
    <!-- 以下的大概意思是：1.先按日期存日志，日期变了，将前一天的日志文件名重命名为XXX%日期%索引，新的日志仍然是sys.log -->
    <!--             2.如果日期没有发生变化，但是当前日志的文件大小超过1KB时，对当前日志进行分割 重命名-->
    <appender name="syslog"
              class="ch.qos.logback.core.rolling.RollingFileAppender">
        <File>/logs/current/sys.log</File>
        <!-- rollingPolicy:当发生滚动时，决定 RollingFileAppender 的行为，涉及文件移动和重命名。 -->
        <!-- TimeBasedRollingPolicy： 最常用的滚动策略，它根据时间来制定滚动策略，既负责滚动也负责出发滚动 -->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!-- 活动文件的名字会根据fileNamePattern的值，每隔一段时间改变一次 -->
            <!-- 文件名：log/sys.2017-12-05.0.log -->
            <fileNamePattern>/logs/day/%d/sys.%d.%i.log</fileNamePattern>
            <!-- 每产生一个日志文件，该日志文件的保存期限为30天 -->
            <!--<maxHistory>360</maxHistory>-->
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <!-- maxFileSize:这是活动文件的大小，默认值是10MB,本篇设置为1KB，只是为了演示 -->
                <maxFileSize>200MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
        </rollingPolicy>
        <encoder>
            <!-- pattern节点，用来设置日志的输入格式 -->
            <pattern>
                %d{yyyy-MM-dd HH:mm:ss.SSS} %-5level [%-15.15thread] %-40.40logger{50} : %msg%n
            </pattern>
            <!-- 记录日志的编码 -->
            <charset>UTF-8</charset> <!-- 此处设置字符集 -->
        </encoder>
    </appender>

    <!-- com.abc 包日志单独输出 -->
    <appender name="abc" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>/logs/current/abc.log</file>
        <!-- rollingPolicy:当发生滚动时，决定 RollingFileAppender 的行为，涉及文件移动和重命名。 -->
        <!-- TimeBasedRollingPolicy： 最常用的滚动策略，它根据时间来制定滚动策略，既负责滚动也负责出发滚动 -->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!-- 活动文件的名字会根据fileNamePattern的值，每隔一段时间改变一次 -->
            <!-- 文件名：log/sys.2017-12-05.0.log -->
            <fileNamePattern>/logs/day/%d/abc.%d.%i.log</fileNamePattern>
            <!-- 每产生一个日志文件，该日志文件的保存期限为30天 -->
            <!--<maxHistory>360</maxHistory>-->
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <!-- maxFileSize:这是活动文件的大小，默认值是10MB,本篇设置为1KB，只是为了演示 -->
                <maxFileSize>200MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
        </rollingPolicy>
        <encoder>
            <!-- pattern节点，用来设置日志的输入格式 -->
            <pattern>
                %d{yyyy-MM-dd HH:mm:ss.SSS} %-5level [%-15.15thread] %-40.40logger{50} : %msg%n
            </pattern>
            <!-- 记录日志的编码 -->
            <charset>UTF-8</charset> <!-- 此处设置字符集 -->
        </encoder>
    </appender>

    <!-- 控制台输出日志级别 -->
    <root level="info">
        <appender-ref ref="STDOUT"/>
        <!-- 指定默认所有都走自定义的 -->
        <appender-ref ref="syslog"/>
    </root>

    <!-- com.abc 包日志 配置了 additivity = false 则不会进入 默认的 root 配置 -->
    <logger name="com.candidate" level="INFO" additivity="false">
        <!-- 指定控制台输出，否则控制台不会输出 -->
        <appender-ref ref="STDOUT"/>
        <!-- 指定新创建的 name = abc 的 appender  -->
        <appender-ref ref="abc" />
    </logger>

</configuration>
```

### 根据Spring环境动态配置

使用 logback-spring.xml 作为配置文件，可以使用 SpringBoot 提供的一些动态配置功能。下面介绍如何根据环境动态配置 日志文件存储路径。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <!-- 本地日志路径配置 -->
  <springProfile name="dev">
      <property name="dateFileName" value="D://logtest//%d//sys.%d.%i.log" />
      <property name="localFileName" value="D://logtest//sys.log" />
  </springProfile>

  <!-- 测试日志路径配置 -->
  <springProfile name="test">
      <property name="dateFileName" value="/logs/day/%d/sys/sys.%d.%i.log" />
      <property name="localFileName" value="/logs/current/sys.log" />
  </springProfile>

  <!-- 生产日志路径配置 -->
  <springProfile name="prod">
      <property name="dateFileName" value="/logs/day/%d/sys/sys.%d.%i.log" />
      <property name="localFileName" value="/logs/current/sys.log" />
  </springProfile>

  ... 省略
  <appender name="syslog"
            class="ch.qos.logback.core.rolling.RollingFileAppender">
      <!-- 使用 ${`name`} 获取上面配置的 property 的 value -->
      <File>${localFileName}</File>
      <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
          <!-- 使用 ${`name`} 获取上面配置的 property 的 value -->
          <fileNamePattern>${dateFileName}</fileNamePattern>
          <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
              <maxFileSize>200MB</maxFileSize>
          </timeBasedFileNamingAndTriggeringPolicy>
      </rollingPolicy>
      <encoder>
          <pattern>
              %d{yyyy-MM-dd HH:mm:ss.SSS} %-5level [%-15.15thread] %-40.40logger{50} : %msg%n
          </pattern>
          <charset>UTF-8</charset>
      </encoder>
  </appender>
  ... 省略

</configuration>
```
