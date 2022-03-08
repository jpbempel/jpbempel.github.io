# JVM debug symbols

Recently I stumble upon this tweet:

https://twitter.com/ne_skazu/status/1491158726258364416


When you are used to work on the JVM it's sure that's very convenient and magic.
Also, you are familiar with stacktraces like for example the ones that are associated with exceptions:

```
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

```
Exception in thread "main" java.lang.RuntimeException: boo
	at LineNumbers.main(Unknown Source)
```

we can inspect with `javap` the classfile compiled with `-g:none`
```
javap -v LineNumbers.class
Classfile LineNumbers.class
  Last modified Mar 2, 2022; size 269 bytes
  MD5 checksum b62074a0ce6b44279a6fc3b347132cd0
public class LineNumbers
  minor version: 0
  major version: 55
  flags: (0x0021) ACC_PUBLIC, ACC_SUPER
  this_class: #5                          // LineNumbers
  super_class: #6                         // java/lang/Object
  interfaces: 0, fields: 0, methods: 2, attributes: 0
Constant pool:
   #1 = Methodref          #6.#12         // java/lang/Object."<init>":()V
   #2 = Class              #13            // java/lang/RuntimeException
   #3 = String             #14            // boo
   #4 = Methodref          #2.#15         // java/lang/RuntimeException."<init>":(Ljava/lang/String;)V
   #5 = Class              #16            // LineNumbers
   #6 = Class              #17            // java/lang/Object
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               main
  #11 = Utf8               ([Ljava/lang/String;)V
  #12 = NameAndType        #7:#8          // "<init>":()V
  #13 = Utf8               java/lang/RuntimeException
  #14 = Utf8               boo
  #15 = NameAndType        #7:#18         // "<init>":(Ljava/lang/String;)V
  #16 = Utf8               LineNumbers
  #17 = Utf8               java/lang/Object
  #18 = Utf8               (Ljava/lang/String;)V
{
  public LineNumbers();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=3, locals=1, args_size=1
         0: new           #2                  // class java/lang/RuntimeException
         3: dup
         4: ldc           #3                  // String boo
         6: invokespecial #4                  // Method java/lang/RuntimeException."<init>":(Ljava/lang/String;)V
         9: athrow
}
```

now with  `-g` option (full debug info):
```
Classfile /Users/jean-philippe.bempel/projects/tmp/LineNumbers.class
  Last modified Mar 2, 2022; size 460 bytes
  MD5 checksum fb5afda47b438ece75b4e4121d8e4786
  Compiled from "LineNumbers.java"
public class LineNumbers
  minor version: 0
  major version: 55
  flags: (0x0021) ACC_PUBLIC, ACC_SUPER
  this_class: #5                          // LineNumbers
  super_class: #6                         // java/lang/Object
  interfaces: 0, fields: 0, methods: 2, attributes: 1
Constant pool:
   #1 = Methodref          #6.#20         // java/lang/Object."<init>":()V
   #2 = Class              #21            // java/lang/RuntimeException
   #3 = String             #22            // boo
   #4 = Methodref          #2.#23         // java/lang/RuntimeException."<init>":(Ljava/lang/String;)V
   #5 = Class              #24            // LineNumbers
   #6 = Class              #25            // java/lang/Object
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               LocalVariableTable
  #12 = Utf8               this
  #13 = Utf8               LLineNumbers;
  #14 = Utf8               main
  #15 = Utf8               ([Ljava/lang/String;)V
  #16 = Utf8               args
  #17 = Utf8               [Ljava/lang/String;
  #18 = Utf8               SourceFile
  #19 = Utf8               LineNumbers.java
  #20 = NameAndType        #7:#8          // "<init>":()V
  #21 = Utf8               java/lang/RuntimeException
  #22 = Utf8               boo
  #23 = NameAndType        #7:#26         // "<init>":(Ljava/lang/String;)V
  #24 = Utf8               LineNumbers
  #25 = Utf8               java/lang/Object
  #26 = Utf8               (Ljava/lang/String;)V
{
  public LineNumbers();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 1: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   LLineNumbers;

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=3, locals=1, args_size=1
         0: new           #2                  // class java/lang/RuntimeException
         3: dup
         4: ldc           #3                  // String boo
         6: invokespecial #4                  // Method java/lang/RuntimeException."<init>":(Ljava/lang/String;)V
         9: athrow
      LineNumberTable:
        line 3: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      10     0  args   [Ljava/lang/String;
}
SourceFile: "LineNumbers.java"
```

We now know that the source filename is `LineNumbers.java`, and for each method we have `LineNumberTable` and `LocalVariableTable`

### sizes
What is the impact on the classfile size:

|SourceFile|-g:none|-g|%|
|-|-|-|-|
|LineNumbers.java|269B|460B|+71%|
|[CommandLine.java](https://github.com/remkop/picocli/blob/main/src/main/java/picocli/CommandLine.java)|64,273B|80,074B|+24%|


### gradle defaults:
gradle invoke javac with -g by default:
- https://docs.gradle.org/current/javadoc/org/gradle/api/tasks/compile/CompileOptions.html?_ga=2.42477413.1942252367.1645032818-155398221.1645032818#isDebug--
- https://github.com/gradle/gradle/blob/37911bb86d02d26a5d2ce3f23e01c0d767e3bb91/subprojects/language-java/src/main/java/org/gradle/api/internal/tasks/compile/JavaCompilerArgumentsBuilder.java#L195

### maven defaults:
maven invoke javac with -g by default:
 - https://maven.apache.org/plugins/maven-compiler-plugin/compile-mojo.html#debug
 - https://maven.apache.org/plugins/maven-compiler-plugin/compile-mojo.html#debuglevel
 
## Exception stacktraces

We have line numbers inside the classfile, now when and where those line are resolved?

For exception, stacktraces are in fact collected when they are instantiated through the call to [`fillInStackTrace()`](https://github.com/openjdk/jdk/blob/5d5bf16b0af419781fd336fe33d8eab5adf8be5a/src/java.base/share/classes/java/lang/Throwable.java#L271)

And this is calling the JVM internally to [`java_lang_Throwable::fill_in_stack_trace`](https://github.com/openjdk/jdk/blob/5d5bf16b0af419781fd336fe33d8eab5adf8be5a/src/hotspot/share/classfile/javaClasses.cpp#L2403-L2538)

and store it into the [`backtrace`](https://github.com/openjdk/jdk/blob/master/src/java.base/share/classes/java/lang/Throwable.java#L122) field in `Throwable` class a list of pointer (or handle) to method metadata from the interpreter state and the BCI (ByteCode Index).

Then, either if you call `getStackTrace()` or `printStackTrace()` on an exception, it will take this backtrace to fill out StackTraceElement array:
[`java_lang_Throwable::get_stack_trace_elements`](https://github.com/openjdk/jdk/blob/1581e3faa06358f192799b3a89718028c7f6a24b/src/hotspot/share/classfile/javaClasses.cpp#L2608-L2643)
and try to resolve symbol names and line numbers with the help of [`Backtrace::get_source_file_name`](https://github.com/openjdk/jdk/blob/5d5bf16b0af419781fd336fe33d8eab5adf8be5a/src/hotspot/share/classfile/javaClasses.inline.hpp#L323-L334) and [`Backtrace::get_line_number`](https://github.com/openjdk/jdk/blob/5d5bf16b0af419781fd336fe33d8eab5adf8be5a/src/hotspot/share/classfile/javaClasses.inline.hpp#L309-L321)
that will get information from the debug information stored in the classfile (SourceFile & LineNumber table).

For the interpreter it seems obvious that there is a 1:1 mapping between the current state of execution of the bytecode and the source file/line number. but for JITed code?

## C1/C2
When compiling a method, a debug info recorder is started and at each safepoint inserted, information about current execution context is recorded into the stream. used for GC, stacktraces (inclufind exception), and deoptimization.
see comment:
https://github.com/openjdk/jdk/blob/5d5bf16b0af419781fd336fe33d8eab5adf8be5a/src/hotspot/share/code/debugInfoRec.hpp#L40-L63

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




