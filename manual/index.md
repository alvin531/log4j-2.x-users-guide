# Welcome to Log4j 2!

## 简介

几乎每个大型应用程序都会有自己日志或跟踪API。E.U. [SEMPER](http://www.semper.org/)项目也不例外，早在1996年初，它就编写了自己的跟踪API。经过无数次的提升、几次大的调整以及大量的开发工作，这个API就发展为了log4j，一个非常流行的日志工具包。该软件包是基于Apache Software License开源协议的。最新的log4j版本，包括全部源代码，类文件和文档可以在 http://logging.apache.org/log4j/2.x/index.html 找到。

在代码里插入日志记录语句是低成本的调试方法。它可能也是唯一的方法，因为调试器不总是可用的。特别是在多线程应用程序和分布式应用程序下。

大量的实践表明，日志记录是软件开发周期中的重要组成部分。它有几大好处。它提供了应用程序运行期间的精确上下文。一旦插入代码，日志的输出就不需要人工干预了。此外，日志可以被持久化到硬盘中，以便日后的分析。除了在开发周期中使用之外，一个丰富的日志包还可以作为一个审计工具。

正如Brian W. Kernighan和Rob Pike在著名的“_编程实践（The Practice of Programming）_”一书中所说的：

> 作为个人选择，我们倾向于不使用调试器，除了跟踪堆栈或获取一个或两个变量的值。 其中一个原因是，在复杂的数据结构和控制流的细节上很容易迷失; 我们发现，逐步调试应用程序相对于好好思考以及添加输出语句和自我检查代码在关键的地方来说，更没有生产力。逐步调试语句所需的时间比查看日志文件所需的时间更长。决定在何处放置日志输出语句而不是断点到代码的关键部分，即使假设我们知道在哪里，它需要更少的时间。更重要的是，调试语句会留在程序中；而调试会话是暂时的。

日志记录确实也有其缺点。它会影响到应用程序的性能。如果日志输出太详细了，日志输出不停滚动而让人没法查阅。为了减轻这些问题，log4j设计得可靠，快速，而且可扩展。由于日志记录很少是应用程序的主体部分，因此log4j API设计得简单易懂和易于使用。

## Log4j 2

Log4j 1.x已经被广泛使用。但是，这几年Log4j 1.x版本已做没什么提升。由于需要兼容老的Java版本，它变得更加难以维护；在2015年8月[终止对它的维护](https://blogs.apache.org/foundation/entry/apache_logging_services_project_announces)。它的替代方案是使用SLF4J/Logback，它们对框架做了很多改进。那么为什么还要用Log4j 2呢？这里有几个原因。

1. Log4j 2设计成可用于审计功能。Log4j 1.x和Logback在重新配置时都会丢失数据；而Log4j 2就不会。在Logback中，应用程序对Appender里的异常是不可见的；但在Log4j 2中，可以在Appender中进行配置，以允许异常通知到应用程序中。

2. Log4j 2基于[LMAX Disruptor](https://lmax-exchange.github.io/disruptor/)库开发了全新的[Asynchronous Loggers](./async.md)（异步日志记录）。在多线程场景中，Asynchronous Loggers的吞吐量比Log4j 1.x和Logback的高10倍，而且延迟更低。

3. Log4j 2对于独立应用程序是[无GC（garbage free）](./garbagefree.md)的；对于稳定的日志期间，Web应用程序也是尽可能少的GC。这减少了垃圾收集器上的压力，并提供更好的响应性能。

4. Log4j 2使用[插件系统](./plugins.md)，可以通过增加新的Appender、Filter、Layout、Lookup和Pattern Converter来扩展框架，而不需要对Log4j做任何的更改。

5. 因为插件系统的简单性，所以在配置的时候，可以不用具体指定所要处理的类型。

6. 支持[自定义日志级别](./customloglevels.md)。自定义的日志级别可以在代码或配置中定义。

7. 支持[lambda表达式](./api.md#LambdaSupport)。在Java 8上运行时，可以使用lambda表达式来延迟构建满足了当前日志级别的日志信息。不需要显式进行级别检查，代码更简洁。

8. 支持[Message对象](./messages.md)。Message可以在日志系统里传递有意义的复杂的消息，而且使用简明。用户可以创建自己的[`Message`](https://logging.apache.org/log4j/2.x/log4j-api/apidocs/org/apache/logging/log4j/message/Message.html)类型，自定义[Layout](./layouts.md)、[Filter](./filters.md)和[Lookup](./lookups.md)都可以操作它们。

9. Log4j 1.x支持在Appender上使用Filter。Logback通过TurboFilter来实现事件（events）被Logger处理之前进行过滤操作。Log4j 2支持更灵活的Filter，可配置为在事件被Logger处理之前进行相应的过滤操作，Filter配置成由Logger或在Appender上处理。

10. Logback里多数Appender不能配置一个Layout，只会以固定格式传输数据。绝大多数Log4j 2里的Appender都可以使用Layout，允许以任何所需的格式进行传输数据。

11. Log4j 1.x和Logback中的Layout返回一个String。这导致出[Logback Encoders](http://logback.qos.ch/manual/encoders.html)里描述的一起问题。Log4j 2采用更简单的方案，[Layout](./layouts.md)总是返回一个字节数组。这具有的优点是，它意味着它们可以在几乎任何Appender中使用，而不仅仅是写入OutputStream里。

12. [Syslog Appender](./appenders.md#SyslogAppender)支持TCP和UDP协议，以及支持BSD syslog和[RFC 5424](http://tools.ietf.org/html/rfc5424)格式。

13. Log4j 2利用Java 5并发技术，尽可能使用细粒度的锁。Log4j 1.x中存在死锁问题。其中许多都已在Logback中修复，但是Logback类仍然需要在比较高的级别上进行同步（synchronization）。

14. 它是一个Apache软件基金会项目，遵循所有ASF项目使用的社区和支持。如果您想要贡献或获得提交更改的权利，请按照[Contributing](http://jakarta.apache.org/site/contributing.html)中概述的进行操作。
