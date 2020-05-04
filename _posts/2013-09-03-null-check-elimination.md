---
title: "Null check elimination"
layout: default
---
# Null check elimination
Among of all kinds of optimization performed by the JIT, one is the most cited as example: null check elimination. Maybe because dealing with null is such a common task in Java.

Regarding performance what is the cost of null checking anyway. Let's take a basic example here:
```java
import java.util.ArrayList;
import java.util.List;
 
public class NullCheck
{
    private static void bench(List<string> list)
    {
        list.contains("foo");
    }
     
    public static void main(String[] args) throws InterruptedException
    {
        List<string> list = new ArrayList<string>();
        list.add("bar");
        for (int i = 0; i < 10000; i++)
        {
            bench(list);
        }
        Thread.sleep(1000);
    }
}
```

Classic example where in `bench` method I call `contains` method on list instance passed as parameter. As usual we examine the [PrintAssembly](https://jpbempel.github.io/2012/10/16/how-to-print-disassembly-from-JIT-code.html) output of this method (64 bits version now since I have my [hsdis-amd64.dll](https://jpbempel.github.io/2013/07/02/how-to-build-hsdis-amd64-dll.html) ;-)):

```
[Disassembling for mach='i386:x86-64']
[Entry Point]
[Verified Entry Point]
  # {method} 'bench' '(Ljava/util/List;)V' in 'com/bempel/sandbox/NullCheck'
  # parm0:    rdx:rdx   = 'java/util/List'
  #           [sp+0x40]  (sp of caller)
  0x00000000025302a0: mov    DWORD PTR [rsp-0x6000],eax
  0x00000000025302a7: push   rbp
  0x00000000025302a8: sub    rsp,0x30           ;*synchronization entry
                                                ; - com.bempel.sandbox.NullCheck::bench@-1 (line 10)
  0x00000000025302ac: mov    r8,rdx
  0x00000000025302af: mov    r10,QWORD PTR [rdx+0x8]  ; implicit exception: dispatches to 0x0000000002530621
  0x00000000025302b3: movabs r11,0x5777e60      ;   {oop('java/util/ArrayList')}
  0x00000000025302bd: cmp    r10,r11
  0x00000000025302c0: jne    0x00000000025305fc  ;*invokeinterface contains
                                                ; - com.bempel.sandbox.NullCheck::bench@3 (line 10)
[...]
```
Nothing fancy here, contains call is inlined, and before that a test against ArrayList class for the instance, to make sure we can apply safely the devirtualization of the call as explained [here](https://jpbempel.github.io/2012/10/24/virtual-call-911.html).

If we now change our example to add a null check on the list instance passed as parameter:
```java
import java.util.ArrayList;
import java.util.List;
 
public class NullCheck
{
    private static void bench(List<string> list)
    {
        if (list != null)
        {
            list.contains("foo");
        }
    }
     
    public static void main(String[] args) throws InterruptedException
    {
        List<string> list = new ArrayList<string>();
        list.add("bar");
        for (int i = 0; i < 10000; i++)
        {
            bench(list);
        }
        Thread.sleep(1000);
    }
}
```

The output is in fact exactly the same.
```
[Disassembling for mach='i386:x86-64']
[Entry Point]
[Verified Entry Point]
  # {method} 'bench' '(Ljava/util/List;)V' in 'com/bempel/sandbox/NullCheck'
  # parm0:    rdx:rdx   = 'java/util/List'
  #           [sp+0x40]  (sp of caller)
  0x00000000025402a0: mov    DWORD PTR [rsp-0x6000],eax
  0x00000000025402a7: push   rbp
  0x00000000025402a8: sub    rsp,0x30           ;*synchronization entry
                                                ; - com.bempel.sandbox.NullCheck::bench@-1 (line 10)
  0x00000000025402ac: mov    r8,rdx
  0x00000000025402af: mov    r10,QWORD PTR [rdx+0x8]  ; implicit exception: dispatches to 0x000000000254062d
  0x00000000025402b3: movabs r11,0x5787e60      ;   {oop('java/util/ArrayList')}
  0x00000000025402bd: cmp    r10,r11
  0x00000000025402c0: jne    0x00000000025405fc  ;*invokeinterface contains
                                                ; - com.bempel.sandbox.NullCheck::bench@7 (line 12)
[...]
```
However the semantic of our `bench` method is totally different: first version we can have a Null Pointer Exception thrown if we pass `null` to the `bench` method. Now it is impossible.
So how JIT can handle this since there is no special instruction to check this in generated code?

Well, `null` is in fact a very special value (billion dollars mistake, blah blah blah...). When trying to access this value (I mean dereferencing it), whatever OS you are using you end up with an error (`ACCESS_VIOLATION`, `SEGMENTATION FAULT`, ...). Most of the time this error leads to an application crash.
However it is still possible to intercept this kind of error. On Unix, `SEGMENTATION FAULT` is just a signal like `SIQ_QUIT`, `SIG_BRK`, etc. So you can provide an handler for this signal and execute some code when it is raised. On Windows there is a mechanism for exception handlers. See [`RtlAddFunctionTable`](http://msdn.microsoft.com/en-us/library/windows/desktop/ms680588%28v=vs.85%29.aspx).

So JVM uses those mechanism to handle nulls. If you look carefully at the generated code, you can see a comment on line
```
mov r10,QWORD PTR [rdx+0x8] ;implicit exception: dispatches to 0x000000000254062d
```
If we follow the address we get:
```
0x000000000254062d: mov    edx,0xffffffad
0x0000000002540632: mov    QWORD PTR [rsp],r8
0x0000000002540636: nop
0x0000000002540637: call   0x0000000002516f60  ; OopMap{[0]=Oop off=924}
                                              ;*ifnull
                                              ; - com.bempel.sandbox.NullCheck::bench@1 (line 10)
                                              ;   {runtime_call}
0x000000000254063c: int3                      ;*ifnull
                                              ; - com.bempel.sandbox.NullCheck::bench@1 (line 10)
```
This code is added also into then generated code along with our `bench` method. We can see that in this case there is a null check (bytecode instruction `ifnull`).

Let's add a call to `bench` method with a null parameter to see what happens:
```java
public static void main(String[] args) throws InterruptedException
{
    List list = new ArrayList();
    list.add("bar");
    for (int i = 0; i < 10000; i++)
    {
        bench(list);
    }
    Thread.sleep(1000);
    bench(null);
}
```
Calling the test with `-XX:+PrintCompilation` we get the following result:
```
  1       java.util.ArrayList::indexOf (67 bytes)
  2       com.bempel.sandbox.NullCheck::bench (14 bytes)
  3       java.util.ArrayList::contains (14 bytes)
  2   made not entrant  com.bempel.sandbox.NullCheck::bench (14 bytes)
```
So `bench` method was the second method compiled here, but then was `made not entrant`, it means deoptimized. The call with null parameter does not cause any `NullPointerException` or any application crash but a trigger to deoptimization of that method that was optimized too aggressively.
If we call again 10,000 times bench with our previous list instance, we get `bench` method compiled again:
```java
    public static void main(String[] args) throws InterruptedException
    {
        List<string> list = new ArrayList<string>();
        list.add("bar");
        for (int i = 0; i < 10000; i++)
        {
            bench(list);
        }
        Thread.sleep(1000);
        bench(null);
        for (int i = 0; i < 10000; i++)
        {
            bench(list);
        }
        Thread.sleep(1000);
    }
```

PrintCompilation output:
```
  1       java.util.ArrayList::indexOf (67 bytes)
  2       com.bempel.sandbox.NullCheck::bench (14 bytes)
  3       java.util.ArrayList::contains (14 bytes)
  2   made not entrant  com.bempel.sandbox.NullCheck::bench (14 bytes)
  4       com.bempel.sandbox.NullCheck::bench (14 bytes)
```
PrintAssembly output:
```
[Disassembling for mach='i386:x86-64']
[Entry Point]
[Verified Entry Point]
  # {method} 'bench' '(Ljava/util/List;)V' in 'com/bempel/sandbox/NullCheck'
  # parm0:    rdx:rdx   = 'java/util/List'
  #           [sp+0x40]  (sp of caller)
  0x0000000002500da0: mov    DWORD PTR [rsp-0x6000],eax
  0x0000000002500da7: push   rbp
  0x0000000002500da8: sub    rsp,0x30           ;*synchronization entry
                                                ; - com.bempel.sandbox.NullCheck::bench@-1 (line 10)
  0x0000000002500dac: mov    r8,rdx
  0x0000000002500daf: test   rdx,rdx
  0x0000000002500db2: je     0x0000000002501104  ;*ifnull
                                                ; - com.bempel.sandbox.NullCheck::bench@1 (line 10)
  0x0000000002500db8: mov    r10,QWORD PTR [rdx+0x8]
  0x0000000002500dbc: movabs r11,0x5747e60      ;   {oop('java/util/ArrayList')}
  0x0000000002500dc6: cmp    r10,r11
  0x0000000002500dc9: jne    0x0000000002501110  ;*invokeinterface contains
                                                ; - com.bempel.sandbox.NullCheck::bench@7 (line 12)
```
We have now an explicit null check:
```
0x0000000002500daf: test   rdx,rdx
0x0000000002500db2: je     0x0000000002501104  ;*ifnull
```
JIT falls back to a classic way to handle nulls, but if the method was never called with a `null` value, the optimization is good deal!

So not fear about null check in your code, it helps you protect against unexpected value and its costs is virtually zero!
