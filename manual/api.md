# Log4j 2 API - Overview

## 概述

Log4j 2 API提供了编程的接口，并提供了一系列的实现具体的日志处理所需的组件。虽然Log4j 2对API和实现之前进行了分离，但这个做法最主要的目的不是希望存在多个实现（尽管也可能这样做），而是更清晰地表达有哪些类和方法可以在应用程序里被更安全地调用（确保向前兼容性）。

### Hello World!

按传统惯例，这里列举Hello，World示例。首先，从[LogManager](https://logging.apache.org/log4j/2.x/log4j-api/apidocs/org/apache/logging/log4j/LogManager.html)获取名为“HelloWorld”的Logger。接下来，logger输出“Hello，World！” 消息，当然只有当Logger配置启用了相应的日志级别（为INFO以下级别）时，才会输出该消息。

```java
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

public class HelloWorld {
    private static final Logger logger = LogManager.getLogger("HelloWorld");
    public static void main(String[] args) {
        logger.info("Hello, World!");
    }
}
```

调用logger.info()的输出效果取决于具体所使用的配置，配置不同，效果也不一样。有关更多详细信息，请参阅[Configuration](./configuration.md)章节。

### 参数替换 -- Substituting Parameters

通常，日志的目的就是反映当时系统的运行时的状态信息，这需要包含当时正在执行的对象的相关信息。在Log4j 1.x里，可以这样做：

```java
if (logger.isDebugEnabled()) {
    logger.debug("Logging in user " + user.getName() + " with birthday " + user.getBirthdayCalendar());
}
```

若代码里充斥着这样的逻辑（一堆的if语句），会本末倒置，让程序更像是日志记录，而不是原有的业务逻辑。另外，这里会导致两次的日志级别的判断；一次是调用isDebugEnabled方法，一次是在debug方法里面。而更好的做法是：

```java
logger.debug("Logging in user {} with birthday {}", user.getName(), user.getBirthdayCalendar());
```

使用上面的代码，日志级别只被检查一次，并且String拼接仅在启用了相应的日志级别（日志级别为Debug以下）时才会发生。

### 参数格式化 -- Formatting Parameters

如果`toString()`输出并不是你想要的效果，Formatter Loggers可以让你决定输出的格式。格式化操作是很通俗易明的，它使用与Java的[Formatter](http://docs.oracle.com/javase/6/docs/api/java/util/Formatter.html#syntax)相同的格式字符串。例如：

```java
public static Logger logger = LogManager.getFormatterLogger("Foo");

logger.debug("Logging in user %s with birthday %s", user.getName(), user.getBirthdayCalendar());
logger.debug("Logging in user %1$s with birthday %2$tm %2$te,%2$tY", user.getName(), user.getBirthdayCalendar());
logger.debug("Integer.MAX_VALUE = %,d", Integer.MAX_VALUE);
logger.debug("Long.MAX_VALUE = %,d", Long.MAX_VALUE);
```

要使用格式化Logger，必须调用LogManager的`getFormatterLogger`方法。此示例的输出以下，与自定义格式相比，Calendar的toString()就显得非常的啰嗦：

```text
2012-12-12 11:56:19,633 [main] DEBUG: User John Smith with birthday java.util.GregorianCalendar[time=?,areFieldsSet=false,areAllFieldsSet=false,lenient=true,zone=sun.util.calendar.ZoneInfo[id="America/New_York",offset=-18000000,dstSavings=3600000,useDaylight=true,transitions=235,lastRule=java.util.SimpleTimeZone[id=America/New_York,offset=-18000000,dstSavings=3600000,useDaylight=true,startYear=0,startMode=3,startMonth=2,startDay=8,startDayOfWeek=1,startTime=7200000,startTimeMode=0,endMode=3,endMonth=10,endDay=1,endDayOfWeek=1,endTime=7200000,endTimeMode=0]],firstDayOfWeek=1,minimalDaysInFirstWeek=1,ERA=?,YEAR=1995,MONTH=4,WEEK_OF_YEAR=?,WEEK_OF_MONTH=?,DAY_OF_MONTH=23,DAY_OF_YEAR=?,DAY_OF_WEEK=?,DAY_OF_WEEK_IN_MONTH=?,AM_PM=0,HOUR=0,HOUR_OF_DAY=0,MINUTE=0,SECOND=0,MILLISECOND=?,ZONE_OFFSET=?,DST_OFFSET=?]
2012-12-12 11:56:19,643 [main] DEBUG: User John Smith with birthday 05 23, 1995
2012-12-12 11:56:19,643 [main] DEBUG: Integer.MAX_VALUE = 2,147,483,647
2012-12-12 11:56:19,643 [main] DEBUG: Long.MAX_VALUE = 9,223,372,036,854,775,807
```

### 混合使用Logger和Formatter Logger（格式化Logger）

格式化Logger对输出格式进行更细粒度的控制，但缺点是必须指定正确的类型（例如，给%d格式参数传递除了非十进制整数的其他内容都会产生异常）。

如果你的主要用法是使用`{}-style`参数，但偶尔需要细粒度的控制输出格式，你可以使用`printf`方法：

```java
public static Logger logger = LogManager.getLogger("Foo");

logger.debug("Opening connection to {}...", someDataSource);
logger.printf(Level.INFO, "Logging in user %1$s with birthday %2$tm %2$te,%2$tY", user.getName(), user.getBirthdayCalendar());
```

### 使用Java 8 lambda支持惰性（延迟）log

在版本2.4中，Logger接口添加了对lambda表达式的支持。这允许客户端代码实现惰性计算，延迟输出日志消息，而不用显式检查是否启用了相应的日志级别。例如，之前你会这样写：

```java
// pre-Java 8 style optimization: explicitly check the log level
// to make sure the expensiveOperation() method is only called if necessary
if (logger.isTraceEnabled()) {
    logger.trace("Some long-running operation returned {}", expensiveOperation());
}
```

在Java 8环境下，您可以使用lambda表达式来实现相同的效果。您不再需要显式检查日志级别：

```java
// Java-8 style optimization: no need to explicitly check the log level:
// the lambda expression is not evaluated if the TRACE level is not enabled
logger.trace("Some long-running operation returned {}", () -> expensiveOperation());
```

### Logger Names

大多数日志实现使用分层方案来匹配logger名字与日志配置相匹配。在此方案中，logger名字层次结构由“.”表示。其方式非常类似于Java包名的层次结构。例如，这两个org.apache.logging.appender和org.apache.logging.filter名字，org.apache.logging就是它们的父级。在大多数情况下，应用程序通过将当前类的名字传递给`LogManager.getLogger`来命名其logger。由于这种用法非常普遍，Log4j 2提供了便捷方法，当logger name参数被省略或为null时，默认使用当前类的名字。例如，在下面的示例中，Logger的名字都是“org.apache.test.MyTest”。

```java
package org.apache.test;

public class MyTest {
    private static final Logger logger = LogManager.getLogger(MyTest.class);
}
```

```java
package org.apache.test;

public class MyTest {
    private static final Logger logger = LogManager.getLogger(MyTest.class.getName());
}
```

```java
package org.apache.test;

public class MyTest {
    private static final Logger logger = LogManager.getLogger();
}
```
