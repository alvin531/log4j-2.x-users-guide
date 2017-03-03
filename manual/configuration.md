# Configuration 配置

在应用程序中插入日志记录功能，需要很好的规划和大量的开发工作。按经验，大约4%的代码专用于日志记录。因此，即使是中等规模的应用程序，也会在代码中嵌入成千上万的日志语句。考虑到它们庞大的数量，无需手工修改就可以管理这些日志语句变得势在必行。

Log4j 2的配置可以通过以下4种方法中的任意1种完成：

1. 通过以XML、JSON、YAML或properties格式文件来配置。
2. 以编程方式，通过创建ConfigurationFactory和Configuration实现。
3. 以编程方式，通过调用Configuration接口的API将组件添加进来。
4. 以编程方式，通过调用内部Logger类上的方法。

本章节主要介绍通过配置文件配置Log4j。关于编程方式，可以在[Extending Log4j 2](./extending.md)和[Programmatic Log4j Configuration](./customconfig.md)中找到。

注意，与Log4j 1.x不同，Log4j 2 API并没有暴露add，modify或remove appender和filter或以任何方式操作配置的方法。

## 自动配置

Log4j在初始化间可以自动地进行配置。当Log4j启动时，它将定位所有ConfigurationFactory插件，并按优先顺序来发现配置文件。目前Log4j有4种ConfigurationFactory实现：一个用于JSON，一个用于YAML，一个用于properties，一个用于XML。

1. Log4j首先检查是否对`"log4j.configurationFile"`系统属性进行了设置，如果设置了，将尝试使用与文件扩展名匹配的`ConfigurationFactory`组件来加载配置。
2. 如果没有设置系统属性，ConfigurationFactory将在类路径中查找`log4j2-test.properties`文件。
3. 如果没有找到上面的文件，YAML ConfigurationFactory将在类路径中查找`log4j2-test.yaml`或`log4j2-test.yml`。
4. 如果没有找到上面的文件，JSON ConfigurationFactory将在类路径中查找`log4j2-test.json`或`log4j2-test.jsn`。
5. 如果没有找到上面的文件，XML ConfigurationFactory将在类路径中查找`log4j2-test.xml`。
6. 如果test的配置都没找到，则ConfigurationFactory将在类路径上查找`log4j2.properties`。
7. 如果没找到properties文件，YAML ConfigurationFactory将在类路径中查找`log4j2.yaml`或`log4j2.yml`。
8. 如果没找到YAML文件，JSON ConfigurationFactory将在类路径上查找`log4j2.json`或`log4j2.jsn`。
9. 如果找不到JSON文件，XML ConfigurationFactory将尝试在类路径上找到`log4j2.xml`。
10. 如果没有找到配置文件，将使用`DefaultConfiguration`。这将导致日志被输出到控制台。

下面的示例`MyApp`，展示整个配置流程。

```java
import com.foo.Bar;

// Import log4j classes.
import org.apache.logging.log4j.Logger;
import org.apache.logging.log4j.LogManager;

public class MyApp {

    // Define a static logger variable so that it references the
    // Logger instance named "MyApp".
    private static final Logger logger = LogManager.getLogger(MyApp.class);

    public static void main(final String... args) {

        // Set up a simple configuration that logs on the console.

        logger.trace("Entering application.");
        Bar bar = new Bar();
        if (!bar.doIt()) {
            logger.error("Didn't do it.");
        }
        logger.trace("Exiting application.");
    }
}
```

`MyApp`首先导入log4j相关的类。 然后它定义一个名为`MyApp`的静态logger变量，使用类的完全限定名。

`MyApp`调用`com.foo`包中的`Bar`类。

```java
package com.foo;
import org.apache.logging.log4j.Logger;
import org.apache.logging.log4j.LogManager;

public class Bar {
  static final Logger logger = LogManager.getLogger(Bar.class.getName());

  public boolean doIt() {
    logger.entry();
    logger.error("Did it again!");
    return logger.exit(false);
  }
}
```

如果Log4j找不到任何配置文件，它将使用默认配置。`DefaultConfiguration`类提供了默认配置：

* root logger使用[ConsoleAppender](https://logging.apache.org/log4j/2.x/log4j-core/apidocs/org/apache/logging/log4j/core/appender/ConsoleAppender.html)。
* ConsoleAppender使用"`%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n`"格式的[PatternLayout](https://logging.apache.org/log4j/2.x/log4j-core/apidocs/org/apache/logging/log4j/core/layout/PatternLayout.html)。

注意，默认情况下Log4j将root logger使用`Level.ERROR`。

MyApp的输出类似于：

```text
17:13:01.540 [main] ERROR com.foo.Bar - Did it again!
17:13:01.540 [main] ERROR MyApp - Didn\'t do it.
```

如前所述，Log4j会尝试从配置文件来进行配置。以下的配置等同于默认配置实现：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN">
  <Appenders>
    <Console name="Console" target="SYSTEM_OUT">
      <PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n"/>
    </Console>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="Console"/>
    </Root>
  </Loggers>
</Configuration>
```

把上面的xml文件命名为log4j2.xml，并放在类路径下，将得到输出结果和上面的示例结果是一致的。将root级别更改为trace，输出结果将会是：

```text
17:13:01.540 [main] TRACE MyApp - Entering application.
17:13:01.540 [main] TRACE com.foo.Bar - entry
17:13:01.540 [main] ERROR com.foo.Bar - Did it again!
17:13:01.540 [main] TRACE com.foo.Bar - exit with (false)
17:13:01.540 [main] ERROR MyApp - Didn\'t do it.
17:13:01.540 [main] TRACE MyApp - Exiting application.
```

请注意，使用默认配置时，将status是不生效的。

## 附加性（Additivity）

若想达到这样的效果：去除`com.foo.Bar`之外的所有TRACE输出。简单地更改上面xml配置里日志级别是达不到目的的。解决方案是向配置添加新的记录器定义：

```xml
<Logger name="com.foo.Bar" level="TRACE"/>
<Root level="ERROR">
  <AppenderRef ref="STDOUT">
</Root>
```

使用此配置，`com.foo.Bar`的所有日志事件都会被记录，而其他组件只会记录error级别的。

在上一个示例中，`com.foo.Bar`的所有日志是输出到控制台的。因为`com.foo.Bar`的logger没有配置任何appender，而其父类（root）配置了。而下面这个配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN">
  <Appenders>
    <Console name="Console" target="SYSTEM_OUT">
      <PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n"/>
    </Console>
  </Appenders>
  <Loggers>
    <Logger name="com.foo.Bar" level="trace">
      <AppenderRef ref="Console"/>
    </Logger>
    <Root level="error">
      <AppenderRef ref="Console"/>
    </Root>
  </Loggers>
</Configuration>
```

将输出：

```text
17:13:01.540 [main] TRACE com.foo.Bar - entry
17:13:01.540 [main] TRACE com.foo.Bar - entry
17:13:01.540 [main] ERROR com.foo.Bar - Did it again!
17:13:01.540 [main] TRACE com.foo.Bar - exit (false)
17:13:01.540 [main] TRACE com.foo.Bar - exit (false)
17:13:01.540 [main] ERROR MyApp - Didn\'t do it.
```

请留意，`com.foo.Bar`的trace消息会出现两次。这是因为首先使用与`com.foo.Bar`相关联的appender，它输出到Console。然后，使用`com.foo.Bar`的父对象(即root logger)，把消息事件传递给它的appender，它也写输出到Console，导致第2次输出。 这称为附加性（Additivity）。虽然附加性是一个很好用的功能（如前面第一个例子，可以不配置appender），但是大多数情景下这种行为并不是想要的，因此可以通过在logger上设置`additivity="false"`来屏蔽这个行为：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN">
  <Appenders>
    <Console name="Console" target="SYSTEM_OUT">
      <PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n"/>
    </Console>
  </Appenders>
  <Loggers>
    <Logger name="com.foo.Bar" level="trace" additivity="false">
      <AppenderRef ref="Console"/>
    </Logger>
    <Root level="error">
      <AppenderRef ref="Console"/>
    </Root>
  </Loggers>
</Configuration>
```

一旦logger的additivity设置为false，事件将不会被传递给它的任何父logger上（不管他们有没有对additivity进行设置）。

## 自动更新配置（Automatic Reconfiguration）

使用文件进行配置时，Log4j能够自动检测配置文件的更改并自动更新配置。若在Configuration节点上指定了`monitorInterval`属性并给它设了个非0值，那么在下次进行日志事件处理期间并且离上次检查后时间间隔已经超过了`monitorInterval`时，将重新检查该文件是否被更新了。下面的示例显示如何配置该属性，在至少30秒后检查配置文件的更改。最小间隔为5秒。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration monitorInterval="30">
...
</Configuration>
```

## 使用Chainsaw自动处理日志文件（配置appender的advertise）

对于所有的基于file的appender和基于socket的appender，log4j提供了通知appender配置的详细信息的功能。例如，基于file的appender，文件地址和格式化信息都包含在了log4j的通知里面。Chainsaw和其他外部系统能发现这些通知并聪明地利用这些通知去处理日志文件。

这种通知暴露的机制和通知格式是特定于每个通知者（Advertiser）实现的。想要使用这些特定于通知者（Advertiser）实现的外部系统必须清楚，如何定位通知的配置和通知的格式。例如，“database”通知者可以在数据库表中存储配置细节。外部系统可以读取该数据库表，以便定位文件位置和文件格式。

Log4j提供了一种通知者（Advertiser）实现：multicastdns，它通过使用[http://jmdns.sourceforge.net](http://jmdns.sourceforge.net/)库的IP多播组来通告appender配置的详细信息。

Chainsaw自动发现log4j通过`multicastdns`生成的通知，并在Chainsaw的Zeroconf选项卡中显示这些发现的通知（如果jmdns库在Chainsaw类路径中）。在Chainsaw的Zeroconf标签中双击通知入口，就可以解析和跟踪通知里的日志文件了。现在Chainsaw只支持FileAppender的通知。

通知一个输出appender配置，你需要如下步骤：

* 从[http://jmdns.sourceforge.net/](http://jmdns.sourceforge.net/)下载JmDns库，加入到应用的类路径中
* 设置configuration节点的"advertiser"属性值为"multicastdns"
* 设置appender节点的"advertise"属性值为"true"
* 如果通知基于FileAppender的配置，设置appender节点的"advertiseURI"属性值为一个适当的URI

基于FileAppender的配置需要在appender上指定一个额外的“advertiseURI”属性。“advertiseURI”属性向Chainsaw提供了有关如何访问文件的信息。例如，属性值是[Commons VFS](http://commons.apache.org/proper/commons-vfs/)的格式`sftp://URI`，就通过ssh/sftp远程访问文件；若使用`http://URI`，就可以通过Web服务器访问；使用`file://URI`，就从Chainsaw实例本地访问文件。

下面是一个示例，启用了通知机制，配置了从Chainsaw实例本地访问文件（注意advertiseURI属性值是file://）：

**请注意，为了用“multicastdns”来通知，你必须在你的应用路径加上Jmdns的库（你可以在[http://jmdns.sourceforge.net/](http://jmdns.sourceforge.net/)找到）**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration advertiser="multicastdns">
  ...
  <Appenders>
    <File name="File1" fileName="output.log" bufferedIO="false" advertiseURI="file://path/to/output.log" advertise="true">
    ...
    </File>
  </Appenders>
</Configuration>
```

## 配置语法

正如这些示例里表现的，Log4j可以让您轻松地更新日志记录行为，而无需更改应用程序。可以禁止应用程序的某些部分的日志记录，仅在满足特定条件（例如特定用户执行的动作）时记录日志，将日志输出到Flume或日志报告系统等。更好地使用配置，需清晰配置文件的语法。

XML配置文件中configuration节点有几个属性：

| 属性名 | 描述 |
| ---- | ---- |
| advertiser | （可选）通知者插件名称，用于通知各个FileAppender或SocketAppender配置。唯一提供的Advertiser插件是“multicastdns”。 |
| dest | 要么“err”，它将输出发送到stderr，或文件路径或URL。 |
| monitorInterval | 检查文件配置更改的最短时间周期（以秒为单位）。 |
| name | 该配置的名字 |
| packages | 以逗号分隔的包名称列表，用于搜索插件。每个类加载器只加载一次插件，因此更改此值可能对更新配置没有任何影响。 |
| schema | 标识类加载器定位XML模式的位置，用于验证配置。仅当strict设置为true时有效。如果未设置，将不会进行模式验证。 |
| shutdownHook | 指定当JVM关闭时Log4j是否应该自动关闭。shutdownHook默认情况下启用，但可以禁用，通过将此属性设置为“disable”。 |
| shutdownTimeout | 指定JVM关闭时执行关闭appender和后台任务的超时毫秒数。默认值为零，这意味着每个appender使用其默认超时，且不等待后台任务。并不是所有的appender都会遵守这点，因此，shutdown过程不会花很长时间，这只是一个提示，而不能绝对的保证。把值设置过低会导致更高的丢失尚未写入最终目的的日志事件的风险。请参阅[LoggerContext.stop(long, java.util.concurrent.TimeUnit)](https://logging.apache.org/log4j/2.x/log4j-core/apidocs/org/apache/logging/log4j/core/LoggerContext.html#stop%28long,%20java.util.concurrent.TimeUnit%29)。（如果shutdownHook设置为“disable”，则该配置不生效）。 |
| status | 输出到控制台的内部Log4j事件的级别。此属性的有效值为“trace”、“debug”、“info”、“warn”、“error”和“fatal”。Log4j会把有关初始化，rollover和其他内部操作的详细信息记录到status logger上。如果您需要对log4j进行故障排除，则可以设置`status="trace"`进行分析。 |
| strict | 启用严格的XML格式。JSON配置不支持。 |
| verbose | 在加载插件时开启输出诊断信息。 |

Log4j可以使用两种XML风格进行配置：简洁（concise）或严谨（strict）。简洁格式的配置更容易，因为使用节点名称匹配它们表示的组件，而不用进行XML模式验证。例如，ConsoleAppender通过在其父appenders节点下声明名为Console的XML节点来配置。 而且，节点和属性名称是不区分大小写的。此外，属性既可以指定为正常的XML属性，也可以指定为没有属性但有文本值的XML节点。如：

```xml
<PatternLayout pattern="%m%n"/>
```

和

```xml
<PatternLayout>
  <Pattern>%m%n</Pattern>
</PatternLayout>
```

这两个是等效的。

下面的文件表示XML配置的结构，但请注意，下面小写的节点表示将出现在其位置的简洁格式下的节点名称。

```xml
<?xml version="1.0" encoding="UTF-8"?>;
<Configuration>

  <Properties>
    <Property name="name1">value</property>
    <Property name="name2" value="value2"/>
  </Properties>

  <filter  ... />

  <Appenders>
    <appender ... >
      <filter  ... />
    </appender>
    ...
  </Appenders>

  <Loggers>
    <Logger name="name1">
      <filter  ... />
    </Logger>
    ...
    <Root level="level">
      <AppenderRef ref="name"/>
    </Root>
  </Loggers>

</Configuration>
```

请参见本章节上的appender、filter和logger的更多示例。

### 严谨模式（Strict XML）

除了上面简洁的XML格式，Log4j可以以更“正常”的XML方式指定配置，这可以使用XML模式进行验证。通过用如下所示的对象类型（type）替换上面的更友好节点名称来实现的。例如，ConsoleAppender配置不再使用名为Console的节点来进行配置，而是配置为type属性值为“Console”的appender节点。

```xml
<?xml version="1.0" encoding="UTF-8"?>;
<Configuration>

  <Properties>
    <Property name="name1">value</property>
    <Property name="name2" value="value2"/>
  </Properties>

  <Filter type="type" ... />

  <Appenders>
    <Appender type="type" name="name">
      <Filter type="type" ... />
    </Appender>
    ...
  </Appenders>

  <Loggers>
    <Logger name="name1">
      <Filter type="type" ... />
    </Logger>
    ...
    <Root level="level">
      <AppenderRef ref="name"/>
    </Root>
  </Loggers>

</Configuration>
```

下面是使用严谨模式的配置示例。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="debug" strict="true" name="XMLConfigTest"
               packages="org.apache.logging.log4j.test">

  <Properties>
    <Property name="filename">target/test.log</Property>
  </Properties>

  <Filter type="ThresholdFilter" level="trace"/>

  <Appenders>
    <Appender type="Console" name="STDOUT">
      <Layout type="PatternLayout" pattern="%m MDC%X%n"/>
      <Filters>
        <Filter type="MarkerFilter" marker="FLOW" onMatch="DENY" onMismatch="NEUTRAL"/>
        <Filter type="MarkerFilter" marker="EXCEPTION" onMatch="DENY" onMismatch="ACCEPT"/>
      </Filters>
    </Appender>
    <Appender type="Console" name="FLOW">
      <Layout type="PatternLayout" pattern="%C{1}.%M %m %ex%n"/><!-- class and line number -->
      <Filters>
        <Filter type="MarkerFilter" marker="FLOW" onMatch="ACCEPT" onMismatch="NEUTRAL"/>
        <Filter type="MarkerFilter" marker="EXCEPTION" onMatch="ACCEPT" onMismatch="DENY"/>
      </Filters>
    </Appender>
    <Appender type="File" name="File" fileName="${filename}">
      <Layout type="PatternLayout">
        <Pattern>%d %p %C{1.} [%t] %m%n</Pattern>
      </Layout>
    </Appender>
    <Appender type="List" name="List">
    </Appender>
  </Appenders>

  <Loggers>
    <Logger name="org.apache.logging.log4j.test1" level="debug" additivity="false">
      <Filter type="ThreadContextMapFilter">
        <KeyValuePair key="test" value="123"/>
      </Filter>
      <AppenderRef ref="STDOUT"/>
    </Logger>

    <Logger name="org.apache.logging.log4j.test2" level="debug" additivity="false">
      <AppenderRef ref="File"/>
    </Logger>

    <Root level="trace">
      <AppenderRef ref="List"/>
    </Root>
  </Loggers>

</Configuration>
```

### 使用JSON配置

除了XML外，Log4j可以使用JSON进行配置。JSON格式非常类似于简洁模式的XML。 每个key表示插件的名称，里面的key/value对是属性和属性值。如果一个key包含多个值，它本身就表示一个从属插件。在下面的示例中，ThresholdFilter，Console和PatternLayout都是插件，而Console插件的名称是STDOUT，且ThresholdFilter的日志级别是debug。

```json
{ "configuration": { "status": "error", "name": "RoutingTest",
                     "packages": "org.apache.logging.log4j.test",

      "properties": {
        "property": { "name": "filename",
                      "value" : "target/rolling1/rollingtest-$${sd:type}.log" }
      },

    "ThresholdFilter": { "level": "debug" },

    "appenders": {
      "Console": { "name": "STDOUT",
        "PatternLayout": { "pattern": "%m%n" }
      },
      "List": { "name": "List",
        "ThresholdFilter": { "level": "debug" }
      },
      "Routing": { "name": "Routing",
        "Routes": { "pattern": "$${sd:type}",
          "Route": [
            {
              "RollingFile": {
                "name": "Rolling-${sd:type}", "fileName": "${filename}",
                "filePattern": "target/rolling1/test1-${sd:type}.%i.log.gz",
                "PatternLayout": {"pattern": "%d %p %c{1.} [%t] %m%n"},
                "SizeBasedTriggeringPolicy": { "size": "500" }
              }
            },
            { "AppenderRef": "STDOUT", "key": "Audit"},
            { "AppenderRef": "List", "key": "Service"}
          ]
        }
      }
    },

    "loggers": {
      "logger": { "name": "EventLogger", "level": "info", "additivity": "false",
                  "AppenderRef": { "ref": "Routing" }},
      "root": { "level": "error", "AppenderRef": { "ref": "STDOUT" }}
    }

  }
}
```

请注意，在RoutingAppender中，Route元素已声明为数组。因为在这里每个数组元素都是一个Route组件。若appender和filter中，在简洁模式下每个节点元素有不同的名称，则不使用数组。如果每个appender或filter使用“type”属性来表示appender的类型，那么appender和filter可以定义为数组元素。以下示例展示了如何将appender声明为数组。

```json
{ "configuration": { "status": "debug", "name": "RoutingTest",
                      "packages": "org.apache.logging.log4j.test",

      "properties": {
        "property": { "name": "filename",
                      "value" : "target/rolling1/rollingtest-$${sd:type}.log" }
      },

    "ThresholdFilter": { "level": "debug" },

    "appenders": {
      "appender": [
         { "type": "Console", "name": "STDOUT", "PatternLayout": { "pattern": "%m%n" }},
         { "type": "List", "name": "List", "ThresholdFilter": { "level": "debug" }},
         { "type": "Routing",  "name": "Routing",
          "Routes": { "pattern": "$${sd:type}",
            "Route": [
              {
                "RollingFile": {
                  "name": "Rolling-${sd:type}", "fileName": "${filename}",
                  "filePattern": "target/rolling1/test1-${sd:type}.%i.log.gz",
                  "PatternLayout": {"pattern": "%d %p %c{1.} [%t] %m%n"},
                  "SizeBasedTriggeringPolicy": { "size": "500" }
                }
              },
              { "AppenderRef": "STDOUT", "key": "Audit"},
              { "AppenderRef": "List", "key": "Service"}
            ]
          }
        }
      ]
    },

    "loggers": {
      "logger": [
        { "name": "EventLogger", "level": "info", "additivity": "false",
          "AppenderRef": { "ref": "Routing" }},
        { "name": "com.foo.bar", "level": "error", "additivity": "false",
          "AppenderRef": { "ref": "Console" }}
      ],
      "root": { "level": "error", "AppenderRef": { "ref": "STDOUT" }}
    }
  }
}
```

JSON的支持是使用Jackson Data Processor来解析JSON文件的。这些依赖项必须添加到要使用JSON进行配置的项目中：

```xml
<dependency>
  <groupId>com.fasterxml.jackson.core</groupId>
  <artifactId>jackson-core</artifactId>
  <version>2.8.5</version>
</dependency>

<dependency>
  <groupId>com.fasterxml.jackson.core</groupId>
  <artifactId>jackson-databind</artifactId>
  <version>2.8.5</version>
</dependency>

<dependency>
  <groupId>com.fasterxml.jackson.core</groupId>
  <artifactId>jackson-annotations</artifactId>
  <version>2.8.5</version>
</dependency>
```

### 使用YAML配置

Log4j还支持使用YAML配置文件。YAML配置格式与XML配置格式类似，采用相同的结构模式。如：

```yaml
Configuration:
  status: warn
  name: YAMLConfigTest
  properties:
    property:
      name: filename
      value: target/test-yaml.log
  thresholdFilter:
    level: debug
  appenders:
    Console:
      name: STDOUT
      PatternLayout:
        Pattern: "%m%n"
    File:
      name: File
      fileName: ${filename}
      bufferedIO: false
      PatternLayout:
        Pattern: "%d %p %C{1.} [%t] %m%n"
    List:
      name: List
      Filters:
        ThresholdFilter:
          level: error

  Loggers:
    logger:
      -
        name: org.apache.logging.log4j.test1
        level: debug
        additivity: false
        ThreadContextMapFilter:
          KeyValuePair:
            key: test
            value: 123
        AppenderRef:
          ref: STDOUT
      -
        name: org.apache.logging.log4j.test2
        level: debug
        additivity: false
        AppenderRef:
          ref: File
    Root:
      level: error
      AppenderRef:
        ref: STDOUT
```

为了使用YAML配置文件，必须包括Jackson YAML库：

```xml
<dependency>
    <groupId>com.fasterxml.jackson.dataformat</groupId>
    <artifactId>jackson-dataformat-yaml</artifactId>
    <version>2.8.5</version>
</dependency>
```

### logger配置

在尝试配置之前，首先必须理解Log4j中Logger的工作原理，这点非常重要。请先熟悉Log4j[架构](./architecture.md)章节。没明白里面的那些概念而瞎目去配置，将困难重重。

使用`logger`节点来配置LoggerConfig。`logger`节点必须设置name属性，一般情况下还要设置level属性，并且还可以设置additivity属性。level属性可以设置为TRACE，DEBUG，INFO，WARN，ERROR，ALL或OFF中的其中一个。若没有指定level值，将默认使用ERROR。additivity属性值为true或false，默认值是false。

LoggerConfig（包括root LoggerConfig）可以配置properties，这些properties将添加到从ThreadContextMap里复制而来的properties中。配置的这些properties可以在Appender，Filters，Layouts等使用，就像它们是ThreadContext Map的一部分一样。properties可以是在解析配置时就可以解析出来的变量，或者是在处理每个事件时动态地解析出来的变量。有关变量的使用，请阅读下面的属性替换（Property Substitution）小节。

LoggerConfig可以配置一个或多个AppenderRef节点。每个引用的appender都与相应的LoggerConfig相关联。如果在LoggerConfig上配置多个appender，则在处理记录事件时逐个调用它们。

_**每个配置都必须有root logger。**_ 如果没有配置，则使用默认root LoggerConfig，它的级别是ERROR且使用Console appender。root logger和其他logger之间的主要区别是：

1. root logger没有name属性。
2. root logger不支持additivity属性，因为它没有父代。

### appender配置

配置appender有两种方式：一是直接使用特定的appender插件的名称作为节点，二是使用appender节点，且type属性值是该appender插件的名称。此外，每个appender必须给name属性赋值，而且该名称值在appender集合内必须唯一。logger将使用该名称来进行引用该appender。

大多数appender还支持要配置layout（配置layoutb也有两种方式：一是直接使用特定的layout插件的名称作为节点，二是使用layout节点，且type属性值是该layout插件的名称。）。appender还可能会需要其他的属性或节点配置，视具体而定。

### filter配置

Log4j里可以在4处地方配置filter：

1. 跟appenders、loggers和properties节点同层上配置。这些filters可以在events传递到LoggerConfig之前进行过滤拦截。
2. 在logger节点中。这些filters针对特定的logger进行events的过滤。
3. 在appender节点中。这些filters可以阻止或产生由appender处理的events。
4. 在appender引用节点中。这些filters用于确定logger是否应将events路由到appender上。

虽然每处地方只能配置一个`filter`节点，但可以使用`filters`节点来包括该节点，这个`filters`节点实际上是`CompositeFilter`。这个`filters`节点就可以达到配置多个`filter`节点的目的。以下示例显示如何在ConsoleAppender上配置多个filters。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="debug" name="XMLConfigTest" packages="org.apache.logging.log4j.test">

  <Properties>
    <Property name="filename">target/test.log</Property>
  </Properties>

  <ThresholdFilter level="trace"/>

  <Appenders>
    <Console name="STDOUT">
      <PatternLayout pattern="%m MDC%X%n"/>
    </Console>
    <Console name="FLOW">
      <!-- this pattern outputs class name and line number -->
      <PatternLayout pattern="%C{1}.%M %m %ex%n"/>
      <filters>
        <MarkerFilter marker="FLOW" onMatch="ACCEPT" onMismatch="NEUTRAL"/>
        <MarkerFilter marker="EXCEPTION" onMatch="ACCEPT" onMismatch="DENY"/>
      </filters>
    </Console>
    <File name="File" fileName="${filename}">
      <PatternLayout>
        <pattern>%d %p %C{1.} [%t] %m%n</pattern>
      </PatternLayout>
    </File>
    <List name="List">
    </List>
  </Appenders>

  <Loggers>
    <Logger name="org.apache.logging.log4j.test1" level="debug" additivity="false">
      <ThreadContextMapFilter>
        <KeyValuePair key="test" value="123"/>
      </ThreadContextMapFilter>
      <AppenderRef ref="STDOUT"/>
    </Logger>

    <Logger name="org.apache.logging.log4j.test2" level="debug" additivity="false">
      <Property name="user">${sys:user.name}</Property>
      <AppenderRef ref="File">
        <ThreadContextMapFilter>
          <KeyValuePair key="test" value="123"/>
        </ThreadContextMapFilter>
      </AppenderRef>
      <AppenderRef ref="STDOUT" level="error"/>
    </Logger>

    <Root level="trace">
      <AppenderRef ref="List"/>
    </Root>
  </Loggers>

</Configuration>
```

### 使用properties配置

从2.4版本开始，Log4j支持通过properties文件进行配置。请注意，properties语法与Log4j 1中使用的语法不同。与XML和JSON配置一样，properties配置根据插件和插件的属性来定义配置。

在2.6版之前，properties配置要求列出appenders、filters和loggers的标识符列表（identifiers），这些标识符用逗号进行分隔。这些组件都以 _component.<.identifier>._ 的格式在properties文件里进行定义。该identifier不必与组件的名称匹配，但必须唯一地标识出该组件的所有属性和子组件。如果没有列出标识符列表（identifiers），identifier不能使用“.”。此时每个组件必须具有指定“type”属性，用于标识组件的插件类型。

从版本2.6起，不再需要定义标识符列表（identifiers），因为在首次使用标识时会进行名称的推断，但是如果您希望使用更复杂的标识，则必须仍然使用该列表。如果列表存在，它就会被使用。

与基本组件不同，创建子组件时，不能给节点指定标识符列表。而是您必须使用type属性来包装该节点（也就是说，type之前的identifier一开始log4j是不认识的，但可以通过type属性来告知log4j如何处理），如下面rolling file appender中policies的定义。然后，定义该包装器节点下面的每个子组件，如下面定义的TimeBasedTriggeringPolicy和SizeBasedTriggeringPolicy。

properties配置文件支持advertiser、monitorInterval、name、packages、shutdownHook、shutdownTimeout、status、verbose和dest等属性。有关这些属性的定义，请参阅上面的配置语法。

```properties
status = error
dest = err
name = PropertiesConfig

property.filename = target/rolling/rollingtest.log

filter.threshold.type = ThresholdFilter
filter.threshold.level = debug

appender.console.type = Console
appender.console.name = STDOUT
appender.console.layout.type = PatternLayout
appender.console.layout.pattern = %m%n

appender.rolling.type = RollingFile
appender.rolling.name = RollingFile
appender.rolling.fileName = ${filename}
appender.rolling.filePattern = target/rolling2/test1-%d{MM-dd-yy-HH-mm-ss}-%i.log.gz
appender.rolling.layout.type = PatternLayout
appender.rolling.layout.pattern = %d %p %C{1.} [%t] %m%n
appender.rolling.policies.type = Policies
appender.rolling.policies.time.type = TimeBasedTriggeringPolicy
appender.rolling.policies.time.interval = 2
appender.rolling.policies.time.modulate = true
appender.rolling.policies.size.type = SizeBasedTriggeringPolicy
appender.rolling.policies.size.size=100MB
appender.rolling.strategy.type = DefaultRolloverStrategy
appender.rolling.strategy.max = 5

appender.list.type = List
appender.list.name = List
appender.list.filter.threshold.type = ThresholdFilter
appender.list.filter.threshold.level = error

logger.rolling.name = com.example.my.app
logger.rolling.level = debug
logger.rolling.additivity = false
logger.rolling.appenderRef.rolling.ref = RollingFile

rootLogger.level = info
rootLogger.appenderRef.stdout.ref = STDOUT
```

## 属性替换（Property Substitution）

Log4j 2支持在配置中使用特殊标记来引用其他地方定义的属性值。这些属性中有的可以在解释配置文件时解析，而有的属性可能会在运行时进行解析处理。为了实现这一点，Log4j使用类似于[Apache Commons Lang](https://commons.apache.org/proper/commons-lang/)的[StrSubstitutor](https://logging.apache.org/log4j/2.x/log4j-core/apidocs/org/apache/logging/log4j/core/lookup/StrSubstitutor.html)和[StrLookup](https://logging.apache.org/log4j/2.x/log4j-core/apidocs/org/apache/logging/log4j/core/lookup/StrLookup.html)类的实现。以类似于Ant或Maven的方式，这允许使用在配置本身中声明的属性来解析`${name}`格式的变量。例如，以下示例显示rolling file appender使用了flename这个属性。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="debug" name="RoutingTest" packages="org.apache.logging.log4j.test">

  <Properties>
    <Property name="filename">target/rolling1/rollingtest-$${sd:type}.log</Property>
  </Properties>

  <ThresholdFilter level="debug"/>

  <Appenders>
    <Console name="STDOUT">
      <PatternLayout pattern="%m%n"/>
    </Console>
    <List name="List">
      <ThresholdFilter level="debug"/>
    </List>
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
        <Route ref="STDOUT" key="Audit"/>
        <Route ref="List" key="Service"/>
      </Routes>
    </Routing>
  </Appenders>

  <Loggers>
    <Logger name="EventLogger" level="info" additivity="false">
      <AppenderRef ref="Routing"/>
    </Logger>

    <Root level="error">
      <AppenderRef ref="STDOUT"/>
    </Root>
  </Loggers>

</Configuration>
```

这种用法很有用，不过还可以更强大的指明属性值的查找上下文。Log4j还支持语法`${prefix:name}`，其中前缀标识（prefix）告诉Log4j应该在特定上下文中解析变量名。有关更多详细信息，请参阅[Lookups](./lookups.md)章节。Logj4中内置的上下文有：

| Prefix | Context |
| ---- | ---- |
| bundle | 资源绑定。格式为`${bundle:BundleName:BundleKey}`。BundleName使用包命名规则，例如：`${bundle:com.domain.Messages:MyKey}`。 |
| ctx | Thread Context Map (MDC) |
| date | 使用指定的格式插入当前日期和/或时间 |
| env | 系统环境变量 |
| jndi | 从JNDI上下文里获取值 |
| jvmrunargs | 通过JMX访问的JVM参数，但不是main参数；请参阅[RuntimeMXBean.getInputArguments()](http://docs.oracle.com/javase/6/docs/api/java/lang/management/RuntimeMXBean.html#getInputArguments--)。Android中无法使用。  |
| log4j | Log4j配置属性。 表达式`${log4j:configLocation}`和`${log4j:configParentLocation}`分别提供了log4j配置文件及其父文件夹的绝对路径。 |
| main | 从[MapLookup.setMainArguments(String\[\])](https://logging.apache.org/log4j/2.x/log4j-core/apidocs/org/apache/logging/log4j/core/lookup/MapLookup.html#setMainArguments-java.lang.String:A-)上获取值 |
| map | 从MapMessage上获取值 |
| sd | 从StructuredDataMessage上获取值。“id”的key将返回StructuredDataId的名称而非应用程序里的id。“type”的key将返回消息类型。其他key将从Map查找。 |
| sys | 系统属性 |

可以在配置文件中声明默认属性映射。如果属性值在指定的上下文里打不到，则将使用默认属性映射中的值。默认映射预先生成“hostName”属性（该值使用当前系统的主机名或IP地址），以及“contextName”属性（该值为当前日志记录上下文）。

## 多个前置'$'的变量查找

StrLookup处理的一个有趣的特性是，每当变量被解析时，一个变量引用在声明时使用了多个前置'$'，最前面的那个'$'被简单粗暴地删除。在前面的例子中，“Routes”元素能够在运行时进行变量解析。为了让其在运行时解析，前缀值使用了两个前置'$'字符的变量。当在解析配置文件时，第一个'$'被简单粗暴地删除。因此，当在运行时处理Routes元素时，它是变量声明就变成了“${sd:type}”，这就表示将使用StructuredDataMessage进行解析，并使用type作为key进行值查找。并非所有节点都支持在运行时解析变量。具体能否支持就要看组件自身的文档说明了。

如果在与前缀相关的上下文里找不到该key的值，则将在配置文件中的属性声明中查找该值。如果还是没有找到值，则将变量名作为值返回。在配置中声明默认值可以这样做：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration>
  <Properties>
    <Property name="type">Audit</property>
  </Properties>
  ...
</Configuration>
```

_在最后，值得指明的是，当解析配置时，不会对RollingFile appender声明中的变量进行处理。这是因为整个RollingFile节点的解析被推迟了，只有匹配上了才会进行解析处理。有关详细信息，请参阅[RoutingAppender](./appenders.md#RoutingAppender)。_

## 脚本（Scripts）

Log4j支持[JSR 223](http://docs.oracle.com/javase/6/docs/technotes/guides/scripting/)脚本语言，这可以在某些组件里使用。也可以使用任何JSR 223脚本引擎提供支持的语言。在[Scripting Engine](https://java.net/projects/scripting/sources/svn/show/trunk/engines)网站上列举了所支持的语言。不过，其中列出的一些语言（如JavaScript，Groovy和Beanshell）直接支持JSR 223脚本框架，只需简单地引入该语言的jar包。

支持使用脚本的组件可以通过在其上配置`<script>`，`<scriptFile>`或`<scriptRef>`节点来实现。script节点包含name属性，language属性（脚本语言类别）和脚本文本。scriptFile节点包含ame属性，path属性（文件路径），language属性，charset属性（字符编码），以及是否应该监视文件的更改。scriptRef节点包含ref属性，该属性指向在`<scripts>`配置节点中定义的脚本名称。name属性用于索引脚本及其ScriptEngine，因此每次需要运行脚本时都可以快速定位到它。虽然name属性不是必需的，但设置它将有助于在脚本运行时调试问题。language属性在script节点上是必须设置的，且必须指定一个在Configuration status日志中显示的语言名称，如下一节所述。如果未在scriptFile元素上指定language属性，则自动由脚本路径的文件扩展名来进行估算。如果需要进行文件监视，则在配置节点上指定非零的monitorInterval就可以启用。该间隔将用于检查文件的更改。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="debug" name="RoutingTest">
  <Scripts>
    <Script name="selector" language="javascript"><![CDATA[
            var result;
            if (logEvent.getLoggerName().equals("JavascriptNoLocation")) {
                result = "NoLocation";
            } else if (logEvent.getMarker() != null && logEvent.getMarker().isInstanceOf("FLOW")) {
                result = "Flow";
            }
            result;
            ]]></Script>
    <ScriptFile name="groovy.filter" path="scripts/filter.groovy"/>
  </Scripts>

  <Appenders>
    <Console name="STDOUT">
      <ScriptPatternSelector defaultPattern="%d %p %m%n">
        <ScriptRef ref="selector"/>
          <PatternMatch key="NoLocation" pattern="[%-5level] %c{1.} %msg%n"/>
          <PatternMatch key="Flow" pattern="[%-5level] %c{1.} ====== %C{1.}.%M:%L %msg ======%n"/>
      </ScriptPatternSelector>
      <PatternLayout pattern="%m%n"/>
    </Console>
  </Appenders>

  <Loggers>
    <Logger name="EventLogger" level="info" additivity="false">
        <ScriptFilter onMatch="ACCEPT" onMisMatch="DENY">
          <Script name="GroovyFilter" language="groovy"><![CDATA[
            if (logEvent.getMarker() != null && logEvent.getMarker().isInstanceOf("FLOW")) {
                return true;
            } else if (logEvent.getContextMap().containsKey("UserId")) {
                return true;
            }
            return false;
            ]]>
          </Script>
        </ScriptFilter>
      <AppenderRef ref="STDOUT"/>
    </Logger>

    <Root level="error">
      <ScriptFilter onMatch="ACCEPT" onMisMatch="DENY">
        <ScriptRef ref="groovy.filter"/>
      </ScriptFilter>
      <AppenderRef ref="STDOUT"/>
    </Root>
  </Loggers>

</Configuration>
```

如果Configuration节点上的status属性设置为DEBUG，那么将列出当前安装的脚本引擎列表，并列出其属性。虽然有些引擎可能不是线程安全的，但Log4j采取措施来确保脚本（如果引擎判断它不是线程安全的话）将以线程安全的方式运行。

```text
2015-09-27 16:13:22,925 main DEBUG Installed script engines
2015-09-27 16:13:22,963 main DEBUG AppleScriptEngine Version: 1.1, Language: AppleScript, Threading: Not Thread Safe,
            Compile: false, Names: {AppleScriptEngine, AppleScript, OSA}
2015-09-27 16:13:22,983 main DEBUG Groovy Scripting Engine Version: 2.0, Language: Groovy, Threading: MULTITHREADED,
            Compile: true, Names: {groovy, Groovy}
2015-09-27 16:13:23,030 main DEBUG BeanShell Engine Version: 1.0, Language: BeanShell, Threading: MULTITHREADED,
            Compile: true, Names: {beanshell, bsh, java}
2015-09-27 16:13:23,039 main DEBUG Mozilla Rhino Version: 1.7 release 3 PRERELEASE, Language: ECMAScript, Threading: MULTITHREADED,
            Compile: true, Names: {js, rhino, JavaScript, javascript, ECMAScript, ecmascript}
```

当执行脚本时，脚本可以使用一组变量，使用这些变量可以实现各种功能。有关脚本可用的变量列表，请参阅各个组件的文档。

支持脚本的组件会要求脚本输出返回值，回传给Java代码。对于一些脚本语言而言，输出返回值是完全可以做到的，但是Javascript不允许有return语句，除非它在一个函数内。不过，Javascript可以返回在脚本中执行的最后一个语句的值。因此，如下所示的代码就可以达到输出返回值的目的。

```javascript
var result;
if (logEvent.getLoggerName().equals("JavascriptNoLocation")) {
    result = "NoLocation";
} else if (logEvent.getMarker() != null && logEvent.getMarker().isInstanceOf("FLOW")) {
    result = "Flow";
}
result;
```

### 关于Beanshell的特别说明

如果JSR 223脚本引擎支持编译他们的脚本，那么就需要标识它们支持Compilable接口。例如Beanshell。但是，无论何时调用compile方法，它都会抛出一个Error（而非Exception）。Log4j会捕获到它，会在尝试编译每个Beanshell脚本时输出下面显示的警告。然后在每次执行时解释所有Beanshell脚本。

```text
2015-09-27 16:13:23,095 main DEBUG Script BeanShellSelector is compilable
2015-09-27 16:13:23,096 main WARN Error compiling script java.lang.Error: unimplemented
            at bsh.engine.BshScriptEngine.compile(BshScriptEngine.java:175)
            at bsh.engine.BshScriptEngine.compile(BshScriptEngine.java:154)
            at org.apache.logging.log4j.core.script.ScriptManager$MainScriptRunner.<init>(ScriptManager.java:125)
            at org.apache.logging.log4j.core.script.ScriptManager.addScript(ScriptManager.java:94)
```

## XInclude

XML配置文件使用[XInclude](https://en.wikipedia.org/wiki/XInclude)来导入其他的文件。下面是一个包含两个其他文件的log4j2.xml示例：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration xmlns:xi="http://www.w3.org/2001/XInclude"
               status="warn" name="XIncludeDemo">
  <properties>
    <property name="filename">xinclude-demo.log</property>
  </properties>
  <ThresholdFilter level="debug"/>
  <xi:include href="log4j-xinclude-appenders.xml" />
  <xi:include href="log4j-xinclude-loggers.xml" />
</configuration>
```

log4j-xinclude-appenders.xml：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<appenders>
  <Console name="STDOUT">
    <PatternLayout pattern="%m%n" />
  </Console>
  <File name="File" fileName="${filename}" bufferedIO="true" immediateFlush="true">
    <PatternLayout>
      <pattern>%d %p %C{1.} [%t] %m%n</pattern>
    </PatternLayout>
  </File>
</appenders>
```

log4j-xinclude-loggers.xml：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<loggers>
  <logger name="org.apache.logging.log4j.test1" level="debug" additivity="false">
    <ThreadContextMapFilter>
      <KeyValuePair key="test" value="123" />
    </ThreadContextMapFilter>
    <AppenderRef ref="STDOUT" />
  </logger>

  <logger name="org.apache.logging.log4j.test2" level="debug" additivity="false">
    <AppenderRef ref="File" />
  </logger>

  <root level="error">
    <AppenderRef ref="STDOUT" />
  </root>
</loggers>
```

## 组合配置

Log4j允许使用多个配置文件，通过在log4j.configurationFile上配置逗号分隔文件路径列表来实现。可以通过在log4j.mergeStrategy属性上指定实现了MergeStrategy接口的类来控制合并逻辑。默认合并策略将使用以下规则来合并文件：

1. 全局配置属性与后面配置文件中属性相聚合，后面的配置属性覆盖之前的配置属性；不过，只使用最高的status级别和最小monitorInterval值（该值必须大于0）。
2. 聚合所有配置文件里的属性。重复的属性覆盖之前配置中的那些属性。
3. 如果定义了多个filters，则filters在CompositeFilter下聚合。由于filters不必命名，因此重复是可能存在的。
4. Scripts和ScriptFile引用被聚合。重复定义覆盖之前配置中的定义。
5. Appenders是聚合的。具有相同名称的appender将被稍后配置中的appender覆盖，包括所有Appender的子组件。
6. Loggers也是聚合的。Logger属性单独合并，重复的属性将被后续配置中的属性覆盖。Logger上的Appender引用（AppenderRef）被聚合，重复的引用被后来的配置中的引用所取代。如果定义了多个Filters，则Logger上的Filters将聚合在CompositeFilter下。由于Filters不必命名，因此重复是可能存在的。在Appenders引用（AppenderRef）下的Filters，是保留还是舍弃根据其父Appender引用下是保留还是丢弃。

## 状态消息（Status Messages）

> **故障排除提示：**
> * 在使用配置之前，可以使用系统属性`org.apache.logging.log4j.simplelog.StatusLogger.level`控制status logger级别。
> * 使用配置后，可以使用"status"属性在配置文件中控制status logger级别，例如：`<Configuration status="trace">`。

正如期望能够诊断应用程序中的问题一样，经常需要能够诊断日志配置或配置的组件中的问题。由于没配置日志，为此在初始化期间无法使用“正常”日志记录。此外，在appender内的日志记录可能导致无限递归，Log4j会检测并忽略这种递归事件。为了满足这种需求，Log4j 2 API包括一个[StatusLogger](https://logging.apache.org/log4j/2.x/log4j-api/apidocs/org/apache/logging/log4j/status/StatusLogger.html)。组件会使用StatusLogger实例，类似于：

```java
protected final static Logger logger = StatusLogger.getLogger();
```

由于StatusLogger实现了Log4j 2 API的Logger接口，因此可以使用所有的Logger方法。

配置Log4j时，有时需要查看status事件。这可以在Configuration节点上配置status属性来实现，或者可以通过设置“Log4jDefaultStatusLevel”系统属性来提供默认值。status属性的只能设置为“trace”，“debug”，“info”，“warn”，“error”和“fatal”。 以下配置将status属性设置为debug。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="debug" name="RoutingTest">
  <Properties>
    <Property name="filename">target/rolling1/rollingtest-$${sd:type}.log</Property>
  </Properties>
  <ThresholdFilter level="debug"/>

  <Appenders>
    <Console name="STDOUT">
      <PatternLayout pattern="%m%n"/>
    </Console>
    <List name="List">
      <ThresholdFilter level="debug"/>
    </List>
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
        <Route ref="STDOUT" key="Audit"/>
        <Route ref="List" key="Service"/>
      </Routes>
    </Routing>
  </Appenders>

  <Loggers>
    <Logger name="EventLogger" level="info" additivity="false">
      <AppenderRef ref="Routing"/>
    </Logger>

    <Root level="error">
      <AppenderRef ref="STDOUT"/>
    </Root>
  </Loggers>

</Configuration>
```

在启动期间，此配置可能产生：

```text
2011-11-23 17:08:00,769 DEBUG Generated plugins in 0.003374000 seconds
2011-11-23 17:08:00,789 DEBUG Calling createProperty on class org.apache.logging.log4j.core.config.Property for element property with params(name="filename", value="target/rolling1/rollingtest-${sd:type}.log")
2011-11-23 17:08:00,792 DEBUG Calling configureSubstitutor on class org.apache.logging.log4j.core.config.PropertiesPlugin for element properties with params(properties={filename=target/rolling1/rollingtest-${sd:type}.log})
2011-11-23 17:08:00,794 DEBUG Generated plugins in 0.001362000 seconds
2011-11-23 17:08:00,797 DEBUG Calling createFilter on class org.apache.logging.log4j.core.filter.ThresholdFilter for element ThresholdFilter with params(level="debug", onMatch="null", onMismatch="null")
2011-11-23 17:08:00,800 DEBUG Calling createLayout on class org.apache.logging.log4j.core.layout.PatternLayout for element PatternLayout with params(pattern="%m%n", Configuration(RoutingTest), null, charset="null")
2011-11-23 17:08:00,802 DEBUG Generated plugins in 0.001349000 seconds
2011-11-23 17:08:00,804 DEBUG Calling createAppender on class org.apache.logging.log4j.core.appender.ConsoleAppender for element Console with params(PatternLayout(%m%n), null, target="null", name="STDOUT", ignoreExceptions="null")
2011-11-23 17:08:00,804 DEBUG Calling createFilter on class org.apache.logging.log4j.core.filter.ThresholdFilter for element ThresholdFilter with params(level="debug", onMatch="null", onMismatch="null")
2011-11-23 17:08:00,806 DEBUG Calling createAppender on class org.apache.logging.log4j.test.appender.ListAppender for element List with params(name="List", entryPerNewLine="null", raw="null", null, ThresholdFilter(DEBUG))
2011-11-23 17:08:00,813 DEBUG Calling createRoute on class org.apache.logging.log4j.core.appender.routing.Route for element Route with params(AppenderRef="null", key="null", Node=Route)
2011-11-23 17:08:00,823 DEBUG Calling createRoute on class org.apache.logging.log4j.core.appender.routing.Route for element Route with params(AppenderRef="STDOUT", key="Audit", Node=Route)
2011-11-23 17:08:00,824 DEBUG Calling createRoute on class org.apache.logging.log4j.core.appender.routing.Route for element Route with params(AppenderRef="List", key="Service", Node=Route)
2011-11-23 17:08:00,825 DEBUG Calling createRoutes on class org.apache.logging.log4j.core.appender.routing.Routes for element Routes with params(pattern="${sd:type}", routes={Route(type=dynamic default), Route(type=static Reference=STDOUT key='Audit'), Route(type=static Reference=List key='Service')})
2011-11-23 17:08:00,827 DEBUG Calling createAppender on class org.apache.logging.log4j.core.appender.routing.RoutingAppender for element Routing with params(name="Routing", ignoreExceptions="null", Routes({Route(type=dynamic default),Route(type=static Reference=STDOUT key='Audit'),Route(type=static Reference=List key='Service')}), Configuration(RoutingTest), null, null)
2011-11-23 17:08:00,827 DEBUG Calling createAppenders on class org.apache.logging.log4j.core.config.AppendersPlugin for element appenders with params(appenders={STDOUT, List, Routing})
2011-11-23 17:08:00,828 DEBUG Calling createAppenderRef on class org.apache.logging.log4j.core.config.plugins.AppenderRefPlugin for element AppenderRef with params(ref="Routing")
2011-11-23 17:08:00,829 DEBUG Calling createLogger on class org.apache.logging.log4j.core.config.LoggerConfig for element logger with params(additivity="false", level="info", name="EventLogger", AppenderRef={Routing}, null)
2011-11-23 17:08:00,830 DEBUG Calling createAppenderRef on class org.apache.logging.log4j.core.config.plugins.AppenderRefPlugin for element AppenderRef with params(ref="STDOUT")
2011-11-23 17:08:00,831 DEBUG Calling createLogger on class org.apache.logging.log4j.core.config.LoggerConfig$RootLogger for element root with params(additivity="null", level="error", AppenderRef={STDOUT}, null)
2011-11-23 17:08:00,833 DEBUG Calling createLoggers on class org.apache.logging.log4j.core.config.LoggersPlugin for element loggers with params(loggers={EventLogger, root})
2011-11-23 17:08:00,834 DEBUG Reconfiguration completed
2011-11-23 17:08:00,846 DEBUG Calling createLayout on class org.apache.logging.log4j.core.layout.PatternLayout for element PatternLayout with params(pattern="%d %p %c{1.} [%t] %m%n", Configuration(RoutingTest), null, charset="null")
2011-11-23 17:08:00,849 DEBUG Calling createPolicy on class org.apache.logging.log4j.core.appender.rolling.SizeBasedTriggeringPolicy for element SizeBasedTriggeringPolicy with params(size="500")
2011-11-23 17:08:00,851 DEBUG Calling createAppender on class org.apache.logging.log4j.core.appender.RollingFileAppender for element RollingFile with params(fileName="target/rolling1/rollingtest-Unknown.log", filePattern="target/rolling1/test1-Unknown.%i.log.gz", append="null", name="Rolling-Unknown", bufferedIO="null", immediateFlush="null", SizeBasedTriggeringPolicy(SizeBasedTriggeringPolicy(size=500)), null, PatternLayout(%d %p %c{1.} [%t] %m%n), null, ignoreExceptions="null")
2011-11-23 17:08:00,858 DEBUG Generated plugins in 0.002014000 seconds
2011-11-23 17:08:00,889 DEBUG Reconfiguration started for context sun.misc.Launcher$AppClassLoader@37b90b39
2011-11-23 17:08:00,890 DEBUG Generated plugins in 0.001355000 seconds
2011-11-23 17:08:00,959 DEBUG Generated plugins in 0.001239000 seconds
2011-11-23 17:08:00,961 DEBUG Generated plugins in 0.001197000 seconds
2011-11-23 17:08:00,965 WARN No Loggers were configured, using default
2011-11-23 17:08:00,976 DEBUG Reconfiguration completed
```

如果status属性设置为error，则只有error消息将写入console。这使得排除配置错误故障成为可能。例如，如果上面的配置将status设置为error，且logger声明为：

```xml
<logger name="EventLogger" level="info" additivity="false">
  <AppenderRef ref="Routng"/>
</logger>
```

将产生以下错误消息：

```text
2011-11-24 23:21:25,517 ERROR Unable to locate appender Routng for logger EventLogger
```

应用程序可能希望将status输出定向到其他目的地。这可以通过将dest属性设置为“err”（这将status消息输出发送到stderr）抑或文件路径或URL来完成。这也可以通过确保配置的status设置为OFF，然后以编程方式配置应用程序来完成，例如：

```java
StatusConsoleListener listener = new StatusConsoleListener(Level.ERROR);
StatusLogger.getLogger().registerListener(listener);
```

## 在Maven中单元测试

Maven可以在构建周期中运行单元和功能测试。默认情况下，放置在`src/test/resources`中的任何文件都会自动复制到`target/test-classes`，并在任何测试执行期间包含在类路径中。为此，将`log4j2-test.xml`放入此目录将导致优先使用它而不是可能存在的`log4j2.xml`或`log4j2.json`。因此，在测试期间可以使用与生产不同的日志配置。

Log4j 2广泛使用的第二种方法是，在junit测试类中，在使用@BeforeClass注释的方法里设置log4j.configurationFile属性。这就可以在测试期间使用任意命名的文件。

Log4j 2广泛使用的第三种方法是，使用`LoggerContextRule`JUnit测试规则，该规则为测试提供了额外的便利方法。这需要将`log4j-core` `test-jar`库加入到测试类路径中。例如：

```java
public class AwesomeTest {
    @Rule
    public LoggerContextRule init = new LoggerContextRule("MyTestConfig.xml");

    @Test
    public void testSomeAwesomeFeature() {
        final LoggerContext ctx = init.getContext();
        final Logger logger = init.getLogger("org.apache.logging.log4j.my.awesome.test.logger");
        final Configuration cfg = init.getConfiguration();
        final ListAppender app = init.getListAppender("List");
        logger.warn("Test message");
        final List<LogEvent> events = app.getEvents();
        // etc.
    }
}
```

## 系统属性

Log4j文档列举了诸多系统属性，可用于控制Log4j 2行为的方方面面。下表列出了这些属性及其默认值及其控制的描述。去除属性名中存在的任何空格。

也可以通过创建名为`log4j2.component.properties`的文件，将所需的键和值添加到文件，然后将该文件包含在应用程序的类路径中，来指定使用下面列出的哪些属性。

Log4j 2系统属性：

| System Property | Default Value | Description |
| ---- | ---- | ---- |
| log4j.configurationFile | - | XML或JSON Log4j 2配置文件的路径。也可以包含逗号分隔的配置文件名列表。 |
| log4j.mergeFactory | - | 实现MergeStrategy接口的类的名称。如果未指定，在创建CompositeConfiguration时使用`DefaultMergeStrategy`。 |
| log4jContextSelector | ClassLoaderContextSelector | 创建LoggerContexts。根据情况，应用程序可以具有一个或多个LoggerContext。有关详细信息，请参阅[日志分隔](./logsep.md)章节。可用的上下文选择器实现类：<br> `org.apache.logging.log4j.core.async.AsyncLoggerContextSelector` - 使所有[logger异步](./async.md)。<br> `org.apache.logging.log4j.core.selector.BasicContextSelector` - 创建一个共享的LoggerContext。<br> `org.apache.logging.log4j.core.selector.ClassLoaderContextSelector` - 为每个Web应用程序创建单独的LoggerContexts。<br> `org.apache.logging.log4j.core.selector.JndiContextSelector` - 使用JNDI来定位每个Web应用程序的LoggerContext。<br> `org.apache.logging.log4j.core.osgi.BundleContextSelector` - 为每个OSGi包分离LoggerContexts。 |
| Log4jLogEventFactory | org.apache.logging.log4j.core.impl.DefaultLogEventFactory | LoggerConfig使用的工厂类来创建`LogEvent`实例。（使用`AsyncLoggerContextSelector`时忽略该值）。 |
| log4j2.loggerContextFactory | org.apache.logging.log4j.simple.SimpleLoggerContextFactory | LogManager使用的工厂类来启动日志实现。core jar包提供了`org.apache.logging.log4j.core.impl.Log4jContextFactory`。 |
| log4j.configurationFactory | - | 扩展`org.apache.logging.log4j.core.config.ConfigurationFactory`的类的完全限定类名。如果指定，此类的实例将追加到配置工厂的列表中。 |
| log4j.shutdownHookEnabled | true | 覆盖全局标志是否应在shutdown hook里stop掉LoggerContext。默认情况下，这是启用的，可以配置为禁用。当使用`log4j-web`时，会自动禁用。 |
| log4j.shutdownCallbackRegistry | org.apache.logging.log4j.core.util.DefaultShutdownCallbackRegistry | 实现[ShutdownCallbackRegistry](https://logging.apache.org/log4j/2.x/log4j-core/apidocs/org/apache/logging/log4j/core/util/ShutdownCallbackRegistry.html)的类的完全限定类名。如果指定，则使用此类的实例而不是`DefaultShutdownCallbackRegistry`。指定的类必须有一个默认的构造函数。 |
| log4j.Clock | SystemClock | 用于对日志事件加时间戳，是`org.apache.logging.log4j.core.util.Clock`接口的具体实现。<br>默认情况下，对每个日志事件调用`System.currentTimeMillis`。<br>您还可以指定实现`Clock`接口的自定义类的完全限定类名。 |
| org.apache.logging.log4j.level | ERROR | 默认配置的日志级别。如果ConfigurationFactory无法成功创建配置（例如，找不到log4j2.xml文件），则使用默认配置。 |
| disableThreadContext | false | 如果为`true`，则会禁用ThreadContext stack和map。（如果指定了自定义ThreadContext map，可能会被忽略。） |
| disableThreadContextStack | false | 如果为`true`，则会禁用ThreadContext stack。 |
| disableThreadContextMap | false | 如果为`true`，则会禁用ThreadContext map。（如果指定了自定义ThreadContext map，可能会被忽略。） |
| log4j2.threadContextMap | - | 自定义ThreadContextMap实现类的完全限定类名。 |
| isThreadContextMapInheritable | false | 如果为`true`，则使用`InheritableThreadLocal`来实现ThreadContext map。否则，使用普通`ThreadLocal`。（如果指定了自定义ThreadContext map，可能会被忽略。） |
| log4j2.ContextDataInjector | - | 自定义`ContextDataInjector`实现类的完全限定类名。 |
| log4j2.garbagefree.threadContextMap | false | 指定“true”使ThreadContext map无GC。 |
| log4j2.disable.jmx | false | 如果为`true`，Log4j配置对象（如LoggerContexts，Appender，Loggers等）将不会被MBean使用，且不能进行远程监视和管理。 |
| log4j2.jmx.notify.async | 对于webapps为false，其它为true | 如果为`true`，log4j的JMX通知启用单独线程发送，否则它们在调用线程里发送。如果系统属性`log4j2.is.webapp`为`true`或`javax.servlet.Servlet`在类路径上，则默认行为是使用调用线程发送JMX通知。 |
| log4j.skipJansi | false | 如果为`true`，ConsoleAppender将不会尝试在Windows上使用Jansi输出流。 |
| log4j.ignoreTCL | false | 如果为`true`，那么默认类加载器加载类。否则，将尝试使用当前线程的上下文类加载器（thread's context class loader）加载类，然后再返回到默认类加载器。 |
| org.apache.logging.log4j.uuidSequence | 0 | 用于为生成整数值UUID提供种子。 |
| org.apache.logging.log4j.simplelog.showlogname | false | 如果为`true`，则logger名称包含在每个SimpleLogger日志消息中。 |
| org.apache.logging.log4j.simplelog.showShortLogname | true | 如果为`true`，则只有logger名称的最后一个组件包含在SimpleLogger日志消息中。（例如，如果logger名称为“mycompany.myproject.mycomponent”，则只记录“mycomponent”。 |
| org.apache.logging.log4j.simplelog.showdatetime | false | 如果为`true`，则SimpleLogger日志消息包含时间戳信息。 |
| org.apache.logging.log4j.simplelog.dateTimeFormat | "yyyy/MM/dd HH:mm:ss:SSS zzz" | 要使用的日期时间格式。如果`org.apache.logging.log4j.simplelog.showdatetime`为`false`，则忽略。 |
| org.apache.logging.log4j.simplelog.logFile | system.err | “system.err”（不区分大小写）日志输出到System.err，“system.out”（不区分大小写）日志输出到System.out，任何其他值当作文件名，用于保存SimpleLogger消息。 |
| org.apache.logging.log4j.simplelog.level | ERROR | 新SimpleLogger实例的默认级别。 |
| org.apache.logging.log4j.simplelog.&lt;loggerName&gt;level | SimpleLogger默认级别 | 特定名称的SimpleLogger实例的日志级别。 |
| org.apache.logging.log4j.simplelog.StatusLogger.level | ERROR | 此属性用于控制初始StatusLogger级别，并且可以通过调用`StatusLogger.getLogger().setLevel(someLevel)`在代码中改写。请注意，StatusLogger级别仅用于确定status日志输出级别，直到注册了listener。实际上，当解析配置时发现了listener，从那时起，status消息仅被发送到listener（取决于它们的statusLevel）。 |
| Log4jDefaultStatusLevel | ERROR | StatusLogger将日志记录系统中发生的事件记录到console。在配置期间，AbstractConfiguration向StatusLogger注册StatusConsoleListener，可以将status日志事件从默认控制台输出重定向到文件。listener（监听器）还支持细粒度过滤。如果配置未指定status级别，则使用此系统属性指定日志级别。<br><br> 注意：只有在找到配置文件之后，log4j-core实现才会使用此属性。 |
| log4j2.StatusLogger.level | WARN | 初始化StatusLogger的“listenersLevel”。如果添加了StatusLogger listeners，则“listenerLevel”将更改为listeners中最低级别的。如果注册了listeners，listenerLevel可以快速判断是否存在匹配上的listener。<br><br> 默认情况下，在解析配置和通过JMX StatusLoggerAdmin MBean来添加StatusLogger listeners。例如，如果配置是`<Configuration status="trace">`，则会注册statusLevel TRACE的listener，且StatusLogger listenerLevel设置为TRACE，在console上显示详细的status消息。<br><br> 如果没有注册listeners，则listenersLevel不会被使用，StatusLogger输出级别由`StatusLogger.getLogger().getLevel()`决定（请参阅属性`org.apache.logging.log4j.simplelog.StatusLogger.level`）。 |
| log4j2.status.entries | 200 | 保存在缓冲区中且可以使用`StatusLogger.getStatusData()`获取到的StatusLogger事件数。 |
| AsyncLogger.ExceptionHandler | default handler | 详情看[Async Logger](./async.md#SysPropsAllAsync)章节。 |
| AsyncLogger.RingBufferSize | 256 * 1024 | 详情看[Async Logger](./async.md#SysPropsAllAsync)章节。 |
| AsyncLogger.WaitStrategy | Timeout | 详情看[Async Logger](./async.md#SysPropsAllAsync)章节。 |
| AsyncLogger.ThreadNameStrategy | CACHED | 详情看[Async Logger](./async.md#SysPropsAllAsync)章节。 |
| AsyncLoggerConfig.ExceptionHandler | default handler | 详情看[Async Logger](./async.md#SysPropsMixedSync-Async)章节。 |
| AsyncLoggerConfig.RingBufferSize | 256 * 1024 | 详情看[Async Logger](./async.md#SysPropsMixedSync-Async)章节。 |
| AsyncLoggerConfig.WaitStrategy | Timeout | 详情看[Async Logger](./async.md#SysPropsMixedSync-Async)章节。 |
