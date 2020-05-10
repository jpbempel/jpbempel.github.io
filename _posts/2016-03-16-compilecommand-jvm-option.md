---
title: "CompileCommand JVM option"
layout: default
date: 2016-03-16
---
# CompileCommand JVM option
CompileCommand is another JVM option that you can play with in conjunction (or not) with the famous [PrintAssembly](https://jpbempel.github.io/2012/10/16/how-to-print-disassembly-from-JIT-code.html). But there is more...

In this article I will give you a description of all useful commands available with this option.

## help
The first command you can try is the following:
```
java -XX:CompileCommand=help -version
  CompileCommand and the CompilerOracle allows simple control over
  what's allowed to be compiled.  The standard supported directives
  are exclude and compileonly.  The exclude directive stops a method
  from being compiled and compileonly excludes all methods except for
  the ones mentioned by compileonly directives.  The basic form of
  all commands is a command name followed by the name of the method
  in one of two forms: the standard class file format as in
  class/name.methodName or the PrintCompilation format
  class.name::methodName.  The method name can optionally be followed
  by a space then the signature of the method in the class file
  format.  Otherwise the directive applies to all methods with the
  same name and class regardless of signature.  Leading and trailing
  *'s in the class and/or method name allows a small amount of
  wildcarding.

  Examples:

  exclude java/lang/StringBuffer.append
  compileonly java/lang/StringBuffer.toString ()Ljava/lang/String;
  exclude java/lang/String*.*
  exclude *.toString
java version "1.8.0_51"
Java(TM) SE Runtime Environment (build 1.8.0_51-b16)
Java HotSpot(TM) 64-Bit Server VM (build 25.51-b03, mixed mode)
```
This is a start. But this online help is in fact quite incomplete. There is other commands/directives than those mentioned above.
To find out the others, we have to look at the documentation page [here](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/java.html) which is complete in this case.

## dontinline
This command is useful if you want to control some behavior related to inlining. For example, JMH is using this directive through the [CompilerControl annotation](http://hg.openjdk.java.net/code-tools/jmh/file/f4e8d0d61f1f/jmh-samples/src/main/java/org/openjdk/jmh/samples/JMHSample_16_CompilerControl.java).
Let's take an example:
```java
    public static void main(String[] args) throws InterruptedException {
        List<String> list = new ArrayList<>();
        for (int i = 0; i < 10_000; i++)
        {
            list.add(String.valueOf(i));
        }
        Thread.sleep(1000);
    }
```

If we execute this code with the following JVM options:
```
-XX:UnlockDiagnosticVMOptions -XX:+PrintCompilation -XX:+PrintInlining
```
```
233   65    java.util.ArrayList::add (29 bytes)
              @ 7   java.util.ArrayList::ensureCapacityInternal (23 bytes)   inline (hot)
                @ 19   java.util.ArrayList::ensureExplicitCapacity (26 bytes)   inline (hot)
                  @ 22   java.util.ArrayList::grow (45 bytes)   too big
```
If we add this JVM option:
```
-XX:CompileCommand="dontinline java.util.ArrayList::ensureCapacityInternal"
```
we will get:
```
[...]
290  65 java.util.ArrayList::add (29 bytes)
         @ 7 java.util.ArrayList::ensureCapacityInternal (23 bytes) disallowed by CompilerOracle
[...]
```
No more inlining because disallowed by CompilerOracle where CompilerOracle is the CompileCommand JVM option.

## inline
Same here with this command which try to force inlining on a specific method.
If we execute the previous code with the following JVM options:
```
-XX:UnlockDiagnosticVMOptions -XX:+PrintCompilation -XX:+PrintInlining
```
```
612   64      java.util.ArrayList::ensureExplicitCapacity (26 bytes)
                @ 22   java.util.ArrayList::grow (45 bytes)   too big
                @ 19   java.util.ArrayList::ensureExplicitCapacity (26 bytes)   inline (hot)
                  @ 22   java.util.ArrayList::grow (45 bytes)   too big
```

We can see that grow method is not inlined in this context because considered as too big (above 35 byte codes). But if we add the CompileCommand:
```
-XX:CompileCommand="inline java.util.ArrayList::grow"
```
The result is the following:
```
361   64     java.util.ArrayList::ensureExplicitCapacity (26 bytes)
                @ 22   java.util.ArrayList::grow (45 bytes)   force inline by CompilerOracle
                  @ 28   java.util.ArrayList::hugeCapacity (26 bytes)   never executed
                  @ 38   java.util.Arrays::copyOf (13 bytes)   executed < MinInliningThreshold times
                @ 19   java.util.ArrayList::ensureExplicitCapacity (26 bytes)   inline (hot)
                  @ 22   java.util.ArrayList::grow (45 bytes)   force inline by CompilerOracle
                    @ 28   java.util.ArrayList::hugeCapacity (26 bytes)   never executed
                    @ 38   java.util.Arrays::copyOf (13 bytes)   executed < MinInliningThreshold times
```
The reason for inlining the method is now clear: `force inline by CompilerOracle`


## print

The print command is in fact similar to the PrintAssembly JVM option. But the added value is the ability to specify the exact or prefix name method that we want to disassemble.  
```
-XX:CompileCommand="print java.util.ArrayList::add"
```
```
CompilerOracle: print java/util/ArrayList.add
Compiled method (c2)     176   65             java.util.ArrayList::add (29 bytes)
 total in heap  [0x0000000002558a10,0x0000000002558fb0] = 1440
 relocation     [0x0000000002558b30,0x0000000002558b60] = 48
 main code      [0x0000000002558b60,0x0000000002558ca0] = 320
 stub code      [0x0000000002558ca0,0x0000000002558cc8] = 40
 oops           [0x0000000002558cc8,0x0000000002558cd0] = 8
 metadata       [0x0000000002558cd0,0x0000000002558cf0] = 32
 scopes data    [0x0000000002558cf0,0x0000000002558dc0] = 208
 scopes pcs     [0x0000000002558dc0,0x0000000002558f80] = 448
 dependencies   [0x0000000002558f80,0x0000000002558f88] = 8
 handler table  [0x0000000002558f88,0x0000000002558fa0] = 24
 nul chk table  [0x0000000002558fa0,0x0000000002558fb0] = 16
Loaded disassembler from C:\Program Files\Java\jdk1.8.0_40\jre\bin\server\hsdis-amd64.dll
Decoding compiled method 0x0000000002558a10:
Code:
[Disassembling for mach='i386:x86-64']
[Entry Point]
[Constants]
  # {method} {0x000000000a4840f8} 'add' '(Ljava/lang/Object;)Z' in 'java/util/ArrayList'
  # this:     rdx:rdx   = 'java/util/ArrayList'
  # parm0:    r8:r8     = 'java/lang/Object'
  #           [sp+0x40]  (sp of caller)

[...]

[Stub Code]
  0x0000000002558ca0: movabs rbx,0x0            ;   {no_reloc}
  0x0000000002558caa: jmp    0x0000000002558caa  ;   {runtime_call}
[Exception Handler]
  0x0000000002558caf: jmp    0x000000000253c9e0  ;   {runtime_call}
[Deopt Handler Code]
  0x0000000002558cb4: call   0x0000000002558cb9
  0x0000000002558cb9: sub    QWORD PTR [rsp],0x5
  0x0000000002558cbe: jmp    0x0000000002517600  ;   {runtime_call}
  0x0000000002558cc3: hlt    
  0x0000000002558cc4: hlt    
  0x0000000002558cc5: hlt    
  0x0000000002558cc6: hlt    
  0x0000000002558cc7: hlt    
OopMapSet contains 6 OopMaps

#0 
OopMap{rbp=Oop [8]=Oop off=180}
#1 
OopMap{[8]=Oop off=216}
#2 
OopMap{rbp=NarrowOop [8]=Oop off=236}
#3 
OopMap{rbp=NarrowOop [8]=Oop off=256}
#4 
OopMap{rbp=Oop [0]=Oop [20]=NarrowOop off=288}
#5 
OopMap{off=300}
Java HotSpot(TM) 64-Bit Server VM warning: printing of assembly code is enabled; turning on DebugNonSafepoints to gain additional output
```
Wildcards are also allowed like mentioned in the online help:
```
-XX:CompileCommand="print java.util.ArrayList::*"
```

## exclude
By default, all methods can be compiled if the compile threshold is reached. However we can exclude one or set of methods with the exclude command. Let's try with an example:
```
-XX:+PrintCompilation -XX:CompileCommand="exclude java.util.ArrayList::add"
```
The output will be:
```
[...]
    237   61             java.lang.Integer::getChars (131 bytes)
    249   62             java.lang.String::<init> (10 bytes)
    250   63             java.util.ArrayList::ensureCapacityInternal (23 bytes)
    250   64             java.util.ArrayList::ensureExplicitCapacity (26 bytes)
### Excluding compile: java.util.ArrayList::add
made not compilable on all levels  java.util.ArrayList::add (29 bytes)   excluded by CompilerOracle
    251   65             java.lang.String::valueOf (5 bytes)
    251   66             java.lang.Integer::toString (48 bytes)
```

## compileonly
On the opposite, compileonly command allow to compile only the method specified in argument. Example:
```
-XX:+PrintCompilation -XX:CompileCommand="compileonly java.util.ArrayList::add"
```
The output will be:
```
[...]
    196   43     n       java.lang.invoke.MethodHandle::invokeBasic(ILL)L (native)   
    196   44     n       java.lang.invoke.MethodHandle::linkToSpecial(LILLL)L (native)   (static)
    227   45             java.util.ArrayList::add (29 bytes)
```
Native methods are printed but no special operation is performed as JIT has nothing to do with them.

## log
By default when you are using LogCompilation to output all compilation information from JIT, all methods compiled are included in this log. If we want to only output a specified method, we can use the log command in the same way than previous one.
Example:
```
-XX:+UnlockDiagnosticVMOptions -XX:+LogCompilation -XX:CompileCommand="log java.util.ArrayList::add"
```
The output file can be opened with [JITWatch](https://github.com/AdoptOpenJDK/jitwatch).

## option
You can use this command to enable a JVM option only for the specified method or set of methods.
For example let's print the inlining tree for the `ArrayList.add` method:
```
-XX:CompileCommand="option java.util.ArrayList::add,PrintInlining"
```

The output will be:
```
CompilerOracle: option java/util/ArrayList.add bool PrintInlining = true
            @ 7   java.util.ArrayList::ensureCapacityInternal (23 bytes)   inline (hot)
              @ 19   java.util.ArrayList::ensureExplicitCapacity (26 bytes)   inline (hot)
                @ 22   java.util.ArrayList::grow (45 bytes)   too big
```
Very handy to avoid to be flooded by information.

## CompileCommandFile
The previous commands can be combine into a file for convenience. By default the file is read from current directory with the name .hotspot_compiler. But you can specify your own file name with:
```
-XX:CompileCommandFile=MyCompilerFile.txt
```
This is an example of content:
```
compileonly java.util.ArrayList::add
compileonly java.util.ArrayList::ensureCapacityInternal
compileonly java.util.ArrayList::ensureExplicitCapacity
compileonly java.util.ArrayList::grow
inline java.util.ArrayList::grow
option java.util.ArrayList::add,PrintInlining
```
And here is the output:
```
CompilerOracle: compileonly java/util/ArrayList.add
CompilerOracle: compileonly java/util/ArrayList.ensureCapacityInternal
CompilerOracle: compileonly java/util/ArrayList.ensureExplicitCapacity
CompilerOracle: compileonly java/util/ArrayList.grow
CompilerOracle: inline java/util/ArrayList.grow
CompilerOracle: option java/util/ArrayList.add bool PrintInlining = true
          @ 7   java.util.ArrayList::ensureCapacityInternal (23 bytes)   inline (hot)
            @ 19   java.util.ArrayList::ensureExplicitCapacity (26 bytes)   inline (hot)
              @ 22   java.util.ArrayList::grow (45 bytes)   force inline by CompilerOracle
                @ 28   java.util.ArrayList::hugeCapacity (26 bytes)   failed initial checks
                @ 38   java.util.Arrays::copyOf (13 bytes)   failed initial checks
```

*Update*:
In JDK 9 there will be a new mechanism: [Compiler Control](http://openjdk.java.net/jeps/165). Thanks [Mark Price](https://twitter.com/epickrram) for pointing me this JEP.

References
* [Documentation of java command with lots of JVM options](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/java.html)
* [Source code of CompileCommand](http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/compiler/compilerOracle.cpp)
* [JEP 165: Compiler Control](http://openjdk.java.net/jeps/165)
