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

### 示例1 - 使用Console Appender的简单配置

### 示例2 - 使用File Appender的简单配置

### 示例3 - SocketAppender

### 示例4 - AsyncAppender

### 示例4 - AsyncAppender + Console + File
