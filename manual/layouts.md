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
