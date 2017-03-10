# Filters - 过滤器

Filters（过滤器）日志事件进行评估，以确定是否发布它们或者如何发布它们。调用filter里的方法，返回Result，该Result是一个枚举类型，具有3个值（ACCEPT，DENY或NEUTRAL）。

Filters可以配置在四个位置中：

1. 上下文范围的过滤器（Context-wide Filters）在配置文件中直接配置。被这些filters拒绝（DENY）的事件不会被传递给日志记录器上。一旦事件被上下文范围的过滤器接受（ACCEPT），它不会被其他任何上下文范围的过滤器处理了，也不会使用记录器的级别来过滤事件。除此之外，事件将由Logger和Appender里的Filters进行评估。
2. Logger Filters在指定的Logger上配置。这些在上下文范围过滤器（Context-wide Filters）和记录器的日志级别（Log Level）之后进行评估。被这些过滤器拒绝的事件将被丢弃，且事件不会被传递到父记录器，而与附加性（additivity）设置无关。
3. Appender Filters用于确定特定的Appender是否应处理事件的格式化和输出。
4. AppenderRef Filters用于确定logger是否应将事件路由到appender上。

## BurstFilter

BurstFilter（爆炸过滤器）提供了一种机制，通过在达到最大限制（maximum limit）后静默地丢弃事件来控制LogEvents的输出速率（rate）。

换句话说，你可以通过设置这个过滤器来控制日志的最大输出量，当超过设置的值之后，这过滤器将会把队列中的满足消息级别的日志丢弃掉（防止日志爆炸）。

BurstFilter参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| leve | String | 要过滤的日志级别。如果超过`maxBurst`，那么任何等于或低于此级别的内容将被过滤掉。默认值为WARN，意味着任何高于warn的消息都将被记录，而不考虑爆炸（burst）的大小。 |
| rate | float | （速率）每秒允许的平均事件输出数。。 |
| maxBurst | integer | 事件的最大数值，在过滤器生效之前的事件数。 默认值是速率的10倍。 |
| onMatch | String | 当满足过滤器设置的条件后进行的操作。可能是ACCEPT，DENY或NEUTRAL。默认值为NEUTRAL。 |
| onMismatch | String | 当不满足过滤器设置的条件后进行的操作。可能是ACCEPT，DENY或NEUTRAL。默认值为DENY。 |

包含BurstFilter的配置可以如下所示：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <RollingFile name="RollingFile" fileName="logs/app.log"
                 filePattern="logs/app-%d{MM-dd-yyyy}.log.gz">
      <BurstFilter level="INFO" rate="16" maxBurst="100"/>
      <PatternLayout>
        <pattern>%d %p %c{1.} [%t] %m%n</pattern>
      </PatternLayout>
      <TimeBasedTriggeringPolicy />
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
    </Root>
  </Loggers>
</Configuration>
```

【译者言】这个示例说明：日志级别为INFO、DEBUG、TRACE、ALL的可以被过滤掉，输出速率为每秒16条日志，最大队列为100条。那么就说明：(100 / 16 = 6.25)秒里只能输出100条，在6.25秒内超过100条的会被抛弃掉。

## CompositeFilter

CompositeFilter（组合过滤器）提供了一种可以集合多个filter的方法。它在配置文件中使用filters节点，里面包含其他filter。filters节点不用设属性。

CompositeFilter的配置如下所示：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Filters> <!-- CompositeFilter -->
    <MarkerFilter marker="EVENT" onMatch="ACCEPT" onMismatch="NEUTRAL"/>
    <DynamicThresholdFilter key="loginId" defaultThreshold="ERROR"
                            onMatch="ACCEPT" onMismatch="NEUTRAL">
      <KeyValuePair key="User1" value="DEBUG"/>
    </DynamicThresholdFilter>
  </Filters>
  <Appenders>
    <File name="Audit" fileName="logs/audit.log">
      <PatternLayout>
        <pattern>%d %p %c{1.} [%t] %m%n</pattern>
      </PatternLayout>
    </File>
    <RollingFile name="RollingFile" fileName="logs/app.log"
                 filePattern="logs/app-%d{MM-dd-yyyy}.log.gz">
      <BurstFilter level="INFO" rate="16" maxBurst="100"/>
      <PatternLayout>
        <pattern>%d %p %c{1.} [%t] %m%n</pattern>
      </PatternLayout>
      <TimeBasedTriggeringPolicy />
    </RollingFile>
  </Appenders>
  <Loggers>
    <Logger name="EventLogger" level="info">
      <AppenderRef ref="Audit"/>
    </Logger>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
    </Root>
  </Loggers>
</Configuration>
```

## DynamicThresholdFilter

DynamicThresholdFilter（动态阈值过滤器）可以使用特定属性按指定的日志级别进行过滤。例如，若在ThreadContext Map中有用户的loginId这个属性，那么可以为某个特定的用户输出debug级别的日志。

DynamicThresholdFilter参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| defaultThreshold | String | 要过滤的日志级别。如果key属性得到的值没有匹配中配置的那些key/avlue对，则该级别将与事件的级别进行对比。 |
| keyValuePair | KeyValuePair\[\] | 配置一个或多个KeyValuePair元素，定于匹配key时的值（value），若匹配中了就使用新的Log Level（value值）。 |
| onMatch | String | 当满足过滤器设置的条件后进行的操作。可能是ACCEPT，DENY或NEUTRAL。默认值为NEUTRAL。 |
| onMismatch | String | 当不满足过滤器设置的条件后进行的操作。可能是ACCEPT，DENY或NEUTRAL。默认值为DENY。 |

下面是DynamicThresholdFilter的配置示例：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <DynamicThresholdFilter key="loginId" defaultThreshold="ERROR"
                          onMatch="ACCEPT" onMismatch="NEUTRAL">
    <KeyValuePair key="User1" value="DEBUG"/>
  </DynamicThresholdFilter>
  <Appenders>
    <RollingFile name="RollingFile" fileName="logs/app.log"
                 filePattern="logs/app-%d{MM-dd-yyyy}.log.gz">
      <BurstFilter level="INFO" rate="16" maxBurst="100"/>
      <PatternLayout>
        <pattern>%d %p %c{1.} [%t] %m%n</pattern>
      </PatternLayout>
      <TimeBasedTriggeringPolicy />
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
    </Root>
  </Loggers>
</Configuration>
```

## MapFilter

MapFilter可以对MapMessage中的数据元素进行过滤。

MapFilter参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| keyValuePair | KeyValuePair\[\] | 定义一个或多个KeyValuePair元素，其中key为map里的键，value是要匹配的值。如果指定多次相同的key，则对该key的检查将自动为“or”逻辑，因为一个Map只能包含一个value。 |
| operator | String | 如果operator是“or”，则只要匹配中任何一个key/value对则认为是匹配中了，否则所有key/value对必须全匹配。 |
| onMatch | String | 当满足过滤器设置的条件后进行的操作。可能是ACCEPT，DENY或NEUTRAL。默认值为NEUTRAL。 |
| onMismatch | String | 当不满足过滤器设置的条件后进行的操作。可能是ACCEPT，DENY或NEUTRAL。默认值为DENY。 |

在此配置中，MapFilter可用于记录特定事件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <MapFilter onMatch="ACCEPT" onMismatch="NEUTRAL" operator="or">
    <KeyValuePair key="eventId" value="Login"/>
    <KeyValuePair key="eventId" value="Logout"/>
  </MapFilter>
  <Appenders>
    <RollingFile name="RollingFile" fileName="logs/app.log"
                 filePattern="logs/app-%d{MM-dd-yyyy}.log.gz">
      <BurstFilter level="INFO" rate="16" maxBurst="100"/>
      <PatternLayout>
        <pattern>%d %p %c{1.} [%t] %m%n</pattern>
      </PatternLayout>
      <TimeBasedTriggeringPolicy />
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
    </Root>
  </Loggers>
</Configuration>
```

此示例配置与前面示例功能相同，因为配置的唯一logger是root。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <RollingFile name="RollingFile" fileName="logs/app.log"
                 filePattern="logs/app-%d{MM-dd-yyyy}.log.gz">
      <BurstFilter level="INFO" rate="16" maxBurst="100"/>
      <PatternLayout>
        <pattern>%d %p %c{1.} [%t] %m%n</pattern>
      </PatternLayout>
      <TimeBasedTriggeringPolicy />
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <MapFilter onMatch="ACCEPT" onMismatch="NEUTRAL" operator="or">
        <KeyValuePair key="eventId" value="Login"/>
        <KeyValuePair key="eventId" value="Logout"/>
      </MapFilter>
      <AppenderRef ref="RollingFile">
      </AppenderRef>
    </Root>
  </Loggers>
</Configuration>
```

这第三个示例配置与前面的示例也功能相同，因为配置的logger仅是root，并且root仅配置了单个appender引用。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <RollingFile name="RollingFile" fileName="logs/app.log"
                 filePattern="logs/app-%d{MM-dd-yyyy}.log.gz">
      <BurstFilter level="INFO" rate="16" maxBurst="100"/>
      <PatternLayout>
        <pattern>%d %p %c{1.} [%t] %m%n</pattern>
      </PatternLayout>
      <TimeBasedTriggeringPolicy />
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile">
        <MapFilter onMatch="ACCEPT" onMismatch="NEUTRAL" operator="or">
          <KeyValuePair key="eventId" value="Login"/>
          <KeyValuePair key="eventId" value="Logout"/>
        </MapFilter>
      </AppenderRef>
    </Root>
  </Loggers>
</Configuration>
```

## MarkerFilter

MarkerFilter使用配置的Marker值与包含在LogEvent中的Marker进行比较。当Marker名与日志事件的Marker或其父Marker相匹配中了，就表示成功匹配了。

MarkerFilter参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| marker | String | 要比较的Marker名。 |
| onMatch | String | 当满足过滤器设置的条件后进行的操作。可能是ACCEPT，DENY或NEUTRAL。默认值为NEUTRAL。 |
| onMismatch | String | 当不满足过滤器设置的条件后进行的操作。可能是ACCEPT，DENY或NEUTRAL。默认值为DENY。 |

示例，只有当标记匹配时，才允许写入日志：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <RollingFile name="RollingFile" fileName="logs/app.log"
                 filePattern="logs/app-%d{MM-dd-yyyy}.log.gz">
      <MarkerFilter marker="FLOW" onMatch="ACCEPT" onMismatch="DENY"/>
      <PatternLayout>
        <pattern>%d %p %c{1.} [%t] %m%n</pattern>
      </PatternLayout>
      <TimeBasedTriggeringPolicy />
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
    </Root>
  </Loggers>
</Configuration>
```

## RegexFilter

RegexFilter可以将格式化或未格式化的消息与正则表达式进行比较。

RegexFilter参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| regex | String | 正则表达式。 |
| useRawMsg | boolean | 如果为true，则将使用未格式化的消息，否则将使用格式化的消息。默认值为false。 |
| onMatch | String | 当满足过滤器设置的条件后进行的操作。可能是ACCEPT，DENY或NEUTRAL。默认值为NEUTRAL。 |
| onMismatch | String | 当不满足过滤器设置的条件后进行的操作。可能是ACCEPT，DENY或NEUTRAL。默认值为DENY。 |

示例，只允许事件里包含“test”的输入appender中：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <RollingFile name="RollingFile" fileName="logs/app.log"
                 filePattern="logs/app-%d{MM-dd-yyyy}.log.gz">
      <RegexFilter regex=".* test .*" onMatch="ACCEPT" onMismatch="DENY"/>
      <PatternLayout>
        <pattern>%d %p %c{1.} [%t] %m%n</pattern>
      </PatternLayout>
      <TimeBasedTriggeringPolicy />
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
    </Root>
  </Loggers>
</Configuration>
```

## Script

ScriptFilter执行一个返回true或false的脚本。

ScriptFilter参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| script | Script, ScriptFile or ScriptRef | 指定要执行实际逻辑的Script元素。 |
| onMatch | String | 当满足过滤器设置的条件后进行的操作。可能是ACCEPT，DENY或NEUTRAL。默认值为NEUTRAL。 |
| onMismatch | String | 当不满足过滤器设置的条件后进行的操作。可能是ACCEPT，DENY或NEUTRAL。默认值为DENY。 |

Script参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| configuration | Configuration | 有ScriptFilter的配置文件。 |
| level | Level | 与事件相关联的日志记录级别。仅当配置为全局过滤器（global filter）的script时有效。 |
| loggerName | String | 记录器的名称。仅当配置为全局过滤器（global filter）的script时有效。 |
| logEvent | LogEvent | 要处理的LogEvent。配置为全局过滤器时不能使用。 |
| marker | Marker | 传递给log调用的Marker。仅当配置为全局过滤器（global filter）的script时有效。 |
| message | Message | 与log调用相关联的消息。仅当配置为全局过滤器（global filter）的script时有效。 |
| parameters | Object\[\] | 传递给log调用的参数。仅当配置为全局过滤器（global filter）的script时有效。有些Message可能包含有一些参数（parameters）。 |
| throwable | Throwable | 传递给log调用的Throwable。仅当配置为全局过滤器（global filter）的script时有效。有些Message可能包含有Throwable。 |
| substitutor | StrSubstitutor | StrSubstitutor用于替换变量。 |

下面的示例显示了如何声明脚本，然后在特定组件中引用它们。有关如何使用`Script`节点直接在配置文件中嵌入脚本代码的示例，请参见[ScriptCondition](./appenders.md#ScriptCondition)。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="ERROR">
  <Scripts>
    <ScriptFile name="filter.js" language="JavaScript" path="src/test/resources/scripts/filter.js" charset="UTF-8" />
    <ScriptFile name="filter.groovy" language="groovy" path="src/test/resources/scripts/filter.groovy" charset="UTF-8" />
  </Scripts>
  <Appenders>
    <List name="List">
      <PatternLayout pattern="[%-5level] %c{1.} %msg%n"/>
    </List>
  </Appenders>
  <Loggers>
    <Logger name="TestJavaScriptFilter" level="trace" additivity="false">
      <AppenderRef ref="List">
        <ScriptFilter onMatch="ACCEPT" onMisMatch="DENY">
          <ScriptRef ref="filter.js" />
        </ScriptFilter>
      </AppenderRef>
    </Logger>
    <Logger name="TestGroovyFilter" level="trace" additivity="false">
      <AppenderRef ref="List">
        <ScriptFilter onMatch="ACCEPT" onMisMatch="DENY">
          <ScriptRef ref="filter.groovy" />
        </ScriptFilter>
      </AppenderRef>
    </Logger>
    <Root level="trace">
      <AppenderRef ref="List" />
    </Root>
  </Loggers>
</Configuration>
```

## StructuredDataFilter

StructuredDataFilter是一个MapFilter，它还可以对事件id，type和message进行过滤。

StructuredDataFilter参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| keyValuePair | KeyValuePair\[\] | 定义一个或多个KeyValuePair元素，其中key为map里的键，value是要匹配的值。“id”，“id.name”，“type”和“message”分别用于在StructuredDataId，StructuredDataId的name，type和格式化消息（formatted message）上匹配。如果指定多次相同的key，则对该key的检查将自动为“or”逻辑，因为一个Map只能包含一个value。 |
| operator | String | 如果operator是“or”，则只要匹配中任何一个key/value对则认为是匹配中了，否则所有key/value对必须全匹配。 |
| onMatch | String | 当满足过滤器设置的条件后进行的操作。可能是ACCEPT，DENY或NEUTRAL。默认值为NEUTRAL。 |
| onMismatch | String | 当不满足过滤器设置的条件后进行的操作。可能是ACCEPT，DENY或NEUTRAL。默认值为DENY。 |

配置中，StructuredDataFilter可用于记录特定事件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <StructuredDataFilter onMatch="ACCEPT" onMismatch="NEUTRAL" operator="or">
    <KeyValuePair key="id" value="Login"/>
    <KeyValuePair key="id" value="Logout"/>
  </StructuredDataFilter>
  <Appenders>
    <RollingFile name="RollingFile" fileName="logs/app.log"
                 filePattern="logs/app-%d{MM-dd-yyyy}.log.gz">
      <BurstFilter level="INFO" rate="16" maxBurst="100"/>
      <PatternLayout>
        <pattern>%d %p %c{1.} [%t] %m%n</pattern>
      </PatternLayout>
      <TimeBasedTriggeringPolicy />
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
    </Root>
  </Loggers>
</Configuration>
```

## ThreadContextMapFilter (或ContextMapFilter)

ThreadContextMapFilter或ContextMapFilter可以对当前上下文中的数据元素进行过滤。默认情况下是ThreadContextMap。

ContextMapFilter参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| keyValuePair | KeyValuePair\[\] | 定义一个或多个KeyValuePair元素，其中key为map里的键，value是要匹配的值。如果指定多次相同的key，则对该key的检查将自动为“or”逻辑，因为一个Map只能包含一个value。 |
| operator | String | 如果operator是“or”，则只要匹配中任何一个key/value对则认为是匹配中了，否则所有key/value对必须全匹配。 |
| onMatch | String | 当满足过滤器设置的条件后进行的操作。可能是ACCEPT，DENY或NEUTRAL。默认值为NEUTRAL。 |
| onMismatch | String | 当不满足过滤器设置的条件后进行的操作。可能是ACCEPT，DENY或NEUTRAL。默认值为DENY。 |

示例：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <ContextMapFilter onMatch="ACCEPT" onMismatch="NEUTRAL" operator="or">
    <KeyValuePair key="User1" value="DEBUG"/>
    <KeyValuePair key="User2" value="WARN"/>
  </ContextMapFilter>
  <Appenders>
    <RollingFile name="RollingFile" fileName="logs/app.log"
                 filePattern="logs/app-%d{MM-dd-yyyy}.log.gz">
      <BurstFilter level="INFO" rate="16" maxBurst="100"/>
      <PatternLayout>
        <pattern>%d %p %c{1.} [%t] %m%n</pattern>
      </PatternLayout>
      <TimeBasedTriggeringPolicy />
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
    </Root>
  </Loggers>
</Configuration>
```

ContextMapFilter也可以应用于logger进行过滤：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <RollingFile name="RollingFile" fileName="logs/app.log"
                 filePattern="logs/app-%d{MM-dd-yyyy}.log.gz">
      <BurstFilter level="INFO" rate="16" maxBurst="100"/>
      <PatternLayout>
        <pattern>%d %p %c{1.} [%t] %m%n</pattern>
      </PatternLayout>
      <TimeBasedTriggeringPolicy />
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
      <ContextMapFilter onMatch="ACCEPT" onMismatch="NEUTRAL" operator="or">
        <KeyValuePair key="foo" value="bar"/>
        <KeyValuePair key="User2" value="WARN"/>
      </ContextMapFilter>
    </Root>
  </Loggers>
</Configuration>
```

## ThresholdFilter

如果LogEvent中的级别与配置的级别相同或更高，则此filter返回onMatch值，否则返回onMismatch值。例如，如果ThresholdFilter配置了级别ERROR，并且LogEvent包含级别DEBUG，那么将返回onMismatch值，因为ERROR事件比DEBUG级别更高。

ThresholdFilter参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| level | String | 要匹配的有效Level名。 |
| onMatch | String | 当满足过滤器设置的条件后进行的操作。可能是ACCEPT，DENY或NEUTRAL。默认值为NEUTRAL。 |
| onMismatch | String | 当不满足过滤器设置的条件后进行的操作。可能是ACCEPT，DENY或NEUTRAL。默认值为DENY。 |

示例，只有当级别匹配时，才允许appender写入事件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <RollingFile name="RollingFile" fileName="logs/app.log"
                 filePattern="logs/app-%d{MM-dd-yyyy}.log.gz">
      <ThresholdFilter level="TRACE" onMatch="ACCEPT" onMismatch="DENY"/>
      <PatternLayout>
        <pattern>%d %p %c{1.} [%t] %m%n</pattern>
      </PatternLayout>
      <TimeBasedTriggeringPolicy />
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
    </Root>
  </Loggers>
</Configuration>
```

## TimeFilter

时间过滤器可以用于限制到一天中的具体时间点。

TimeFilter参数：

| 参数名 | 类型 | 描述 |
| ---- | ---- | ---- |
| start | String | HH:mm:ss格式的开始时间点。 |
| end | String | HH:mm:ss格式的结束时间点。 |
| timezone | String | 与事件时间戳进行比较时使用的时区。 |
| onMatch | String | 当满足过滤器设置的条件后进行的操作。可能是ACCEPT，DENY或NEUTRAL。默认值为NEUTRAL。 |
| onMismatch | String | 当不满足过滤器设置的条件后进行的操作。可能是ACCEPT，DENY或NEUTRAL。默认值为DENY。 |

示例，只允许由appender在每天的5:00到5:30使用默认时区写入事件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <RollingFile name="RollingFile" fileName="logs/app.log"
                 filePattern="logs/app-%d{MM-dd-yyyy}.log.gz">
      <TimeFilter start="05:00:00" end="05:30:00" onMatch="ACCEPT" onMismatch="DENY"/>
      <PatternLayout>
        <pattern>%d %p %c{1.} [%t] %m%n</pattern>
      </PatternLayout>
      <TimeBasedTriggeringPolicy />
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
    </Root>
  </Loggers>
</Configuration>
```
