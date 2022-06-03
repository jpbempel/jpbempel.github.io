# profiling stacktraces bias

# Introduction

I presented in my [previous blog post](https://jpbempel.github.io/2022/03/22/jvm-debug-symbols.html) how debug symbols are generated and used to resolve frames in exception stacktraces. 
Beside exceptions, stacktraces are also used extensively in profilers. The old profiler generation was based on JVMTI GetAllStackTraces Api (or equivalent) with the known issue related to this technique (safepoint bias).
The new one is based on the AsyncGetCallTrace un documented API which does not require to be at safepoint to collect stacktraces.
In this article we will explore the consequence on those profilers to rely on debug symbols resolution described earlier.

# Async-Profiler / Honest profiler

One of the most popular profiler based on `AsyncGetCallTrace` is Async-Profiler by Andrei Pangin. Async-Profiler has different method to trigger ticks for collecting stacktraces:
- Perf events
- `itimer`
- Wall clock (regular interval)

Once the tick is triggered, it calls `AsyncGetCallTrace` which will collect stacktraces even if threads are not at safepoint.

Honest Profiler is using the same call with `itimer`.


# JFR

JDK Flight Recorder, for method sampling, use a timer and at each regular interval pick max 5 Java threads, verify that those threads are executing Java code 
(internal state maintined by the JVM) and collect stacktraces in a very similar way than AsyncGetCallTrace, though they don't share the same code.
No safepoint requires also for JFR which allow to collect at any point of the code.


# Last frame resolution
Once stacktraces are collected, they are resolved against debug symbol emitted by the JVM (interpreter or JIT) as described in my previous post.
If you look at stacktraces, bottom of the stack is just a stack of calls, and as we learned previously, calls are safepoint with all debug information
required to resolve them as a pair of method name and line number.

Only the last frame where the code is currently executed can be anywhere inside a method, including outside of a safepoint. If we are outside a 
safepoint we don't have debug information so we cannot resolve this last frame correctly. 

How JFR & AsyncGetCallTrace manage to get this last frame? is the information accurate? Do we have line number precision?

## Async-Profiler
Async-Profiler as several output formats:
- flamegraph (svg/html)
- text (flat/traces/collapsed)
- JFR

For the first 2, where `AsyncGetCallTrace` is used, you will never see line numbers as the author chose to not process/display them.

![](/assets/2022/06/AP_flamegraph.png)

![](/assets/2022/06/AP_traces.png)

For JFR output, line numbers are transfered/preserved from `AsyncGetCallTrace` to the JFR file format. This way we can open the file with JDK Mission Control (JMC) 
to see what line is resolved for the last frame.

![](/assets/2022/06/AP_JFR_JMC.png)

## Honest Profiler

Honest Profiler reports line numbers with flat profiles:

```
Flat Profile (by line):
	(t 51.1,s 51.1) AGCT::UnknownJavaErr5 @ (bci=-1,line=-100)
	(t 18.7,s 18.7) safepoint.profiling.generated.AgctProfilingFails_systemArrayCopy_jmhTest::systemArrayCopy_avgt_jmhStub @ (bci=29,line=165)
	(t  5.3,s  5.3) safepoint.profiling.AgctProfilingFails::systemArrayCopy @ (bci=23,line=42)
	(t  5.0,s  5.0) AGCT::UnknownNotJava3 @ (bci=-1,line=-100)
	(t  1.2,s  1.2) safepoint.profiling.AgctProfilingFails::systemArrayCopy @ (bci=15,line=41)
	(t  1.2,s  1.2) safepoint.profiling.generated.AgctProfilingFails_systemArrayCopy_jmhTest::systemArrayCopy_avgt_jmhStub @ (bci=13,line=163)
	(t  1.2,s  1.2) org.openjdk.jmh.infra.Blackhole::consume @ (bci=16,line=404)
```

## JFR

The recording format allow line number information to be stored for each frame and in JMC you can distinguish by line number. 
Useful if you have a method that have multiple calls to the same method. Without the line information it will be difficult to correlate

![](/assets/2022/06/JFR_JMC_Lines.png)

## Debug information and safepoints

By default, debug information are emitted for the JIT where safepoint are emitted, for example when a method is called, which is convenient for building stacktraces for exception.
However with sampling profiling like done with `AsyncGetCallTrace` if we want to resolve the last frame with debug information, we need to look for nearby safepoints.
And this is exactly what JVM is doing when the actual Program Counter (PC) is not exactly on a safepoint.

To demonstrate this beahvior let's take a contrive example:

```java
L117 public static int noLoopBench(int idx) {
L118     int res = idx * idx;
         res += (idx % 7) + idx * 31;
	 res -= (idx * 53) % 13 - idx;
	 res += idx * 1003 - (idx * 13 % 7);
	 res += (idx % 19) + idx * 37;
	 res -= (idx * 71) % 7 - idx;
	 res += idx * 97 - (idx * 53 % 29);
	 res += (idx % 7) + idx * 31;
	 res -= (idx * 53) % 13 - idx;
	 // ... skip for brevity ...
	 res += idx * 1003 - (idx * 13 % 7);
	 res += (idx % 19) + idx * 37;
	 res -= (idx * 71) % 7 - idx;
	 res += idx * 97 - (idx * 53 % 29);
L167	 return res;
    }
```

Full code [here](https://gist.github.com/jpbempel/b40e5081b98d9021116f845d8adf0be1)

We profile this method with JFR and here how it looks in JMC:

![](/assets/2022/06/JMC_Profile_noLoop.png)

PC of the last frame need to be resolved to a line debug info. => by default only for Safepoint, except if DebugNonSafepoint activated!


# DebugNonSafepoint
There is an interesting flag that modifiy slightly the behavior described above: `-XX:+UnlockDiagnosticVMOptions -XX:+DebugNonSafepoints`.  This flag activates the recording of more debug information about PC even if it's not at safepoint. It means we can have a more precise location for stacktraces. It's not affecting exceptions, but profiling tools based on `AsyncGetCallTrace` like [JFR/JMC](https://github.com/openjdk/jmc), [AsyncProfiler](https://github.com/jvm-profiling-tools/async-profiler) or [HonestProfiler](https://github.com/jvm-profiling-tools/honest-profiler).

This flag seems like magic but tough there is some caveat about it. We may have information about stacktraces outside of safepoint, tough, it does not mean it's more accurate about time spent on a method reported by profiling tools. See [JDK-8201516](https://bugs.openjdk.java.net/browse/JDK-8201516) and [JDK-8281677](https://bugs.openjdk.java.net/browse/JDK-8281677).

Beware of the auto-activation (AsyncProfiler via JVMTI or PrintAssembly or CompileCommand)
Even though there is auto-activation, if you profile by attaching AP, most of the code is already C2 compiled with debug info generated without DebugNonSafepoint. So you are biased to safepoint for last frame resolution (example noLoop?)


Example of code without Safepoint, and statistical bias with taht (both AP & JFR)
https://gist.github.com/jpbempel/b40e5081b98d9021116f845d8adf0be1

for noLoopBench, add a very expensive operation without loop of call?

# Perf impact?


## References
 - [https://bugs.openjdk.java.net/browse/JDK-8201516](https://bugs.openjdk.java.net/browse/JDK-8201516)
 - [https://bugs.openjdk.java.net/browse/JDK-8281677](https://bugs.openjdk.java.net/browse/JDK-8281677)
 - [http://psy-lob-saw.blogspot.com/2016/06/the-pros-and-cons-of-agct.html](http://psy-lob-saw.blogspot.com/2016/06/the-pros-and-cons-of-agct.html)
 - [Honest profiler](https://github.com/jvm-profiling-tools/honest-profiler/wiki/AsyncGetCallTrace-errors-and-what-they-mean)




