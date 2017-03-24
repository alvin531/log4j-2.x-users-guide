# Appenders - 输出终端（追加器）

Appenders的责任就是将LogEvent输出到目的地。每个Appender必须实现[Appender](https://logging.apache.org/log4j/2.x/log4j-core/apidocs/org/apache/logging/log4j/core/Appender.html)接口。大多数Appenders继承[AbstractAppender](https://logging.apache.org/log4j/2.x/log4j-core/apidocs/org/apache/logging/log4j/core/appender/AbstractAppender.html)，它还实现了[Lifecycle](https://logging.apache.org/log4j/2.x/log4j-core/apidocs/org/apache/logging/log4j/core/LifeCycle.html)和[Filterable](https://logging.apache.org/log4j/2.x/log4j-core/apidocs/org/apache/logging/log4j/core/filter/Filterable.html)接口。LifeCycle允许组件在配置后进行初始化，并在关闭期间执行清理动作。Filterable允许组件增加Filter功能，它们在事件处理期间进行判断。

Appenders通常仅负责将事件数据写入目标目的地。在大多数情况下，他们将事件格式化的工作委托给一个[layout](./layouts.md)。一些appenders包装其他appenders，以便他们可以修改LogEvent，处理Appender中的失败，根据Filter的条件将事件路由到下级Appender，或提供类似于不直接格式化事件只是提供查看等这样的功能。

Appenders总是有一个名字，以便可以在Logger中引用它们。

在下面的表中，“类型（Type）”列对应于Java类型。类型里出现的非JDK类，除非另有说明，否则这些类通常应在[Log4j Core](https://logging.apache.org/log4j/2.x/log4j-core/apidocs/index.html)包中。

## AsyncAppender

AsyncAppender包含对其他Appenders的引用，并在单独的线程上将LogEvents写入到那些Appenders中。请留意，写入这些Appender时出现的异常无法知会到应用程序中。AsyncAppender应在其引用的其他Appender之后进行配置，以使其被正常关闭。

默认情况下，AsyncAppender使用[java.util.concurrent.ArrayBlockingQueue](http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/ArrayBlockingQueue.html)，不需要依赖其他的库。请留意，多线程应用程序在使用此appender时应该小心：阻塞队列容易发生锁竞争，我们的[测试](./others/performance.md#asyncLogging)表明，当越多线程并发记录时，性能可能会越差。考虑使用[无锁异步记录器（lock-free Async Loggers）](./async.md)以获得最佳性能。

AsyncAppender参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| AppenderRef | String | 需异步调用的被引用的Appender名字。可以配置多个AppenderRef元素。 |
| blocking | boolean | 如果为true，则appender将堵塞等待，等到队列中有空闲插槽。如果为false，则如果队列已满，则该事件将写入error appender中。默认值为true。 |
| shutdownTimeout | integer | 在shutdown时，Appender应该等待的毫秒数，等待把队列中未完成的日志事件完全处理完。默认值为零，这意味着永远等待。 |
| bufferSize | integer | 指定事件队列的最大容量。默认值为128。请注意，使用Disruptor的`BlockingQueue`时，此缓冲区大小必须为2的幂次方。<br> 当应用程序的记录速度比底层appender可以处理的速度快时，在一段时间里队列将被填满，此时后续的行为由[AsyncQueueFullPolicy](https://logging.apache.org/log4j/2.x/log4j-core/apidocs/org/apache/logging/log4j/core/async/AsyncQueueFullPolicy.html)来确定。 |
| errorRef | String | 错误时调用的Appender的名字。若appender中出现错误或队列已满，此时无法调用任何一个appender，就会调用该errorRef指定的Appender。如果没有指定，那么错误将被忽略。 |
| filter | Filter | 过滤器，用于确定事件是否应由此Appender处理。使用CompositeFilter可以组合多个Filter。 |
| name | String | 该Appender的名字。 |
| ignoreExceptions | boolean | 默认值为`true`，将忽略在内部记录事件时遇到的异常。当设置为`false`异常将被传播回调用者中。将此Appender包装在[FailoverAppender](./appenders.md#FailoverAppender)中时，必须将其设置为`false`。 |
| includeLocation | boolean | 提取location是一项昂贵的操作（可以使日志记录慢5到20倍）。基于性能的考虑，将日志事件添加到队列时，默认情况下不包括location信息。您可以通过设置`includeLocation="true"`来更改此值。 |
| BlockingQueueFactory | BlockingQueueFactory | 此元素覆盖要使用的`BlockingQueue`类型。有关详细信息，请参阅[下面的文档](./appenders.md#BlockingQueueFactory)。 |

还有一些系统属性可用于维护应用程序吞吐量，即使底层appender无法跟上日志记录速率从而导致队列被填满。请参阅系统属性[log4j2.AsyncQueueFullPolicy和log4j2.DiscardThreshold](./configuration.md#log4j2.AsyncQueueFullPolicy)的详细信息。

典型的AsyncAppender配置可能如下所示：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <File name="MyFile" fileName="logs/app.log">
      <PatternLayout>
        <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
      </PatternLayout>
    </File>
    <Async name="Async">
      <AppenderRef ref="MyFile"/>
    </Async>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="Async"/>
    </Root>
  </Loggers>
</Configuration>
```

从Log4j 2.7开始，可以使用[BlockingQueueFactory](https://logging.apache.org/log4j/2.x/log4j-core/apidocs/org/apache/logging/log4j/core/async/BlockingQueueFactory.html)插件指定`BlockingQueue`或`TransferQueue`的自定义实现。要覆盖默认的`BlockingQueueFactory`，请在`<Async />`节点内指定插件，如下所示：

```xml
<Configuration name="LinkedTransferQueueExample">
  <Appenders>
    <List name="List"/>
    <Async name="Async" bufferSize="262144">
      <AppenderRef ref="List"/>
      <LinkedTransferQueue/> <!-- override -->
    </Async>
  </Appenders>
  <Loggers>
    <Root>
      <AppenderRef ref="Async"/>
    </Root>
  </Loggers>
</Configuration>
```

Log4j附带以下实现（BlockingQueueFactory实现）：

| 插件名 | 描述 |
| ---- | ---- |
| ArrayBlockingQueue | 这是使用[ArrayBlockingQueue](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/ArrayBlockingQueue.html)的缺省实现。 |
| DisruptorBlockingQueue | 这使用了[Conversant Disruptor](https://github.com/conversant/disruptor)来实现`BlockingQueue`。这个插件带一个可选的属性，`spinPolicy`。 |
| JCToolsBlockingQueue | 这使用[JCTools](https://jctools.github.io/JCTools/)，特别是MPSC（multiple producer single consumer）有界无锁队列。 |
| LinkedTransferQueue | 这使用Java 7中新增的[LinkedTransferQueue](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/LinkedTransferQueue.html)来实现的功能。请注意，此队列不使用AsyncAppender的`bufferSize`属性，因为`LinkedTransferQueue`没有最大容量的说法。 |

## CassandraAppender

CassandraAppender将其输出写入到[Apache Cassandra](https://cassandra.apache.org/)数据库。必须提前配置密钥空间和表，并将该表的列映射配置在文件中。每列可以指定[StringLayout](./layouts.md)（例如，[PatternLayout](./layouts.md#PatternLayout)）以及可选的转换类型（type），也可以指定`org.apache.logging.log4j.spi.ThreadContextMap`或`org.apache.logging.log4j.spi.ThreadContextStack`这些转换类型，分别在map或list列中存储[MDC或NDC](./thread-context.md)。`java.util.Date`转换类型，将使用转换为该类型的日志事件时间戳（例如，使用`java.util.Date`填充Cassandra中的`timestamp`列类型）。

CassandraAppender参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| batched | boolean | 是否使用批处理语句将日志消息写入Cassandra。默认是`false`。 |
| batchType | [BatchStatement.Type](http://docs.datastax.com/en/drivers/java/3.0/com/datastax/driver/core/BatchStatement.Type.html) | 使用批写入时的批处理类型。默认是`LOGGED`。 |
| bufferSize | int | 写入前要缓冲或批处理的日志消息数。默认不进行缓冲。 |
| clusterName | String | 要连接的Cassandra集群名。 |
| columns | ColumnMapping\[\] | 列映射配置列表。每列必须指定列名。每列可以配置由完全限定类名指定的转换类型。默认的转换类型为`String`。如果配置的类型与[ReadOnlyStringMap](https://logging.apache.org/log4j/2.x/log4j-api/apidocs/org/apache/logging/log4j/util/ReadOnlyStringMap.html)/[ThreadContextMap](https://logging.apache.org/log4j/2.x/log4j-api/apidocs/org/apache/logging/log4j/spi/ThreadContextMap.html)或[ThreadContextStack](https://logging.apache.org/log4j/2.x/log4j-api/apidocs/org/apache/logging/log4j/spi/ThreadContextStack.html)赋值兼容，则该列将分别由MDC或NDC填充。如果配置的类型与`java.util.Date`赋值兼容，则将日志时间戳转换为该配置的日期类型。如果指定了`literal`属性，那么它的值将按照`INSERT`查询中的原样使用，而不进行任何转义。否则，指定的layout或pattern将被转换为配置好的类型并存储在该列中。 |
| contactPoints | SocketAddress\[\] | 要连接的Cassandra节点的主机和端口列表。这些必须是有效的主机名或IP地址。默认情况下，如果没有为主机指定端口或将其设置为0，则将使用默认的Cassandra端口9042。默认使用`localhost:9042`。 |
| filter | Filter | 过滤器，用于确定事件是否应由此Appender处理。使用CompositeFilter可以组合多个Filter。 |
| ignoreExceptions | boolean | 默认值为`true`，将忽略在内部记录事件时遇到的异常。当设置为`false`异常将被传播回调用者中。将此Appender包装在[FailoverAppender](./appenders.md#FailoverAppender)中时，必须将其设置为`false`。 |
| keyspace | String | 包含要写入日志消息的表的键空间名称。 |
| name | String | 该Appender的名字。 |
| password | String | 要使用的密码（连同用户名）连接到Cassandra。 |
| table | String | 要将日志消息写入的表名。 |
| useClockForTimestampGenerator | boolean | 是否使用配置的`org.apache.logging.log4j.core.util.Clock`作为[TimestampGenerator](http://docs.datastax.com/en/drivers/java/3.0/com/datastax/driver/core/TimestampGenerator.html)。默认是`false`。 |
| username | String | 用于连接Cassandra的用户名。默认情况下，不使用用户名或密码。 |
| useTls | boolean | 是否使用TLS/SSL连接Cassandra。默认为`false`。 |

以下是CassandraAppender配置示例：

```xml
<Configuration name="CassandraAppenderTest">
  <Appenders>
    <Cassandra name="Cassandra" clusterName="Test Cluster" keyspace="test" table="logs" bufferSize="10" batched="true">
      <SocketAddress host="localhost" port="9042"/>
      <ColumnMapping name="id" pattern="%uuid{TIME}" type="java.util.UUID"/>
      <ColumnMapping name="timeid" literal="now()"/>
      <ColumnMapping name="message" pattern="%message"/>
      <ColumnMapping name="level" pattern="%level"/>
      <ColumnMapping name="marker" pattern="%marker"/>
      <ColumnMapping name="logger" pattern="%logger"/>
      <ColumnMapping name="timestamp" type="java.util.Date"/>
      <ColumnMapping name="mdc" type="org.apache.logging.log4j.spi.ThreadContextMap"/>
      <ColumnMapping name="ndc" type="org.apache.logging.log4j.spi.ThreadContextStack"/>
    </Cassandra>
  </Appenders>
  <Loggers>
    <Logger name="org.apache.logging.log4j.nosql.appender.cassandra" level="DEBUG">
      <AppenderRef ref="Cassandra"/>
    </Logger>
    <Root level="ERROR"/>
  </Loggers>
</Configuration>
```

此示例配置使用下表模式：

```sql
CREATE TABLE logs (
    id timeuuid PRIMARY KEY,
    timeid timeuuid,
    message text,
    level text,
    marker text,
    logger text,
    timestamp timestamp,
    mdc map<text,text>,
    ndc list<text>
);
```

## ConsoleAppender

很明显，ConsoleAppender将其输出到System.out或System.err，默认使用System.out。必须提供Layout以格式化LogEvent。

ConsoleAppender参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| filter | Filter | 过滤器，用于确定事件是否应由此Appender处理。使用CompositeFilter可以组合多个Filter。 |
| layout | Layout | 用于格式化LogEvent的Layout。如果没有指定Layout，将使用“%m%n”作为默认模式layout。 |
| follow | boolean | 确定appender是否配置后通过System.setOut或System.setErr来重新指定System.out或System.err。请注意，follow属性不能与Windows上的Jansi一起使用。不能与`direct`一起使用。 |
| direct | boolean | 直接写入`java.io.FileDescriptor`而不绕开`java.lang.System.out/.err`。当输出重定向到文件或其他进程时，会损失10倍的性能。在Windows上无法与Jansi一起使用。不能与`follow`一起使用。Output不会对`java.lang.System.setOut()/.setErr()`有响应，且多线程应用程序中其他输出可能与`java.lang.System.out/.err`交织在一起。_2.6.2版本新特性。请注意，这是一个新特性，它迄今仅在Linux和Windows上使用Oracle JVM进行了测试。_ |
| name | String | 该Appender的名字。 |
| ignoreExceptions | boolean | 默认值为`true`，将忽略在内部记录事件时遇到的异常。当设置为`false`异常将被传播回调用者中。将此Appender包装在[FailoverAppender](./appenders.md#FailoverAppender)中时，必须将其设置为`false`。 |
| target | String | “SYSTEM_OUT”或“SYSTEM_ERR”。 默认值为“SYSTEM_OUT”。 |

典型的Console配置所示：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <Console name="STDOUT" target="SYSTEM_OUT">
      <PatternLayout pattern="%m%n"/>
    </Console>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="STDOUT"/>
    </Root>
  </Loggers>
</Configuration>
```

## FailoverAppender

FailoverAppender包装一组Appenders。如果主（primary）Appender失败，次（secondary）Appender将进行尝试，直到成功或再也没有可以尝试的appender了。

FailoverAppender参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| filter | Filter | 过滤器，用于确定事件是否应由此Appender处理。使用CompositeFilter可以组合多个Filter。 |
| primary | String | 要使用的主Appender的名字。 |
| failovers | String\[\] | 要使用的次Appender的名字列表。 |
| name | String | 该Appender的名字。 |
| ignoreExceptions | boolean | 默认值为`true`，将忽略在内部记录事件时遇到的异常。当设置为`false`异常将被传播回调用者中。将此Appender包装在[FailoverAppender](./appenders.md#FailoverAppender)中时，必须将其设置为`false`。 |
| retryIntervalSeconds | integer | 在重试主Appender的间隔时间（秒数）。默认值为60。 |
| target | String | “SYSTEM_OUT”或“SYSTEM_ERR”。 默认值为“SYSTEM_ERR”。 |

Failover的配置可能是：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <RollingFile name="RollingFile" fileName="logs/app.log" filePattern="logs/app-%d{MM-dd-yyyy}.log.gz"
                 ignoreExceptions="false">
      <PatternLayout>
        <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
      </PatternLayout>
      <TimeBasedTriggeringPolicy />
    </RollingFile>
    <Console name="STDOUT" target="SYSTEM_OUT" ignoreExceptions="false">
      <PatternLayout pattern="%m%n"/>
    </Console>
    <Failover name="Failover" primary="RollingFile">
      <Failovers>
        <AppenderRef ref="Console"/>
      </Failovers>
    </Failover>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="Failover"/>
    </Root>
  </Loggers>
</Configuration>
```

## FileAppender

FileAppender是OutputStreamAppender的一种，它输出到fileName属性中设置的文件里。FileAppender使用FileManager（继承于OutputStreamManager）来实际执行文件I/O操作。虽然不能共享来自不同配置的FileAppenders，但是如果Manager可访问到的话，FileManager可以共享。例如，Servlet容器中的两个Web应用程序可各自拥有配置，且如果Log4j位于它们两者共同的ClassLoader中，则可以安全地写入同一个文件。

FileAppender参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| append | boolean | 当为`true`（默认值）时，记录将追加到文件的末尾。当设置为`false`时，文件将在新记录写入之前被清除。 |
| bufferedIO | boolean | 当为`true`（默认值）时，记录将被写入缓冲区，当缓冲区满或者如果设置了`immediateFlush`时，数据将被写入磁盘，写入该记录。文件`locking`不能与`bufferedIO`一起使用。性能测试表明，即使启用`immediateFlush`，使用`bufferedIO`可显着提高性能。 |
| bufferSize | int | 当`bufferedIO`为`true`时，表示缓冲区大小，默认为8192字节。 |
| createOnDemand | boolean | appender按需创建文件。当日志事件通过所有过滤器并被路由到该appender时，appender才创建该文件。默认为`false`。 |
| filter | Filter | 过滤器，用于确定事件是否应由此Appender处理。使用CompositeFilter可以组合多个Filter。 |
| fileName | String | 要写入的文件名。如果文件或其任何父目录不存在，将创建它们。 |
| immediateFlush | boolean | 当为`true`（默认值）时，每次写入后都会进行刷新。这将保证数据写入磁盘，但可能会影响性能。<br><br> 每次写入后刷新入盘仅在使用同步logger的appender时才有效。异步logger和appender将在一批事件结束时自动刷新，即使immediateFlush设置为false。这也保证了数据被写入磁盘，但效率更高。 |
| layout | Layout | 用于格式化LogEvent的Layout。如果没有指定Layout，将使用“%m%n”作为默认模式layout。 |
| locking | boolean | 当设置为`true`时，只有在持有文件锁的才会发生I/O操作，这允许多个JVM中的FileAppenders和多个主机同时写入同一个文件。这对性能有显著影响，所以应该谨慎使用。此外，在许多系统上，文件锁是“咨询（advisory）”，意味着其他应用程序可以对文件执行操作，而无需获取锁。默认值为`false`。 |
| name | String | 该Appender的名字。 |
| ignoreExceptions | boolean | 默认值为`true`，将忽略在内部记录事件时遇到的异常。当设置为`false`异常将被传播回调用者中。将此Appender包装在[FailoverAppender](./appenders.md#FailoverAppender)中时，必须将其设置为`false`。 |

File配置示例：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <File name="MyFile" fileName="logs/app.log">
      <PatternLayout>
        <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
      </PatternLayout>
    </File>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="MyFile"/>
    </Root>
  </Loggers>
</Configuration>
```

## FlumeAppender

_这是单独jar中提供的可选组件。_

[Apache Flume](http://flume.apache.org/index.html)是一种分布式，可靠和高可用的系统，用于从许多不同的源高效收集，聚合和移动大量日志数据到集中式数据存储。FlumeAppender使用LogEvents，并将它们发送到Flume agent，并序列化为Avro进行处理。

Flume Appender支持三种操作模式。

1. 它可以充当远程Flume客户端，通过Avro将Flume事件发送到配置有Avro源的Flume Agent。
2. 它可以充当嵌入式Flume Agent，其中Flume事件直接通过Flume进行处理。
3. 它可以将事件保留到本地BerkeleyDB数据存储，然后异步地将事件发送到Flume，类似于嵌入式Flume Agent，但无需过多依赖于Flume。

作为嵌入式代理的用法将使消息直接传递到Flume Channel，然后立即把控制权返回到应用程序。远程代理的所有交互都是异步处理的。将“type”属性设置为“Embedded”将强制使用嵌入式代理。另外，在appender配置中配置agent属性也将导致使用嵌入式代理。

FlumeAppender参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| agents | Agent\[\] | 要发送日志记录事件的一组代理。如果指定了多个代理，则第一个代理为主代理，后续的代理将按主代理失败后以指定的次序使用。每个代理定义提供了代理主机和端口。`agents`和`properties`是互斥的。如果两者都配置了会出错。 |
| agentRetries | integer | 使用下一个代理时，重试当前代理的次数。当设定`type="persistent"`时，将忽略此参数（代理程序尝试一次后才使用下一个代理） |
| batchSize | integer | 指定按批处理的事件数。默认值为1。_此参数仅适用于Flume Appender。_ |
| compress | boolean | 当设置为true时，消息正文将使用gzip进行压缩。 |
| connectTimeoutMillis | integer | Flume连接超时设置（毫秒）。 |
| dataDir | String | Flume写入日志的目录。仅当嵌入设置为true且使用`agents`而非`properties`参数时才有效。 |
| filter | Filter | 过滤器，用于确定事件是否应由此Appender处理。使用CompositeFilter可以组合多个Filter。 |
| eventPrefix | String | 要为每个事件属性添加的前缀字符串，以便将其与MDC属性区分开来。默认为空字符串。 |
| flumeEventFactory | FlumeEventFactory | 从Log4j事件转化为Flume事件的工厂。默认工厂是FlumeAvroAppender本身。 |
| layout | Layout | 用于格式化LogEvent的Layout。若没有指定layout，则使用RFC5424Layout。 |
| lockTimeoutRetries | integer | 在写入Berkeley DB时发生LockConflictException异常时进行重试的次数。默认值为5。 |
| maxDelayMillis | integer | 等待完整批次（填满batchSize）的最大毫秒数。 |
| mdcExcludes | String | 从FlumeEvent中排除的mdc键的列表，以逗号分隔。这与`mdcIncludes`是互斥的。 |
| mdcIncludes | String | 应该包含在FlumeEvent中的mdc键的列表，以逗号分隔。若列表中的键在MDC里不存在，则被忽略。此选项与`mdcExcludes`互斥。 |
| mdcRequired | String | MDC中必须存在的mdc键的列表，以逗号分隔。如果键不存在，将抛出LoggingException。 |
| mdcPrefix | String | 为了将其与事件属性区分开来，应该将其添加到每个MDC键前面的字符串。默认字符串为`mdc:`。 |
| name | String | 该Appender的名字。 |
| properties | Property\[\] | 用于配置Flume Agent的一个或多个Property节点。properties不使用agent name来配置（在此，是使用appender的name），且不能配置sources。可以使用“sources.log4j-source.interceptors”为sources指定拦截器。所有其他Flume的properties都是允许的。同时指定`Agent`和`Property`节点，会出错。<br><br> 当在`Persistent`模式下时，有效属性为：<br><br> 1. “keyProvider”来指定插件的名称，以提供加密的密钥。 |
| requestTimeoutMillis | integer | Flume请求超时设置（毫秒）。 |
| ignoreExceptions | boolean | 默认值为`true`，将忽略在内部记录事件时遇到的异常。当设置为`false`异常将被传播回调用者中。将此Appender包装在[FailoverAppender](./appenders.md#FailoverAppender)中时，必须将其设置为`false`。 |
| type | enumeration | 值为“Avro”，“Embedded”或“Persistent”三者之一，以表示Appender使用哪种操作模式。 |

FlumeAppender配置示例 -- 使用主代理和辅助代理，使用RFC5424Layout格式化正文，并压缩正文：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <Flume name="eventLogger" compress="true">
      <Agent host="192.168.10.101" port="8800"/>
      <Agent host="192.168.10.102" port="8800"/>
      <RFC5424Layout enterpriseNumber="18060" includeMDC="true" appName="MyApp"/>
    </Flume>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="eventLogger"/>
    </Root>
  </Loggers>
</Configuration>
```

FlumeAppender配置示例 -- 使用主代理和辅助代理，压缩正文，使用RFC5424Layout格式化正文，并对事件进行加密持续到磁盘：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <Flume name="eventLogger" compress="true" type="persistent" dataDir="./logData">
      <Agent host="192.168.10.101" port="8800"/>
      <Agent host="192.168.10.102" port="8800"/>
      <RFC5424Layout enterpriseNumber="18060" includeMDC="true" appName="MyApp"/>
      <Property name="keyProvider">MySecretProvider</Property>
    </Flume>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="eventLogger"/>
    </Root>
  </Loggers>
</Configuration>
```

FlumeAppender配置示例 -- 使用主代理和辅助代理，压缩正文，使用RFC5424Layout格式化正文，并将事件传递到嵌入式Flume代理。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <Flume name="eventLogger" compress="true" type="Embedded">
      <Agent host="192.168.10.101" port="8800"/>
      <Agent host="192.168.10.102" port="8800"/>
      <RFC5424Layout enterpriseNumber="18060" includeMDC="true" appName="MyApp"/>
    </Flume>
    <Console name="STDOUT">
      <PatternLayout pattern="%d [%p] %c %m%n"/>
    </Console>
  </Appenders>
  <Loggers>
    <Logger name="EventLogger" level="info">
      <AppenderRef ref="eventLogger"/>
    </Logger>
    <Root level="warn">
      <AppenderRef ref="STDOUT"/>
    </Root>
  </Loggers>
</Configuration>
```

FlumeAppender配置示例 -- 使用Flume配置属性配置主代理和辅助代理，压缩正文，使用RFC5424Layout格式化正文，并将事件传递到嵌入式Flume代理。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="error" name="MyApp" packages="">
  <Appenders>
    <Flume name="eventLogger" compress="true" type="Embedded">
      <Property name="channels">file</Property>
      <Property name="channels.file.type">file</Property>
      <Property name="channels.file.checkpointDir">target/file-channel/checkpoint</Property>
      <Property name="channels.file.dataDirs">target/file-channel/data</Property>
      <Property name="sinks">agent1 agent2</Property>
      <Property name="sinks.agent1.channel">file</Property>
      <Property name="sinks.agent1.type">avro</Property>
      <Property name="sinks.agent1.hostname">192.168.10.101</Property>
      <Property name="sinks.agent1.port">8800</Property>
      <Property name="sinks.agent1.batch-size">100</Property>
      <Property name="sinks.agent2.channel">file</Property>
      <Property name="sinks.agent2.type">avro</Property>
      <Property name="sinks.agent2.hostname">192.168.10.102</Property>
      <Property name="sinks.agent2.port">8800</Property>
      <Property name="sinks.agent2.batch-size">100</Property>
      <Property name="sinkgroups">group1</Property>
      <Property name="sinkgroups.group1.sinks">agent1 agent2</Property>
      <Property name="sinkgroups.group1.processor.type">failover</Property>
      <Property name="sinkgroups.group1.processor.priority.agent1">10</Property>
      <Property name="sinkgroups.group1.processor.priority.agent2">5</Property>
      <RFC5424Layout enterpriseNumber="18060" includeMDC="true" appName="MyApp"/>
    </Flume>
    <Console name="STDOUT">
      <PatternLayout pattern="%d [%p] %c %m%n"/>
    </Console>
  </Appenders>
  <Loggers>
    <Logger name="EventLogger" level="info">
      <AppenderRef ref="eventLogger"/>
    </Logger>
    <Root level="warn">
      <AppenderRef ref="STDOUT"/>
    </Root>
  </Loggers>
</Configuration>
```

## JDBCAppender

JDBCAppender使用标准JDBC将日志事件写入关系数据库里。可以使用JNDI DataSource或自定义工厂方法来获取JDBC连接。无论采取哪种方式，都 **必须** 使用连接池。否则，记录性能将受到很大影响。如果配置的JDBC驱动程序支持批处理语句，并且将`bufferSize`配置为正数，则日志事件将被批处理。请注意，从Log4j 2.8起，有两种将日志事件配置为列映射的方法：原有的`ColumnConfig`方式，只允许字符串和时间戳；新的`ColumnMapping`插件使用Log4j的内置类型转换，支持更丰富的数据类型（这与[Cassandra Appender](./appenders.md#CassandraAppender)使用相同的插件）。

JDBCAppender参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| name | String | _必需_。该Appender的名字。 |
| ignoreExceptions | boolean | 默认值为`true`，将忽略在内部记录事件时遇到的异常。当设置为`false`异常将被传播回调用者中。将此Appender包装在[FailoverAppender](./appenders.md#FailoverAppender)中时，必须将其设置为`false`。 |
| filter | Filter | 过滤器，用于确定事件是否应由此Appender处理。使用CompositeFilter可以组合多个Filter。 |
| bufferSize | int | 若配置一个大于0的整数，表示appender缓冲日志事件，并在缓冲区达到此大小时进行刷新入库。 |
| connectionSource | ConnectionSource | _必需_。数据库连接源。 |
| tableName | String | _必需_。插入日志事件的数据库表名。 |
| columnConfigs | ColumnConfig\[\] | _必需（或使用columnMappings）_。列配置信息，配置要插入的日志事件数据以及如何插入该数据。由多个`<Column>`节点组成。 |
| columnMappings | ColumnMapping\[\] | _必需（或使用columnConfigs）_。列映射配置列表。每列必须指定列名（name）。每列可以配置由其完全限定类名指定的转换类型。默认情况下，转换类型为String。默认的转换类型为`String`。如果配置的类型与[ReadOnlyStringMap](https://logging.apache.org/log4j/2.x/log4j-api/apidocs/org/apache/logging/log4j/util/ReadOnlyStringMap.html)/[ThreadContextMap](https://logging.apache.org/log4j/2.x/log4j-api/apidocs/org/apache/logging/log4j/spi/ThreadContextMap.html)或[ThreadContextStack](https://logging.apache.org/log4j/2.x/log4j-api/apidocs/org/apache/logging/log4j/spi/ThreadContextStack.html)赋值兼容，则该列将分别由MDC或NDC填充（这要取决于具体数据库是如何处理插入Map或List值）。如果配置的类型与`java.util.Date`赋值兼容，则将日志时间戳转换为该配置的日期类型。如果配置的类型与`java.sql.Clob`或`java.sql.NClob`赋值兼容，则格式化的事件将分别转化为Clob或NClob（类似于ColumnConfig插件）。如果指定了`literal`属性，那么它的值将按照`INSERT`查询中的原样使用，而不进行任何转义。否则，指定的layout或pattern将被转换为配置好的类型并存储在该列中。 |

配置JDBCAppender时，必须指定`ConnectionSource`实现类，Appender从中获取JDBC连接。您必须使用`<DataSource>`或`<ConnectionFactory>`其中一个嵌套节点。

DataSource参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| jndiName | String | _必需_。完整名字，包括绑定`javax.sql.DataSource`的JNDI前缀，如`java:/comp/env/jdbc/LoggingDatabase`。`DataSource`必须使用连接池; 否则，记录将非常慢。 |

ConnectionFactory参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| class | Class | _必需_。完全限定类名，用于获取JDBC连接的静态工厂方法。 |
| method | Method | _必需_。方法名，用于获取JDBC连接的静态工厂方法。此方法必须无参，其返回类型必须是`java.sql.Connection`或`DataSource`。如果方法返回`Connection`，它必须是从连接池中获取到的（当Log4j完成操作时，它们将返回到池中）; 否则，记录将非常慢。如果该方法返回一个`DataSource`，`DataSource`将仅被获取一次，且由相同的原因它必须使用连接池。 |

配置JDBCAppender时，使用嵌套的`<Column>`节点来指定表中应该写入哪些列以及如何写入它们。JDBCAppender使用此信息来生成`PreparedStatement`来插入记录，避免SQL注入漏洞。

Column参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| name | String | _必需_。数据库的列名。 |
| pattern | String | 此属性可以使用`PatternLayout`模式格式化日志事件，然后把值插入该列中。需指定合法有效的模式。**必须** 配置该属性，`literal`或`isEventTimestamp="true"`，但不能混合使用，只能配置其中一个属性。 |
| literal | String | 使用此属性在该列中插入字面量。该值将直接包含在插入SQL中，而不会引用任何引号（这意味着如果您希望将其作为字符串，则该值应包含单引号：`literal="'Literal String'"`）。这对于不支持identity列的数据库特别有用。例如，如果使用Oracle，您可以指定`literal="NAME_OF_YOUR_SEQUENCE.NEXTVAL"`，以在ID列中插入唯一的ID。**必须** 配置该属性，`pattern`或`isEventTimestamp="true"`，但不能混合使用，只能配置其中一个属性。 |
| isEventTimestamp | boolean | 使用此属性可以在该列中插入事件时间戳，该列应为SQL datetime。该值使用`java.sql.Types.TIMESTAMP`插入。 **必须** 配置该属性，`pattern`或`literal`，但不能混合使用，只能配置其中一个属性。 |
| isUnicode | boolean | 除非指定了`pattern`，否则忽略此属性。如果为`true`或省略（默认），该值将作为unicode（`setNString`或`setNClob`）插入。否则，该值将按非unicode插入（`setString`或`setClob`）。 |
| isClob | boolean | 除非指定了`pattern`，否则忽略此属性。使用此属性指示列存储字符大对象（Character Large Object，CLOB）。如果为`true`，则该值将作为CLOB（`setClob`或`setNClob`）插入。如果为`false`或省略（默认），则该值将作为VARCHAR或NVARCHAR（`setString`或`setNString`）插入。 |

以下是JDBCAppender的几个示例配置，以及使用Commons Pooling和Commons DBCP来池化数据库连接：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="error">
  <Appenders>
    <JDBC name="databaseAppender" tableName="dbo.application_log">
      <DataSource jndiName="java:/comp/env/jdbc/LoggingDataSource" />
      <Column name="eventDate" isEventTimestamp="true" />
      <Column name="level" pattern="%level" />
      <Column name="logger" pattern="%logger" />
      <Column name="message" pattern="%message" />
      <Column name="exception" pattern="%ex{full}" />
    </JDBC>
  </Appenders>
  <Loggers>
    <Root level="warn">
      <AppenderRef ref="databaseAppender"/>
    </Root>
  </Loggers>
</Configuration>
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="error">
  <Appenders>
    <JDBC name="databaseAppender" tableName="LOGGING.APPLICATION_LOG">
      <ConnectionFactory class="net.example.db.ConnectionFactory" method="getDatabaseConnection" />
      <Column name="EVENT_ID" literal="LOGGING.APPLICATION_LOG_SEQUENCE.NEXTVAL" />
      <Column name="EVENT_DATE" isEventTimestamp="true" />
      <Column name="LEVEL" pattern="%level" />
      <Column name="LOGGER" pattern="%logger" />
      <Column name="MESSAGE" pattern="%message" />
      <Column name="THROWABLE" pattern="%ex{full}" />
    </JDBC>
  </Appenders>
  <Loggers>
    <Root level="warn">
      <AppenderRef ref="databaseAppender"/>
    </Root>
  </Loggers>
</Configuration>
```

```java
package net.example.db;

import java.sql.Connection;
import java.sql.SQLException;
import java.util.Properties;

import javax.sql.DataSource;

import org.apache.commons.dbcp.DriverManagerConnectionFactory;
import org.apache.commons.dbcp.PoolableConnection;
import org.apache.commons.dbcp.PoolableConnectionFactory;
import org.apache.commons.dbcp.PoolingDataSource;
import org.apache.commons.pool.impl.GenericObjectPool;

public class ConnectionFactory {
    private static interface Singleton {
        final ConnectionFactory INSTANCE = new ConnectionFactory();
    }

    private final DataSource dataSource;

    private ConnectionFactory() {
        Properties properties = new Properties();
        properties.setProperty("user", "logging");
        properties.setProperty("password", "abc123"); // or get properties from some configuration file

        GenericObjectPool<PoolableConnection> pool = new GenericObjectPool<PoolableConnection>();
        DriverManagerConnectionFactory connectionFactory = new DriverManagerConnectionFactory(
                "jdbc:mysql://example.org:3306/exampleDb", properties
        );
        new PoolableConnectionFactory(
                connectionFactory, pool, null, "SELECT 1", 3, false, false, Connection.TRANSACTION_READ_COMMITTED
        );

        this.dataSource = new PoolingDataSource(pool);
    }

    public static Connection getDatabaseConnection() throws SQLException {
        return Singleton.INSTANCE.dataSource.getConnection();
    }
}
```

## JMSAppender

JMSAppender将格式化的日志事件发送到JMS。

请注意，在Log4j 2.0中，此appender被分为JMSQueueAppender和JMSTopicAppender。从Log4j 2.1开始，这些appender被组合到JMSAppender中，不区分queue和topic。但是，2.0的使用`<JMSQueue />`或`<JMSTopic />`节点配置依然和新版本的`<JMS />`节点兼容。

JMSAppender参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| factoryBindingName | String | _必需_。用于在Context中获取[ConnectionFactory](http://docs.oracle.com/javaee/5/api/javax/jms/ConnectionFactory.html)的名字。这可以是`ConnectionFactory`的任何子接口。 |
| factoryName | String | 用于定义的Initial Context Factory的完全限定类名，由[INITIAL_CONTEXT_FACTORY](http://download.oracle.com/javase/6/docs/api/javax/naming/Context.html#INITIAL_CONTEXT_FACTORY)定义。如果没有提供值，将使用默认的InitialContextFactory。如果没有配置`providerURL`而又指定了factoryName，则会得到一个warn信息，因为这可能会出错。 |
| filter | Filter | 过滤器，用于确定事件是否应由此Appender处理。使用CompositeFilter可以组合多个Filter。 |
| layout | Layout | 用于格式化LogEvent的layout。如果没有指定，该appender将使用[SerializedLayout](./layouts.md#SerializedLayout)。 |
| name | String | _必需_。该Appender的名字。 |
| password | String | 创建JMS连接的密码。 |
| providerURL | String | provider URL，由[PROVIDER_URL](http://download.oracle.com/javase/6/docs/api/javax/naming/Context.html#PROVIDER_URL)定义。如果此值为空，则将使用默认系统provider。 |
| destinationBindingName | String | 用于定位[Destination](http://download.oracle.com/javaee/5/api/javax/jms/Destination.html)的名称。这可以是`Queue`或`Topic`，因此，属性`queueBindingName`和`topicBindingName`是保持与Log4j 2.0 JMS Appender兼容的别名。 |
| securityPrincipalName | String | 指定的主体身份名，由[SECURITY_PRINCIPAL](http://download.oracle.com/javase/6/docs/api/javax/naming/Context.html#SECURITY_PRINCIPAL)定义。如果指定`securityPrincipalName`而没有设置`securityCredentials`，则会得到一个warn信息，因为这可能会出错。 |
| securityCredentials | String | 指定主体的安全凭证，由[SECURITY_CREDENTIALS](http://download.oracle.com/javase/6/docs/api/javax/naming/Context.html#SECURITY_CREDENTIALS)定义。 |
| ignoreExceptions | boolean | 默认值为`true`，将忽略在内部记录事件时遇到的异常。当设置为`false`异常将被传播回调用者中。将此Appender包装在[FailoverAppender](./appenders.md#FailoverAppender)中时，必须将其设置为`false`。 |
| urlPkgPrefixes | String | 冒号分隔的包前缀列表，用于定位那些创建URL context factory的工厂类的类名，由[URL_PKG_PREFIXES](http://download.oracle.com/javase/6/docs/api/javax/naming/Context.html#URL_PKG_PREFIXES)定义。。 |
| userName | String | |

以下是JMSAppender示例配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp">
  <Appenders>
    <JMS name="jmsQueue" destinationBindingName="MyQueue" factoryBindingName="MyQueueConnectionFactory"/>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="jmsQueue"/>
    </Root>
  </Loggers>
</Configuration>
```

## JPAAppender

JPAAppender参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| filter | Filter | 过滤器，用于确定事件是否应由此Appender处理。使用CompositeFilter可以组合多个Filter。 |
| name | String | 该Appender的名字。 |
| ignoreExceptions | boolean | 默认值为`true`，将忽略在内部记录事件时遇到的异常。当设置为`false`异常将被传播回调用者中。将此Appender包装在[FailoverAppender](./appenders.md#FailoverAppender)中时，必须将其设置为`false`。 |

## KafkaAppender

KafkaAppender参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| filter | Filter | 过滤器，用于确定事件是否应由此Appender处理。使用CompositeFilter可以组合多个Filter。 |
| name | String | 该Appender的名字。 |
| ignoreExceptions | boolean | 默认值为`true`，将忽略在内部记录事件时遇到的异常。当设置为`false`异常将被传播回调用者中。将此Appender包装在[FailoverAppender](./appenders.md#FailoverAppender)中时，必须将其设置为`false`。 |

## MemoryMappedFileAppender

MemoryMappedFileAppender参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| filter | Filter | 过滤器，用于确定事件是否应由此Appender处理。使用CompositeFilter可以组合多个Filter。 |
| name | String | 该Appender的名字。 |
| ignoreExceptions | boolean | 默认值为`true`，将忽略在内部记录事件时遇到的异常。当设置为`false`异常将被传播回调用者中。将此Appender包装在[FailoverAppender](./appenders.md#FailoverAppender)中时，必须将其设置为`false`。 |

## NoSQLAppender

NoSQLAppender参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| filter | Filter | 过滤器，用于确定事件是否应由此Appender处理。使用CompositeFilter可以组合多个Filter。 |
| name | String | 该Appender的名字。 |
| ignoreExceptions | boolean | 默认值为`true`，将忽略在内部记录事件时遇到的异常。当设置为`false`异常将被传播回调用者中。将此Appender包装在[FailoverAppender](./appenders.md#FailoverAppender)中时，必须将其设置为`false`。 |

## OutputStreamAppender

## RandomAccessFileAppender

RandomAccessFileAppender参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| filter | Filter | 过滤器，用于确定事件是否应由此Appender处理。使用CompositeFilter可以组合多个Filter。 |
| name | String | 该Appender的名字。 |
| ignoreExceptions | boolean | 默认值为`true`，将忽略在内部记录事件时遇到的异常。当设置为`false`异常将被传播回调用者中。将此Appender包装在[FailoverAppender](./appenders.md#FailoverAppender)中时，必须将其设置为`false`。 |

## RewriteAppender

RewriteAppender参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| filter | Filter | 过滤器，用于确定事件是否应由此Appender处理。使用CompositeFilter可以组合多个Filter。 |
| name | String | 该Appender的名字。 |
| ignoreExceptions | boolean | 默认值为`true`，将忽略在内部记录事件时遇到的异常。当设置为`false`异常将被传播回调用者中。将此Appender包装在[FailoverAppender](./appenders.md#FailoverAppender)中时，必须将其设置为`false`。 |

## RollingFileAppender

RollingFileAppender参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| filter | Filter | 过滤器，用于确定事件是否应由此Appender处理。使用CompositeFilter可以组合多个Filter。 |
| name | String | 该Appender的名字。 |
| ignoreExceptions | boolean | 默认值为`true`，将忽略在内部记录事件时遇到的异常。当设置为`false`异常将被传播回调用者中。将此Appender包装在[FailoverAppender](./appenders.md#FailoverAppender)中时，必须将其设置为`false`。 |

## RollingRandomAccessFileAppender

RollingRandomAccessFileAppender参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| filter | Filter | 过滤器，用于确定事件是否应由此Appender处理。使用CompositeFilter可以组合多个Filter。 |
| name | String | 该Appender的名字。 |
| ignoreExceptions | boolean | 默认值为`true`，将忽略在内部记录事件时遇到的异常。当设置为`false`异常将被传播回调用者中。将此Appender包装在[FailoverAppender](./appenders.md#FailoverAppender)中时，必须将其设置为`false`。 |

## RoutingAppender

RoutingAppender参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| filter | Filter | 过滤器，用于确定事件是否应由此Appender处理。使用CompositeFilter可以组合多个Filter。 |
| name | String | 该Appender的名字。 |
| ignoreExceptions | boolean | 默认值为`true`，将忽略在内部记录事件时遇到的异常。当设置为`false`异常将被传播回调用者中。将此Appender包装在[FailoverAppender](./appenders.md#FailoverAppender)中时，必须将其设置为`false`。 |

## SMTPAppender

SMTPAppender参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| filter | Filter | 过滤器，用于确定事件是否应由此Appender处理。使用CompositeFilter可以组合多个Filter。 |
| name | String | 该Appender的名字。 |
| ignoreExceptions | boolean | 默认值为`true`，将忽略在内部记录事件时遇到的异常。当设置为`false`异常将被传播回调用者中。将此Appender包装在[FailoverAppender](./appenders.md#FailoverAppender)中时，必须将其设置为`false`。 |

## ScriptAppenderSelector

## SocketAppender

SocketAppender参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| filter | Filter | 过滤器，用于确定事件是否应由此Appender处理。使用CompositeFilter可以组合多个Filter。 |
| name | String | 该Appender的名字。 |
| ignoreExceptions | boolean | 默认值为`true`，将忽略在内部记录事件时遇到的异常。当设置为`false`异常将被传播回调用者中。将此Appender包装在[FailoverAppender](./appenders.md#FailoverAppender)中时，必须将其设置为`false`。 |

## SyslogAppender

SyslogAppender参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| filter | Filter | 过滤器，用于确定事件是否应由此Appender处理。使用CompositeFilter可以组合多个Filter。 |
| name | String | 该Appender的名字。 |
| ignoreExceptions | boolean | 默认值为`true`，将忽略在内部记录事件时遇到的异常。当设置为`false`异常将被传播回调用者中。将此Appender包装在[FailoverAppender](./appenders.md#FailoverAppender)中时，必须将其设置为`false`。 |

## ZeroMQ/JeroMQ Appender

JeroMQ参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| filter | Filter | 过滤器，用于确定事件是否应由此Appender处理。使用CompositeFilter可以组合多个Filter。 |
| name | String | 该Appender的名字。 |
| ignoreExceptions | boolean | 默认值为`true`，将忽略在内部记录事件时遇到的异常。当设置为`false`异常将被传播回调用者中。将此Appender包装在[FailoverAppender](./appenders.md#FailoverAppender)中时，必须将其设置为`false`。 |
