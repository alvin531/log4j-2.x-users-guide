# Layouts - 布局

Appender使用Layout将LogEvent格式化，从而满足消费该日志事件的组件需求。在Log4j 1.x和Logback Layouts中，将事件转换为String。在Log4j 2中，Layouts返回一个字节数组。这就让Layout的结果可以在更多类型的Appender中使用。但是，这意味着您需要使用[Charset](http://docs.oracle.com/javase/6/docs/api/java/nio/charset/Charset.html)配置大多数Layouts，以确保字节数组里的值是正确无误的。

使用Charset的Layout的基类是`org.apache.logging.log4j.core.layout.AbstractStringLayout`，默认为`UTF-8`。继承`AbstractStringLayout`的每个Layout都可以提供自己的默认值。请参阅下面Layout具体说明。

在Log4j 2.4.1中，对于ISO-8859-1和US-ASCII字符集，可以自定义字符编码器（character encoder），全将Java 8中的一些性能改进内置到Log4j以供Java 7的项目使用。若应用程序仅记录ISO-8859-1字符，指定此字符集将显着提高性能。

## CSV Layouts

此Layout创建[逗号分隔值（CSV）](https://en.wikipedia.org/wiki/Comma-separated_values)记录，并依赖[Apache Commons CSV](https://commons.apache.org/proper/commons-csv/) 1.4。

CSV layout有两种使用方式：首先，使用`CsvParameterLayout`记录事件参数以创建自定义数据库，通常为了达到此目而唯一地配置logger和file appender。其次，使用`CsvLogEventLayout`来记录事件以创建数据库，作为使用完整DBMS或使用支持CSV格式的JDBC驱动程序的替代方法。

`CsvParameterLayout`将事件的参数转换为CSV记录，而忽略消息（message）。要记录CSV信息，可以使用变普通的Logger方法`info()`，`debug()`等等：

```java
logger.info("Ignored", value1, value2, value3);
```

将创建CSV记录：

```csv
value1, value2, value3
```

或者，您可以使用仅包含参数的`ObjectArrayMessage`：

```java
logger.info(new ObjectArrayMessage(value1, value2, value3));
```

`CsvParameterLayout`和`CsvLogEventLayout`可以配置以下参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| format | String | 预定格式之一：`Default`，`Excel`，`MySQL`，`RFC4180`，`TDF`。请参阅[CSVFormat.Predefined](https://commons.apache.org/proper/commons-csv/archives/1.4/apidocs/org/apache/commons/csv/CSVFormat.Predefined.html)。 |
| delimiter | Character | 分隔符设置为指定的字符。 |
| escape | Character | 转义字符设置为指定的字符。 |
| quote | Character | quoteChar（引用字符）设置为指定的字符。 |
| quoteMode | String | 输出结果的引用策略（quote policy）设置为指定的值。其中一个值：`ALL`，`MINIMAL`，`NON_NUMERIC`，`NONE`。 |
| nullString | String | 在写入记录时将null作为给定的nullString代替。 |
| recordSeparator | String | 记录分隔符设置为指定的String。 |
| charset | Charset | 输出的Charset。 |
| header | 设置当流打开时包括的标题 | - |
| footer | 设置当流打开时包括的页脚 | - |

以CSV事件记录为例：

```java
logger.debug("one={}, two={}, three={}", 1, 2, 3);
```

使用以下字段生成CSV记录：

1. Nanos时间
2. Millis时间
3. 日志级别（Level）
4. 线程ID
5. 线程名称
6. 线程优先级
7. 格式化的消息
8. Logger FQCN
9. Logger名称
10. Marker
11. 异常
12. 源
13. Context Map
14. Context Stack

```csv
0,1441617184044,DEBUG,main,"one=1, two=2, three=3",org.apache.logging.log4j.spi.AbstractLogger,,,,org.apache.logging.log4j.core.layout.CsvLogEventLayoutTest.testLayout(CsvLogEventLayoutTest.java:98),{},[]
```

## GELF Layout

在Graylog扩展日志格式（Graylog Extended Log Format，GELF）1.1中格式事件。

如果日志事件数据大于1024字节（`compressionThreshold`），则此Layout会将JSON压缩为GZIP或ZLIB（`compressionType`）。此Layout不实现分块（chunking）。

配置如下发送到Graylog2服务器：

```xml
<Appenders>
  <Socket name="Graylog" protocol="udp" host="graylog.domain.com" port="12201">
      <GelfLayout host="someserver" compressionType="GZIP" compressionThreshold="1024">
        <KeyValuePair key="additionalField1" value="constant value"/>
        <KeyValuePair key="additionalField2" value="$${ctx:key}"/>
      </GelfLayout>
  </Socket>
</Appenders>
```

GELF Layout参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| host | String | `host`值（可选，默认为本地主机名）。 |
| compressionType | `GZIP`，`ZLIB`或`OFF` | 压缩类型（可选，默认为`GZIP`） |
| compressionThrehold | int | 如果数据大于此字节数，则进行压缩（可选，默认为1024） |
| includeStacktrace | boolean | 是否包括记录的Throwables的完整堆栈跟踪信息（可选，默认为true）。如果设置为false，则只包括[Throwable](http://docs.oracle.com/javase/6/docs/api/java/lang/Throwable.html)的类名和信息。 |
| includeThreadContext | boolean | 是否将线程上下文作为附加字段（可选，默认为true）。 |

参考：

* [GELF规范](http://docs.graylog.org/en/latest/pages/gelf.html#gelf)

## HTMLLayout

HTMLLayout生成一个HTML页面，每个LogEvent作为表的行记录。

HTMLLayout参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| charset | String | 将HTML字符串转换为字节数组时使用的字符集。该值必须是有效的[字符集](http://docs.oracle.com/javase/6/docs/api/java/nio/charset/Charset.html)。如果未指定，则此Layout使用UTF-8。 |
| contentType | String | Content-Type值。默认值为“text/html”。 |
| locationInfo | boolean | 如果为true，文件名和行号将包含在HTML输出中。默认值为false。<br><br>生成[位置信息](./layouts.md#LocationInformation)是一项昂贵的操作，可能会影响性能。谨慎使用。 |
| title | String | 将显示为HTML标题的字符串。 |

## JSONLayout

使用JSON格式来表示一系列事件，然后序列化为字节。此Layout需要依赖Jackson jar（有关详细信息，请参阅pom.xml）。

### 完整格式的JSON与片段JSON

如果配置`complete="true"`，则appender会输出格式完整格式的JSON文档。默认情况下，使用`complete="false"`，应将输出作为 _外部文件_ 包含在单独的文件中，以形成一个完整格式的JSON文档。

一个完整格式的JSON文档遵循以下模式：

```json
[
  {
    "logger":"com.foo.Bar",
    "timestamp":"1376681196470",
    "level":"INFO",
    "threadId":1,
    "thread":"main",
    "threadPriority":1,
    "message":"Message flushed with immediate flush=true"
  },
  {
    "logger":"com.foo.Bar",
    "timestamp":"1376681196471",
    "level":"ERROR",
    "threadId":1,
    "thread":"main",
    "threadPriority":1,
    "message":"Message flushed with immediate flush=true",
    "throwable":"java.lang.IllegalArgumentException: badarg\\n\\tat org.apache.logging.log4j.core.appender.JSONCompleteFileAppenderTest.testFlushAtEndOfBatch(JSONCompleteFileAppenderTest.java:54)\\n\\tat sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)\\n\\tat sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:57)\\n\\tat sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)\\n\\tat java.lang.reflect.Method.invoke(Method.java:606)\\n\\tat org.junit.runners.model.FrameworkMethod$1.runReflectiveCall(FrameworkMethod.java:47)\\n\\tat org.junit.internal.runners.model.ReflectiveCallable.run(ReflectiveCallable.java:12)\\n\\tat org.junit.runners.model.FrameworkMethod.invokeExplosively(FrameworkMethod.java:44)\\n\\tat org.junit.internal.runners.statements.InvokeMethod.evaluate(InvokeMethod.java:17)\\n\\tat org.junit.runners.ParentRunner.runLeaf(ParentRunner.java:271)\\n\\tat org.junit.runners.BlockJUnit4ClassRunner.runChild(BlockJUnit4ClassRunner.java:70)\\n\\tat org.junit.runners.BlockJUnit4ClassRunner.runChild(BlockJUnit4ClassRunner.java:50)\\n\\tat org.junit.runners.ParentRunner$3.run(ParentRunner.java:238)\\n\\tat org.junit.runners.ParentRunner$1.schedule(ParentRunner.java:63)\\n\\tat org.junit.runners.ParentRunner.runChildren(ParentRunner.java:236)\\n\\tat org.junit.runners.ParentRunner.access$000(ParentRunner.java:53)\\n\\tat org.junit.runners.ParentRunner$2.evaluate(ParentRunner.java:229)\\n\\tat org.junit.internal.runners.statements.RunBefores.evaluate(RunBefores.java:26)\\n\\tat org.junit.runners.ParentRunner.run(ParentRunner.java:309)\\n\\tat org.eclipse.jdt.internal.junit4.runner.JUnit4TestReference.run(JUnit4TestReference.java:50)\\n\\tat org.eclipse.jdt.internal.junit.runner.TestExecution.run(TestExecution.java:38)\\n\\tat org.eclipse.jdt.internal.junit.runner.RemoteTestRunner.runTests(RemoteTestRunner.java:467)\\n\\tat org.eclipse.jdt.internal.junit.runner.RemoteTestRunner.runTests(RemoteTestRunner.java:683)\\n\\tat org.eclipse.jdt.internal.junit.runner.RemoteTestRunner.run(RemoteTestRunner.java:390)\\n\\tat org.eclipse.jdt.internal.junit.runner.RemoteTestRunner.main(RemoteTestRunner.java:197)\\n"
  }
]
```

若使用`complete="false"`，则在文档不会使用“]”和“[”，记录间也不使用逗号“,”。

此方法使JSONLayout和嵌入它的appender相互独立。

### 漂亮型JSON vs. 紧凑型JSON

默认情况下，使用`compact="false"`，JSON布局不是紧凑的，这意味着appender使用换行和缩进行来格式化文本。如果`compact="true"`，则不使用任何换行符或缩进。Message内容也可能包含转义的换行符。

JSONLayout参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| charset | String | 将JSON字符串转换为字节数组时使用的字符集。该值必须是有效的[字符集](http://docs.oracle.com/javase/6/docs/api/java/nio/charset/Charset.html)。如果未指定，则此Layout使用UTF-8。 |
| compact | boolean | 若true，则appender不使用换行符和缩进。默认为false。 |
| eventEol | boolean | 若为true，则appenders在每个记录之后追加换行符。默认为false。与eventEol=true和compact=true一起使用，可以达到每行一条记录的效果。 |
| complete | boolean | 若为true，则appender包括JSON头和尾，以及记录之间的逗号。默认为false。 |
| properties | boolean | 若为true，则appender会在生成的JSON中包含线程上下文映射。默认为false。 |
| propertiesAsList | boolean | 若为true，则将线程上下文映射作为列表方式，列表里的值为map的entry对象，其中每个entry具有“key”属性（即为map的key）和“value”属性（即为map的value）。默认为false，在这种情况下，线程上下文映射即为普通的键值对映射。 |
| locationInfo | boolean | 如果为true，文件名和行号将包含在HTML输出中。默认值为false。<br><br>生成[位置信息](./layouts.md#LocationInformation)是一项昂贵的操作，可能会影响性能。谨慎使用。 |
| includeStacktrace | boolean | 若为true，则包括Throwable的完整堆栈信息（可选，默认为true）。 |

## PatternLayout

使用模式字符串可配置灵活的布局。此类的目标是格式化LogEvent并返回结果。结果的格式取决于转换模式（_conversion pattern_）。

转换模式与C中的printf函数的转换模式非常类假。转换模式由文字文本和格式控制表达式组成，称为转换表示符（_conversion specifiers_）。

_注意，任何文字（包括特殊字符）都可包含在转换模式中_。特殊字符包括\t，\n，\r，\f。使用\\输出一个反斜杠。

每个转换表示符以百分号（%）开头，后面是可选的格式修饰符（_format modifiers_）和转换字符（_conversion character_）。转换字符指定数据的类型，例如类别（category），优先级（priority），日期（date），线程名称（thread name）。格式修饰符控制诸如字段宽度，填充，左右对齐等等。以下是一个简单的例子。

转换模式为 **"%-5p [%t]: %m%n"**，并假设Log4j环境设置为使用PatternLayout。使用以下语句：

```java
Logger logger = LogManager.getLogger("MyLogger");
logger.debug("Message 1");
logger.warn("Message 2");
```

将输出：

```shell
DEBUG [main]: Message 1
WARN  [main]: Message 2
```

请注意，文本和转换表示符之间没有明确的分隔符。模式解析器在读取转换字符时知道何时达到了转换表示符的结尾处。在上面的示例中，转换表示符%-5p表示记录事件的优先级而保持对齐为五个字符的宽度。

如果模式字符串不包含处理Throwable的表示符，那么该模式的解析将会像“%xEx”表示符添加到字符串末尾一样。要抑制Throwable的格式化，只需在模式字符串中添加“%ex{0}”作为表示符。

PatternLayout参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| charset | String | 将HTML字符串转换为字节数组时使用的字符集。该值必须是有效的[字符集](http://docs.oracle.com/javase/6/docs/api/java/nio/charset/Charset.html)。如果未指定，则此Layout使用UTF-8。 |
| pattern | String | 复合模式字符串，包括下表中的一个或多个转换模式。不能同时使用PatternSelector。  |
| patternSelector | PatternSelector | 分析LogEvent中的信息并确定应使用哪种模式来格式化事件的组件。pattern和patternSelector参数是互斥的。 |
| replace | RegexReplacement | 允许替换部分生成的字符串。若配置，则replace元素必须指定要匹配的正则表达式和替换值。这执行类似于RegexReplacement转换器的功能，但适用于整个消息，而转换器仅适用于其模式生成的字符串。 |
| alwaysWriteExceptions | boolean | 若为`true`（默认），即使模式不包含异常转换，也会始终写入异常。这意味着如果在模式中并不包括输出异常的方式，但也会在模式的末尾添加默认的异常格式化程序。将此设置为`false`将禁用此行为，并允许您从模式输出中排除异常。 |
| header | String | 包含在每个日志文件顶部的可选标题字符串。 |
| footer | String | 包含在每个日志文件底部的可选脚注字符串。 |
| disableAnsi | boolean | 若为`true`（默认为`false`），则不输出ANSI转义码。 |
| noConsoleNoAnsi | boolean | 若为`true`（默认为`false`），且`System.console()`为null，则不输出ANSI转义码。 |

RegexReplacement参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| regex | String | 一个符合Java规范的正则表达式，以在结果字符串中进行匹配运算。见[Pattern](http://docs.oracle.com/javase/6/docs/api/java/util/regex/Pattern.html)。 |
| replacement | String | 该字符串用于替换任何匹配的子字符串。 |

### Patterns

Log4j支持的转换：

| Convertern Pattern | Description |
| ---- | ---- |
| **c**{precision} <br> **logger**{precision} | 输出当前日志记录事件的logger名称。logger转换表示符可以可选地跟随精度表示符，其由十进制整数或以十进制整数开头的模式组成。<br><br> 当精度表示符为整数值时，会减小logger名称的大小。 如果数字为正，layout将输出相应数量的最右边的logger名称。如果为负，layout将删除相应数量的最左边的logger名称。<br><br> 如果精度包含任何非整数字符，则layout将根据模式缩写名称。如果精度整数小于1，则layout仍将打印最右侧的名字。默认情况下，layout打印完整的logger名称。 <br> <table><tr><th>Conversion Pattern</th><th>Logger Name</th><th>Result</th></tr><tr><td>%c{1}</td><td>org.apache.commons.Foo</td><td>Foo</td></tr><tr><td>%c{2}</td><td>org.apache.commons.Foo</td><td>commons.Foo</td></tr><tr><td>%c{10}</td><td>org.apache.commons.Foo</td><td>org.apache.commons.Foo</td></tr><tr><td>%c{-1}</td><td>org.apache.commons.Foo</td><td>apache.commons.Foo</td></tr><tr><td>%c{-2}</td><td>org.apache.commons.Foo</td><td>commons.Foo</td></tr><tr><td>%c{-10}</td><td>org.apache.commons.Foo</td><td>org.apache.commons.Foo</td></tr><tr><td>%c{1.}</td><td>org.apache.commons.Foo</td><td>o.a.c.Foo</td></tr><tr><td>%c{1.1.~.~}</td><td>org.apache.commons.Foo</td><td>o.a.~.~.Foo</td></tr><tr><td>%c{.}</td><td>org.apache.commons.Foo</td><td>....Foo</td></tr></table>|
| **C**{precision} <br> **class**{precision} | 输出当前日志记录请求的调用者完全限定类名。此转换表示符也可以使用精确表示符，遵循与上面logger名称转换器相同的规则。<br><br> 生成调用者的类名（位置信息）是一项昂贵的操作，可能会影响性能。谨慎使用。 |
| **d**{pattern} <br> **date**{pattern} | 输出日志事件的日期。日期转换表示符后面可以使用大括号包含[SimpleDateFormat](http://docs.oracle.com/javase/6/docs/api/java/text/SimpleDateFormat.html)可以解析的日期和时间模式字符串。<br><br> 预定格式为`DEFAULT`，`ABSOLUTE`，`COMPACT`，`DATE`，`ISO8601`和`ISO8601_BASIC`。<br><br> 还可以使用[java.util.TimeZone.getTimeZone](http://docs.oracle.com/javase/6/docs/api/java/util/TimeZone.html#getTimeZone%28java.lang.String%29)的时区ID。如果未指定日期格式表示符，则使用`DEFAULT`格式。<br> <table><tr><th>Pattern</th><th>Example</th></tr><tr><td>%d{DEFAULT}</td><td>2012-11-02 14:34:02,781</td></tr><tr><td>%d{ISO8601}</td><td>2012-11-02T14:34:02,781</td></tr><tr><td>%d{ISO8601_BASIC}</td><td>20121102T143402,781</td></tr><tr><td>%d{ABSOLUTE}</td><td>14:34:02,781</td></tr><tr><td>%d{DATE}</td><td>02 Nov 2012 14:34:02,781</td></tr><tr><td>%d{COMPACT}</td><td>20121101143402781</td></tr><tr><td>%d{HH:mm:ss,SSS}</td><td>14:34:02,781</td></tr><tr><td>%d{dd MMM yyyy HH:mm:ss,SSS}</td><td>02 Nov 2012 14:34:02,781</td></tr><tr><td>%d{HH:mm:ss}{GMT+0}</td><td>18:34:02</td></tr><tr><td>%d{UNIX}</td><td>1351866842</td></tr><tr><td>%d{UNIX_MILLIS}</td><td>1351866842781</td></tr></table> <br> %d{UNIX}以秒为单位输出UNIX时间。%d{UNIX_MILLIS}以毫秒为单位输出UNIX时间。UNIX时间是是从1970年1月1日（UTC/GMT的午夜）开始所经过的时，其中UNIX以秒为单位，UNIX_MILLIS以毫秒为单位。当时间单位为毫秒，粒度取决于操作系统（[Windows](http://msdn.microsoft.com/en-us/windows/hardware/gg463266.aspx)）。这是输出事件时间的有效方法，因为只是发生从long到String的转换，没有涉及Date格式。 |
| **enc**{pattern}{[HTML&#124;JSON] <br> **encode**{pattern}{[HTML&#124;JSON]}} | 编码和转义特殊字符适合于特定标记语言的输出。默认情况下，如果只指定一个选项，则编码HTML。第二个选项用于指定应使用哪种编码格式。该转换器对于编码用户提供的数据特别有用，使得输出的数据不再是不正确的或不安全的。<br><br>一个典型的用法就是对消息`%enc{%m}`进行编码，但用户输入也可能来自其他位置，例如MDC `%enc{%mdc{key}}` <br><br> 使用HTML编码格式，将替换以下字符：<br> <table><tr><th>Character</th><th>Replacement</th></tr><tr><td>'\r', '\n'</td><td>分别转换为转义字符串“\\r”和“\\n”</td></tr><tr><td>&, <, >, ", ', /</td><td>替换为相应的HTML实体</td></tr></table> <br><br> 使用JSON编码格式，遵循[RFC 4627](https://www.ietf.org/rfc/rfc4627.txt)规定的转义规则：<br> <table><tr><th>Character</th><th>Replacement</th></tr><tr><td>U+0000 - U+001F</td><td>\u0000 - \u001F</td></tr><tr><td>其他控制字符</td><td>编码到\uABCD等效的转义代码</td></tr><tr><td>"</td><td>\"</td></tr><tr><td>\</td><td>\\</td></tr></table> <br> 例如，使用模式{"message": %enc{%m}{JSON}”}输出日志消息的有效JSON字符串。 |
| **equals**{pattern}{test}{substitution} <br> **equalsIgnoreCase**{pattern}{test}{substitution} | 由'pattern'运算得到匹配，使用'substitute'替换'test'。 如，"%equals{[%marker]}{[]}{}"将在marker里使用空字符串替换“[]”。<br><br> 该模式可以非常复杂的，特别是可以包含多个转换关键字。 |
| **ex&#124;exception&#124;throwable** <br>  {["none" <br> &#124;"full" <br> &#124;"depth" <br> &#124;"short" <br> &#124;"short.className" <br> &#124;"short.fileName" <br> &#124;"short.lineNumber" <br> &#124;"short.methodName" <br> &#124;"short.message" <br> &#124;"short.localizedMessage"]}<br>{subffix(pattern)} | 输出LoggingEvent中的Throwable堆栈信息，默认情况下将输出完整的堆栈信息，可以通过调用Throwable.printStackTrace()来查看。<br> 您可以使用 **%throwable{option}** 来控制输出格式。<br> **%throwable{short}** 输出Throwable的第一行。<br> **%throwable{short.className}** 输出产生异常的类名。<br> **%throwable{short.methodName}** 输出产生异常的方法名。<br> **%throwable{short.fileName}** 输出产生异常的类名。<br> **%throwable{short.lineNumber}** 输出产生异常的行号。<br> **%throwable{short.message}** 输出消息文本。<br> **%throwable{short.localizedMessage}** 输出本地化消息。<br> **%throwable{n}** 输出堆栈信息的前n行。<br> 指定 **%throwable{none}** 或 **%throwable{0}** 可以禁止异常的输出。<br> 指定 **%throwable{suffix{pattern}}** 只有在有可输出的情况下才将输出的模式(pattern)添加到输出中。此选项与上述转换可一起使用。 |
| **F** <br> **file** | 输出产生日志记录请求的文件名。<br><br> 生成文件信息（位置信息）是一项昂贵的操作，可能会影响性能。谨慎使用。 |
| **highlight**{pattern}{style} | |
| **K**{key} <br> **map**{key} <br> **MAP**{key} | |
| **l** <br> **location** | |
| **L** <br> **line** | |
| **m**{nolooups}{ansi} <br> **msg**{nolookups}{ansi} <br> **message**{nolookups}{ansi} | |
| **M** <br> **method** | |
| **marker** | |
| **markerSimpleName** | |
| **maxLen** <br> **maxLength** | |
| **n** | |
| **N** <br> **nano** | |
| **variablesNotEmpty**{pattern} <br> **varsNotEmpty**{pattern} <br> **notEmpty**{pattern} | |
| **p&#124;level**{level=label,level=label,...} <br> **p&#124;level**{length=n} <br> **p&#124;level**{lowerCase=true&#124;false} | |
| **r** <br> **relative** | |
| **replace**{pattern}{regex}{substitution} | |
| **rEx**{["none"&#124;"short"&#124;"full"&#124;depth]} <br> {[filters(packages)]}{suffix(pattern)} <br> **rException**{["none"&#124;"short"&#124;"full"&#124;depth]} <br> {[filters(packages)]}{suffix(pattern)} <br> **rThrowable**{["none"&#124;"short"&#124;"full"&#124;depth]} <br> {[filters(packages)]}{suffix(pattern)} | |
| **sn** <br> **sequenceNumber** | |
| **style**{pattern}{ANSI style} | |
| **T** <br> **tid** <br> **threadId** | |
| **t** <br> **tn** <br> **thread** <br> **threadName** | |
| **tp** <br> **threadPriority** | |
| **x** <br> **NDC** | |
| **X**{key[,key2...]} <br> **mdc**{key[,key2...]} <br> **MDC**{key[,key2...]} | |
| **u**{"RANDOM" &#124; "TIME"} <br> **uuid** | |
| **xEx&#124;xException&#124;xThrowable** <br> { <br> ["none"&#124;"short"&#124;"full"&#124;depth] <br> [,filters(package,package,...)] <br> } <br> {ansi( <br> Key=Value,Value,... <br> Key=Value,Value,... <br> ...) <br> } <br> {suffix(pattern)} | |
| % | |
