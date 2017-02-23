# Architecture 架构

## 主要组件

![log4j classes](../assets/images/Log4jClasses.jpg)

使用Log4j 2 API时，应用程序从LogManager里请求特定名称的Logger。LogManager将匹配适当的LoggerContext，从中提到Logger。若必须创建Logger，它需与LoggerConfig相关联，LoggerConfig包含其中一个：a）与Logger相同的名称，b）parent package的名称，或c）root LoggerConfig。LoggerConfig对象是从配置中的Logger声明里创建的。LoggerConfig与实际传递LogEvent的Appender相关联。

### Logger层次结构

一个日志API首要优势在于，相对于普通的`System.out.println`来说，日志API可以禁用某些日志语句，同时又允许其他语句无限制的进行输出。此功能假定日志记录（即所有可能的日志记录语句）根据某个标准进行了分类。

在Log4j 1.x里，通过Logger之间的关系来反映Logger层次结构。在Log4j 2中，此关系不再存在。相反，层次结构保持在LoggerConfig对象之间的关系中。

Logger和LoggerConfig都是命名实体。Logger的名称大小写敏感，并且遵循分层命名规则：

> **命名层次结构（Named Hierarchy）**
> 如果LoggerConfig的名字(使用点符号进行分隔)是后代Logger名字的前缀，则LoggerConfig被认为是这个LoggerConfig的祖先。如果LoggerConfig本身和后代LoggerConfig之间没有祖先（即没有断层），则LoggerConfig被称为子LoggerConfig的父级。

例如，名为`"com.foo"`的LoggerConfig是名为`"com.foo.Bar"`的LoggerConfig的父级。类似地，`"java"`是`"java.util"`的父级和`"java.util.Vector"`的祖先。这个命名方式大多数开发人员都熟悉。

根（root）LoggerConfig位于LoggerConfig层次结构的顶级。它是例外的，因为它总是存在，它是每个层次结构的一部分。直接链接到根LoggerConfig的Logger可以这样获取到：

```java
Logger logger = LogManager.getLogger(LogManager.ROOT_LOGGER_NAME);
```

或者，更简单地：

```java
Logger logger = LogManager.getRootLogger();
```

获取其他Logger，可以通过传递所需Logger的名称给[LogManager.getLogger](https://logging.apache.org/log4j/2.x/log4j-api/apidocs/org/apache/logging/log4j/LogManager.html#getLoggerjava.lang.String)静态方法来获取。 有关Logging API的更多信息，请参见[Log4j 2 API](./api.md)。

### LoggerContext

[LoggerContext](https://logging.apache.org/log4j/2.x/log4j-core/apidocs/org/apache/logging/log4j/core/LoggerContext.html)充当Logger系统的锚点。不过，根据实际情况，应用程序可以具有多个LoggerContext。有关LoggerContext的详细信息，请参考[Log Separation](./logsep.md)部分。

### Configuration

每个LoggerContext都包含一个[Configuration](https://logging.apache.org/log4j/2.x/log4j-core/apidocs/org/apache/logging/log4j/core/config/Configuration.html)。Configuration包含所有的Appender、上下文相关（context-wide）的Filter、LoggerConfig、以及对StrSubstitutor的引用。在重新配置期间，将存在两个Configuration对象。一旦所有的Logger启用了新的Configuration，旧的Configuration就将被停止并回收。

### Logger

如前所述，Logger通过调用[LogManager.getLogger](https://logging.apache.org/log4j/2.x/log4j-api/apidocs/org/apache/logging/log4j/LogManager.html#getLoggerjava.lang.String)来创建。Logger本身不直接执行操作。它只是包括一个name，并与LoggerConfig相关联。它直接扩展AbstractLogger。当configuration被更改时，Logger可能会和另一个LoggerConfig相关联，从而导致它们的行为被更改了。

#### 获取Logger

使用相同名字来调用`LogManager.getLogger`，始终返回完全相同的Logger对象。

例如：

```java
Logger x = LogManager.getLogger("wombat");
Logger y = LogManager.getLogger("wombat");
```

`x`和`y`指向完全相同的Logger对象。

log4j环境的Configuration通常在应用程序初始化时就完成。最主要的方式就是读取配置文件。将在[Configuration](./configuration.md)章节进行详述。

Log4j可以便利地通过组件名进行Logger的命名。这可以通过把Logger命名为该类的全名（fully qualified name）来实例化一个Logger对象。这是一个非常有用且直接的定义Logger的方式。由于日志输出基于Logger的名字，所以此命名策略可以很好地识别日志消息的来源。不过，这只是一个推荐的常用的命名策略。Log4j并不限制对Logger的命名。开发人员可以根据需要自定义命名Logger。

由于使用类名来命名一个Logger是最常见的方式，Logj4j提供了便利的`LogManager.getLogger()`方法，它可以自动地使用调用类的全名来命名Logger。

不管如何，使用调用类来命名Logger是迄今为止最好的命名策略了。

### LoggerConfig

当在日志配置中声明Logger时，就创建了[LoggerConfig](https://logging.apache.org/log4j/2.x/log4j-core/apidocs/org/apache/logging/log4j/core/config/LoggerConfig.html)对象。LoggerConfig包含一组Filter，这些Filter对LogEvent进行处理，然后再传递到Appender里。它还包含用于处理事件（LogEvent）的一组Appender的引用。

#### Log级别

LoggerConfig包含一个日志级别（[Log Level](https://logging.apache.org/log4j/2.x/log4j-api/apidocs/org/apache/logging/log4j/Level.html)）。内置的级别有：TRACE、DEBUG、INFO、WARN、ERROR和FATAL。Log4j 2还支持[自定义日志级别](./customloglevels.md)。另一种更细粒度定义级别的机制是使用标记（[Markers](./markers.md)）。

[Log4j 1.x](http://logging.apache.org/log4j/1.2/manual.html)和[logback](http://logback.qos.ch/manual/architecture.html#effectiveLevel)都有“级别继承（Level Inheritance）”的概念。在Log4j 2中，Logger和LoggerConfig是两个不同的对象，因此这个概念的实现并不相同。每个Logger都引用适当的LoggerConfig，而LoggerConfig引用其父级，从而达到相同的效果。

以下示例了五个表，分别指定不同的级别值（Log Level），并展示每个关联的Logger的结果级别。请注意，在所有这些情况下，如果root LoggerConfig未配置，将为其分配默认级别。

| Logger Name | Assigned LoggerConfig | LoggerConfig Level | Logger Level |
| ---- | ---- | ---- | ---- |
| root | root | DEBUG | DEBUG |
| X | root | DEBUG | DEBUG |
| X.Y | root | DEBUG | DEBUG |
| X.Y.Z | root | DEBUG | DEBUG |

在这个示例1中，仅配置了root logger的Log Level。那么所有其他的Logger都引用root LoggerConfig，并使用其Level。

| Logger Name | Assigned LoggerConfig | LoggerConfig Level | Logger Level |
| ---- | ---- | ---- | ---- |
| root | root | DEBUG | DEBUG |
| X | X | ERROR | ERROR |
| X.Y | X.Y | INFO | INFO |
| X.Y.Z | X.Y.Z | WARN | WARN |

在示例2中，所有的Logger都配置了一个LoggerConfig，则从中获取到相应的Level。

| Logger Name | Assigned LoggerConfig | LoggerConfig Level | Logger Level |
| ---- | ---- | ---- | ---- |
| root | root | DEBUG | DEBUG |
| X | X | ERROR | ERROR |
| X.Y | X | ERROR | ERROR |
| X.Y.Z | X.Y.Z | WARN | WARN |

在示例3中，`root`、`X`和`X.Y.Z`这3个Logger都配置了一个跟对应的Logger相同名字的LoggerConfig。`X.Y`这个Logger则没有配置自身的LoggerConfig，那么将使用`X`的LoggerConfig，因为这个LoggerConfig的名字跟`X.Y`这个Logger的名字最大相似匹配的。

| Logger Name | Assigned LoggerConfig | LoggerConfig Level | Logger Level |
| ---- | ---- | ---- | ---- |
| root | root | DEBUG | DEBUG |
| X | X | ERROR | ERROR |
| X.Y | X | ERROR | ERROR |
| X.Y.Z | X | ERROR | ERROR |

在示例4中，`root`和`X`Logger各自都配置相同名字的LoggerConfig。`X.Y`和`X.Y.Z`则没有配置LoggerConfig，那么将使用`X`的LoggerConfig里定义的Level，因为这个LoggerConfig的名字跟它们Logger的名字最大相似匹配的。

| Logger Name | Assigned LoggerConfig | LoggerConfig Level | Logger Level |
| ---- | ---- | ---- | ---- |
| root | root | DEBUG | DEBUG |
| X | X | ERROR | ERROR |
| X.Y | X.Y | INF | INF |
| X.YZ | X | ERROR | ERROR |

在示例5中，`root`、`X`和`X.Y`Logger各自都配置相同名字的LoggerConfig。而`X.YZ`（_注意中间没有逗号_）则没有配置LoggerConfig，那么将使用`X`的LoggerConfig里定义的Level，因为这个LoggerConfig的名字跟其Logger的名字最大相似匹配的。它不与`X.Y`的LoggerConfig相关联，因为其标记必须完全匹配（没有后代关系）。

| Logger Name | Assigned LoggerConfig | LoggerConfig Level | Logger Level |
| ---- | ---- | ---- | ---- |
| root | root | DEBUG | DEBUG |
| X | X | ERROR | ERROR |
| X.Y | X.Y |  | ERROR |
| X.Y.Z | X.Y |  | ERROR |

在示例6中，LoggerConfig `X.Y`没有配置Level，那么它将从LoggerConfig `X`里继承Level过来。Logger `X.Y.Z`使用LoggerConfig `X.Y`。由于它没有完全匹配的LoggerConfig。它的level也将从LoggerConfig `X`里继承下来。

下表说明了Level过滤的工作原理。在表中，垂直头显示LogEvent的Level，而水平头显示与适当LoggerConfig关联的Level。其交集说明是否允许LogEvent通过进一步处理（:heavy_check_mark:）或丢弃（:x:）。

| Event Level | LoggerConfig Level | | | | | | |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| | **TRACE** | **DEBUG** | **INFO** | **WARN** | **ERROR** | **FATAL** | **OFF** |
| **ALL** | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | :x: |
| **TRACE** | :heavy_check_mark: | :x: | :x: | :x: | :x: | :x: | :x: |
| **DEBUG** | :heavy_check_mark: | :heavy_check_mark: | :x: | :x: | :x: | :x: | :x: |
| **INFO** | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | :x: | :x: | :x: | :x: |
| **WARN** | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | :x: | :x: | :x: |
| **ERROR** | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | :x: | :x: |
| **FATAL** | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | :x: |
| **OFF** | :x: | :x: | :x: | :x: | :x: | :x: | :x: |

### Filter

除了上一节描述的自动进行log Level过滤之外，Log4j还可以自定义Filter，这些Filter可以配置在以下周期中：在LoggerConfig处理之前；在LoggerConfig处理之后但在调用任何的Appender之前；在LoggerConfig处理之后，但在调用特定的Appender之前；以及在每个Appender上使用。这非常类似于防火墙的过滤器方式，每个Filter返回三个值中的其中一个：`Accept`、`Deny`和`Neutral`。响应`Accept`意味着不再调用其它的Filter，事件已被处理了。响应`Deny`意味着事件应立即被中止，应该把控制权返回给调用者。响应`Neutral`意味着事件应传递到其他的Filter上；如果没有其他Filter了，事件将被处理。

虽然Filter可能接受事件，但仍可能出现不记录事件的情况。当事件被在LoggerConfig处理之前的Filter接受，但又被定义在LoggerConfig上的Filter或被所有的Appender拒绝时，就是发生这种情况。

### Appender

基于其Logger选择性地启用或禁用日志请求的能力仅是架构里的一部分。Log4j能够把日志请求输出到多个目的地。在Log4j中，输出目的地称之为[Appender](https://logging.apache.org/log4j/2.x/log4j-core/apidocs/org/apache/logging/log4j/core/Appender.html)。目前，Appender类型有：console、file、远程socket服务器、Apache Flume、JMS、远程UNIX Syslog进程、以及各种DB。

可以调用当前的Configuration里的[addLoggerAppender](https://logging.apache.org/log4j/2.x/log4j-core/apidocs/org/apache/logging/log4j/core/config/Configuration.html#addLoggerAppenderorg.apache.logging.log4j.core.Logger20org.apache.logging.log4j.core.Appender)方法，将一个Appender添加一个Logger上。

**logger中所有被允许记录（enabled）的日志请求将都会流转到该Logger里LoggerConfig中的所有Appender上，以及LoggerConfig的父节点的Appender上。** 换句话说，Appender在LoggerConfig层次结构中有继承关系。例如，如果console appender添加到root logger里，则所有被允许记录（enabled）的日志请求将至少在console上输出。如果另外一个file appender被添加到LoggerConfig上，比如C logger，那么C和C的后代们的被允许记录的日志请求都会输出在file和console上（附加性效果）。通过在配置文件中的Logger声明上设置`additivity="false"`，可以改变此默认行为，以便中断这种继承关系，Appender的附加性不再生效。

关于Appender Additivity（附加性）的规则总结如下。

> **Appender Additivity**

> Logger _L_ 的日志语句的输出将转到与L相关联的LoggerConfig中的所有Appender上，以及该LoggerConfig的祖先Appender上。这是术语“appender additivity”的含义。

> 然而，如果与Logger _L_ 相关联的LoggerConfig的祖先（即 _P_）的additivity标识设置为`false`，则L的输出将被转向到L关联的LoggerConfig中的所有appender上，且会转向它的祖先 _P_ 的Appender里，但不是转向 _P_ 的祖先里了（已断层了）。

> 默认情况下，appender的additivity标记设置为`true`。

下表给出更详细的示例：

| Logger Name | Added Appenders | Additivity Flag | Output Targets | Comment |
| ---- | ---- | ---- | ---- | ---- |
| root | A1 | not applicable | A1 | root logger没有父类，所以additivity不能设置。 |
| x | A-x1，A-x2 | true | A1，A-x1，A-x2 | “x”和root的Appenders。 |
| x.y | none | true | A1，A-x1，A-x2 | “x”和root的Appenders。通常不配置没有Appenders的Logger。 |
| x.y.z | A-xyz1 | true | A1，A-x1，A-x2，A-xyz1 | “x.y.z”、“x”和root的Appenders。 |
| security | A-sec | false | A-sec | 由于additivity标识设为了`false`，那么就不再有appender累积。|
| security.access | none | true | A-sec | 只有“security”的appender，因为“security”里的dditivity标识设为了`false`。 |

### Layout

通常，用户不仅希望定制日志输出appender，还希望个性化输出格式。这可以通过给Appender指定一个Layout来达到。Layout的功能就是通过用户的意愿来格式化LogEvent，然后Appender将格式化好的信息输出到目的地。[PatternLayout](https://logging.apache.org/log4j/2.x/log4j-core/apidocs/org/apache/logging/log4j/core/layout/PatternLayout.html)是Log4j标准的一部分，它允许用户根据类似于C语言`printf`函数的转换模式进行输出格式化。

例如，带有转换模式“%r [%t] %-5p %c - %m%n”的PatternLayout将输出类似于：

```shell
176 [main] INFO  org.foo.Bar - Located nearest gas station.
```

第一个字段表示从程序启动以来的毫秒数。第二个字段表示发出日志请求的线程名。第三个字段是日志语句的级别。第四个字段是与日志请求相关联的logger的名字。“-”后面的文件本是具体的日志消息。

Log4j针对各种场景（如JSON、XML、HTML和Syslog（包括新的RFC 5424版本））提供了不同的Layout。其他的Appender（例如DB连接器）会填充到指定的字段，而不是特定的文件格式。

还有一点值得关注的是，log4j可以根据用户指定的标准呈现日志消息的内容。例如，如果您经常需要记录当前项目中使用到的对象类型`Oranges`，您可以创建一个`OrangeMessage`类，它接受Orange实例并传递给Log4j，这样Orange对象就可以按需格式化为一个适当的字节数组。

### StrSubstitutor and StrLookup

[StrSubstitutor](https://logging.apache.org/log4j/2.x/log4j-core/apidocs/org/apache/logging/log4j/core/lookup/StrSubstitutor.html)类和[StrLookup](https://logging.apache.org/log4j/2.x/log4j-core/apidocs/org/apache/logging/log4j/core/lookup/StrLookup.html)接口是从[Apache Commons Lang](https://commons.apache.org/proper/commons-lang/)中借鉴的做法，并对他们进行改造以支持处理LogEvent。另外，[Interpolator](https://logging.apache.org/log4j/2.x/log4j-core/apidocs/org/apache/logging/log4j/core/lookup/Interpolator.html)类是从Apache Commons Configuration中借鉴而来，用来处理StrSubstitutor可以从多个StrLookup里查找变量的具体值。它还可以修改LogEvent。这些类在一起，就提供了一种机制，允许配置引用来自系统属性，配置文件，ThreadContext Map，LogEvent中的StructuredData的变量。当处理配置或处理每个事件时，可以解析这些变量。有关详细信息，请参阅[Lookups](./lookups.md)章节。
