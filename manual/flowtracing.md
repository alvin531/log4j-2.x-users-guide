# Log4j 2 API - Flow Tracing

## 流程跟踪（Flow Tracing）

Logger提供了非常有用的跟踪应用程序执行过程的日志记录方法。这些方法生成的日志事件可以跟其他的调试日志分开过滤。提倡使用这些方法，以达到这些目的：

* 在开发阶段协助诊断问题，无需调试
* 在无法调试的生产环境中协助诊断问题
* 协助新开发人员熟悉应用程序。

最常用的方法是entry()或traceEntry()和exit()或traceExit()。entry()或traceEntry()放在方法开头。entry()接收0到4个参数。通常这些都是传递给方法的参数。traceEntry()接收一个格式化的String和一个可变参数列表，或一个Message。entry()和traceEntry()方法使用TRACE级别记录日志，并使用名字为“ENTER”的Marker，它也是一个“FLOW”Marker，所有消息字符串都以“event”开头，即使使用了格式化String或Message。

entry和traceEntry方法主要的区别是，entry方法接收变量列表，每一个都是方法的参数。而traceEntry方法接收一个格式化String，后面跟一个变量列表，可能会被包含在格式化的String中。无法统一这两个方法，因为传递进来的第一个String到底是一个方法参数，还是一个格式化String，是无法区分的。

exit()或traceExit()方法放在return语句之前，或在没有return语句的方法最后。exit()和traceExit()的调用可以不带参数。通常，返回void的方法使用exit()或traceExit()，而返回Object的方法使用exit(Object obj)或traceExit(object, new SomeMessage(object))。exit()和traceExit()方法使用TRACE级别记录，并使用名字为“EXIT”的Marker，它也是一个“FLOW”Marker，所有消息字符串都以“exit”开头，即使使用了格式化String或Message。

throwing()方法可以在应用程序捕获无法处理的异常处调用，比如RuntimeException。这可确保在需要时进行适当的问题诊断。生成的日志使用ERROR级别，并使用名字为“THROWING”的Marker，它也是一个“EXCEPTION”Marker。

catching()方法可以在应用程序捕获异常处调用，此时这个异常不会被重新抛出，或附加到另一个异常抛出。生成的日志使用ERROR级别，并使用名字为“CATCHING”的Marker，其也是“EXCEPTION”Marker。

下面的例子显示了一个典型的使用这些方法的简单应用程序。没有使用throwing()，因为没有显式抛出和不处理的异常。

```java
package com.test;

import org.apache.logging.log4j.Logger;
import org.apache.logging.log4j.LogManager;

import java.util.Random;

public class TestService {
    private Logger logger = LogManager.getLogger(TestService.class.getName());

    private String[] messages = new String[] {
        "Hello, World",
        "Goodbye Cruel World",
        "You had me at hello"
    };
    private Random rand = new Random(1);

    public void setMessages(String[] messages) {
        logger.traceEntry(new JsonMessage(messages));
        this.messages = messages;
        logger.traceExit();
    }

    public String[] getMessages() {
        logger.traceEntry();
        return logger.traceExit(messages, new JsonMessage(messages));
    }

    public String retrieveMessage() {
        logger.entry();

        String testMsg = getMessage(getKey());

        return logger.exit(testMsg);
    }

    public void exampleException() {
        logger.entry();
        try {
            String msg = messages[messages.length];
            logger.error("An exception should have been thrown");
        } catch (Exception ex) {
            logger.catching(ex);
        }
        logger.exit();
    }

    public String getMessage(int key) {
        logger.entry(key);

        String value = messages[key];

        return logger.exit(value);
    }

    private int getKey() {
        logger.entry();
        int key = rand.nextInt(messages.length);
        return logger.exit(key);
    }
}
```

此测试类使用上面的service来生成日志记录。

```java
package com.test;

public class App {

    public static void main( String[] args ) {
        TestService service = new TestService();
        service.retrieveMessage();
        service.retrieveMessage();
        service.exampleException();
    }
}
```

下面的配置将把所有日志输出到target/test.log。FileAppender的模式包括类名，行号和方法名。在模式中包括这些信息是非常有价值的。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="error">
  <Appenders>
    <Console name="Console" target="SYSTEM_OUT">
      <ThresholdFilter level="ERROR" onMatch="ACCEPT" onMismatch="DENY"/>
      <!-- Flow tracing is most useful with a pattern that shows location.
           Below pattern outputs class, line number and method name. -->
      <PatternLayout pattern="%d{HH:mm:ss.SSS} %-5level %class{36} %L %M - %msg%xEx%n"/>
    </Console>
    <File name="log" fileName="target/test.log" append="false">
      <PatternLayout pattern="%d{HH:mm:ss.SSS} %-5level %class{36} %L %M - %msg%xEx%n"/>
    </File>
  </Appenders>
  <Loggers>
    <Root level="trace">
      <AppenderRef ref="log"/>
    </Root>
  </Loggers>
</Configuration>
```

下面是日志的结果：

```shell
19:08:07.056 TRACE com.test.TestService 19 retrieveMessage -  entry
19:08:07.060 TRACE com.test.TestService 46 getKey -  entry
19:08:07.060 TRACE com.test.TestService 48 getKey -  exit with (0)
19:08:07.060 TRACE com.test.TestService 38 getMessage -  entry parms(0)
19:08:07.060 TRACE com.test.TestService 42 getMessage -  exit with (Hello, World)
19:08:07.060 TRACE com.test.TestService 23 retrieveMessage -  exit with (Hello, World)
19:08:07.061 TRACE com.test.TestService 19 retrieveMessage -  entry
19:08:07.061 TRACE com.test.TestService 46 getKey -  entry
19:08:07.061 TRACE com.test.TestService 48 getKey -  exit with (1)
19:08:07.061 TRACE com.test.TestService 38 getMessage -  entry parms(1)
19:08:07.061 TRACE com.test.TestService 42 getMessage -  exit with (Goodbye Cruel World)
19:08:07.061 TRACE com.test.TestService 23 retrieveMessage -  exit with (Goodbye Cruel World)
19:08:07.062 TRACE com.test.TestService 27 exampleException -  entry
19:08:07.077 DEBUG com.test.TestService 32 exampleException - catching java.lang.ArrayIndexOutOfBoundsException: 3
        at com.test.TestService.exampleException(TestService.java:29) [classes/:?]
        at com.test.App.main(App.java:9) [classes/:?]
        at com.test.AppTest.testApp(AppTest.java:15) [test-classes/:?]
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[?:1.6.0_29]
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:39) ~[?:1.6.0_29]
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:25) ~[?:1.6.0_29]
        at java.lang.reflect.Method.invoke(Method.java:597) ~[?:1.6.0_29]
        at org.junit.internal.runners.TestMethodRunner.executeMethodBody(TestMethodRunner.java:99) [junit-4.3.1.jar:?]
        at org.junit.internal.runners.TestMethodRunner.runUnprotected(TestMethodRunner.java:81) [junit-4.3.1.jar:?]
        at org.junit.internal.runners.BeforeAndAfterRunner.runProtected(BeforeAndAfterRunner.java:34) [junit-4.3.1.jar:?]
        at org.junit.internal.runners.TestMethodRunner.runMethod(TestMethodRunner.java:75) [junit-4.3.1.jar:?]
        at org.junit.internal.runners.TestMethodRunner.run(TestMethodRunner.java:45) [junit-4.3.1.jar:?]
        at org.junit.internal.runners.TestClassMethodsRunner.invokeTestMethod(TestClassMethodsRunner.java:66) [junit-4.3.1.jar:?]
        at org.junit.internal.runners.TestClassMethodsRunner.run(TestClassMethodsRunner.java:35) [junit-4.3.1.jar:?]
        at org.junit.internal.runners.TestClassRunner$1.runUnprotected(TestClassRunner.java:42) [junit-4.3.1.jar:?]
        at org.junit.internal.runners.BeforeAndAfterRunner.runProtected(BeforeAndAfterRunner.java:34) [junit-4.3.1.jar:?]
        at org.junit.internal.runners.TestClassRunner.run(TestClassRunner.java:52) [junit-4.3.1.jar:?]
        at org.apache.maven.surefire.junit4.JUnit4TestSet.execute(JUnit4TestSet.java:35) [surefire-junit4-2.7.2.jar:2.7.2]
        at org.apache.maven.surefire.junit4.JUnit4Provider.executeTestSet(JUnit4Provider.java:115) [surefire-junit4-2.7.2.jar:2.7.2]
        at org.apache.maven.surefire.junit4.JUnit4Provider.invoke(JUnit4Provider.java:97) [surefire-junit4-2.7.2.jar:2.7.2]
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[?:1.6.0_29]
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:39) ~[?:1.6.0_29]
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:25) ~[?:1.6.0_29]
        at java.lang.reflect.Method.invoke(Method.java:597) ~[?:1.6.0_29]
        at org.apache.maven.surefire.booter.ProviderFactory$ClassLoaderProxy.invoke(ProviderFactory.java:103) [surefire-booter-2.7.2.jar:2.7.2]
        at $Proxy0.invoke(Unknown Source) [?:?]
        at org.apache.maven.surefire.booter.SurefireStarter.invokeProvider(SurefireStarter.java:150) [surefire-booter-2.7.2.jar:2.7.2]
        at org.apache.maven.surefire.booter.SurefireStarter.runSuitesInProcess(SurefireStarter.java:91) [surefire-booter-2.7.2.jar:2.7.2]
        at org.apache.maven.surefire.booter.ForkedBooter.main(ForkedBooter.java:69) [surefire-booter-2.7.2.jar:2.7.2]
19:08:07.087 TRACE com.test.TestService 34 exampleException -  exit
```

在上面的例子中，将root logger级别改为DEBUG将大大降低输出。

```shell
19:13:24.963 DEBUG com.test.TestService 32 exampleException - catching java.lang.ArrayIndexOutOfBoundsException: 3
        at com.test.TestService.exampleException(TestService.java:29) [classes/:?]
        at com.test.App.main(App.java:9) [classes/:?]
        at com.test.AppTest.testApp(AppTest.java:15) [test-classes/:?]
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[?:1.6.0_29]
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:39) ~[?:1.6.0_29]
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:25) ~[?:1.6.0_29]
        at java.lang.reflect.Method.invoke(Method.java:597) ~[?:1.6.0_29]
        at org.junit.internal.runners.TestMethodRunner.executeMethodBody(TestMethodRunner.java:99) [junit-4.3.1.jar:?]
        at org.junit.internal.runners.TestMethodRunner.runUnprotected(TestMethodRunner.java:81) [junit-4.3.1.jar:?]
        at org.junit.internal.runners.BeforeAndAfterRunner.runProtected(BeforeAndAfterRunner.java:34) [junit-4.3.1.jar:?]
        at org.junit.internal.runners.TestMethodRunner.runMethod(TestMethodRunner.java:75) [junit-4.3.1.jar:?]
        at org.junit.internal.runners.TestMethodRunner.run(TestMethodRunner.java:45) [junit-4.3.1.jar:?]
        at org.junit.internal.runners.TestClassMethodsRunner.invokeTestMethod(TestClassMethodsRunner.java:66) [junit-4.3.1.jar:?]
        at org.junit.internal.runners.TestClassMethodsRunner.run(TestClassMethodsRunner.java:35) [junit-4.3.1.jar:?]
        at org.junit.internal.runners.TestClassRunner$1.runUnprotected(TestClassRunner.java:42) [junit-4.3.1.jar:?]
        at org.junit.internal.runners.BeforeAndAfterRunner.runProtected(BeforeAndAfterRunner.java:34) [junit-4.3.1.jar:?]
        at org.junit.internal.runners.TestClassRunner.run(TestClassRunner.java:52) [junit-4.3.1.jar:?]
        at org.apache.maven.surefire.junit4.JUnit4TestSet.execute(JUnit4TestSet.java:35) [surefire-junit4-2.7.2.jar:2.7.2]
        at org.apache.maven.surefire.junit4.JUnit4Provider.executeTestSet(JUnit4Provider.java:115) [surefire-junit4-2.7.2.jar:2.7.2]
        at org.apache.maven.surefire.junit4.JUnit4Provider.invoke(JUnit4Provider.java:97) [surefire-junit4-2.7.2.jar:2.7.2]
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[?:1.6.0_29]
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:39) ~[?:1.6.0_29]
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:25) ~[?:1.6.0_29]
        at java.lang.reflect.Method.invoke(Method.java:597) ~[?:1.6.0_29]
        at org.apache.maven.surefire.booter.ProviderFactory$ClassLoaderProxy.invoke(ProviderFactory.java:103) [surefire-booter-2.7.2.jar:2.7.2]
        at $Proxy0.invoke(Unknown Source) [?:?]
        at org.apache.maven.surefire.booter.SurefireStarter.invokeProvider(SurefireStarter.java:150) [surefire-booter-2.7.2.jar:2.7.2]
        at org.apache.maven.surefire.booter.SurefireStarter.runSuitesInProcess(SurefireStarter.java:91) [surefire-booter-2.7.2.jar:2.7.2]
        at org.apache.maven.surefire.booter.ForkedBooter.main(ForkedBooter.java:69) [surefire-booter-2.7.2.jar:2.7.2]
```
