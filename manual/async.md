# Asynchronous Loggers for Low-Latency Logging - 低延迟异步记录器

异步日志记录通过在单独的线程中执行I/O操作来提高应用程序的性能。Log4j 2在这领域上还进一步做了一些改进。

* **异步记录器（Asynchronous Loggers）** 是Log4j 2中新增的一个功能。目的就是尽可能减少Logger.log的调用时间，让控制尽可能快速地返回到应用程序。您可以选择使所有Logger都异步，或混合使用同步和异步Logger。使所有Logger异步将提供最佳性能，而混合为您提供更强灵活性。
* **LMAX Disruptor技术**。异步记录器在内部使用[Disruptor](https://logging.apache.org/log4j/2.x/manual/async.html#UnderTheHood)，一个无锁的线程间通信库，用这个库来代替队列，实现更高的吞吐量和更低的延迟。
* Async Logger还做了一些提升，**异步追加器（Asynchronous Appenders）** 被增强，它实现批处理功能，按批（当队列为空时）把日志刷新到磁盘上。它和配置“immediateFlush=true”起到相同效果，即，所有接收的日志事件总是可以输出到磁盘上，但是却更高效，因为它无需在每个日志事件上进行磁盘操作。（Async Appenders内部使用ArrayBlockingQueue，且无需依赖disruptor jar包）。

## 权衡取舍（Trade-offs）

尽管异步日志记录提供了显著的性能优势，但在某些情况下，您可能需要选择同步日志记录。本节介绍异步日志记录的一些折衷。

### 优势

* 更高的峰值吞吐量。使用异步记录器（Asynchronous Logger），它比命名用同步记录器（Synchronous Logger）提升6~68倍的处理速率。

  这对于偶尔出现日志暴涨的应用程序特别有用。异步日志记录可以通过缩短日志记录的等待时间来防止或抑制延迟峰值。如果队列大小配置足够大以处理突发暴涨，异步日志记录可以有助于防止应用程序在突发暴涨时导致吞吐量下降（很多）。

* 降低日志响应时延。响应时延是在给定工作负载下调用Logger.log返回所花费的时间。异步记录器的时延相对于同步记录器或甚至是基于队列的异步追加器，它的时延始终更低。

### 缺陷

* 错误处理。如果在日志记录过程中发生问题并抛出异常，异步记录器或追加器很难把该问题通知到应用程序。可以通过配置`ExceptionHandler`来部分地减轻，不过这可能仍然不能覆盖所有情况。因此，如果日志是业务逻辑的一部分，例如，如果您使用Log4j作为审计日志框架，我们建议使用同步记录这些审计消息。（注意，您仍然可以混合地使用它们，除了使用同步日志记录来实现审计跟踪外，使用异步日志记录进行调试/跟踪日志信息。）

* 在极少数情景下，需要关注可变消息（mutable messages）。大多数下并不需要担心这个的。Log4j确保`logger.debug("My object is {}", myObject)`等日志消息使用的`myObject`参数的状态是在调用`logger.debug()`时那一刻的。即使`myObject`在后面被修改，日志消息也不会被更改掉。异步记录可变对象是安全的，因为Log4j内置的大多数[Message](https://logging.apache.org/log4j/2.x/log4j-api/apidocs/org/apache/logging/log4j/message/Message.html)实现都使用参数的快照。但有一些例外：[MapMessage](https://logging.apache.org/log4j/2.x/log4j-api/apidocs/org/apache/logging/log4j/message/MapMessage.html)和[StructuredDataMessage](https://logging.apache.org/log4j/2.x/log4j-api/apidocs/org/apache/logging/log4j/message/StructuredDataMessage.html)是可变的：创建Message对象后，依然可以将字段添加到这些消息中。在使用异步记录器或异步追加器记录这些消息后，您不应进行修改操作；在生成的日志输出中所做的修改您可能会看到，也可能看不到，会导致一定的混淆。类似地，自定义[Message](https://logging.apache.org/log4j/2.x/log4j-api/apidocs/org/apache/logging/log4j/message/Message.html)实现应该考虑异步的使用，在构建时就获取参数的快照，或标记它的特性是线程安全的。

* 如果您的应用程序在CPU资源稀缺的环境中运行，例如CPU是单核的机器，则启动另一个线程不太可能提供更好的性能。

* 如果应用程序记录消息的持续速率比日志追加器（appender）的最大持续吞吐量还要快，则队列将被填满，应用程序将以最慢的appender的速度进行日志记录。如果发生这种情况，请考虑选择[更快的appender](https://logging.apache.org/log4j/2.x/performance.html#whichAppender)，或日志更少些。如果这两个方案都不可选，使用同步记录可能会获得更好的吞吐量和更低的峰值延迟。

## 所有的Logger都为异步

_需要依赖disruptor-3.0.0.jar或更高版本。未来版本的Log4j 2将需要disruptor-3.3.3.jar或更高版本。_

这种是配置最简单且可以达到最佳的性能。要使所有的Logger都为异步，请将disruptor jar添加到类路径，并将系统属性`Log4jContextSelector`设置为`org.apache.logging.log4j.core.async.AsyncLoggerContextSelector`。

默认情况下，location无法通过异步logger传递给I/O线程。如果您的某个layout或自定义filter需要location信息，则需要在所有相关logger（包括root logger）的配置中设置`includeLocation=true`。

不需要location的配置如下所示：

```xml
<?xml version="1.0" encoding="UTF-8"?>

<!-- Don't forget to set system property
-DLog4jContextSelector=org.apache.logging.log4j.core.async.AsyncLoggerContextSelector
     to make all loggers asynchronous. -->

<Configuration status="WARN">
  <Appenders>
    <!-- Async Loggers will auto-flush in batches, so switch off immediateFlush. -->
    <RandomAccessFile name="RandomAccessFile" fileName="async.log" immediateFlush="false" append="false">
      <PatternLayout>
        <Pattern>%d %p %c{1.} [%t] %m %ex%n</Pattern>
      </PatternLayout>
    </RandomAccessFile>
  </Appenders>
  <Loggers>
    <Root level="info" includeLocation="false">
      <AppenderRef ref="RandomAccessFile"/>
    </Root>
  </Loggers>
</Configuration>
```

## 混合同步和异步Logger

## Location, location, location...

## 异步Log的性能

### 记录峰值吞吐量

### 异步与其他日志包的吞吐量对比

### 响应时延

## 底层技术点
