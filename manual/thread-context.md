# Log4j 2 API - ThreadContext

## Thread Context（线程上下文）

### 概述

Log4j引入了映射诊断上下文（Mapped Diagnostic Context）或MDC的概念。已经有很多文档简介和讨论了，包括[Log4j MDC: What and Why](http://veerasundar.com/blog/2009/10/log4j-mdc-mapped-diagnostic-context-what-and-why/)和[Log4j and the Mapped Diagnostic Context](http://blog.f12.no/wp/2004/12/09/log4j-and-the-mapped-diagnostic-context/)。此外，Log4j 1.x还提供对嵌套诊断上下文（Nested Diagnostic Context）或NDC的支持。它也已经有很多文档简介和讨论了，如[Log4j NDC](http://lstierneyltd.com/blog/development/log4j-nested-diagnostic-contexts-ndc/)。SLF4J/Logback也有自己的MDC实现，详细文档可看[Mapped Diagnostic Context](https://logback.qos.ch/manual/mdc.html)。

Log4j 2保持了MDC和NDC的概念，但将它们合并到单线程上下文（single Thread Context）中。线程上下文映射（Thread Context Map）等价于MDC，线程上下文栈（Thread Context Stack）等价于NDC。虽然它们常被用于非诊断问题上，但是它们在Log4j 2中仍被称为MDC和NDC，因为这两个概念已深入人心。

### Fish Tagging

现实中的系统几乎都要并发处理多个请求。如典型的多线程运用中，不同的线程处理不同的客户请求。日志记录特别适合于跟踪和调试这种复杂的分布式应用。区分一个客户端请求的日志输出和另一个客户端请求的常见方法是，为每个客户端都实例化独立的logger实例。这将导致logger膨胀，增加了logger的管理开销。

一个简易的做法是对同一个客户端的每个日志请求打上唯一标记。Neil Harrison在“_Pattern Languages of Program Design 3_”一书（由R.Martin，D.Riehle和F.Buschmann（Addison-Wesley，1997）编辑出版）的“Pattern Patterns of Logging Diagnostic Messages”章节里描述了这种方法。正如鱼可以被打上标记且跟踪其行为，用标记或数据元素集合来给日志事件打上标签，就可以跟踪整个请求和处理流程。我们称之为 _鱼标_（_Fish Tagging_）。

Log4j提供了两种机制来实现Fish Tagging；线程上下文映射（Thread Context Map）和线程上下文栈（Thread Context Stack）。线程上下文映射可以添加任何数量的键/值对来标识。线程上下文栈可以在栈上压入一个或多个数据，然后通过它们在栈中的顺序或由数据本身来标识。由于键/值对更灵活，当在请求的处理期间或当存在不只一个项时，推荐使用线程上下文映射。

为了使用线程上下文栈来唯一地标记每个请求，将上下文信息压入栈中。

```java
ThreadContext.push(UUID.randomUUID().toString()); // Add the fishtag;

logger.debug("Message 1");
.
.
.
logger.debug("Message 2");
.
.
ThreadContext.pop();
```

使用线程上下文映射来代替线程上下文栈。使用线程上下文映射时，在请求开始时加入相关联的属性，在处理后清除，如：

```java
ThreadContext.put("id", UUID.randomUUID().toString()); // Add the fishtag;
ThreadContext.put("ipAddress", request.getRemoteAddr());
ThreadContext.put("loginId", session.getAttribute("loginId"));
ThreadContext.put("hostName", request.getServerName());
.
logger.debug("Message 1");
.
.
logger.debug("Message 2");
.
.
ThreadContext.clear();
```

### CloseableThreadContext

在使用stack或map时，必须记得remove掉那些值。为了避免失误，有实现[AutoCloseable](http://docs.oracle.com/javase/7/docs/api/java/lang/AutoCloseable.html)接口的`CloseableThreadContext`类。这就可以把压入stack或放入map里的数据，在调用`close()`方法时被删除，或者使用try-with-resources自动清理。例如stack的使用：

```java
// Add to the ThreadContext stack for this try block only;
try (final CloseableThreadContext.Instance ctc = CloseableThreadContext.push(UUID.randomUUID().toString())) {

    logger.debug("Message 1");
.
.
    logger.debug("Message 2");
.
.
}
```

又例如map的使用：

```java
// Add to the ThreadContext map for this try block only;
try (final CloseableThreadContext.Instance ctc = CloseableThreadContext.put("id", UUID.randomUUID().toString())
                                                                .put("loginId", session.getAttribute("loginId"))) {

    logger.debug("Message 1");
.
.
    logger.debug("Message 2");
.
.
}
```

如果你使用线程池，你可以使用`putAll(final Map<String, String> values)`和/或`pushAll(List<String> messages)`方法来初始化CloseableThreadContext：

```java
for( final Session session : sessions ) {
    try (final CloseableThreadContext.Instance ctc = CloseableThreadContext.put("loginId", session.getAttribute("loginId"))) {
        logger.debug("Starting background thread for user");
        final Map<String, String> values = ThreadContext.getImmutableContext();
        final List<String> messages = ThreadContext.getImmutableStack().asList();
        executor.submit(new Runnable() {
          public void run() {
            try (final CloseableThreadContext.Instance ctc = CloseableThreadContext.putAll(values).pushAll(messages)) {
                logger.debug("Processing for user started");
                .
                logger.debug("Processing for user completed");
            }
          }
        );
    }
}
```

### 实现细节

默认情况下，Stack和Map是基于[ThreadLocal](http://docs.oracle.com/javase/6/docs/api/java/lang/ThreadLocal.html)进行管理的。通过设置系统属性`isThreadContextMapInheritable`为`true`，可以使用[InheritableThreadLocal](http://docs.oracle.com/javase/6/docs/api/java/lang/InheritableThreadLocal.html)来管理Map。这种情况下，Map的内容将被传递给子线程。不过，正如在[Executors](http://docs.oracle.com/javase/6/docs/api/java/util/concurrent/Executors.html#privilegedThreadFactory%28%29)类中讨论的，在使用线程池的情况下，ThreadContext可能不总是可以传递到工作线程。在这些情况下，池机制应当提供必要的措施来解决。那么，getContext()和cloneStack()方法就可以分别用于获取Map和Stack的副本。

注意，[ThreadContext](https://logging.apache.org/log4j/2.x/log4j-api/apidocs/org/apache/logging/log4j/ThreadContext.html)类的所有方法都是静态的。

### 在log里使用ThreadContext

[PatternLayout](https://logging.apache.org/log4j/2.x/log4j-api/apidocs/org/apache/logging/log4j/core/PatternLayout.html)提供了打印[ThreadContext](https://logging.apache.org/log4j/2.x/log4j-api/apidocs/org/apache/logging/log4j/ThreadContext.html)里的Map和Stack内容的机制。

* 使用`%X`，输出Map的全部内容。
* 使用`%X{key}`，输出指定的键值。
* 使用`%x`，输出Stack的全部内容。

### 自定义上下文数据注入器（for non thread-local conte data）

使用ThreadContext可以给日志打标志，因此相关的日志可以使用这些标志链接起来。不过这只适用于在同一线程（或子线程）输出的日志记录。

一些应用程序使用一种线程模型：将工作委派给其他线程去处理。在这种模型中，在一个线程本地（thread-local）map里的标签属性在其他线程是不可见的，那么在其他线程输出的日志也无法使用到这些标签属性。

Log4j 2.7添加了一种灵活的机制，用于从其他的数据源而不是ThreadContext里提取上下文数据，来给日志打标志。有关详细信息，请参阅[Log4j扩展](./extending.md#Custom_ContextDataInjector)章节。
