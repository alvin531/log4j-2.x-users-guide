# Log4j 2 API - Scala API

## Scala API

Log4j 2包含一个便利Scala包装器，让scala更简单地使用[Logger](https://logging.apache.org/log4j/2.x/log4j-api/apidocs/org/apache/logging/log4j/Logger.html) API的。

### 环境要求

Log4j 2 Scala API依赖于Log4j 2 API、Scala运行库和反射机制。它目前支持Scala 2.10和2.11。请参阅将此包含在SBT，Maven，Ivy或Gradle中的[说明](https://logging.apache.org/log4j/2.x/maven-artifacts.html#Scala_API)。

### 示例

```scala
import org.apache.logging.log4j.scala.Logging
import org.apache.logging.log4j.Level

class MyClass extends BaseClass with Logging {
  def doStuff(): Unit = {
    logger.info("Doing stuff")
  }
  def doStuffWithLevel(level: Level): Unit = {
    logger(level, "Doing stuff with arbitrary level")
  }
}
```

调用logger.info()的输出取决于所使用的配置，不同的配置，使用也将不同。有关更多详细信息，请参阅[Configuration](./configuration.md)部分。

### 参数替换

通常，记录的目的是提供关于系统中正在发生什么的信息，这需要包括关于正在运行的对象的信息。在Scala中，可以使用[字符串插值（string interpolation）](http://docs.scala-lang.org/overviews/core/string-interpolation.html)来实现：

```scala
logger.debug(s"Logging in user ${user.getName} with birthday ${user.calcBirthday}")
```

由于Scala Logger是用宏实现的，因此String构造和方法调用只会在启用debug级别时才会发生。

### Logger Names

大多数日志实现使用分层方案来匹配logger名字与日志配置相匹配。在此方案中，logger名字层次结构由“.”表示。其方式非常类似于Java包名的层次结构。例如，这两个org.apache.logging.appender和org.apache.logging.filter名字，org.apache.logging就是它们的父级。在大多数情况下，应用程序通过将当前类的名字传递给`LogManager.getLogger`来命名其logger。由于这种用法非常普遍，Log4j 2提供了便捷方法，当logger name参数被省略或为null时，默认使用当前类的名字。例如，在下面的示例中，Logger的名字都是“org.apache.test.MyTest”。

大多数日志实现使用分层方案来匹配logger名字与日志配置相匹配。在此方案中，logger名字层次结构由“.”表示。其方式非常类似于Java/Scala包名的层次结构。[Logging trait](https://logging.apache.org/log4j/2.x/log4j-api-scala_2.11/scaladocs/index.html#org.apache.logging.log4j.scala.Logging)将自动将logger命名为其使用的类。

### ScalaDoc

[ScalaDoc](https://logging.apache.org/log4j/2.x/log4j-api-scala_2.11/scaladocs/index.html#org.apache.logging.log4j.scala.package)。
