---
title:  "Safety First: Safepoints"
layout: default
---

# Safety First: Safepoints
You may have never heard about them since your are programming in Java, but it is very important feature of the JVM.
Your Java (or other JVM language) application encounters safepoints every day. Let's talk about it.

## What is a Safepoint ?
A safepoint is a state of your application execution where all references to objects are perfectly reachable by the VM.

Some operations of the VM require that all threads reach a safepoint to be performed. The most common operation that need it is the GC.

A safepoint means that all threads need to reach a certain point of execution before they are stopped. Then the VM operation is performed. After that all threads are resumed.

## VM operations
When a GC occurs, a safepoint is requested. Then, when all threads are stopped, GC threads can performed their algorithm.
But GC is not the only operation that requires a safepoint. Here is a non exhaustive list of operations that your application can encounter during execution:

* Deoptimization
* PrintThreads
* PrintJNI
* FindDeadlock
* ThreadDump
* EnableBiasLocking
* RevokeBias
* HeapDumper
* GetAllStackTrace
* ...
You can find some VM Operations here: vm_operations.hpp But note that all VM operations do not require a safepoint.

## What is the impact?

If you run your application with following JVM options:
`-XX:+PrintSafepointStatistics -XX:+PrintGCApplicationStoppedTime -XX:PrintSafepointStatisticsCount=1`

You will find a output similar to:
```
vmop [threads: total initially_running wait_to_block]    [time: spin block sync cleanup vmop] page_trap_count
0.255: ParallelGCFailedAllocation       [9    0    0]      [0     0     0     0     1]  0  
Total time for which application threads were stopped: 0.0018988 seconds
vmop [threads: total initially_running wait_to_block]    [time: spin block sync cleanup vmop] page_trap_count 
0.267: ParallelGCFailedAllocation       [9    0    1]      [0     0     0     0     1]  0  
Total time for which application threads were stopped: 0.0014882 seconds 
vmop [threads: total initially_running wait_to_block]    [time: spin block sync cleanup vmop] page_trap_count 
0.278: FindDeadlocks                    [9    0    0]      [0     0     0     0     0]  0  
Total time for which application threads were stopped: 0.0001193 seconds
```

This output indicates that 3 safepoints were run: 2 for GC 1 for `findDeadlock`.
The 2 GCs (`ParallelGCFailedAllocation` stopped threads for 1 to 2 ms.
`FindDeadlock` stopped threads for 119 us.

If `findDeadlock` is a lighter operation than a GC it is not negligible for some kind of sensitive applications. Some other VM operations like ThreadDump can take a while (up to several ms) depending on the number of threads involved.

So check on your application what are the effects of such operations.
