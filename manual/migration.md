# Migrating from Log4j 1.x 从Log4j 1.x迁移Log4j 2

## 使用Logj 1.x桥接

最简单的方法就是把log4j 1.x这个jar包替换为`log4j-1.2-api.jar`。不过，必须满足以下条件：

1. 没有调用到Log4j 1.x实现的内部方法和类，例如`Appender`，`LoggerRepository`或`Category`的`callAppenders`方法。
2. 没有使用编程方式配置Log4j。
3. 没有通过调用类`DOMConfigurator`或`PropertyConfigurator`来配置。

## 转换为Log4j 2 API

在大多数情况下，从Log4j 1.x API迁移到Log4j 2是相当简单的。绝大多数日志语句不需要做任何修改。不过，有时候需要的话，做以下代码更改。

1. 1.x版本的总包名是`org.apache.log4j`，而版本2的总包名是`org.apache.logging.log4j`。
2. `org.apache.log4j.Logger.getLogger()`的调用必须转变为`org.apache.logging.log4j.LogManager.getLogger()`。
3. `org.apache.log4j.Logger.getRootLogger()`或`org.apache.log4j.LogManager.getRootLogger()`的调用必须转变为`org.apache.logging.log4j.LogManager.getRootLogger()`。
4. `org.apache.log4j.Logger.getLogger`使用了`LoggerFactory`，必须移除`org.apache.log4j.spi.LoggerFactory`，使用Log4j 2里的一个扩展机制。
5. `org.apache.log4j.Logger.getEffectiveLevel()`替换为`org.apache.logging.log4j.Logger.getLevel()`。
6. 移除`org.apache.log4j.LogManager.shutdown()`的调用，在版本2里不再需要，因为Log4j Core现在会在启动时自动添加JVM的shutdown hook，以执行Core的清除工作。
  1. 从Log4j 2.1开始，可以自定义[ShutdownCallbackRegistry](https://logging.apache.org/log4j/2.x/log4j-core/apidocs/org/apache/logging/log4j/core/util/ShutdownCallbackRegistry.html)的实现，来覆盖默认的JVM shutdown hook策略。
  2. 从Log4j 2.6开始，您可以调用`org.apache.logging.log4j.LogManager.shutdown()`来手动关闭。
7. 调用`org.apache.log4j.Logger.setLevel()`或相似的功能已不再支持。应用程序应该删除这些代码。在Log4j 2实现类中提供了等效功能，请参阅`org.apache.logging.log4j.core.config.Configurator.setLevel()`，但是可能会使应用程序对Log4j 2内部结构的更改变得很敏感。
8. 在适当情况下，应用程序应转换为使用参数化消息（parameterized messages），而不是字符串连接（String concatenation）。
9. `org.apache.log4j.MDC`和`org.apache.log4j.NDC`已经被[Thread Context](./thread-context.md)取代。

## 配置Log4j 2

虽然Log4j 2的配置语法跟Log4j 1.x是很大不同的，但绝大多数的功能是一致的。

注意，通过``${foo}``语法来获取系统属性值已经被扩展了，可以从许多不同源的属性里查找相应的值。有关更多详细信息，请参阅[Lookups](./lookups.md)章节。例如，使用对名为`catalina.base`的系统属性取值，在Log4j 1.x中，语法将是`${catalina.base}`。 在Log4j 2中，语法将是`${syscatalina.base}`。

以下是Log4j 1.x及其对应的Log4j 2配置示例。

### 示例1 - 使用Console Appender的简单配置

Log4j 1.x XML配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE log4j:configuration PUBLIC "-//APACHE//DTD LOG4J 1.2//EN" "log4j.dtd">
<log4j:configuration xmlns:log4j='http://jakarta.apache.org/log4j/'>
  <appender name="STDOUT" class="org.apache.log4j.ConsoleAppender">
    <layout class="org.apache.log4j.PatternLayout">
      <param name="ConversionPattern" value="%d %-5p [%t] %C{2} (%F:%L) - %m%n"/>
    </layout>
  </appender>
  <category name="org.apache.log4j.xml">
    <priority value="info" />
  </category>
  <Root>
    <priority value ="debug" />
    <appender-ref ref="STDOUT" />
  </Root>
</log4j:configuration>
```

Log2j 2 XML配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration>
  <Appenders>
    <Console name="STDOUT" target="SYSTEM_OUT">
      <PatternLayout pattern="%d %-5p [%t] %C{2} (%F:%L) - %m%n"/>
    </Console>
  </Appenders>
  <Loggers>
    <Logger name="org.apache.log4j.xml" level="info"/>
    <Root level="debug">
      <AppenderRef ref="STDOUT"/>
    </Root>
  </Loggers>
</Configuration>
```

### 示例2 - 使用File Appender的简单配置

Log4j 1.x XML配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE log4j:configuration PUBLIC "-//APACHE//DTD LOG4J 1.2//EN" "log4j.dtd">
<log4j:configuration xmlns:log4j="http://jakarta.apache.org/log4j/">
  <appender name="A1" class="org.apache.log4j.FileAppender">
    <param name="File"   value="A1.log" />
    <param name="Append" value="false" />
    <layout class="org.apache.log4j.PatternLayout">
      <param name="ConversionPattern" value="%t %-5p %c{2} - %m%n"/>
    </layout>
  </appender>
  <appender name="STDOUT" class="org.apache.log4j.ConsoleAppender">
    <layout class="org.apache.log4j.PatternLayout">
      <param name="ConversionPattern" value="%d %-5p [%t] %C{2} (%F:%L) - %m%n"/>
    </layout>
  </appender>
  <category name="org.apache.log4j.xml">
    <priority value="debug" />
    <appender-ref ref="A1" />
  </category>
  <root>
    <priority value ="debug" />
    <appender-ref ref="STDOUT" />
  </Root>
</log4j:configuration>
```

Log2j 2 XML配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration>
  <Appenders>
    <File name="A1" fileName="A1.log" append="false">
      <PatternLayout pattern="%t %-5p %c{2} - %m%n"/>
    </File>
    <Console name="STDOUT" target="SYSTEM_OUT">
      <PatternLayout pattern="%d %-5p [%t] %C{2} (%F:%L) - %m%n"/>
    </Console>
  </Appenders>
  <Loggers>
    <Logger name="org.apache.log4j.xml" level="debug">
      <AppenderRef ref="A1"/>
    </Logger>
    <Root level="debug">
      <AppenderRef ref="STDOUT"/>
    </Root>
  </Loggers>
</Configuration>
```

### 示例3 - SocketAppender

Log4j 1.x XML配置。这个例子在Log4j 1.x中配置是有问题的。SocketAppender实际上不使用Layout。这里配置的Layout，实际上是没有效果的。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE log4j:configuration PUBLIC "-//APACHE//DTD LOG4J 1.2//EN" "log4j.dtd">
<log4j:configuration xmlns:log4j="http://jakarta.apache.org/log4j/">
  <appender name="A1" class="org.apache.log4j.net.SocketAppender">
    <param name="RemoteHost" value="localhost"/>
    <param name="Port" value="5000"/>
    <param name="LocationInfo" value="true"/>
    <layout class="org.apache.log4j.PatternLayout">
      <param name="ConversionPattern" value="%t %-5p %c{2} - %m%n"/>
    </layout>
  </appender>
  <appender name="STDOUT" class="org.apache.log4j.ConsoleAppender">
    <layout class="org.apache.log4j.PatternLayout">
      <param name="ConversionPattern" value="%d %-5p [%t] %C{2} (%F:%L) - %m%n"/>
    </layout>
  </appender>
  <category name="org.apache.log4j.xml">
    <priority value="debug"/>
    <appender-ref ref="A1"/>
  </category>
  <root>
    <priority value="debug"/>
    <appender-ref ref="STDOUT"/>
  </Root>
</log4j:configuration>
```

Log2j 2 XML配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration>
  <Appenders>
    <Socket name="A1" host="localHost" port="5000">
      <SerializedLayout/>
    </Socket>
    <Console name="STDOUT" target="SYSTEM_OUT">
      <PatternLayout pattern="%d %-5p [%t] %C{2} (%F:%L) - %m%n"/>
    </Console>
  </Appenders>
  <Loggers>
    <Logger name="org.apache.log4j.xml" level="debug">
      <AppenderRef ref="A1"/>
    </Logger>
    <Root level="debug">
      <AppenderRef ref="STDOUT"/>
    </Root>
  </Loggers>
</Configuration>
```

### 示例4 - AsyncAppender

Log4j 1.x XML配置，使用AsyncAppender进行配置。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE log4j:configuration PUBLIC "-//APACHE//DTD LOG4J 1.2//EN" "log4j.dtd">
<log4j:configuration xmlns:log4j="http://jakarta.apache.org/log4j/" configDebug="true">
  <appender name="ASYNC" class="org.apache.log4j.AsyncAppender">
    <appender-ref ref="TEMP"/>
  </appender>
  <appender name="TEMP" class="org.apache.log4j.FileAppender">
    <param name="File" value="temp"/>
    <layout class="org.apache.log4j.PatternLayout">
      <param name="ConversionPattern" value="%d %-5p [%t] %C{2} (%F:%L) - %m%n"/>
    </layout>
  </appender>
  <root>
    <priority value="debug"/>
    <appender-ref ref="ASYNC"/>
  </Root>
</log4j:configuration>
```

Log2j 2 XML配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="debug">
  <Appenders>
    <File name="TEMP" fileName="temp">
      <PatternLayout pattern="%d %-5p [%t] %C{2} (%F:%L) - %m%n"/>
    </File>
    <Async name="ASYNC">
      <AppenderRef ref="TEMP"/>
    </Async>
  </Appenders>
  <Loggers>
    <Root level="debug">
      <AppenderRef ref="ASYNC"/>
    </Root>
  </Loggers>
</Configuration>
```

### 示例5 - AsyncAppender + Console + File

Log4j 1.x XML配置，使用AsyncAppender进行配置。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE log4j:configuration PUBLIC "-//APACHE//DTD LOG4J 1.2//EN" "log4j.dtd">
<log4j:configuration xmlns:log4j="http://jakarta.apache.org/log4j/" configDebug="true">
  <appender name="ASYNC" class="org.apache.log4j.AsyncAppender">
    <appender-ref ref="TEMP"/>
    <appender-ref ref="CONSOLE"/>
  </appender>
  <appender name="CONSOLE" class="org.apache.log4j.ConsoleAppender">
    <layout class="org.apache.log4j.PatternLayout">
      <param name="ConversionPattern" value="%d %-5p [%t] %C{2} (%F:%L) - %m%n"/>
    </layout>
  </appender>
  <appender name="TEMP" class="org.apache.log4j.FileAppender">
    <param name="File" value="temp"/>
    <layout class="org.apache.log4j.PatternLayout">
      <param name="ConversionPattern" value="%d %-5p [%t] %C{2} (%F:%L) - %m%n"/>
    </layout>
  </appender>
  <root>
    <priority value="debug"/>
    <appender-ref ref="ASYNC"/>
  </Root>
</log4j:configuration>
```

Log2j 2 XML配置。请注意，Async Appender应该在它引用的appender之后配置。这样才可以让它正常关闭。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="debug">
  <Appenders>
    <Console name="CONSOLE" target="SYSTEM_OUT">
      <PatternLayout pattern="%d %-5p [%t] %C{2} (%F:%L) - %m%n"/>
    </Console>
    <File name="TEMP" fileName="temp">
      <PatternLayout pattern="%d %-5p [%t] %C{2} (%F:%L) - %m%n"/>
    </File>
    <Async name="ASYNC">
      <AppenderRef ref="TEMP"/>
      <AppenderRef ref="CONSOLE"/>
    </Async>
  </Appenders>
  <Loggers>
    <Root level="debug">
      <AppenderRef ref="ASYNC"/>
    </Root>
  </Loggers>
</Configuration>
```
