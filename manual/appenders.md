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
| userName | String | 创建JMS连接的用户名。 |

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

JPAAppender使用Java Persistence API 2.1将日志事件写入关系数据库表里。它需依赖API和具体的提供者实现jar包。它还需要一个实体，用于持久到具体的表上。该实体应继承`org.apache.logging.log4j.core.appender.db.jpa.BasicLogEventEntity`（如果您想使用默认映射），并提供至少一个`@Id`属性，或继承`org.apache.logging.log4j .core.appender.db.jpa.AbstractLogEventWrapperEntity`（如果要自定义映射）。有关更多信息，请参阅这两个类的Javadoc。您还可以参考这两个类的源代码，作为如何实现实体的示例参考。

JPAAppender参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| name | String | _必需_。该Appender的名字。 |
| ignoreExceptions | boolean | 默认值为`true`，将忽略在内部记录事件时遇到的异常。当设置为`false`异常将被传播回调用者中。将此Appender包装在[FailoverAppender](./appenders.md#FailoverAppender)中时，必须将其设置为`false`。 |
| filter | Filter | 过滤器，用于确定事件是否应由此Appender处理。使用CompositeFilter可以组合多个Filter。 |
| bufferSize | int | 若配置一个大于0的整数，表示appender缓冲日志事件，并在缓冲区达到此大小时进行刷新入库。 |
| entityClassName | String | _必需_。实现LogAventWrapperEntity的完全限定类名，它使用JPA注解将其映射到数据库表。 |
| persistenceUnitName | String | _必需_。应用于持久化日志事件的JPA持久化单元（persistence unit）的名称。 |

以下是JPAAppender的示例配置。第一个XML示例是Log4j配置文件，第二个是`persistence.xml`文件。假定使用EclipseLink，使用JPA 2.1或更高版本的实现提供者。您应该 _始终_ 为记录创建一个 _独立_ 的持久性单元，原因有两个。首先，_必须_ 将`<shared-cache-mode>`设置为“NONE”，这在JPA的普通使用中是不需要的。此外，出于性能原因，记录实体应该将其自身的持久化单元（persistence unit）与其他所有实体相隔离，且应使用非JTA数据源。请注意，您的持久化单元（persistence unit）还必须包含`<class>`元素，指定所有`org.apache.logging.log4j.core.appender.db.jpa.converter`转换器类。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="error">
  <Appenders>
    <JPA name="databaseAppender" persistenceUnitName="loggingPersistenceUnit"
         entityClassName="com.example.logging.JpaLogEntity" />
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
<persistence xmlns="http://xmlns.jcp.org/xml/ns/persistence"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/persistence
                                 http://xmlns.jcp.org/xml/ns/persistence/persistence_2_1.xsd"
             version="2.1">

  <persistence-unit name="loggingPersistenceUnit" transaction-type="RESOURCE_LOCAL">
    <provider>org.eclipse.persistence.jpa.PersistenceProvider</provider>
    <class>org.apache.logging.log4j.core.appender.db.jpa.converter.ContextMapAttributeConverter</class>
    <class>org.apache.logging.log4j.core.appender.db.jpa.converter.ContextMapJsonAttributeConverter</class>
    <class>org.apache.logging.log4j.core.appender.db.jpa.converter.ContextStackAttributeConverter</class>
    <class>org.apache.logging.log4j.core.appender.db.jpa.converter.ContextStackJsonAttributeConverter</class>
    <class>org.apache.logging.log4j.core.appender.db.jpa.converter.MarkerAttributeConverter</class>
    <class>org.apache.logging.log4j.core.appender.db.jpa.converter.MessageAttributeConverter</class>
    <class>org.apache.logging.log4j.core.appender.db.jpa.converter.StackTraceElementAttributeConverter</class>
    <class>org.apache.logging.log4j.core.appender.db.jpa.converter.ThrowableAttributeConverter</class>
    <class>com.example.logging.JpaLogEntity</class>
    <non-jta-data-source>jdbc/LoggingDataSource</non-jta-data-source>
    <shared-cache-mode>NONE</shared-cache-mode>
  </persistence-unit>

</persistence>
```

```java
package com.example.logging;
...
@Entity
@Table(name="application_log", schema="dbo")
public class JpaLogEntity extends BasicLogEventEntity {
    private static final long serialVersionUID = 1L;
    private long id = 0L;

    public TestEntity() {
        super(null);
    }
    public TestEntity(LogEvent wrappedEvent) {
        super(wrappedEvent);
    }

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "id")
    public long getId() {
        return this.id;
    }

    public void setId(long id) {
        this.id = id;
    }

    // If you want to override the mapping of any properties mapped in BasicLogEventEntity,
    // just override the getters and re-specify the annotations.
}
```

```java
package com.example.logging;
...
@Entity
@Table(name="application_log", schema="dbo")
public class JpaLogEntity extends AbstractLogEventWrapperEntity {
    private static final long serialVersionUID = 1L;
    private long id = 0L;

    public TestEntity() {
        super(null);
    }
    public TestEntity(LogEvent wrappedEvent) {
        super(wrappedEvent);
    }

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "logEventId")
    public long getId() {
        return this.id;
    }

    public void setId(long id) {
        this.id = id;
    }

    @Override
    @Enumerated(EnumType.STRING)
    @Column(name = "level")
    public Level getLevel() {
        return this.getWrappedEvent().getLevel();
    }

    @Override
    @Column(name = "logger")
    public String getLoggerName() {
        return this.getWrappedEvent().getLoggerName();
    }

    @Override
    @Column(name = "message")
    @Convert(converter = MyMessageConverter.class)
    public Message getMessage() {
        return this.getWrappedEvent().getMessage();
    }
    ...
}
```

## KafkaAppender

KafkaAppender将事件记录到[Apache Kafka](https://kafka.apache.org/)的topic中。每个日志事件作为没有密钥的Kafka记录发送。

KafkaAppender参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| topic | String | _必需_。Kafka使用的topic。 |
| filter | Filter | 过滤器，用于确定事件是否应由此Appender处理。使用CompositeFilter可以组合多个Filter。 |
| layout | Layout | 用于格式化LogEvent的layout。如果没有指定，该appender将[格式化的消息](http://logging.apache.org/log4j/2.x/log4j-api/apidocs/org/apache/logging/log4j/message/Message.html#getFormattedMessage%28%29)作为UTF-8编码的字符串发送到Kafka。 |
| name | String | 该Appender的名字。 |
| ignoreExceptions | boolean | 默认值为`true`，将忽略在内部记录事件时遇到的异常。当设置为`false`异常将被传播回调用者中。将此Appender包装在[FailoverAppender](./appenders.md#FailoverAppender)中时，必须将其设置为`false`。 |
| syncSend | boolean | 默认值为`true`，阻塞发送，直到Kafka服务器确认了记录。当设置为`false`时，立即发送并返回，达到更低的延迟和显著的更高吞吐量。_从2.8版本新增。请注意，这是一个新的扩展，并没有被广泛的测试。发送到Kafka的任何故障将输出给StatusLogger，且日志事件将被丢弃（ignoreExceptions参数将无效）。日志事件可能无法到达Kafka服务器。_ |
| properties | Property\[\] | 您可以在[Kafka producer properties](http://kafka.apache.org/documentation.html#producerconfigs)中设置属性。您需要设置`bootstrap.servers`属性，其他属性都可以使用其默认值。不要设置`value.serializer`属性。 |

以下是KafkaAppender示例配置代码片段：

```xml
<?xml version="1.0" encoding="UTF-8"?>
  ...
  <Appenders>
    <Kafka name="Kafka" topic="log-test">
      <PatternLayout pattern="%date %message"/>
      <Property name="bootstrap.servers">localhost:9092</Property>
    </Kafka>
  </Appenders>
  ```

该appender默认情况下是同步的，将阻塞，直到Kafka服务器确认记录为止，可以使用`timeout.ms`属性设置超时时间（默认为30秒）。使用[异步appender](./appenders.md#AsyncAppender)包装和/或将`syncSend`设置为`false`，将以异步记录日志。

这个appender需要[Kafka客户端库](http://kafka.apache.org/)。请注意，您需要使用与Kafka服务器相匹配的Kafka客户端库版本。

_注意：_ 确保不能把`org.apache.kafka`的日志设为`DEBUG`级别并输出到Kafka appender上，因为这将导致递归记录：

```xml
<?xml version="1.0" encoding="UTF-8"?>
  ...
  <Loggers>
    <Root level="DEBUG">
      <AppenderRef ref="Kafka"/>
    </Root>
    <Logger name="org.apache.kafka" level="INFO" /> <!-- avoid recursive logging -->
  </Loggers>
```

## MemoryMappedFileAppender

_从2.1版本新增。请注意，这是一个新的扩展，虽然它已经在几个平台上进行了测试，但还没有像其他文件appender那样进行过长期的跟踪分析。_

MemoryMappedFileAppender将指定文件的一部分映射到内存中，并将日志事件写入该内存，依靠操作系统的虚拟内存管理器将数据同步到存储设备。使用内存映射文件的主要优点是I/O性能。而不是通过系统调用来把数据写入磁盘，这个appender可以简单地操作程序的本地内存，这就快了好几个数量级了。此外，在大多数操作系统中，映射的内存区域实际上是内核的[页面缓存](http://en.wikipedia.org/wiki/Page_cache)（文件高速缓存），这意味着无需在用户空间中创建副本。（TODO：性能测试，将此appender的性能与RandomAccessFileAppender和FileAppender进行比较）

将文件区域映射到内存中会导致一些开销，特别是非常大的区域（半千兆字节或更多）。默认区域大小为32MB，这应该在重新映射操作的频率和持续时间之间达到合理的平衡。 （TODO：性能测试重新映射各种尺寸）

与FileAppender和RandomAccessFileAppender类似，MemoryMappedFileAppender使用MemoryMappedFileManager来实际执行文件I/O。虽然不同配置的MemoryMappedFileAppender无法共享，但如果Manager可共享访问，则MemoryMappedFileManagers可以共享使用。例如，Servlet容器中的两个Web应用程序可各自拥有配置，且如果Log4j位于它们两者共同的ClassLoader中，则可以安全地写入同一个文件。

MemoryMappedFileAppender参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| append | boolean | 当为`true`（默认值）时，记录将追加到文件的末尾。当设置为`false`时，文件将在新记录写入之前被清除。 |
| fileName | String | 要写入的文件名。如果文件或其任何父目录不存在，将创建它们。 |
| filter | Filter | 过滤器，用于确定事件是否应由此Appender处理。使用CompositeFilter可以组合多个Filter。 |
| immediateFlush | boolean | 当为`true`时，每次写入后都会调用[MappedByteBuffer.force()](http://docs.oracle.com/javase/7/docs/api/java/nio/MappedByteBuffer.html#force%28%29)。这将保证数据写入磁盘。<br> 该参数的默认值为`false`。这意味着即使Java进程崩溃也会将数据写入存储设备，但如果操作系统崩溃，则可能会丢失数据。<br> 请注意，使用内存映射文件，在每个日志事件上手动强制进行同步调用，会导致较大的性能问题。<br> 每次写入后刷新入盘仅在使用同步logger的appender时才有效。异步logger和appender将在一批事件结束时自动刷新，即使immediateFlush设置为false。这也保证了数据被写入磁盘，但效率更高。 |
| regionLength | int | 映射区域的长度默认为32MB（32 * 1024 * 1024字节）。此参数必须是256到1,073,741,824（1GB或2^30）之间的值; 超出该范围的值将被调整到最接近的有效值。Log4j将指定的值舍入到最接近的2的幂次方。 |
| layout | Layout | 用于格式化LogEvent的Layout。如果没有指定Layout，将使用“%m%n”作为默认模式layout。 |
| name | String | 该Appender的名字。 |
| ignoreExceptions | boolean | 默认值为`true`，将忽略在内部记录事件时遇到的异常。当设置为`false`异常将被传播回调用者中。将此Appender包装在[FailoverAppender](./appenders.md#FailoverAppender)中时，必须将其设置为`false`。 |

MemoryMappedFile示例配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <MemoryMappedFile name="MyFile" fileName="logs/app.log">
      <PatternLayout>
        <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
      </PatternLayout>
    </MemoryMappedFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="MyFile"/>
    </Root>
  </Loggers>
</Configuration>
```

## NoSQLAppender

NoSQLAppender使用内部轻量级provider接口将日志事件写入NoSQL数据库。目前提了MongoDB和Apache CouchDB的Provider实现，且编写自定义Provider也非常便利。

NoSQLAppender参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| name | String | _必需_。该Appender的名字。 |
| ignoreExceptions | boolean | 默认值为`true`，将忽略在内部记录事件时遇到的异常。当设置为`false`异常将被传播回调用者中。将此Appender包装在[FailoverAppender](./appenders.md#FailoverAppender)中时，必须将其设置为`false`。 |
| filter | Filter | 过滤器，用于确定事件是否应由此Appender处理。使用CompositeFilter可以组合多个Filter。 |
| bufferSize | int | 若配置一个大于0的整数，表示appender缓冲日志事件，并在缓冲区达到此大小时进行刷新入库。 |
| NoSqlProvider | NoSQLProvider<C extends NoSQLConnection<W, T extends NoSQLObject<W>>> | _必需_。NoSQL Provider程序，提供与具体的NoSQL数据库的连接。 |

通过在`<NoSql>`元素节点中指定相应的配置信息，指定要使用哪个NoSQL Provider。当前支持的类型是`<MongoDb>`和`<CouchDb>`。要创建自己的自定义Provider，请阅读`NoSQLProvider`，`NoSQLConnection`和`NoSQLObject`类的JavaDoc以及有关创建Log4j插件的文档。我们建议您查看MongoDB和CouchDB的Provider源代码，作为自定义Provider的参考指南。

MongoDB Provider参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| collectionName | String | _必需_。要插入日志事件信息的MongoDB collection名。 |
| writeConcernConstant | Field | 默认下，MongoDB使用`com.mongodb.WriteConcern.ACKNOWLEDGED`来插入记录。使用此可选属性，可以指定除了`ACKNOWLEDGED`之外的常量名。 |
| writeConcernConstantClass | Class | 如果设定了`writeConcernConstant`，则可以使用此属性来指定除`com.mongodb.WriteConcern`之外的类以找到其他常量（以自定义创建常量） |
| factoryClassName | Class | 可以使用此属性和`factoryMethodName`来指定类和静态方法来获取连接，用于提供具体的MongoDB数据库的连接。该方法必须返回`com.mongodb.DB`或`com.mongodb.MongoClient`。如果`DB`未被认证，还必须指定`username`和`password`。如果使用此factory方法提供连接，则不能指定`databaseName`，`server`或`port`属性。 |
| factoryMethodName | Method | 请参阅属性`factoryClassName`的说明。 |
| databaseName | String | 如果不指定`factoryClassName`和`factoryMethodName`来提供MongoDB连接，则必须使用此属性来指定MongoDB数据库名称。您还必须指定`username`和`password`。您还可选指定`server`（默认为localhost）和`port`（默认为MongoDB的默认端口）。 |
| server | String | 请参阅属性`databaseName`的说明。 |
| port | int | 请参阅属性`databaseName`的说明。 |
| username | String | 请参阅属性`databaseName`和`factoryClassName`的说明。 |
| passwrod | String | 请参阅属性`databaseName`和`factoryClassName`的说明。 |

CouchDB Provider参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| factoryClassName | Class | 可以使用此属性和`factoryMethodName`来指定类和静态方法来获取连接，用于提供具体的CouchDB数据库的连接。该方法必须返回`org.lightcouch.CouchDbClient`或`org.lightcouch.CouchDbProperties`。如果使用此factory方法提供连接，则不能指定`databaseName`，`protocol`，`server`，`port`，`username`或`password`属性。 |
| factoryMethodName | Method | 请参阅属性`factoryClassName`的说明。 |
| databaseName | String | 如果不指定`factoryClassName`和`factoryMethodName`来提供CouchDB连接，则必须使用此属性来指定CouchDB数据库名称。您还必须指定`username`和`password`。您还可选指定`protocol`（默认为http），`server`（默认为localhost）和`port`（默认为http的80端口或https的443端口）。 |
| protocol | String | 必须是“http”或“https”。请参阅属性`databaseName`的说明。 |
| server | String | 请参阅属性`databaseName`的说明。 |
| port | int | 请参阅属性`databaseName`的说明。 |
| username | String | 请参阅属性`databaseName`的说明。 |
| passwrod | String | 请参阅属性`databaseName`的说明。 |

NoSQLAppender的几个示例配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="error">
  <Appenders>
    <NoSql name="databaseAppender">
      <MongoDb databaseName="applicationDb" collectionName="applicationLog" server="mongo.example.org"
               username="loggingUser" password="abc123" />
    </NoSql>
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
    <NoSql name="databaseAppender">
      <MongoDb collectionName="applicationLog" factoryClassName="org.example.db.ConnectionFactory"
               factoryMethodName="getNewMongoClient" />
    </NoSql>
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
    <NoSql name="databaseAppender">
      <CouchDb databaseName="applicationDb" protocol="https" server="couch.example.org"
               username="loggingUser" password="abc123" />
    </NoSql>
  </Appenders>
  <Loggers>
    <Root level="warn">
      <AppenderRef ref="databaseAppender"/>
    </Root>
  </Loggers>
</Configuration>
```

以下示例演示如何在NoSQL数据库中持久化，存储为JSON格式：

```json
{
    "level": "WARN",
    "loggerName": "com.example.application.MyClass",
    "message": "Something happened that you might want to know about.",
    "source": {
        "className": "com.example.application.MyClass",
        "methodName": "exampleMethod",
        "fileName": "MyClass.java",
        "lineNumber": 81
    },
    "marker": {
        "name": "SomeMarker",
        "parent" {
            "name": "SomeParentMarker"
        }
    },
    "threadName": "Thread-1",
    "millis": 1368844166761,
    "date": "2013-05-18T02:29:26.761Z",
    "thrown": {
        "type": "java.sql.SQLException",
        "message": "Could not insert record. Connection lost.",
        "stackTrace": [
                { "className": "org.example.sql.driver.PreparedStatement$1", "methodName": "responder", "fileName": "PreparedStatement.java", "lineNumber": 1049 },
                { "className": "org.example.sql.driver.PreparedStatement", "methodName": "executeUpdate", "fileName": "PreparedStatement.java", "lineNumber": 738 },
                { "className": "com.example.application.MyClass", "methodName": "exampleMethod", "fileName": "MyClass.java", "lineNumber": 81 },
                { "className": "com.example.application.MainClass", "methodName": "main", "fileName": "MainClass.java", "lineNumber": 52 }
        ],
        "cause": {
            "type": "java.io.IOException",
            "message": "Connection lost.",
            "stackTrace": [
                { "className": "java.nio.channels.SocketChannel", "methodName": "write", "fileName": null, "lineNumber": -1 },
                { "className": "org.example.sql.driver.PreparedStatement$1", "methodName": "responder", "fileName": "PreparedStatement.java", "lineNumber": 1032 },
                { "className": "org.example.sql.driver.PreparedStatement", "methodName": "executeUpdate", "fileName": "PreparedStatement.java", "lineNumber": 738 },
                { "className": "com.example.application.MyClass", "methodName": "exampleMethod", "fileName": "MyClass.java", "lineNumber": 81 },
                { "className": "com.example.application.MainClass", "methodName": "main", "fileName": "MainClass.java", "lineNumber": 52 }
            ]
        }
    },
    "contextMap": {
        "ID": "86c3a497-4e67-4eed-9d6a-2e5797324d7b",
        "username": "JohnDoe"
    },
    "contextStack": [
        "topItem",
        "anotherItem",
        "bottomItem"
    ]
}
```

## OutputStreamAppender

OutputStreamAppender为其他Appender提供了基础功能实现，例如File和Socket appender，将事件写入Output Stream。不能对它直接配置。OutputStreamAppender支持immediateFlush和缓冲。 OutputStreamAppender使用OutputStreamManager来处理实际的I/O操作，可以在多个配置中共享Appender流。

## RandomAccessFileAppender

RandomAccessFileAppender类似于标准的[FileAppender](./appenders.md#FileAppender)，但它总是使用缓冲区（这不能关闭），且内部使用`ByteBuffer + RandomAccessFile`而不是`BufferedOutputStream`。在[性能测试](../others/performance.md#whichAppender)中，可以看到，与“bufferedIO=true”的FileAppender相比，有20-200%的性能提升。与FileAppender类似，RandomAccessFileAppender使用RandomAccessFileManager来实际执行文件I/O操作。不同配置的RandomAccessFileAppender无法共享，但如果Manager可共享访问，则RandomAccessFileManagers可以共享使用。例如，Servlet容器中的两个Web应用程序可各自拥有配置，且如果Log4j位于它们两者共同的ClassLoader中，则可以安全地写入同一个文件。

RandomAccessFileAppender参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| append | boolean | 当为`true`（默认值）时，记录将追加到文件的末尾。当设置为`false`时，文件将在新记录写入之前被清除。 |
| fileName | String | 要写入的文件名。如果文件或其任何父目录不存在，将创建它们。 |
| filter | Filter | 过滤器，用于确定事件是否应由此Appender处理。使用CompositeFilter可以组合多个Filter。 |
| immediateFlush | boolean | 当为`true`时，每次写入后都会调用flush操作。这将保证数据写入磁盘，但会影响到性能。<br> 每次写入后刷新入盘仅在使用同步logger的appender时才有效。异步logger和appender将在一批事件结束时自动刷新，即使immediateFlush设置为false。这也保证了数据被写入磁盘，但效率更高。 |
| bufferSize | int | 缓冲区大小，默认为262，144字节（256 * 1024字节）。此参数必须是256到1,073,741,824（1GB或2^30）之间的值; 超出该范围的值将被调整到最接近的有效值。Log4j将指定的值舍入到最接近的2的幂次方。 |
| layout | Layout | 用于格式化LogEvent的Layout。如果没有指定Layout，将使用“%m%n”作为默认模式layout。 |
| name | String | 该Appender的名字。 |
| ignoreExceptions | boolean | 默认值为`true`，将忽略在内部记录事件时遇到的异常。当设置为`false`异常将被传播回调用者中。将此Appender包装在[FailoverAppender](./appenders.md#FailoverAppender)中时，必须将其设置为`false`。 |

RandomAccessFile示例配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <RandomAccessFile name="MyFile" fileName="logs/app.log">
      <PatternLayout>
        <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
      </PatternLayout>
    </RandomAccessFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="MyFile"/>
    </Root>
  </Loggers>
</Configuration>
```

## RewriteAppender

RewriteAppender可以在由另一个Appender处理之前操纵LogEvent。这可以用来屏蔽敏感信息（如密码）或将信息注入到每个事件中。必须使用[RewritePolicy](./appenders.md#RewritePolicy)配置RewriteAppender。RewriteAppender应在其引用的任何Appenders之后进行配置，以使其可以被正确关闭。

RewriteAppender参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| AppenderRef | String | 需异步调用的被引用的Appender名字。可以配置多个AppenderRef元素。 |
| filter | Filter | 过滤器，用于确定事件是否应由此Appender处理。使用CompositeFilter可以组合多个Filter。 |
| name | String | 该Appender的名字。 |
| rewritePolicy | RewritePolicy | 将重写LogEvent的RewritePolicy。 |
| ignoreExceptions | boolean | 默认值为`true`，将忽略在内部记录事件时遇到的异常。当设置为`false`异常将被传播回调用者中。将此Appender包装在[FailoverAppender](./appenders.md#FailoverAppender)中时，必须将其设置为`false`。 |

### RewritePolicy

RewritePolicy是一个接口，允许具体的实现在将它们传递给Appender之前进行检索并修改LogEvent。RewritePolicy接口有一个名为rewrite的方法。该方法接收LogEvent参数，可以返回相同的LogEvent或创建一个全新的LogEvent。

#### MapRewritePolicy

MapRewritePolicy将处理包含MapMessage的LogEvent，可以添加或更新Map的元素。

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| mode | String | "Add"或"Update"。 |
| keyValuePair | KeyValuePair\[\] | 一个键/值对数组。 |

以下配置显示RewriteAppender配置，将产品密钥及其值添加到MapMessage中：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <Console name="STDOUT" target="SYSTEM_OUT">
      <PatternLayout pattern="%m%n"/>
    </Console>
    <Rewrite name="rewrite">
      <AppenderRef ref="STDOUT"/>
      <MapRewritePolicy mode="Add">
        <KeyValuePair key="product" value="TestProduct"/>
      </MapRewritePolicy>
    </Rewrite>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="Rewrite"/>
    </Root>
  </Loggers>
</Configuration>
```

#### PropertiesRewritePolicy

PropertiesRewritePolicy将在策略上配置的属性添加到正在记录的ThreadContext Map中。这些属性并不会添加到实际的ThreadContext Map中。属性值可能包含在处理配置期间以及在处理记录事件期间被动态计算的变量。

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| properties | Property\[\] | 用于定义一个或多个要添加到ThreadContext Map的键/值对信息。 |

以下配置示例，将系统用户名和环境信息添加到ThreadContext Map中：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <Console name="STDOUT" target="SYSTEM_OUT">
      <PatternLayout pattern="%m%n"/>
    </Console>
    <Rewrite name="rewrite">
      <AppenderRef ref="STDOUT"/>
      <PropertiesRewritePolicy>
        <Property name="user">${sys:user.name}</Property>
        <Property name="env">${sys:environment}</Property>
      </PropertiesRewritePolicy>
    </Rewrite>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="Rewrite"/>
    </Root>
  </Loggers>
</Configuration>
```

#### LoggerNameLevelRewritePolicy

您可以使用此策略通过更改事件级别使第三方代码中的日志记录变少。LoggerNameLevelRewritePolicy将重写给定的logger名前缀的日志事件级别。您可以对LoggerNameLevelRewritePolicy配置一个logger名前缀和一对级别信息，其中这对级别信息定义了源级别和目标级别。

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| loggerName | String | logger名的前缀，用来判断每个事件的logger名是否匹配。 |
| LevelPair | KeyValuePair\[\] | 一个键/值对数组，每个键都是源级别，每个值都是目标级别。 |

以下配置示例，将对`com.foo.bar`的所有日志记录，级别INFO映射到DEBUG，并将WARN映射为INFO。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp">
  <Appenders>
    <Console name="STDOUT" target="SYSTEM_OUT">
      <PatternLayout pattern="%m%n"/>
    </Console>
    <Rewrite name="rewrite">
      <AppenderRef ref="STDOUT"/>
      <LoggerNameLevelRewritePolicy loggerName="com.foo.bar">
        <KeyValuePair key="INFO" value="DEBUG"/>
        <KeyValuePair key="WARN" value="INFO"/>
      </LoggerNameLevelRewritePolicy>
    </Rewrite>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="Rewrite"/>
    </Root>
  </Loggers>
</Configuration>
```

## RollingFileAppender

RollingFileAppender是一个OutputStreamAppender，它写入FileName参数设置的文件，并根据TriggeringPolicy和RolloverStrategy翻转（rollover）文件。RollingFileAppender使用RollingFileManager（它扩展了OutputStreamManager）来实际执行文件I/O操作，并执行翻转（rollover）。不同配置的RolloverFileAppender无法共享，但如果Manager可共享访问，则RollingFileManagers可以共享使用。例如，Servlet容器中的两个Web应用程序可各自拥有配置，且如果Log4j位于它们两者共同的ClassLoader中，则可以安全地写入同一个文件。

RollingFileAppender需要配置[TriggeringPolicy](./appenders.md#Triggering%20Policies)和[RolloverStrategy](./appenders.md#Rollover%20Strategies)。TriggeringPolicy确定是否应该执行翻转（rollover），而在RolloverStrategy定义了如何进行翻转（rollover）。如果没有配置RolloverStrategy，RollingFileAppender将使用[DefaultRolloverStrategy](./appenders.md#DefaultRolloverStrategy)。从log4j-2.5开始，可以配置DefaultRolloverStrategy在执行翻转（rollover）中[自定义删除操作](./appenders.md#CustomDeleteOnRollover)。从2.8开始，若没有配置文件名，那可以使用[DirectWriteRolloverStrategy](./appenders.md#DirectWriteRolloverStrategy)来代替[DefaultRolloverStrategy](./appenders.md#DefaultRolloverStrategy)。

RollingFileAppender不支持文件锁定（file locking）。

RollingFileAppender参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| append | boolean | 当为`true`（默认值）时，记录将追加到文件的末尾。当设置为`false`时，文件将在新记录写入之前被清除。 |
| bufferedIO | boolean | 当为`true`（默认值）时，记录将被写入缓冲区，当缓冲区满或者如果设置了`immediateFlush`时，数据将被写入磁盘，写入该记录。文件`locking`不能与`bufferedIO`一起使用。性能测试表明，即使启用`immediateFlush`，使用`bufferedIO`可显着提高性能。 |
| bufferSize | int | 当`bufferedIO`为`true`时，表示缓冲区大小，默认为8192字节。 |
| createOnDemand | boolean | appender按需创建文件。当日志事件通过所有过滤器并被路由到该appender时，appender才创建该文件。默认为`false`。 |
| filter | Filter | 过滤器，用于确定事件是否应由此Appender处理。使用CompositeFilter可以组合多个Filter。 |
| fileName | String | 要写入的文件名。如果文件或其任何父目录不存在，将创建它们。 |
| filePattern | String | 归档日志文件的文件名模式。模式的格式取决于所使用的RolloverStrategy。DefaultRolloverStrategy使用与[SimpleDateFormat](http://download.oracle.com/javase/6/docs/api/java/text/SimpleDateFormat.html)兼容的日期/时间模式，和/或表示整数计数器的%i。该模式还支持运行时插值，所以在模式中可以包含Lookups（如[DateLookup](./lookups.md#DateLookup)）。 |
| immediateFlush | boolean | 当为`true`（默认值）时，每次写入后都会进行刷新。这将保证数据写入磁盘，但可能会影响性能。<br><br> 每次写入后刷新入盘仅在使用同步logger的appender时才有效。异步logger和appender将在一批事件结束时自动刷新，即使immediateFlush设置为false。这也保证了数据被写入磁盘，但效率更高。 |
| layout | Layout | 用于格式化LogEvent的Layout。如果没有指定Layout，将使用“%m%n”作为默认模式layout。 |
| name | String | 该Appender的名字。 |
| policy | TriggeringPolicy | 用于确定是否执行翻转的规则。 |
| strategy | RolloverStrategy | 用于确定归档文件的名称和位置的策略。 |
| ignoreExceptions | boolean | 默认值为`true`，将忽略在内部记录事件时遇到的异常。当设置为`false`异常将被传播回调用者中。将此Appender包装在[FailoverAppender](./appenders.md#FailoverAppender)中时，必须将其设置为`false`。 |

### Triggering Policies

#### CompositeTriggeringPolicy

`CompositeTriggeringPolicy`组合多个触发策略（triggering policies），只要有其中的一个策略返回true，则结果为true。CompositeTriggeringPolicy简单地通过在`Policies`元素节点中包括其他策略来配置。

例如，以下XML片段定义了3个策略，当JVM启动时，日志大小达到20M时以及当前日期与日志的开始日期不匹配时翻转日志文件。

```xml
<Policies>
  <OnStartupTriggeringPolicy />
  <SizeBasedTriggeringPolicy size="20 MB" />
  <TimeBasedTriggeringPolicy />
</Policies>
```

#### CronTriggeringPolicy

`CronTriggeringPolicy`基于cron表达式触发翻转。

CronTriggeringPolicy参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| schedule | String | cron表达式。表达式与Quartz调度程序中的表达式相同。有关表达式的完整描述，请参阅[CronExpression](https://logging.apache.org/log4j/2.x/log4j-core/apidocs/org/apache/logging/log4j/core/util/CronExpression.html)。 |
| evaluateOnStartup | boolean | 在启动时，将根据文件的最后修改时间戳评估cron表达式。如果cron表达式通过该时间和当前时间进行评估，判断出应该发生翻转，则文件将立即翻转。 |

#### OnStartupTriggeringPolicy

如果日志文件比当前JVM的开始时间更早且满足或超过最小文件大小，则`OnStartupTriggeringPolicy`策略将导致翻转。

OnStartupTriggeringPolicy参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| minSize | long | 文件必须翻转的最小尺寸。无论文件大小是多少，都会导致翻转。默认值为1，这可避免翻转空文件。 |

#### SizeBasedTriggeringPolicy

一旦文件达到指定的大小，`SizeBasedTriggeringPolicy`就会导致翻转。大小可以指定，后缀单位可为KB，MB或GB，例如`20MB`。

#### TimeBasedTriggeringPolicy

一旦日期/时间模式对当前的文件无效了，`TimeBasedTriggeringPolicy`会导致翻转。此策略可配`interval`属性，基于时间模式和`modulate`布尔属性，该属性表示了应该发生翻转的频率。

TimeBasedTriggeringPolicy参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| interval | integer | 根据日期格式中最小的时间单位，应该多长时间翻转一次。例如，日期模式最小时间单位为小时，配置间隔为4，则表示每4小时发生翻转。 默认值为1。 |
| modulate | boolean | 指示是否应调整间隔（interval）以正确地在间隔边界发生下一次翻转。例如，如果模式为小时，当前时间为凌晨3点，间隔为4，则第一次翻转将在上午4点发生，接下来的时间将在上午8点，中午12点，下午4点等发生。 |

### Rollover Strategies

#### DefaultRolloverStrategy

默认的翻转策略可以同时使用RollingFileAppender在filePattern属性上指定的日期/时间模式和整数模式。如果日期/时间模式存在，它将被替换为当前日期和时间值。如果模式包含一个整数，它将在每次翻转时递增。如果在模式中同时包含日期/时间和整数，则在日期/时间模式的结果发生更改时才进行整数的递增。如果文件模式以“.gz”，“.zip”，“.bz2”，“.deflate”，“.pack200”或“.xz”结尾，则使用与后缀匹配的压缩方案。格式bzip2，Deflate，Pack200和XZ需要依赖[Apache Commons Compress](http://commons.apache.org/proper/commons-compress/)。另外XZ需要依赖[XZ Java](http://tukaani.org/xz/java.html)。该模式还可以包含运行时解析的lookup，如下面的示例所示。

默认的翻转策略的递增计数器有三种变体。第一个是“固定窗口（fixed window）”策略。为了说明它的工作原理，假设min属性设置为1，max属性设置为3，文件名为“foo.log”，文件名模式为“foo-%i.log”。

| 翻转的次数 | 输出目标 | 归档日志文件 | 描述 |
| ---- | ---- | ---- | ---- |
| 0 | foo.log | - | 初始文件，所有日志都记录在此。 |
| 1 | foo.log | foo-1.log | 在第一次翻转期间，foo.log被重命名为foo-1.log。创建一个新的foo.log文件并开始接受写入。 |
| 2 | foo.log | foo-1.log，foo-2.log | 在第二次翻转期间，foo-1.log被重命名为foo-2.log，foo.log被重命名为foo-1.log。创建一个新的foo.log文件并开始接受写入。 |
| 3 | foo.log | foo-1.log，foo-2.logg，foo-3.log | 在第三次翻转期间，foo-2.log被重命名为foo-3.log，foo-1.log被重命名为foo-2.log，foo.log被重命名为foo-1.log。创建一个新的foo.log文件并开始接受写入。 |
| 4 | foo.log | foo-1.log，foo-2.logg，foo-3.log | 在第四个和随后的翻转中，foo-3.log被删除，foo-2.log被重命名为foo-3.log，foo-1.log被重命名为foo-2.log，foo.log被重命名为foo-1.log。创建一个新的foo.log文件并开始接受写入。  |

比较下面的，当fileIndex属性设置为“max”，而其他所有设置都相同时，将有以下的效果。

| 翻转的次数 | 输出目标 | 归档日志文件 | 描述 |
| ---- | ---- | ---- | ---- |
| 0 | foo.log | - | 初始文件，所有日志都记录在此。 |
| 1 | foo.log | foo-1.log | 在第一次翻转期间，foo.log被重命名为foo-1.log。创建一个新的foo.log文件并开始接受写入。 |
| 2 | foo.log | foo-1.log，foo-2.log | 在第二次翻转期间，foo.log被重命名为foo-2.log。创建一个新的foo.log文件并开始接受写入。 |
| 3 | foo.log | foo-1.log，foo-2.logg，foo-3.log | 在第三次翻转期间，foo.log被重命名为foo-3.log。创建一个新的foo.log文件并开始接受写入。 |
| 4 | foo.log | foo-1.log，foo-2.logg，foo-3.log | 在第四个和随后的翻转中，foo-1.log被删除，foo-2.log被重命名为foo-1.log，foo-3.log被重命名为foo-2.log，foo.log被重命名为foo-3.log。创建一个新的foo.log文件并开始接受写入。  |

最后，从版本2.8起，如果fileIndex属性设置为“nomax”，那么min和max将被忽略，文件编号将递增1，并且每次翻转将递增更大的值，没有最大文件数。

DefaultRolloverStrategy参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| fileIndex | String | 如果设置为“max”（默认值），索引号较高的文件比索引号较小的文件记录更新。如果设置为“min”，文件重命名和计数器将遵循上述“固定窗口（Fixed Window）”策略。 |
| min | integer | 计数器的最小值。默认值为1。 |
| max | integer | 计数器的最大值，默认值为7。一旦达到这个值，旧的归档文件将在随后的翻转中被删除。 |
| compressionLevel | integer | 设置压缩级别，0-9，其中0=无，1=最佳速度，而直到9=最佳压缩。只适用于ZIP格式。 |

#### DirectWriteRolloverStrategy

DirectWriteRolloverStrategy会将日志事件直接写入由文件模式定义的文件中。使用此策略文件不会被重命名。如果基于大小的触发策略会在指定的时间段内写入多个文件，它们将从编号1开始，并持续递增，直到出现基于时间的翻转。

警告：如果文件模式具有指示压缩的后缀，那么当应用程序关闭时，当前文件将不被压缩。此外，如果更新时间使得文件模式匹配不中当前文件，则它也不会在启动时被压缩。

DirectWriteRolloverStrategy参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| maxFiles | String | 在与文件模式匹配的时间段内允许的最大文件数。如果文件数超过，最旧的文件将被删除。如果指定，则该值必须大于1。如果该值小于零或省略，则文件数不受限制。 |
| compressionLevel | integer | 设置压缩级别，0-9，其中0=无，1=最佳速度，而直到9=最佳压缩。只适用于ZIP格式。 |

#### 示例

下面是一个使用RollingFileAppender的示例配置，它同时使用基于时间和大小的触发策略，每天翻转归档，且文件大小大于250MB时也会翻转归档，在当天（1-7）中创建多达7个归档，这些存档位于基于当前年份和月份的目录中，并使用gzip压缩每个存档：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <RollingFile name="RollingFile" fileName="logs/app.log"
                 filePattern="logs/$${date:yyyy-MM}/app-%d{MM-dd-yyyy}-%i.log.gz">
      <PatternLayout>
        <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
      </PatternLayout>
      <Policies>
        <TimeBasedTriggeringPolicy />
        <SizeBasedTriggeringPolicy size="250 MB"/>
      </Policies>
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
    </Root>
  </Loggers>
</Configuration>
```

第二个例子显示的翻转策略，最多可以保留20个文件，然后再删除它们。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <RollingFile name="RollingFile" fileName="logs/app.log"
                 filePattern="logs/$${date:yyyy-MM}/app-%d{MM-dd-yyyy}-%i.log.gz">
      <PatternLayout>
        <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
      </PatternLayout>
      <Policies>
        <TimeBasedTriggeringPolicy />
        <SizeBasedTriggeringPolicy size="250 MB"/>
      </Policies>
      <DefaultRolloverStrategy max="20"/>
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
    </Root>
  </Loggers>
</Configuration>
```

下面的示例配置，它同时使用基于时间和大小的触发策略，将当文件大小大于250MB时会翻转归档，且每6小时也会翻转归档，在当天（1-7）中创建多达7个归档，这些存档位于基于当前年份和月份的目录中，并将会使用gzip压缩每个存档：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <RollingFile name="RollingFile" fileName="logs/app.log"
                 filePattern="logs/$${date:yyyy-MM}/app-%d{yyyy-MM-dd-HH}-%i.log.gz">
      <PatternLayout>
        <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
      </PatternLayout>
      <Policies>
        <TimeBasedTriggeringPolicy interval="6" modulate="true"/>
        <SizeBasedTriggeringPolicy size="250 MB"/>
      </Policies>
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
    </Root>
  </Loggers>
</Configuration>
```

此示例配置使用基于cron和基于大小的触发策略的RollingFileAppender，使用Direct Write方式（fileName没有配置），归档文件数量无限制。cron触发器导致每小时翻转一次，而文件大小限制为250MB：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <RollingFile name="RollingFile" filePattern="logs/app-%d{yyyy-MM-dd-HH}-%i.log.gz">
      <PatternLayout>
        <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
      </PatternLayout>
      <Policies>
        <CronTriggeringPolicy schedule="0 0 * * * ?"/>
        <SizeBasedTriggeringPolicy size="250 MB"/>
      </Policies>
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
    </Root>
  </Loggers>
</Configuration>
```

此示例配置与之前的配置相同，但使用将每小时保存的文件数限制为10：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <RollingFile name="RollingFile" filePattern="logs/app-%d{yyyy-MM-dd-HH}-%i.log.gz">
      <PatternLayout>
        <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
      </PatternLayout>
      <Policies>
        <CronTriggeringPolicy schedule="0 0 * * * ?"/>
        <SizeBasedTriggeringPolicy size="250 MB"/>
      </Policies>
      <DirectWriteRolloverStrategy maxFiles="10"/>
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
    </Root>
  </Loggers>
</Configuration>
```

#### 日志存档保留策略：翻转时删除（Delete）操作

Log4j-2.5引入了一个`Delete`操作，使用户可以更有效地控制在翻转时删除的文件，而不是使用DefaultRolloverStrategy的`max`属性进行删除。Delete操作允许用户配置一个或多个条件，用于选择相对于基本目录要删除的文件。

请注意，它可以删除任何文件，而不仅仅是翻转的日志文件，因此请谨慎使用此操作！使用testMode参数可以测试您的配置，而不会意外删除了错误的文件。

Delete参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| basePath | String | _必需。_ 需扫描要删除文件的基本路径。 |
| maxDepth | int | 要访问的目录的最大层级数。值为0表示仅访问起始文件（基本路径本身），除非被安全管理器拒绝。Integer.MAX_VALUE的值表示应该访问所有级别。默认为1，仅指指定的基本目录中的文件。 |
| followLinks | boolean | 是否访问符号链接，默认值为`false`。 |
| testMode | boolean | 如果为true，则文件不会被删除，而是在INFO级别将消息打印到[status logger](./configuration.md#StatusMessages)里。使用它来进行演习，以测试配置是否按预期工作。默认为`false`。 |
| pathSorter | PathSorter | 实现[PathSorter](https://logging.apache.org/log4j/2.x/log4j-core/apidocs/org/apache/logging/log4j/core/appender/rolling/action/PathSorter.html)接口的插件，用于在选择要删除的文件之前对文件进行排序。默认是按最近修改时间倒序。 |
| pathConditions | PathCondition\[\] | _如果没有指定ScriptCondition，则为必需。配置一个或多个PathCondition元素。_ <br><br>如果指定了多个条件，则所有的条件都accept了该路径才可以删除。条件可以嵌套，在这种情况下，仅当外部条件accept该路径时才会评估内部条件。如果条件不嵌套，则可以按任何顺序进行评估。<br><br>条件也可以通过使用`IfAll`，`IfAny`和`IfNot`复合条件进行组合，实现AND，OR和NOT逻辑运算。<br><br>用户可以创建自定义条件或使用内置条件：<br><br> * **IfFileName** - 判断文件路径（相对于基本路径）是否与[正则表达式](https://docs.oracle.com/javase/7/docs/api/java/util/regex/Pattern.html)或[glob](https://docs.oracle.com/javase/7/docs/api/java/nio/file/FileSystem.html#getPathMatcher%28java.lang.String%29)匹配。<br> * **IfLastModified** - 判断文件修改时间是否与指定的[持续时间](https://logging.apache.org/log4j/2.x/log4j-core/apidocs/org/apache/logging/log4j/core/appender/rolling/action/Duration.html#parse%28java.lang.CharSequence%29)相同或更早。<br> **IfAccumulatedFileCount**：判断路径目录树下的文件数是否已超过某个计数阈值。<br> **IfAccumulatedFileSize**：判断路径目录树下的累积文件大小是否已超过某个阈值。<br> **IfAll**：如果所有嵌套条件都accept（逻辑AND），则accept该路径。嵌套条件可以按任何顺序进行评估。<br> **IfAny**：如果其中一个嵌套条件accept（逻辑OR），则accept该路径。嵌套条件可以按任何顺序进行评估。<br> **IfNot**：如果嵌套条件不accept（逻辑NOT），则accept该路径。 |
| scriptCondition | ScriptCondition | _如果没有指定PathConditions，则为必需。配置ScriptCondition元素。_ <br><br>ScriptCondition应包含一个Script，ScriptRef或ScriptFile元素，用于指定要执行的逻辑。（有关配置ScriptFiles和ScriptRefs的更多示例，请参阅[ScriptFilter](./filters.md#Script)文档。）<br><br>给脚本传递了诸多参数（见以下的具体说明），包括在基本路径下找到的路径列表（最多为maxDepth），并且必须返回要删除的路径列表。 |

IfFileName条件参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| glob | String | _如果没有指定regex，则为必需。_ 使用类似于正则表达式但具有[更简单语法](https://docs.oracle.com/javase/7/docs/api/java/nio/file/FileSystem.html#getPathMatcher%28java.lang.String%29)的有限模式语言来匹配相对路径（相对于基本路径）。 |
| regex | String | _如果没有指定glob，则为必需。_ 使用由[Pattern](https://docs.oracle.com/javase/7/docs/api/java/util/regex/Pattern.html)类定义的正则表达式来匹配相对路径（相对于基本路径）。 |
| nestedConditons | PathCondition\[\] | 一组可选的嵌套PathCondition。如果有嵌套条件，则所有的条件都accept了该路径（IfAll）才可以删除。仅当外部条件accept了文件（如果路径名称匹配）时，才会评估嵌套条件。 |

IfLastModified条件参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| age | String | _必需。_ 指定持续时间。判断文件修改时间是否与指定的[持续时间](https://logging.apache.org/log4j/2.x/log4j-core/apidocs/org/apache/logging/log4j/core/appender/rolling/action/Duration.html#parse%28java.lang.CharSequence%29)相同或更早。 |
| nestedConditons | PathCondition\[\] | 一组可选的嵌套PathCondition。如果有嵌套条件，则所有的条件都accept了该路径（IfAll）才可以删除。仅当外部条件accept了文件（如果文件已“年久”）时，才会评估嵌套条件。 |

IfAccumulatedFileCount条件参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| exceeds | int | _必需。_ 将删除文件的阈值计数。 |
| nestedConditons | PathCondition\[\] | 一组可选的嵌套PathCondition。如果有嵌套条件，则所有的条件都accept了该路径（IfAll）才可以删除。仅当外部条件accept了文件（如果已超过阈值计数）时，才会评估嵌套条件。 |

IfAccumulatedFileCount条件参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| exceeds | int | _必需。_ 将删除文件的阈值计数。 |
| nestedConditons | PathCondition\[\] | 一组可选的嵌套PathCondition。如果有嵌套条件，则所有的条件都accept了该路径（IfAll）才可以删除。仅当外部条件accept了文件（如果已超过阈值计数）时，才会评估嵌套条件。 |

IfAccumulatedFileSize条件参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| exceeds | int | _必需。_ 将删除文件的累积文件大小阈值。大小可以字节（bytes）指定，后缀可为KB，MB或GB，例如`20MB`。 |
| nestedConditons | PathCondition\[\] | 一组可选的嵌套PathCondition。如果有嵌套条件，则所有的条件都accept了该路径（IfAll）才可以删除。仅当外部条件accept了文件（如果已超过累积文件大小阈值）时，才会评估嵌套条件。 |

以下的示例配置，其中cron触发策略配置为在每一天午夜触发。根据当前年份和月份确定目录来进行存档。基本目录下与`*/app-*.log.gz`glob进行匹配，且文件最近修改时间为60天或以上，这些文件在翻转间会被删除。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Properties>
    <Property name="baseDir">logs</Property>
  </Properties>
  <Appenders>
    <RollingFile name="RollingFile" fileName="${baseDir}/app.log"
          filePattern="${baseDir}/$${date:yyyy-MM}/app-%d{yyyy-MM-dd}.log.gz">
      <PatternLayout pattern="%d %p %c{1.} [%t] %m%n" />
      <CronTriggeringPolicy schedule="0 0 0 * * ?"/>
      <DefaultRolloverStrategy>
        <Delete basePath="${baseDir}" maxDepth="2">
          <IfFileName glob="*/app-*.log.gz" />
          <IfLastModified age="60d" />
        </Delete>
      </DefaultRolloverStrategy>
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
    </Root>
  </Loggers>
</Configuration>
```

以下的示例配置，同时基于时间和大小的触发策略，将在当天（1-100）内创建多达100个归档，这些存档位于基于当前年份和月份的目录中，并使用gzip压缩每个存档，且每小时翻转。在每次翻转期间，此配置将删除与`*/app-*.log.gz`匹配的文件，并且是30天或更早的文件，但保留最近的100GB或最近的10个文件（以最先评估的条件（IfAny）为准）。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Properties>
    <Property name="baseDir">logs</Property>
  </Properties>
  <Appenders>
    <RollingFile name="RollingFile" fileName="${baseDir}/app.log"
          filePattern="${baseDir}/$${date:yyyy-MM}/app-%d{yyyy-MM-dd-HH}-%i.log.gz">
      <PatternLayout pattern="%d %p %c{1.} [%t] %m%n" />
      <Policies>
        <TimeBasedTriggeringPolicy />
        <SizeBasedTriggeringPolicy size="250 MB"/>
      </Policies>
      <DefaultRolloverStrategy max="100">
        <!--
        Nested conditions: the inner condition is only evaluated on files
        for which the outer conditions are true.
        -->
        <Delete basePath="${baseDir}" maxDepth="2">
          <IfFileName glob="*/app-*.log.gz">
            <IfLastModified age="30d">
              <IfAny>
                <IfAccumulatedFileSize exceeds="100 GB" />
                <IfAccumulatedFileCount exceeds="10" />
              </IfAny>
            </IfLastModified>
          </IfFileName>
        </Delete>
      </DefaultRolloverStrategy>
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
    </Root>
  </Loggers>
</Configuration>
```

ScriptCondition参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| script | Script, ScriptFile or ScriptRef | 指定要执行实际逻辑的Script元素。该脚本接受一份在基本路径下找到的路径列表参数，并必须返回`java.util.List<PathWithAttributes>`要被删除的路径列表。 另请参阅[ScriptFilter](./filters.md#Script)文档，了解可以如何配置ScriptFiles和ScriptRefs的示例。 |

Script参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| basePath | `java.nio.file.Path` | 扫描要删除的文件的目录。可用于相对pathList中的路径。 |
| pathList | `java.util.List<PathWithAttributes>` | 在basePath下找到的路径列表，直到指定的最大深度，按最近修改时间倒序。script可以自由修改并返回此列表。 |
| statusLogger | StatusLogger | StatusLogger可用于在脚本执行期间记录内部事件。 |
| configuration | Configuration | 拥有此ScriptCondition的配置。 |
| substitutor | StrSubstitutor | 用于替换查找变量的StrSubstitutor。 |
| ? | String | 在配置中声明的任何属性。 |

以下的示例配置，其中cron触发策略配置为在每天午夜触发。这些存档位于基于当前年份和月份的目录中，该脚本返回在第13个星期五的基础目录下翻转的文件列表。删除操作将删除脚本返回的所有文件。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="trace" name="MyApp" packages="">
  <Properties>
    <Property name="baseDir">logs</Property>
  </Properties>
  <Appenders>
    <RollingFile name="RollingFile" fileName="${baseDir}/app.log"
          filePattern="${baseDir}/$${date:yyyy-MM}/app-%d{yyyyMMdd}.log.gz">
      <PatternLayout pattern="%d %p %c{1.} [%t] %m%n" />
      <CronTriggeringPolicy schedule="0 0 0 * * ?"/>
      <DefaultRolloverStrategy>
        <Delete basePath="${baseDir}" maxDepth="2">
          <ScriptCondition>
            <Script name="superstitious" language="groovy">
              < ![CDATA[
                import java.nio.file.*;

                def result = [];
                def pattern = ~/\d*\/app-(\d*)\.log\.gz/;

                pathList.each { pathWithAttributes ->
                  def relative = basePath.relativize pathWithAttributes.path
                  statusLogger.trace 'SCRIPT: relative path=' + relative + " (base=$basePath)";

                  // remove files dated Friday the 13th

                  def matcher = pattern.matcher(relative.toString());
                  if (matcher.find()) {
                    def dateString = matcher.group(1);
                    def calendar = Date.parse("yyyyMMdd", dateString).toCalendar();
                    def friday13th = calendar.get(Calendar.DAY_OF_MONTH) == 13 \
                                  && calendar.get(Calendar.DAY_OF_WEEK) == Calendar.FRIDAY;
                    if (friday13th) {
                      result.add pathWithAttributes;
                      statusLogger.trace 'SCRIPT: deleting path ' + pathWithAttributes;
                    }
                  }
                }
                statusLogger.trace 'SCRIPT: returning ' + result;
                result;
              ]] >
            </Script>
          </ScriptCondition>
        </Delete>
      </DefaultRolloverStrategy>
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
    </Root>
  </Loggers>
</Configuration>
```

## RollingRandomAccessFileAppender

RollingRandomAccessFileAppender类似于标准的[RollingFileAppender](./appenders.md#RollingFileAppender)，除了它总是使用缓冲区（这不能关闭），且内部使用`ByteBuffer + RandomAccessFile`而不是`BufferedOutputStream`。在[性能测试](../others/performance.md#whichAppender)中，可以看到，与“bufferedIO=true”的FileAppender相比，有20-200%的性能提升。RollingRandomAccessFileAppender写入FileName参数中命名的文件，并根据TriggeringPolicy和RolloverPolicy滚动文件。与RollingFileAppender类似，RollingRandomAccessFileAppender使用RollingRandomAccessFileManager来实际执行文件I/O并执行翻转。不同配置的RollingRandomAccessFileAppender无法共享，但如果Manager可共享访问，则RollingRandomAccessFileManagers可以共享使用。例如，Servlet容器中的两个Web应用程序可各自拥有配置，且如果Log4j位于它们两者共同的ClassLoader中，则可以安全地写入同一个文件。

RollingRandomAccessFileAppender需要配置[TriggeringPolicy](./appenders.md#Triggering%20Policies)和[RolloverStrategy](./appenders.md#Rollover%20Strategies)。TriggeringPolicy确定是否应该执行翻转（rollover），而在RolloverStrategy定义了如何进行翻转（rollover）。如果没有配置RolloverStrategy，RollingRandomAccessFileAppender将使用[DefaultRolloverStrategy](./appenders.md#DefaultRolloverStrategy)。从log4j-2.5开始，可以配置DefaultRolloverStrategy在执行翻转（rollover）中[自定义删除操作](./appenders.md#CustomDeleteOnRollover)。

RollingRandomAccessFileAppender不支持文件锁定（file locking）。

RollingRandomAccessFileAppender参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| append | boolean | 当为`true`（默认值）时，记录将追加到文件的末尾。当设置为`false`时，文件将在新记录写入之前被清除。 |
| filter | Filter | 过滤器，用于确定事件是否应由此Appender处理。使用CompositeFilter可以组合多个Filter。 |
| fileName | String | 要写入的文件名。如果文件或其任何父目录不存在，将创建它们。 |
| filePattern | String | 归档日志文件的文件名模式。模式的格式取决于所使用的RolloverStrategy。DefaultRolloverStrategy使用与[SimpleDateFormat](http://download.oracle.com/javase/6/docs/api/java/text/SimpleDateFormat.html)兼容的日期/时间模式，和/或表示整数计数器的%i。该模式还支持运行时插值，所以在模式中可以包含Lookups（如[DateLookup](./lookups.md#DateLookup)）。 |
| immediateFlush | boolean | 当为`true`（默认值）时，每次写入后都会进行刷新。这将保证数据写入磁盘，但可能会影响性能。<br><br> 每次写入后刷新入盘仅在使用同步logger的appender时才有效。异步logger和appender将在一批事件结束时自动刷新，即使immediateFlush设置为false。这也保证了数据被写入磁盘，但效率更高。 |
| bufferSize | int | 当`bufferedIO`为`true`时，表示缓冲区大小，默认为262,144字节（256 * 1024）。 |
| layout | Layout | 用于格式化LogEvent的Layout。如果没有指定Layout，将使用“%m%n”作为默认模式layout。 |
| name | String | 该Appender的名字。 |
| policy | TriggeringPolicy | 用于确定是否执行翻转的规则。 |
| strategy | RolloverStrategy | 用于确定归档文件的名称和位置的策略。 |
| ignoreExceptions | boolean | 默认值为`true`，将忽略在内部记录事件时遇到的异常。当设置为`false`异常将被传播回调用者中。将此Appender包装在[FailoverAppender](./appenders.md#FailoverAppender)中时，必须将其设置为`false`。 |

### Triggering Policies

见上面的RollingFileAppender Triggering Policies。

### Rollover Strategies

见上面的RollingFileAppender Rollover Strategies。

下面是一个使用RollingRandomAccessFileAppender的示例配置，它同时使用基于时间和大小的触发策略，使用当前年份和月份作为目录，存储同一天（1-7）上最多可创建7个归档，并使用gzip压缩每个存档：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <RollingRandomAccessFile name="RollingRandomAccessFile" fileName="logs/app.log"
                 filePattern="logs/$${date:yyyy-MM}/app-%d{MM-dd-yyyy}-%i.log.gz">
      <PatternLayout>
        <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
      </PatternLayout>
      <Policies>
        <TimeBasedTriggeringPolicy />
        <SizeBasedTriggeringPolicy size="250 MB"/>
      </Policies>
    </RollingRandomAccessFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingRandomAccessFile"/>
    </Root>
  </Loggers>
</Configuration>
```

第二个例子，最多可以保留20个文件，然后再删除它们。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <RollingRandomAccessFile name="RollingRandomAccessFile" fileName="logs/app.log"
                 filePattern="logs/$${date:yyyy-MM}/app-%d{MM-dd-yyyy}-%i.log.gz">
      <PatternLayout>
        <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
      </PatternLayout>
      <Policies>
        <TimeBasedTriggeringPolicy />
        <SizeBasedTriggeringPolicy size="250 MB"/>
      </Policies>
      <DefaultRolloverStrategy max="20"/>
    </RollingRandomAccessFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingRandomAccessFile"/>
    </Root>
  </Loggers>
</Configuration>
```

下面的示例配置，它同时使用基于时间和大小的触发策略，使用当前年份和月份作为目录，存储同一天（1-7）上最多可创建7个归档，并使用gzip压缩每个存档，且每6小时翻转日志文件：

```xml
?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <RollingRandomAccessFile name="RollingRandomAccessFile" fileName="logs/app.log"
                 filePattern="logs/$${date:yyyy-MM}/app-%d{yyyy-MM-dd-HH}-%i.log.gz">
      <PatternLayout>
        <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
      </PatternLayout>
      <Policies>
        <TimeBasedTriggeringPolicy interval="6" modulate="true"/>
        <SizeBasedTriggeringPolicy size="250 MB"/>
      </Policies>
    </RollingRandomAccessFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingRandomAccessFile"/>
    </Root>
  </Loggers>
</Configuration>
```

## RoutingAppender

RoutingAppender评估LogEvents，然后将它们路由到下级Appender。目标Appender可能是先前配置的appender然后再通过其名称进行引用，也可以按需动态创建Appender。RoutingAppender应在其引用的任何Appenders之后进行配置，以使其正常关闭。

您还可以使用脚本配置RoutingAppender：您可以在appender启动时或者为日志事件选择路由时运行脚本。

RoutingAppender参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| filter | Filter | 过滤器，用于确定事件是否应由此Appender处理。使用CompositeFilter可以组合多个Filter。 |
| name | String | 该Appender的名字。 |
| rewritePolicy | RewritePolicy | 将重写LogEvent的RewritePolicy。 |
| routes | Routes | 包含一个或多个路由声明，以确定选择Appender的条件。 |
| script | Script | 当Log4j启动RoutingAppender时运行此脚本，并返回一个String Route键以确定选择具体的路由。脚本传递变量见以下具体描述。 |
| ignoreExceptions | boolean | 默认值为`true`，将忽略在内部记录事件时遇到的异常。当设置为`false`异常将被传播回调用者中。将此Appender包装在[FailoverAppender](./appenders.md#FailoverAppender)中时，必须将其设置为`false`。 |

RoutingAppender Script参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| configuration | Configuration | 当前的Configuration。 |
| staticVariables | Map | 在此appender实例的所有脚本之间共享的Map信息。跟Routes Script里的是同一个map。 |

在此示例中，脚本把“ServiceWindows”路由成为Windows上的默认路由，并所有其他操作系统使用“ServiceOther”。请注意，ListAppender是只是作为示例的appender之一，任何appender都可以使用。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN" name="RoutingTest">
  <Appenders>
    <Routing name="Routing">
      <Script name="RoutingInit" language="JavaScript"><![CDATA[
        importPackage(java.lang);
        System.getProperty("os.name").search("Windows") > -1 ? "ServiceWindows" : "ServiceOther";]]>
      </Script>
      <Routes>
        <Route key="ServiceOther">
          <List name="List1" />
        </Route>
        <Route key="ServiceWindows">
          <List name="List2" />
        </Route>
      </Routes>
    </Routing>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="Routing" />
    </Root>
  </Loggers>
</Configuration>
```

### Routes

Routes节点有一个名为“pattern”的属性。该pattern可以使用所有注册好的Lookups，结果用于选择Route。每个Route可以配置一个key。如果key与pattern的结果相匹配，那么将选择该Route。如果在Route上没有指定key，那么该Route是默认的。只有一条路由可以配置为默认Route。

Routes节点可能包含Script子节点。如果指定，则为每个日志事件运行该Script，并返回要使用的String Route key。

您必须指定pattern属性或Script元素，但不能同时指定两者。

每个Route必须引用一个Appender。如果Route包含ref属性，那么Route将引用在配置中定义的Appender。如果Route内含Appender定义，则Appender将在RoutingAppender的上下文中创建，并且每当通过Route引用匹配中Appender名称时，它可以被重用。

此Script传递以下变量：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| configuration | Configuration | 当前的Configuration。 |
| staticVariables | Map | 在此appender实例的所有脚本之间共享的Map信息。跟Routes Script里的是同一个map。 |
| logEvent | LogEvent | 日志事件。 |

在此示例中，每个日志事件都运行Script，并根据“AUDIT”的Marker选择路由。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN" name="RoutingTest">
  <Appenders>
    <Console name="STDOUT" target="SYSTEM_OUT" />
    <Flume name="AuditLogger" compress="true">
      <Agent host="192.168.10.101" port="8800"/>
      <Agent host="192.168.10.102" port="8800"/>
      <RFC5424Layout enterpriseNumber="18060" includeMDC="true" appName="MyApp"/>
    </Flume>
    <Routing name="Routing">
      <Routes>
        <Script name="RoutingInit" language="JavaScript"><![CDATA[
          if (logEvent.getMarker() != null && logEvent.getMarker().isInstanceOf("AUDIT")) {
                return "AUDIT";
            } else if (logEvent.getContextMap().containsKey("UserId")) {
                return logEvent.getContextMap().get("UserId");
            }
            return "STDOUT";]]>
        </Script>
        <Route>
          <RollingFile
              name="Rolling-${mdc:UserId}"
              fileName="${mdc:UserId}.log"
              filePattern="${mdc:UserId}.%i.log.gz">
            <PatternLayout>
              <pattern>%d %p %c{1.} [%t] %m%n</pattern>
            </PatternLayout>
            <SizeBasedTriggeringPolicy size="500" />
          </RollingFile>
        </Route>
        <Route ref="AuditLogger" key="AUDIT"/>
        <Route ref="STDOUT" key="STDOUT"/>
      </Routes>
      <IdlePurgePolicy timeToLive="15" timeUnit="minutes"/>
    </Routing>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="Routing" />
    </Root>
  </Loggers>
</Configuration>
```

### 清除策略（Purge Policy）

RoutingAppender可以配置PurgePolicy，其目的是停止并删除由RoutingAppender动态创建的休眠（再也没有被使用了）Appender。Log4j目前提供了唯一的IdlePurgePolicy可用于清理Appender的PurgePolicy。IdlePurgePolicy接受2个属性; timeToLive（这是Appender在没有任何日志事件发送给它的生存时间），timeUnit（timeUnit是与timeToLive属性一起使用，以`java.util.concurrent.TimeUnit`的String形式来表示）。

以下是使用RoutingAppender将所有Audit事件路由到FlumeAppender的示例配置，其他所有事件将路由到特定事件类型（sd:type）的RollingFileAppender上。请注意，这里根需创建了RollingFileAppender，并预先定义了AuditAppender。这里定义了15分钟的清除策略（即特定事件类型的RollingFileAppender最长只能不工作15分钟）。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <Flume name="AuditLogger" compress="true">
      <Agent host="192.168.10.101" port="8800"/>
      <Agent host="192.168.10.102" port="8800"/>
      <RFC5424Layout enterpriseNumber="18060" includeMDC="true" appName="MyApp"/>
    </Flume>
    <Routing name="Routing">
      <Routes pattern="$${sd:type}">
        <Route>
          <RollingFile name="Rolling-${sd:type}" fileName="${sd:type}.log"
                       filePattern="${sd:type}.%i.log.gz">
            <PatternLayout>
              <pattern>%d %p %c{1.} [%t] %m%n</pattern>
            </PatternLayout>
            <SizeBasedTriggeringPolicy size="500" />
          </RollingFile>
        </Route>
        <Route ref="AuditLogger" key="Audit"/>
      </Routes>
      <IdlePurgePolicy timeToLive="15" timeUnit="minutes"/>
    </Routing>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="Routing"/>
    </Root>
  </Loggers>
</Configuration>
```

## SMTPAppender

当发生特定的日志记录事件时，特别是在发生error或fatal时，发送电子邮件。

此电子邮件中传送的日志记录事件数取决于 **BufferSize** 选项的值。`SMTPAppender`只保留`BufferSize`个记录事件在其循环缓冲区中。这就要求将内存占用保持在合理的水平，同时仍然要注意给邮件信息提供有用的应用上下文。缓冲区中的所有事件都包含在电子邮件中。在触发电子邮件的事件之前，缓冲区包含最近级别TRACE到WARN的事件。

默认行为是在记录ERROR或更严重事件时触发发送电子邮件，并将其格式化为HTML。电子邮件的发送可以通过在Appender上设置一个或多个filters进行控制。与其他Appender一样，可以通过为Appender指定Layout来控制格式。

SMTPAppender参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| bcc | String | 逗号分隔的BCC电子邮件地址列表。 |
| cc | String | 逗号分隔的CC电子邮件地址列表。 |
| bufferSize | integer | 要缓冲的包含在消息中的最大日志事件数。默认为512。 |
| filter | Filter | 过滤器，用于确定事件是否应由此Appender处理。使用CompositeFilter可以组合多个Filter。 |
| from | String | 发送者的电子邮件地址。 |
| layout | Layout | 用于格式化LogEvent的Layout。如果没有指定Layout，将使用[HTML Layout](./layouts.md#HTMLLayout)作为默认模式layout。 |
| name | String | 该Appender的名字。 |
| replyTo | String | 以逗号分隔的回复电子邮件地址列表。 |
| smtpDebug | boolean | 当设置为true时，可以在STDOUT上启用会话调试。默认为false。 |
| smtpHost | String | 要发送到的SMTP主机名。此参数是必需的。 |
| smtpPassword | String | 针对SMTP服务器进行身份验证所需的密码。 |
| smtpPort | integer | 要发送的SMTP端口。 |
| smtpProtocol | String | SMTP传输协议（如“smtps”，默认为“smtp”）。 |
| smtpUsername | String | 用于SMTP服务器验证的用户名。 |
| ignoreExceptions | boolean | 默认值为`true`，将忽略在内部记录事件时遇到的异常。当设置为`false`异常将被传播回调用者中。将此Appender包装在[FailoverAppender](./appenders.md#FailoverAppender)中时，必须将其设置为`false`。 |
| to | String | 逗号分隔的收件人电子邮件地址列表。 |

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <SMTP name="Mail" subject="Error Log" to="errors@logging.apache.org" from="test@logging.apache.org"
          smtpHost="localhost" smtpPort="25" bufferSize="50">
    </SMTP>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="Mail"/>
    </Root>
  </Loggers>
</Configuration>
```

## ScriptAppenderSelector

配置解析后，`ScriptAppenderSelector` appender会调用`Script`来计算一个appender名。然后Log4j使用`ScriptAppenderSelector`的名字，并创建一个列举在`AppenderSet`里的appender。配置后，Log4j会忽略`ScriptAppenderSelector`。Log4j仅从配置树构建一个选定的appender，其他`AppenderSet`子节点将被忽略。

在以下示例中，脚本返回名字。appender名称是`ScriptAppenderSelector`的名字，而不是所选的appender的名字，在本示例中为“SelectIt”。

```xml
<Configuration status="WARN" name="ScriptAppenderSelectorExample">
  <Appenders>
    <ScriptAppenderSelector name="SelectIt">
      <Script language="JavaScript"><![CDATA[
        importPackage(java.lang);
        System.getProperty("os.name").search("Windows") > -1 ? "MyCustomWindowsAppender" : "MySyslogAppender";]]>
      </Script>
      <AppenderSet>
        <MyCustomWindowsAppender name="MyAppender" ... />
        <SyslogAppender name="MySyslog" ... />
      </AppenderSet>
    </ScriptAppenderSelector>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="SelectIt" />
    </Root>
  </Loggers>
</Configuration>
```

## SocketAppender

`SocketAppender`是一个OutputStreamAppender，它将其输出写入由主机和端口指定的远程目标上。数据可以通过TCP或UDP发送，并且可以以任何格式发送。默认格式是序列化的LogEvent。Log4j 2包含一个SocketServer，它能够接收序列化的LogEvent并通过服务器上的日志记录系统进行路由。您可使用SSL进行安全通信。

SocketAppender参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| name | String | 该Appender的名字。 |
| host | String | 用于接收日志事件的系统名称或地址。此参数是必需的。 |
| port | integer | 侦听的主机端口。此参数是必需的。 |
| protocol | String | “TCP”（默认），“SSL”或“UDP”。 |
| SSL | SslConfiguration | 包含KeyStore和TrustStore的配置。 |
| filter | Filter | 过滤器，用于确定事件是否应由此Appender处理。使用CompositeFilter可以组合多个Filter。 |
| immediateFail | boolean | 当设置为true时，日志事件不会等待尝试重连，如果socket不可用，将立即失败。 |
| immediateFlush | boolean | 当设置为true（默认）时，每次写入后都会进行刷新。这将保证数据写入磁盘，但可能会影响性能。 |
| bufferedIO | boolean | 当为`true`（默认值）时，记录将被写入缓冲区，当缓冲区满或者如果设置了`immediateFlush`时，数据将被写入socket里。 |
| bufferSize | int | 当`bufferedIO`为`true`时，表示缓冲区大小，默认为8192字节。 |
| layout | Layout | 用于格式化LogEvent的Layout。SerializedLayout作为默认模式layout。 |
| reconnectionDelayMillis | integer | 如果设置为大于0的值，则在等待指定的毫秒数后，SocketManager将尝试重连服务器。如果重连失败，则会抛出异常（如果`ignoreExceptions`设置为`false`，则可以由应用程序捕获异常）。 |
| connectionTimeoutMillis | integer | 连接超时（以毫秒为单位）。默认值为0（不超时，如Socket.connect()方法）。 |
| ignoreExceptions | boolean | 默认值为`true`，将忽略在内部记录事件时遇到的异常。当设置为`false`异常将被传播回调用者中。将此Appender包装在[FailoverAppender](./appenders.md#FailoverAppender)中时，必须将其设置为`false`。 |

非安全的TCP配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <Socket name="socket" host="localhost" port="9500">
      <SerializedLayout />
    </Socket>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="socket"/>
    </Root>
  </Loggers>
</Configuration>
```

安全的SSL配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <Socket name="socket" host="localhost" port="9500">
      <SerializedLayout />
      <SSL>
        <KeyStore location="log4j2-keystore.jks" password="changeme"/>
        <TrustStore location="truststore.jks" password="changeme"/>
      </SSL>
    </Socket>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="socket"/>
    </Root>
  </Loggers>
</Configuration>
```

## SyslogAppender

`SyslogAppender`是一个`SocketAppender`，它将其输出写入由主机和端口指定的远程目标，格式符合BSD Syslog格式或RFC 5424格式。数据可以通过TCP或UDP发送。

SyslogAppender参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| advertise | boolean | 指示appender是否应该被通告（advertised）。 |
| appName | String | 在RFC 5424系统日志记录中用作APP-NAME的值。 |
| charset | String | 将syslog字符串转换为字节数组时使用的字符集。字符串必须是有效的[字符集](http://download.oracle.com/javase/6/docs/api/java/nio/charset/Charset.html)。如果未指定，将使用默认系统字符集。 |
| connectionTimeoutMillis | integer | 连接超时（以毫秒为单位）。默认值为0（不超时，如Socket.connect()方法）。 |
| enterpriseNumber | integer | IANA企业编号，如[RFC 5424](http://tools.ietf.org/html/rfc5424#section-7.2.2)所述。 |
| filter | Filter | 过滤器，用于确定事件是否应由此Appender处理。使用CompositeFilter可以组合多个Filter。 |
| facility | String | 该facility用于尝试对消息进行分类。facility选项必须设置为“KERN”，“USER”，“MAIL”，“DAEMON”，“AUTH”，“SYSLOG”，“LPR”，“NEWS”，“UUCP”，“CRON” “LOCAL2”，“LOCAL3”，“LOCAL4”，“LOCAL5”，“LOCAL6”，“LOCAL2”，“LOCAL2”，“LOCAL2” ，或“LOCAL7”。这些值可以被指定为大写（upper case）或小写（lower case）字符。。 |
| format | String | 如果设置为“RFC5424”，则数据将按照RFC 5424进行格式化。否则，将格式化为BSD进行Syslog记录。请注意，虽然BSD Syslog记录需要1024字节或更短，但SyslogLayout不会截断它们。RFC5424Layout也不会截断记录，因为接收方必须接受最多2048个字节的记录，并且可以接受更长的记录。 |
| host | String | 用于接收日志事件的系统名称或地址。此参数是必需的。 |
| id | String | 根据RFC 5424进行格式化时使用的默认结构化数据标识（structured data id）。如果LogEvent包含一个StructuredDataMessage，则将使用Message中的id代替该值。 |
| ignoreExceptions | boolean | 默认值为`true`，将忽略在内部记录事件时遇到的异常。当设置为`false`异常将被传播回调用者中。将此Appender包装在[FailoverAppender](./appenders.md#FailoverAppender)中时，必须将其设置为`false`。 |
| immediateFail | boolean | 当设置为true时，日志事件不会等待尝试重连，如果socket不可用，将立即失败。 |
| immediateFlush | boolean | 当设置为true（默认）时，每次写入后都会进行刷新。这将保证数据写入磁盘，但可能会影响性能。 |
| includeMDC | boolean | 指示来自ThreadContextMap的数据是否将包含在RFC 5424 Syslog记录中。默认为true。 |
| layout | Layout | 覆盖`format`设置的自定义布局。 |
| loggerFields | List of KeyValuePairs | 允许任意的PatternLayout模式作为指定的ThreadContext字段; 没有默认指定。要使用，请包括一个`LoggerFields`嵌套节点，包含一个或多个`KeyValuePair`节点。每个`KeyValuePair`必须有一个key属性，它指定将用于标识MDC Structured Data元素中的字段的key名称，以及一个value属性，它指定用于解析值的PatternLayout模式。 |
| mdcExcludes | String | 从LogEvent中排除的mdc键的列表，以逗号分隔。这与`mdcIncludes`是互斥的。此属性仅适用于RFC 5424 syslog记录。 |
| mdcIncludes | String | 应该包含在LogEvent中的mdc键的列表，以逗号分隔。若列表中的键在MDC里不存在，则被忽略。此选项与`mdcExcludes`互斥。此属性仅适用于RFC 5424 syslog记录。 |
| mdcRequired | String | MDC中必须存在的mdc键的列表，以逗号分隔。如果键不存在，将抛出LoggingException。此属性仅适用于RFC 5424 syslog记录。 |
| mdcPrefix | String | 为了将其与事件属性区分开来，应该将其添加到每个MDC键前面的字符串。默认字符串为`mdc:`。此属性仅适用于RFC 5424 syslog记录。 |
| messageId | String | 在RFC 5424 syslog记录的MSGID字段中使用的默认值。 |
| name | String | 该Appender的名字。 |
| newLine | boolean | 如果为true，则会在syslog记录的末尾附加一个换行符。默认值为false。 |
| port | integer | 侦听的主机端口。此参数是必需的。 |
| protocol | String | “TCP”或“UDP”。此参数是必需的。 |
| SSL | SslConfiguration | 包含KeyStore和TrustStore的配置。 |
| reconnectionDelayMillis | integer | 如果设置为大于0的值，则在等待指定的毫秒数后，SocketManager将尝试重连服务器。如果重连失败，则会抛出异常（如果`ignoreExceptions`设置为`false`，则可以由应用程序捕获异常）。 |

syslogAppender配置示例，配置有两个`SyslogAppender`，一个使用BSD格式，一个使用RFC 5424。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <Syslog name="bsd" host="localhost" port="514" protocol="TCP"/>
    <Syslog name="RFC5424" format="RFC5424" host="localhost" port="8514"
            protocol="TCP" appName="MyApp" includeMDC="true"
            facility="LOCAL0" enterpriseNumber="18060" newLine="true"
            messageId="Audit" id="App"/>
  </Appenders>
  <Loggers>
    <Logger name="com.mycorp" level="error">
      <AppenderRef ref="RFC5424"/>
    </Logger>
    <Root level="error">
      <AppenderRef ref="bsd"/>
    </Root>
  </Loggers>
</Configuration>
```

对于SSL，该appender将其输出到远程主机和端口，通过SSL传输，格式符合BSD Syslog格式或RFC 5424格式。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <TLSSyslog name="bsd" host="localhost" port="6514">
      <SSL>
        <KeyStore location="log4j2-keystore.jks" password="changeme"/>
        <TrustStore location="truststore.jks" password="changeme"/>
      </SSL>
    </TLSSyslog>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="bsd"/>
    </Root>
  </Loggers>
</Configuration>
```

## ZeroMQ/JeroMQ Appender

ZeroMQ appender使用[JeroMQ](https://github.com/zeromq/jeromq)库将日志事件发送到一个或多个ZeroMQ端点。

这是一个简单的JeroMQ配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration name="JeroMQAppenderTest" status="TRACE">
  <Appenders>
    <JeroMQ name="JeroMQAppender">
      <Property name="endpoint">tcp://*:5556</Property>
      <Property name="endpoint">ipc://info-topic</Property>
    </JeroMQ>
  </Appenders>
  <Loggers>
    <Root level="info">
      <AppenderRef ref="JeroMQAppender"/>
    </Root>
  </Loggers>
</Configuration>
```

JeroMQ参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| name | String | 该Appender的名字。 |
| layout | Layout | 用于格式化LogEvent的Layout。如果没有指定Layout，将使用“%m%n”作为默认模式layout。 |
| filter | Filter | 过滤器，用于确定事件是否应由此Appender处理。 |
| properties | Property\[\] | 一个或多个Property节点，名为`endpoint`。 |
| ignoreExceptions | boolean | 默认值为`true`，将忽略在内部记录事件时遇到的异常。当设置为`false`异常将被传播回调用者中。将此Appender包装在[FailoverAppender](./appenders.md#FailoverAppender)中时，必须将其设置为`false`。 |
| affinity | long | ZMQ_AFFINITY选项。默认值为0。 |
| backlog | long | ZMQ_BACKLOG选项。默认值为100。 |
| delayAttachOnConnect | boolean | ZMQ_DELAY_ATTACH_ON_CONNECT选项。默认值为false。 |
| identity | byte\[\] | ZMQ_IDENTITY选项。默认值为none。 |
| ipv4Only | boolean | ZMQ_IPV4ONLY选项。默认值为true。 |
| linger | long | ZMQ_LINGER选项。默认值为-1。 |
| maxMsgSize | long | ZMQ_MAXMSGSIZE选项。默认值为-1。 |
| rcvHwm | long | ZMQ_RCVHWM选项。默认值为1000。 |
| receiveBufferSize | long | ZMQ_RCVBUF选项。默认值为0。 |
| receiveTimeOut | int | ZMQ_RCVTIMEO选项。默认值为-1。 |
| reconnectIVL | long | ZMQ_RECONNECT_IVL选项。默认值为100。 |
| reconnectIVLMax | long | ZMQ_RECONNECT_IVL_MAX选项。默认值为0。 |
| sendBufferSize | long | ZMQ_SNDBUF选项。默认值为0。 |
| sendTimeOut | int | ZMQ_SNDTIMEO选项。默认值为-1。 |
| sndHwm | long | ZMQ_SNDHWM选项。默认值为1000。 |
| tcpKeepAlive | long | ZMQ_TCP_KEEPALIVE选项。默认值为-1。 |
| tcpKeepAliveCount | long | ZMQ_TCP_KEEPALIVE_CNT选项。默认值为-1。 |
| tcpKeepAliveIdle | long | ZMQ_TCP_KEEPALIVE_IDLE选项。默认值为-1。 |
| tcpKeepAliveInterval | long | ZMQ_TCP_KEEPALIVE_INTVL选项。默认值为-1。 |
| xpubVerbose | boolean | ZMQ_XPUB_VERBOSE选项。默认值为false。 |
