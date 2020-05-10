---
title:  "CompileThreshold is relative!"
layout: default
date: 2013-04-03
---

# CompileThreshold is relative!

## CompileThreshold
Basically, what I have always read and understood about JIT compilation triggering in HotSpot is the following:
1. Execution starts in interpreted mode, collecting statistics per method
2. When a number of method calls is reached (aka CompileThreshold) the JIT kicks in and compile the method.
3. There is special cases for loops with On-Stack Replacement to allow partial compilation for those hot spots !

So the following code behaves as predicted:
```java
public static void main(String[] args) throws Exception
{
    for (int i = 0; i < 10000; i++)
    {
        call();
    }
    Thread.sleep(1000);
}

private static void call()
{
    if (System.currentTimeMillis() == 0)
    {
        System.out.println("foo");
    }
}
```
Executing this class with `java -server -XX:+PrintCompilation` gives the following output:
```
---   n   java.lang.System::currentTimeMillis (static)
  1       com.bempel.sandbox.TestJIT::call (17 bytes)
```
So yes, after exactly 10,000 calls (server compiler C2's CompileThreshold=10000) to call method, it gets compiled.
You can also tweak this behavior by changing CompileThresold with the option `-XX:CompileThreshold=n`

Everything is fine, micro-benchmarks can be warmed up correctly and results are predictable, we are happy ! Until now...

## Welcome to the real world !
Wake up Neo! Real life applications do not execute like our benchmarks! Throughputs are not constant, messages are not coming at sustainable high rates. There are some peaks and some pauses. Flows are chaotics.
Simulating this behavior is less easier than putting in a "monkey" program some instructions in a loop. Some flows are so slow that it will take a day to replay in "real time".
But some times it really matters!
Let's introduce some slowdown into our first example:
```java
public static void main(String[] args) throws Exception
{
    for (int i = 0; i < 10000; i++)
    {
        call();
        Thread.sleep(1);
    }
    Thread.sleep(1000);
}
 
private static void call()
{
    if (System.currentTimeMillis() == 0)
    {
        System.out.println("foo");
    }
}
```  
We only pause 1ms for each loop execution. Running it shows that the call method is no longer compiled!
```
---   n   java.lang.System::currentTimeMillis (static)
---   n   java.lang.Thread::sleep (static)
```
So what happened? Does the CompileThreshold is also time dependent?
Let's add the following option to monitor VM operations during this tests: `-XX:+PrintSafepointStatistics`
``` 
---   n   java.lang.System::currentTimeMillis (static)
---   n   java.lang.Thread::sleep (static)
         vmop                    [threads: total initially_running wait_to_block]    [time: spin block sync cleanup vmop] page_trap_count
4.148: EnableBiasedLocking              [       8          0              0    ]      [     0     0     0     0     0    ]  0  
20.707: no vm operation                  [       6          0              0    ]      [     0     0     0     0     0    ]  0  
 
Polling page always armed
EnableBiasedLocking                1
    0 VM operations coalesced during safepoint
Maximum sync time      0 ms
Maximum vm operation time (except for Exit VM operation)      0 ms
```
As you can see, after 4 seconds there is a VM operation EnabledBiasedLocking which requires a safepoint to be executed. As mentioned in one of my previous post about locks, this operation is performed for biased locking optimization. If we deactivate this kind of optimization (`-XX:-UseBisaedLocking`), we have now our call method compiled:
```
---   n   java.lang.System::currentTimeMillis (static)
  1       com.bempel.sandbox.TestJIT::call (17 bytes)
---   n   java.lang.Thread::sleep (static)
         vmop                    [threads: total initially_running wait_to_block]    [time: spin block sync cleanup vmop] page_trap_count
21.780: no vm operation                  [       6          0              0    ]      [     0     0     0     0     0    ]  0  
 
Polling page always armed
    0 VM operations coalesced during safepoint
Maximum sync time      0 ms
Maximum vm operation time (except for Exit VM operation)      0 ms
```

## Safepoints
So CompileThreshold is in fact sensible to safepoints. There is another kind of safepoints that you encouter everyday: GC. Let's introduce some GCs into our loop:

```java
public static void main(String[] args) throws Exception
{
    for (int i = 0; i < 10000; i++)
    {
        call();
        if (i % 100 == 0)
        {
            System.gc();
        }
        Thread.sleep(10);
    }
    Thread.sleep(1000);
}
```
Again, the method call is not getting compiled here. In fact no method are compiled!

During each safepoint, a special process is also invoked: [CounterDecay](https://github.com/openjdk/jdk/blob/18c01206d00b2d368f737345a5b615bd14743b24/src/hotspot/share/compiler/compilationPolicy.cpp#L218). It divides by 2 the number of method invocations for a small percentage (around 17%) of the classes.
This mechanism is here to balance "Hot Spots" not only based on number of invocations but alos based on time (indirectly by safepoints/GC).

Hopefully this mechanism can be disabled with `-XX:-UseCounterDecay`
Now with the same code we have all our methods compiled:
```
---   n   java.lang.System::currentTimeMillis (static)
  1       com.bempel.sandbox.TestJIT::call (17 bytes)
---   n   java.lang.Thread::sleep (static)
```
