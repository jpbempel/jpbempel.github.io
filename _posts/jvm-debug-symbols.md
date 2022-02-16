# JVM debug symbols

Recently I stumbole upon this tweet:

https://twitter.com/ne_skazu/status/1491158726258364416
When you are used to work on the JVM it's ture that's very convenient and magic.
Also, you are familiar with stacktraces like for example the ones that are associated with exceptions:

```java
java.lang.RuntimeException: Expected: controller used to showcase what happens when an exception is thrown
        at org.springframework.samples.petclinic.system.CrashController.triggerException(CrashController.java:36) ~[classes!/:2.2.0.BUILD-SNAPSHOT]
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[na:1.8.0_272]
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62) ~[na:1.8.0_272]
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[na:1.8.0_272]
        at java.lang.reflect.Method.invoke(Method.java:498) ~[na:1.8.0_272]
        at org.springframework.web.method.support.InvocableHandlerMethod.doInvoke(InvocableHandlerMethod.java:197) ~[spring-web-5.3.6.jar!/:5.3.6]
        at org.springframework.web.method.support.InvocableHandlerMethod.invokeForRequest(InvocableHandlerMethod.java:141) ~[spring-web-5.3.6.jar!/:5.3.6]
        at org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod.invokeAndHandle(ServletInvocableHandlerMethod.java:106) ~[spring-webmvc-5.3.6.jar!/:5.3.6]
 // ... skipped for brevity ...
        at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:49) [tomcat-embed-core-9.0.45.jar!/:na]
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149) [na:1.8.0_272]
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624) [na:1.8.0_272]
        at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61) [tomcat-embed-core-9.0.45.jar!/:na]
        at java.lang.Thread.run(Thread.java:748) [na:1.8.0_272]
```
But how the JVM managed to keep and gather those line information, moreover with JITed code?

## javac

javac by default
test with an exception to see the difference

### gradle defaults:
gradle invoke javac with -g by default:
- https://docs.gradle.org/current/javadoc/org/gradle/api/tasks/compile/CompileOptions.html?_ga=2.42477413.1942252367.1645032818-155398221.1645032818#isDebug--
- https://github.com/gradle/gradle/blob/37911bb86d02d26a5d2ce3f23e01c0d767e3bb91/subprojects/language-java/src/main/java/org/gradle/api/internal/tasks/compile/JavaCompilerArgumentsBuilder.java#L195

### maven defaults:
maven invoke javac with -g by default:
 - https://maven.apache.org/plugins/maven-compiler-plugin/compile-mojo.html#debug
 - https://maven.apache.org/plugins/maven-compiler-plugin/compile-mojo.html#debuglevel
 

javap on classfile, LineNumber Table
what is the overhead in size, table to compare, %

## C2
debug info recorder:
https://github.com/openjdk/jdk/blob/master/src/hotspot/share/code/debugInfoRec.hpp
debug info at safepoint
issues with DebugNonSafepoint flag regarding stacktrace accuracy


## References
 - [https://github.com/openjdk/jdk/blob/master/src/hotspot/share/code/debugInfoRec.hpp](https://github.com/openjdk/jdk/blob/master/src/hotspot/share/code/debugInfoRec.hpp)
 - [https://bugs.openjdk.java.net/browse/JDK-8201516](https://bugs.openjdk.java.net/browse/JDK-8201516)
 - [https://bugs.openjdk.java.net/browse/JDK-8281677](https://bugs.openjdk.java.net/browse/JDK-8281677)
 - [https://felixge.de/2022/02/11/profiling-improvements-in-go-1.18/](https://felixge.de/2022/02/11/profiling-improvements-in-go-1.18/)
 - [http://psy-lob-saw.blogspot.com/2016/06/the-pros-and-cons-of-agct.html](http://psy-lob-saw.blogspot.com/2016/06/the-pros-and-cons-of-agct.html)




