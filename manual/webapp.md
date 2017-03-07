# 在Web应用程序中使用Log4j 2

在Java EE Web应用程序中使用Log4j或任何其他日志记录框架时，必须格外小心。在容器关闭或Web应用程序取消部署时，对日志资源进行正确清理（关闭数据库连接，关闭文件等）非常重要。由于Web应用程序中的类加载器的性质，Log4j资源无法通过正常方式清除。 当Web应用程序部署时，Log4j必须“启动”；而在Web应用程序取消部署时，则必须“关闭”。它的工作原理取决于您的应用程序是Servlet 3.0或更高版本，还是Servlet 2.5 Web应用程序。

无论哪种情况，您都需要将`log4j-web`模块添加到部署中，详细信息请参阅[Maven，Ivy和Gradle Artifacts](https://logging.apache.org/log4j/2.x/maven-artifacts.html)手册页。

**为了避免出现问题，当包括`log4j-web`jar时，Log4j将自动禁用shutdown hook。**

## 配置

Log4j允许在web.xml中使用`log4jConfiguration`这个context参数来指定配置文件。Log4j将通过以下方式来搜索配置文件：

1. 如果提供了一个文件位置，它将作为一个servlet context资源进行搜索。例如，如果`log4jConfiguration`包含“logging.xml”，则Log4j将在Web应用程序的根目录中查找具有该名称的文件。
2. 如果没有定义位置，Log4j将搜索在WEB-INF目录中以“log4j2”开头的文件。如果找到多个文件，且如果存在以“log4j2-_name_”开头的文件，其中 _name_ 是Web应用程序的名称，则将使用它。否则将使用第一个文件。
3. 按类路径（classpath）和文件URL等方式进行搜索和定位配置文件。

## Servlet 3.0或更高版本的Web应用程序

Servlet 3.0或更高版本的Web应用程序是指在`<web-app>`里的`version`属性具的值是“3.0”或更高。当然，应用程序也必须在兼容的Web容器中运行。比如：Tomcat 7.0及更高版本，GlassFish 3.0及更高版本，JBoss 7.0及更高版本，Oracle WebLogic 12c及更高版本，以及IBM WebSphere 8.0及更高版本。

### 简介

Log4j 2“只是可以工作”在Servlet 3.0或更高版本的Web应用程序中。它能够在应用程序部署时自动启动，且在应用程序取消部署时关闭。得益于Servlet 3.0增加的[ServletContainerInitializer](http://docs.oracle.com/javaee/6/api/javax/servlet/ServletContainerInitializer.html) API添加到Servlet 3.0，`Filter`和`ServletContextListener`类可以在Web应用程序启动时动态注册。

**请注意了！** 出于性能原因，容器经常忽略某些的不包含TLDs或`ServletContainerInitializers`的特定JAR，且不会对它们进行扫描，从而无法获取Web描述符片段（web-fragment）和进行初始化。值得留意的是，Tomcat 7 < 7.0.43的版本忽略名为log4j\*.jar的所有JAR包，这会让这个功能无法正常运行。不过，这已在Tomcat 7.0.43，Tomcat 8及更高版本中得到修复。若在使用Tomcat 7 < 7.0.4的版本，您需要更改`catalina.properties`并从`jarsToSkip`属性中删除“log4j\*.jar”。如果在其他容器跳过扫描Log4j JAR文件，您可能需要在其上面执行类似的操作。

### 描述

Log4j 2 Web JAR需要在应用程序中配置其次序（order）高于其他Web描述符片段（web-fragment）。它包含了`ServletContainerInitializer`（[Log4jServletContainerInitializer](https://logging.apache.org/log4j/2.x/log4j-web/apidocs/org/apache/logging/log4j/web/Log4jServletContainerInitializer.html)），容器会自动发现和初始化。该类将[Log4jServletContextListener](https://logging.apache.org/log4j/2.x/log4j-web/apidocs/org/apache/logging/log4j/web/Log4jServletContextListener.html)和[Log4jServletFilter](https://logging.apache.org/log4j/2.x/log4j-web/apidocs/org/apache/logging/log4j/web/Log4jServletFilter.html)添加到`ServletContext`中。这些类会正确地初始化和消毁Log4j配置。

对于某些用户，可能无法自动启动Log4j。您可以使用`isLog4jAutoInitializationDisabled`这个context参数里禁用此功能。只需将其添加到部署描述符中，设为“true”即可禁用自动初始化功能。您 _必须_ 在`web.xml`中进行设置。如果以编程方式设置，Log4j将无法检测到该设置。

```xml
<context-param>
    <param-name>isLog4jAutoInitializationDisabled</param-name>
    <param-value>true</param-value>
</context-param>
```

一旦禁用自动初始化后，必须像Servlet 2.5 Web应用程序一样初始化Log4j。您必须保证这种初始化发生在任何其他应用程序代码（如Spring Framework启动代码）执行之前。

您可以使用`log4jContextName`，`log4jConfiguration`和/或`isLog4jContextSelectorNamed`等context参数来自定义listener和filter的功能。有关详情，请参阅下面的Context Parameters部分。您 _不能_ 在部署描述符（web.xml）中，或在Servlet 3.0或更高版本应用程序中的另一个初始化类或listener中，手动配置`Log4jServletContextListener`或`Log4jServletFilter`，_除非您使用`isLog4jAutoInitializationDisabled`禁用自动初始化_。否则将导致启动错误和未知的错误。

## Servlet 2.5 Web应用程序

Servlet 2.5的Web应用程序是指在`<web-app>`里的`version`属性具的值是“2.5”。`version`属性是唯一关注的点; 即使Web应用程序在Servlet 3.0或更高版本的容器中运行，如果`version`属性为“2.5”，则它是Servlet 2.5的Web应用程序。注意，Log4j 2不支持Servlet 2.4和较低版本的Web应用程序。

如果在Servlet 2.5的Web应用程序中使用Log4j，或者已使用`isLog4jAutoInitializationDisabled`这个context参数禁用了自动初始化，则必须在部署描述符中或以编程方式配置[Log4jServletContextListener](https://logging.apache.org/log4j/2.x/log4j-web/apidocs/org/apache/logging/log4j/web/Log4jServletContextListener.html)和[Log4jServletFilter](https://logging.apache.org/log4j/2.x/log4j-web/apidocs/org/apache/logging/log4j/web/Log4jServletFilter.html)。filter应匹配任何类型的所有请求。listener应该是应用程序中的第一个侦听器，filter应该是在应用程序中定义（define）和映射（map）的第一个filter。如以下web.xml代码：

```xml
<listener>
    <listener-class>org.apache.logging.log4j.web.Log4jServletContextListener</listener-class>
</listener>

<filter>
    <filter-name>log4jServletFilter</filter-name>
    <filter-class>org.apache.logging.log4j.web.Log4jServletFilter</filter-class>
</filter>
<filter-mapping>
    <filter-name>log4jServletFilter</filter-name>
    <url-pattern>/*</url-pattern>
    <dispatcher>REQUEST</dispatcher>
    <dispatcher>FORWARD</dispatcher>
    <dispatcher>INCLUDE</dispatcher>
    <dispatcher>ERROR</dispatcher>
    <dispatcher>ASYNC</dispatcher><!-- Servlet 3.0 w/ disabled auto-initialization only; not supported in 2.5 -->
</filter-mapping>
```

您可以使用`log4jContextName`，`log4jConfiguration`和/或`isLog4jContextSelectorNamed`等context参数来自定义listener和filter的功能。有关详情，请参阅下面的Context Parameters部分。

## Context Parameters

默认情况下，Log4j 2使用ServletContext的[context name](http://docs.oracle.com/javaee/6/api/javax/servlet/ServletContext.html#getServletContextName%28%29)来设置`LoggerContext`名称，并使用标准模式定位Log4j配置文件。有三个context参数可用于控制此功能。第一个是`isLog4jContextSelectorNamed`，指定是否应使用[JndiContextSelector](https://logging.apache.org/log4j/2.x/log4j-core/apidocs/org/apache/logging/log4j/core/selector/JndiContextSelector.html)选择context。如果未指定`isLog4jContextSelectorNamed`或除`true`之外的任何值，则假定为`false`。

如果`isLog4jContextSelectorNamed`为`true`，则必须指定`log4jContextName`或在`web.xml`中指定`display-name`；否则，应用程序将启动异常。在这种情况下，还可以指定`log4jConfiguration`，且必须是有效URI来定位配置文件；但是，此参数不是必需的。

如果`isLog4jContextSelectorNamed`不为`true`，则可以可选地指定`log4jConfiguration`且必须有效URI或路径，或者以“classpath:”开头以表示可在类路径上找到的配置文件。如果没有此参数，Log4j将使用标准机制来定位配置文件。

要指定这些context参数，您必须在部署描述符（web.xml）中指定它们，即使在Servlet 3.0环境中，不能在应用程序中编程设置。如果在一个listener里将它们添加到`ServletContext`，Log4j将在context参数可用之前被初始化了，它们就不会起作用。见以下示例。

### 设“myApplication”为Logging Context Name

```xml
<context-param>
    <param-name>log4jContextName</param-name>
    <param-value>myApplication</param-value>
</context-param>
```

### 设置Configuration Path为“/etc/myApp/myLogging.xml”

```xml
<context-param>
    <param-name>log4jConfiguration</param-name>
    <param-value>file:///etc/myApp/myLogging.xml</param-value>
</context-param>
```

### 使用`JndiContextSelector`

```xml
<context-param>
    <param-name>isLog4jContextSelectorNamed</param-name>
    <param-value>true</param-value>
</context-param>
<context-param>
    <param-name>log4jContextName</param-name>
    <param-value>appWithJndiSelector</param-value>
</context-param>
<context-param>
    <param-name>log4jConfiguration</param-name>
    <param-value>file:///D:/conf/myLogging.xml</param-value>
</context-param>
```

请注意，在这种情况下，您还必须将“Log4jContextSelector”系统属性设置为“org.apache.logging.log4j.core.selector.JndiContextSelector”。

## 在配置里使用Web应用程序信息

您可能需要在配置里使用有关Web应用程序的信息。例如，您可以将Web应用程序的context path嵌入到Rolling File Appender的名称中。有关详细信息，请参阅[Lookups](./lookups.md#WebLookup)章节中的WebLookup。

## JSP日志记录

您可以在JSP中使用Log4j 2，就如在其他Java代码中一样。只需获取一个`Logger`并调用它的方法来记录日志。但是，这需要您在JSP中使用Java代码，但一些开发团队是不允许这样做的。比如您有一个不熟悉使用Java的专用UI开发团队，您甚至可能会在JSP中禁用Java代码。

因此，Log4j 2提供了一个JSP标签库，使您能够在不使用任何Java代码的情况下记录日志。要阅读有关使用此标签库的更多信息，请阅读[Log4j标签库文档](https://logging.apache.org/log4j/2.x/log4j-taglib/index.html)。

**请注意了！** 如上所述，容器经常忽略一些不包含TLDs的某些JAR，且不会扫描TLD文件。重要的是，Tomcat 7 < 7.0.43会忽略名为log4j\*.jar的所有JAR，这就无法自动发现JSP标签库。Tomcat 6.x没有影响，并已在Tomcat 7.0.43，Tomcat 8及更高版本中修复。在Tomcat 7 < 7.0.43中，您需要更改`catalina.properties`并从`jarsToSkip`属性中删除“log4j\*.jar”。如果在其他容器跳过扫描Log4j JAR文件，您可能需要在其上面执行类似的操作。

## 异步请求和线程

异步请求的处理是相当棘手的，不管Servlet容器版本或还是配置，Log4j无法自动处理异步请求。当处理标准requests、forward、includes和error resources时，`Log4jServletFilter`将`LoggerContext`绑定到处理请求的线程。请求处理完成后，filter从线程移除`LoggerContext`。

类似地，当使用`javax.servlet.AsyncContext`分派内部请求时，`Log4jServletFilter`也将`LoggerContext`绑定到处理请求的线程，且在请求处理完成时移除绑定。但是，这只发生在通过`AsyncContext`分派的请求。除了内部分派的请求之外，还有其他异步操作。

例如，在启动`AsyncContext`后，您可以启动一个单独的线程来在后台处理请求，可能使用`ServletOutputStream`来响应。Filters无法在此线程拦截到该执行。Filters也不能拦截在后台非异步请求里启动的线程。无论您启动的全新线程还是从线程池取到的线程，都是如此。那么对于这些特殊线程要如何处理呢？

你可能不需要做任何事情。如果没有使用`isLog4jContextSelectorNamed`该context参数，则不需将`LoggerContext`绑定到线程。Log4j可以安全地定位到`LoggerContext`。在这些情况下，filter会适当地得到性能增益，且仅在创建新的logger时会产生。但是，如果指定了`isLog4jContextSelectorNamed`的值为“true”，则需要手动将`LoggerContext`绑定到异步线程。否则，Log4j将无法找到它。

幸运的是，Log4j在这些特殊情况下提供了一种将`LoggerContext`绑定到异步线程的简单机制。最简单的方法是调用`AsyncContext.start()`方法，给参数传递Runnable实例。

```java
import java.io.IOException;
import javax.servlet.AsyncContext;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.apache.logging.log4j.web.WebLoggerContextUtils;

public class TestAsyncServlet extends HttpServlet {

    @Override
    protected void doGet(final HttpServletRequest req, final HttpServletResponse resp) throws ServletException, IOException {
        final AsyncContext asyncContext = req.startAsync();
        asyncContext.start(WebLoggerContextUtils.wrapExecutionContext(this.getServletContext(), new Runnable() {
            @Override
            public void run() {
                final Logger logger = LogManager.getLogger(TestAsyncServlet.class);
                logger.info("Hello, servlet!");
            }
        }));
    }

    @Override
    protected void doPost(final HttpServletRequest req, final HttpServletResponse resp) throws ServletException, IOException {
        final AsyncContext asyncContext = req.startAsync();
        asyncContext.start(new Runnable() {
            @Override
            public void run() {
                final Log4jWebSupport webSupport =
                    WebLoggerContextUtils.getWebLifeCycle(TestAsyncServlet.this.getServletContext());
                webSupport.setLoggerContext();
                // do stuff
                webSupport.clearLoggerContext();
            }
        });
    }
}
```

当使用Java 1.8和lambda函数时，这可以稍微更方便一些，如下所示。

```java
import java.io.IOException;
import javax.servlet.AsyncContext;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.apache.logging.log4j.web.WebLoggerContextUtils;

public class TestAsyncServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        final AsyncContext asyncContext = req.startAsync();
        asyncContext.start(WebLoggerContextUtils.wrapExecutionContext(this.getServletContext(), () -> {
            final Logger logger = LogManager.getLogger(TestAsyncServlet.class);
            logger.info("Hello, servlet!");
        }));
    }
}
```

或者，您可以从`ServletContext`属性获取[Log4jWebLifeCycle](https://logging.apache.org/log4j/2.x/log4j-web/apidocs/org/apache/logging/log4j/web/Log4jWebLifeCycle.html)实例，在异步线程中的第一行代码里调用其`setLoggerContext`方法，并在异步线程中最后一行代码里调用`clearLoggerContext`方法。下面的代码演示了这一点。它使用容器线程池执行异步请求处理，将这个匿名`Runnable`类传递给`start`方法。

```java
import java.io.IOException;
import javax.servlet.AsyncContext;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.apache.logging.log4j.web.Log4jWebLifeCycle;
import org.apache.logging.log4j.web.WebLoggerContextUtils;

public class TestAsyncServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
         final AsyncContext asyncContext = req.startAsync();
        asyncContext.start(new Runnable() {
            @Override
            public void run() {
                final Log4jWebLifeCycle webLifeCycle =
                    WebLoggerContextUtils.getWebLifeCycle(TestAsyncServlet.this.getServletContext());
                webLifeCycle.setLoggerContext();
                try {
                    final Logger logger = LogManager.getLogger(TestAsyncServlet.class);
                    logger.info("Hello, servlet!");
                } finally {
                    webLifeCycle.clearLoggerContext();
                }
            }
        });
   }
}
```

注意，一旦你的线程完成处理，你必须调用`clearLoggerContext`。如果不这样做，将导致内存泄漏。如果使用线程池，它甚至可能中断您的容器中其他Web应用程序的日志记录。因此，这里的示例显示了在`finally`块中执行清除操作，这就可以始终被执行到。

## 使用Servlet Appender

Log4j提供了一个Servlet Appender，它使用servlet context作为日志目标。例如：

```xml
<Configuration status="WARN" name="ServletTest">

    <Appenders>
        <Servlet name="Servlet">
            <PatternLayout pattern="%m%n%ex{none}"/>
        </Servlet>
    </Appenders>

    <Loggers>
        <Root level="debug">
            <AppenderRef ref="Servlet"/>
        </Root>
    </Loggers>

</Configuration>
```

要避免对servlet context的异常被重复记录，必须在`PatternLayout`中使用`%ex{none}`，如示例所示。异常将从消息文本中省略，但作为Throwable对象传递到servlet context中。
