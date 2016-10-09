---
layout: post
title:  "以Log4j切分Tomcat日志"
categories: Tomcat
description: 使用log4j切分tomcat日志，按时间或者按大小，Tomcat For Ubuntu16.04。
keywords: Tomcat, log4j, 日志, log, log4j切分tomcat日志,log4j切分log
---

# 一、Log4j.properties解析 #

```properties  
# 根日志的级别定义为 DEBUG，并将名为 CATALINA 的 appender 添加其上。
log4j.rootLogger = DEBUG, CATALINA

# 将名为 CATALINA 的 appender 设置为DailyRollingFileAppender。
log4j.appender.CATALINA=org.apache.Log4j.DailyRollingFileAppender

# 他将日志写入 log 目录下一个名为 log.out 的文件中。
log4j.appender.CATALINA.File=${log}/log.out

# 该标志位默认为 true，意味着每次日志追加操作都将输出流刷新至文件。
log4j.appender.CATALINA.ImmediateFlush=true

# 设置appender 对象的阀值。
log4j.appender.CATALINA.Threshold=debug

# 该值默认为 true，其含义是让日志追加至文件末尾。
log4j.appender.CATALINA.Append=true

# 设置回滚规则，在此我们设置为每分钟回滚方便测试。
log4j.appender.CATALINA.DatePattern='.' yyyy-MM-dd-HH-mm

# 设置 appender CATALINA 的 layout。
log4j.appender.CATALINA.layout=org.apache.Log4j.PatternLayout

# layout 被定义为 %m%n，即打印出来的日志信息末尾加入换行。
log4j.appender.CATALINA.layout.conversionPattern=%m%n

```

`conversionPattern`可以自定义log的格式，相关参数有：

- %p 输出优先级，即`DEBUG，INFO，WARN，ERROR，FATAL`。
- %r 输出自应用启动到输出该log信息耗费的毫秒数。
- %c 输出所属的类目，通常就是所在类的全名。
- %t 输出产生该日志事件的线程名。
- %m 输出代码中指定的消息。
- %n 输出一个回车换行符，Windows平台为“rn”，Unix平台为“n”。
- %d 输出日志时间点的日期或时间，默认格式为ISO8601，也可以在其后指定格式，比如：`%d{yyyy MMM dd HH:mm:ss,SSS}`，输出类似：2016年10月09日 ：15：41，628。
- %l 输出日志事件的发生位置，包括类目名、发生的线程，以及在代码中的行数。
- %L 输出当前记录器所在的文件行号，如输出: “51”。
- %F 输出当前记录器所在的文件名称，如输出: “main.cpp”。
- %x 输出和当前线程相关联的NDC(嵌套诊断环境),尤其用到像java servlets这样的多客户多线程的应用中。


# 二、使用Log4j接管catalina.out #

## 2.1 接管前置准备 ##

**1、**修改`$CATALINA_HOME/bin/`下的`catalina.sh`

```
# 1)将下条隐藏。
# touch "$CATALINA_OUT"

# 2)下条有两处需要隐藏。
#  >> "$CATALINA_OUT" 2>&1 "&"

# 3)下条有两处需要修改。
org.apache.catalina.startup.Bootstrap "$@" start \
# 修改为
org.apache.catalina.startup.Bootstrap "$@" start &\

```

![](http://i.imgur.com/syYTpXt.png)


**2、**重命名`$CATALINA_HOME/bin/`下的`logging.properties`成其他名字，该文件不需要了，建议重命名:

```
mv logging.properties loggin.properties
```

## 2.2 切分方式 ##

### 2.2.1 按时间切分 ###

如需按时段生成日志文件，需要使用`org.apache.Log4j.DailyRollingFileAppender`，该类继承了`FileAppender`类。该类多包涵了一个重要的属性：`DatePattern`,该属性表明什么时间回滚文件，以及文件的命名约定。缺省情况下，在每天午夜回滚文件。

`DatePattern`使用如下规则控制回滚计划：

- `yyyy-MM`				在本月末，下月初回滚文件。
- `yyyy-MM-dd	`		在每天午夜回滚文件，这是缺省值。
- `yyyy-MM-dd-a`			在每天中午和午夜回滚文件。
- `yyyy-MM-dd-HH`		在每个整点回滚文件。
- `yyyy-MM-dd-HH-mm`		每分钟回滚文件。
- `yyyy-ww`				根据地域，在每周的第一天回滚。

在项目中添加或修改`log4j.properties`文件:

```properties
log4j.rootLogger=INFO, CATALINA

log4j.appender.CATALINA=org.apache.log4j.DailyRollingFileAppender
log4j.appender.CATALINA.File=${catalina.base}/logs/catalina.
log4j.appender.CATALINA.Append=true
log4j.appender.CATALINA.Encoding=UTF-8
# 为方便测试，此处设置为每分钟分割一次
log4j.appender.CATALINA.DatePattern=yyyy-MM-dd-HH-mm'.log'
log4j.appender.CATALINA.layout = org.apache.log4j.PatternLayout
log4j.appender.CATALINA.layout.ConversionPattern = %d{yyyy-MM-dd HH:mm:ss} [%t] %-5p %c{1}:%L - %m%n

log4j.logger.org.springframework=DEBUG

# 以下按需要选择
log4j.appender.LOCALHOST=org.apache.log4j.DailyRollingFileAppender
log4j.appender.LOCALHOST.File=${catalina.base}/logs/localhost.
log4j.appender.LOCALHOST.Append=true
log4j.appender.LOCALHOST.Encoding=UTF-8
log4j.appender.LOCALHOST.DatePattern=yyyy-MM-dd-HH-mm'.log'
log4j.appender.LOCALHOST.layout = org.apache.log4j.PatternLayout
log4j.appender.LOCALHOST.layout.ConversionPattern = %d [%t] %-5p %c- %m%n

log4j.appender.MANAGER=org.apache.log4j.DailyRollingFileAppender
log4j.appender.MANAGER.File=${catalina.base}/logs/manager.
log4j.appender.MANAGER.Append=true
log4j.appender.MANAGER.Encoding=UTF-8
log4j.appender.MANAGER.DatePattern=yyyy-MM-dd-HH-mm'.log'
log4j.appender.MANAGER.layout = org.apache.log4j.PatternLayout
log4j.appender.MANAGER.layout.ConversionPattern = %d [%t] %-5p %c- %m%n

log4j.appender.HOST-MANAGER=org.apache.log4j.DailyRollingFileAppender
log4j.appender.HOST-MANAGER.File=${catalina.base}/logs/host-manager.
log4j.appender.HOST-MANAGER.Append=true
log4j.appender.HOST-MANAGER.Encoding=UTF-8
log4j.appender.HOST-MANAGER.DatePattern=yyyy-MM-dd-HH-mm'.log'
log4j.appender.HOST-MANAGER.layout = org.apache.log4j.PatternLayout
log4j.appender.HOST-MANAGER.layout.ConversionPattern = %d [%t] %-5p %c- %m%n

# 配置哪个loggers属于哪个appenders，按需选择配置
log4j.logger.org.apache.catalina.core.ContainerBase.[Catalina].[localhost]=INFO, LOCALHOST
log4j.logger.org.apache.catalina.core.ContainerBase.[Catalina].[localhost].[/manager]=\
  INFO, MANAGER
log4j.logger.org.apache.catalina.core.ContainerBase.[Catalina].[localhost].[/host-manager]=\
  INFO, HOST-MANAGER

```

### 2.2.2 按日志文件大小拆分 ###

如需按日志文件大小分割日志，则需要使用 org.apache.Log4j.RollingFileAppender，该类继承了 FileAppender 类，继承了它的所有属性。
该类多包括了以下可配置属性：

- `maxFileSize`：这是文件大小的阀值，大于该值时，文件会回滚。该值默认为 10 MB。
- `maxBackupIndex`：该值表示备份文件的个数，默认为 1。

对log4j.properties配置引入以下关键项：

```properties
# 设置日志文件阀值最大为5MB
Log4j.appender.CATALINA.MaxFileSize=5MB

# 设置日志文件个数为2
Log4j.appender.CATALINA.MaxBackupIndex=2
```

该示例配置文件展示了每个日志文件最大为 5 MB，如果超过该最大值，则会生成一个新的日志文件。由于 `maxBackupIndex` 的值为 2，当第二个文件的大小超过最大值时，会擦去第一个日志文件的内容，所有的日志都回滚至第一个日志文件。

# 参考 #
本文参考以下文章，在此对原作者表示感谢！

[Tomcat下使用Log4j 接管 catalina.out 日志文件生成方式](https://my.oschina.net/jsan/blog/205669)