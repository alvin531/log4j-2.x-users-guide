# Lookups - 查找

Lookups提供了一种可以在任意位置向Log4j配置添加值的方法。它们是实现[StrLookup](https://logging.apache.org/log4j/2.x/log4j-core/apidocs/org/apache/logging/log4j/core/lookup/StrLookup.html)接口的特定类型的插件。有关如何在配置文件中使用查找的信息，请参阅[Configuration](./configuration.md#PropertySubstitution)章节里的属性替换（Property Substitution）部分。

## Context Map查找

ContextMapLookup允许应用程序在Log4j的hreadContext Map中存储数据，然后Log4j配置中检索值。在下面的示例中，应用程序将使用键“loginId”把当前用户的登录ID存储在ThreadContext Map中。在初始配置处理时，第一个“$”将被删除。PatternLayout支持使用Lookups进行插值，然后解析每个事件的变量。请注意，模式“%X{loginId}”起相同的效果。

```xml
<File name="Application" fileName="application.log">
  <PatternLayout>
    <pattern>%d %p %c{1.} [%t] $${ctx:loginId} %m%n</pattern>
  </PatternLayout>
</File>
```

## Date查找

DateLookup与其他查找有些不同，因为它不使用键来定位值。相反，该键可用于指定[SimpleDateFormat](http://docs.oracle.com/javase/6/docs/api/java/text/SimpleDateFormat.html)的日期格式字符串。使用当前日期或与当前日志事件关联的日期按指定的格式输出。

```xml
<RollingFile name="Rolling-${map:type}" fileName="${filename}" filePattern="target/rolling1/test1-$${date:MM-dd-yyyy}.%i.log.gz">
  <PatternLayout>
    <pattern>%d %p %c{1.} [%t] %m%n</pattern>
  </PatternLayout>
  <SizeBasedTriggeringPolicy size="500" />
</RollingFile>
```

## 环境变量查找

EnvironmentLookup允许系统配置环境变量，或全局文件（如/etc/profile）或应用程序的启动脚本中的变量，然后从日志配置中检索这些变量。下面的示例包括应用程序日志中当前登录的用户名。

```xml
<File name="Application" fileName="application.log">
  <PatternLayout>
    <pattern>%d %p %c{1.} [%t] $${env:USER} %m%n</pattern>
  </PatternLayout>
</File>
```

## Java变量查找

JavaLookup允许使用`java:`前缀格式来检索Java环境信息。

| Key | 描述 |
| ---- | ---- |
| version | Java简短版本信息，如：<br> `Java version 1.7.0_67` |
| runtime | Java运行环境版本信息，如：<br> `Java(TM) SE Runtime Environment (build 1.7.0_67-b01) from Oracle Corporation` |
| vm | Java VM版本信息，如：<br> `Java HotSpot(TM) 64-Bit Server VM (build 24.65-b04, mixed mode)` |
| os | OS版本信息，如：<br> `Windows 7 6.1 Service Pack 1, architecture: amd64-64` |
| locale | 本地语言信息，如：<br> `default locale: en_US, platform encoding: Cp1252` |
| hw | 硬件信息，如：<br> `processors: 4, architecture: amd64-64, instruction sets: amd64` |

例如：

```xml
<File name="Application" fileName="application.log">
  <PatternLayout header="${java:runtime} - ${java:vm} - ${java:os}">
    <Pattern>%d %m%n</Pattern>
  </PatternLayout>
</File>
```

## Jndi查找

JndiLookup允许通过JNDI检索变量。默认情况下，键将以java:comp/env/作为前缀，但是如果键包含“:”，则不用添加前缀。

```xml
<File name="Application" fileName="application.log">
  <PatternLayout>
    <pattern>%d %p %c{1.} [%t] $${jndi:logging/context-name} %m%n</pattern>
  </PatternLayout>
</File>
```

**Java的JNDI模块在Android上不可用。**

## JVM输入参数查找（JMX）

映射JVM输入参数 - 但不是 _main_ 方法里的参数 - 而是使用JMX获取JVM参数。

使用前缀`jvmrunargs`来访问JVM参数。

请参阅[java.lang.management.RuntimeMXBean.getInputArguments()](http://docs.oracle.com/javase/8/docs/api/java/lang/management/RuntimeMXBean.html#getInputArguments--)的Javadocs。

**Java的JMX模块不适用于Android或Google App Engine。**

## Log4j配置位置查找

Log4j配置属性。表达式`${log4j:configLocation}`和`${log4j:configParentLocation}`分别提供了log4j配置文件及其父文件夹的绝对路径。

下面的示例使用此查找将日志文件放置在相对于log4j配置文件的目录中。

```xml
<File name="Application" fileName="${log4j:configParentLocation}/logs/application.log">
  <PatternLayout>
    <pattern>%d %p %c{1.} [%t] %m%n</pattern>
  </PatternLayout>
</File>
```

## Main方法参数查找

此查找要求您手动将应用程序的主要参数提供给Log4j：

```java
import org.apache.logging.log4j.core.lookup.MainMapLookup;

public static void main(String args[]) {
  MainMapLookup.setMainArguments(args);
  ...
}
```

如果已设置main参数，则此查找允许应用程序从日志配置中检索这些main参数值。`main:`前缀之后的键可以是参数列表中基于0的索引，也可以是一个字符串，其中`${main:myString}`使用main参数列表中`myString`后面的值替换。

例如，假设static void main String\[\] arguments是：

```text
--file foo.txt --verbose -x bar
```

然后，以下替换是可能的：

| 表达式 | 值 |
| ---- | ---- |
| ${main:0} | `--file` |
| ${main:1} | `foo.txt` |
| ${main:2} | `--verbose` |
| ${main:3} | `-x` |
| ${main:4} | `bar` |
| ${main:--file} | `foo.txt` |
| ${main:-x} | `bar` |
| ${main:bar} | `null` |

示例：

```xml
<File name="Application" fileName="application.log">
  <PatternLayout header="File: ${main:--file}">
    <Pattern>%d %m%n</Pattern>
  </PatternLayout>
</File>
```

## Map查找

MapLookup有几个用途。

1. 提供在配置文件中声明的基础属性。
2. 在LogEvents中从MapMessage检索值。
3. 检索使用[MapLookup.setMainArguments(String \[\])](https://logging.apache.org/log4j/2.x/log4j-core/apidocs/org/apache/logging/log4j/core/lookup/MapLookup.html#setMainArguments%28java.lang.String...%29)设置的值

第一个用途意味着MapLookup用于替换在配置文件中定义的属性。这些变量没有前缀，例如`${name}`。第二个用法是允许使用当前[MapMessage](https://logging.apache.org/log4j/2.x/log4j-api/apidocs/org/apache/logging/log4j/message/MapMessage.html)的值（如果是当前日志事件的一部分）替换。在下面的示例中，RoutingAppender使用MapMessage中名为“type”的键的值，每个不同的值表示使用不同的RollingFileAppender。注意，当以这种方式使用“type”的值时，应在属性声明中声明一个默认值，以便在消息并非MapMessage类型或MapMessage不包含该键的情况下。有关如何设置默认值的信息，请参阅[Configuration](./configuration.md#PropertySubstitution)章节里的属性替换（Property Substitution）部分。

```xml
<Routing name="Routing">
  <Routes pattern="$${map:type}">
    <Route>
      <RollingFile name="Rolling-${map:type}" fileName="${filename}"
                   filePattern="target/rolling1/test1-${map:type}.%i.log.gz">
        <PatternLayout>
          <pattern>%d %p %c{1.} [%t] %m%n</pattern>
        </PatternLayout>
        <SizeBasedTriggeringPolicy size="500" />
      </RollingFile>
    </Route>
  </Routes>
</Routing>
```

## Marker查找

标记（Marker）查找允许您在一些配置中使用Marker，如路由输出终端（routing appender）。查阅以下YAML配置，基于Marker记录到不同文件：

```yaml
Configuration:
  status: debug

  Appenders:
    Console:
    RandomAccessFile:
      - name: SQL_APPENDER
        fileName: logs/sql.log
        PatternLayout:
          Pattern: "%d{ISO8601_BASIC} %-5level %logger{1} %X %msg%n"
      - name: PAYLOAD_APPENDER
        fileName: logs/payload.log
        PatternLayout:
          Pattern: "%d{ISO8601_BASIC} %-5level %logger{1} %X %msg%n"
      - name: PERFORMANCE_APPENDER
        fileName: logs/performance.log
        PatternLayout:
          Pattern: "%d{ISO8601_BASIC} %-5level %logger{1} %X %msg%n"

    Routing:
      name: ROUTING_APPENDER
      Routes:
        pattern: "$${marker:}"
        Route:
        - key: PERFORMANCE
          ref: PERFORMANCE_APPENDER
        - key: PAYLOAD
          ref: PAYLOAD_APPENDER
        - key: SQL
          ref: SQL_APPENDER

  Loggers:
    Root:
      level: trace
      AppenderRef:
        - ref: ROUTING_APPENDER
```

```java
public static final Marker SQL = MarkerFactory.getMarker("SQL");
public static final Marker PAYLOAD = MarkerFactory.getMarker("PAYLOAD");
public static final Marker PERFORMANCE = MarkerFactory.getMarker("PERFORMANCE");

final Logger logger = LoggerFactory.getLogger(Logger.ROOT_LOGGER_NAME);

logger.info(SQL, "Message in Sql.log");
logger.info(PAYLOAD, "Message in Payload.log");
logger.info(PERFORMANCE, "Message in Performance.log");
```

注意配置的关键部分是`pattern: "$${marker:}"`。这将产生三个日志文件，每个日志文件都有特定marker的日志事件。Log4j将`SQL`标记的日志事件路由到`sql.log`，将带有`PAYLOAD`标记的日志事件路由到`payload.log`，等等。

您可以使用符号“`${marker:name}`”和“`$${marker:name}`”来检查marker是否是`name`这个标记名。如果标记存在，则表达式返回名称，否则返回`null`。

## 结构数据查找（Structured Data Lookup）

StructuredDataLookup非常类似于MapLookup，因为它将从StructuredDataMessages中检索值。除了Map值，它还将返回id（此id非应用程序里的id）和type的值。下面的示例和MapMessage的示例之间的主要区别是：“type”是[StructuredDataMessage](https://logging.apache.org/log4j/2.x/log4j-api/apidocs/org/apache/logging/log4j/message/StructuredDataMessage.html)的属性，而“type”必须是MapMessage中的Map中存在的项。

```xml
<Routing name="Routing">
  <Routes pattern="$${sd:type}">
    <Route>
      <RollingFile name="Rolling-${sd:type}" fileName="${filename}"
                   filePattern="target/rolling1/test1-${sd:type}.%i.log.gz">
        <PatternLayout>
          <pattern>%d %p %c{1.} [%t] %m%n</pattern>
        </PatternLayout>
        <SizeBasedTriggeringPolicy size="500" />
      </RollingFile>
    </Route>
  </Routes>
</Routing>
```

## 系统属性查找

由于通过使用系统属性定义应用程序内部和外部的值是相当常见的，自然地，在Log4j中也可以通过Lookup进行检索。由于系统属性通常在应用程序外部定义，所以常见的用法是：

```xml
<Appenders>
  <File name="ApplicationLog" fileName="${sys:logPath}/app.log"/>
</Appenders>
```

## Web查找

WebLookup允许应用程序检索在ServletContext中的变量。除了能够检索ServletContext中的各个变量，WebLookup支持查找作为属性存储或作为初始化参数的值。下表列出了可以检索的各种键：

| Key | 描述 |
| ---- | ---- |
| attr._name_ | 返回指定名称的ServletContext属性 |
| contextPath | web应用程序的context path |
| effectiveMajorVersion | 获取由ServletContext应用程序所基于的Servlet规范的主要版本。 |
| effectiveMinorVersion | 获取由ServletContext应用程序所基于的Servlet规范的次要版本。 |
| initParam._name_ | 返回指定名称的ServletContext初始化参数 |
| majorVersion | 返回此servlet容器支持的Servlet API的主要版本。 |
| minorVersion | 返回此servlet容器支持的Servlet API的次要版本。 |
| rootDir | 返回调用getRealPath方法的结果，值带“/”。 |
| serverInfo | 返回运行servlet的servlet容器的名称和版本。 |
| servletContextName | 返回在部署描述符的display-name元素中定义的Web应用程序的名称。 |

其他任何键首先在ServletContext属性检查是否存在该键名，然后在初始化参数检查是否存在该键名。如果键存在，则返回相应的值。

```xml
<Appenders>
  <File name="ApplicationLog" fileName="${web:rootDir}/app.log"/>
</Appenders>
```
