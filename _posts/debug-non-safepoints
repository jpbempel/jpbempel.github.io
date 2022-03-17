# profiling stacktraces bias



## DebugNonSafepoint
There is an interesting flag that modifiy slightly the behavior described above: `-XX:+UnlockDiagnosticVMOptions -XX:+DebugNonSafepoints`.  This flag activates the recording of more debug information about PC even if it's not at safepoint. It means we can have a more precise location for stacktraces. It's not affecting exceptions, but profiling tools based on `AsyncGetCallTrace` like [JFR/JMC](https://github.com/openjdk/jmc), [AsyncProfiler](https://github.com/jvm-profiling-tools/async-profiler) or [HonestProfiler](https://github.com/jvm-profiling-tools/honest-profiler).

This flag seems like magic but tough there is some caveat about it. We may have information about stacktraces outside of safepoint, tough, it does not mean it's more accurate about time spent on a method reported by profiling tools. See [JDK-8201516](https://bugs.openjdk.java.net/browse/JDK-8201516) and [JDK-8281677](https://bugs.openjdk.java.net/browse/JDK-8281677).

## References
 - [https://bugs.openjdk.java.net/browse/JDK-8201516](https://bugs.openjdk.java.net/browse/JDK-8201516)
 - [https://bugs.openjdk.java.net/browse/JDK-8281677](https://bugs.openjdk.java.net/browse/JDK-8281677)
 - [http://psy-lob-saw.blogspot.com/2016/06/the-pros-and-cons-of-agct.html](http://psy-lob-saw.blogspot.com/2016/06/the-pros-and-cons-of-agct.html)




