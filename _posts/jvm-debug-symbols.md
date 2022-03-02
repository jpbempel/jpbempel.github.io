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

First steps are in java compiler `javac`. According to the inline documentation, you can specify `-g` option to manage the debug information that will be included into the compiled classfile:

```
javac --help
...
  -g                           Generate all debugging info
  -g:{lines,vars,source}       Generate only some debugging info
  -g:none                      Generate no debugging info
...
```

### javac defaults:
By default, javac is generating line number and source file but no local variable information.
see https://docs.oracle.com/en/java/javase/17/docs/specs/man/javac.html
> By default, only line number and source file information is generated.

let's take a simple example:

```java
public class LineNumbers {
    public static void main(String[] args) {
        throw new RuntimeException("boo");
    }
}
```

`javac LineNumbers.java && java LineNumbers`

```sh
Exception in thread "main" java.lang.RuntimeException: boo
	at LineNumbers.main(LineNumbers.java:3)
```
here we have line number information from where the exception was thrown.

`javac -g:none LineNumbers.java && java LineNumbers`

```sh
Exception in thread "main" java.lang.RuntimeException: boo
	at LineNumbers.main(Unknown Source)
```

we can inspect with `javap`


### sizes
what is the impact on the classfile size:
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
 - https://docs.oracle.com/en/java/javase/17/docs/specs/man/javac.html
 - [https://github.com/openjdk/jdk/blob/master/src/hotspot/share/code/debugInfoRec.hpp](https://github.com/openjdk/jdk/blob/master/src/hotspot/share/code/debugInfoRec.hpp)
 - [https://bugs.openjdk.java.net/browse/JDK-8201516](https://bugs.openjdk.java.net/browse/JDK-8201516)
 - [https://bugs.openjdk.java.net/browse/JDK-8281677](https://bugs.openjdk.java.net/browse/JDK-8281677)
 - [https://felixge.de/2022/02/11/profiling-improvements-in-go-1.18/](https://felixge.de/2022/02/11/profiling-improvements-in-go-1.18/)
 - [http://psy-lob-saw.blogspot.com/2016/06/the-pros-and-cons-of-agct.html](http://psy-lob-saw.blogspot.com/2016/06/the-pros-and-cons-of-agct.html)




