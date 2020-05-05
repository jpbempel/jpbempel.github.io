---
title: "Hardware Performance Counters: atomic vs standard incrementation"
layout: default
---

# Hardware Performance Counters: atomic vs standard incrementation
In a previous post on [Hardware  Performance Counters](https://jpbempel.github.io/2013/08/02/hardware-performance-counters.html), I have said that the main problem of the perf command is that only program from command line can be monitored like this.
It means when you want to monitor HPC for Java program you will also monitor the JVM and the startup phase, including warmup. Obviously it will affect negatively the performance compared to the steady state of the application when the code is JITed.

The perfect illustration of this flaw is presented in this [blog post](http://brooker.co.za/blog/2012/11/13/increment.html). In an attempt to understand the cost of a memory barrier, the author use the perf command in a micro-benchmark comparing simple incrementation of a regular variable to an incremention of an AtomicInteger. The author is fully aware of this methodlogy contains flaws:

> We must be careful interpreting these results, because they are polluted with data not related to our program under test, like the JVM startup sequence.

To avoid this bias in the micro-benchmark we can use the HPC more precisely with the help of the overseer library presented in my previous post.

I have written a similar micro-benchmark with overseer and try to eliminate all the startup & warmup bias when measuring.

Here the code:
```java
import ch.usi.overseer.OverHpc;
import java.util.concurrent.atomic.AtomicInteger;
 
public class Increment
{
    private static String[] EVENTS = {
        "UNHALTED_CORE_CYCLES",
        "INSTRUCTION_RETIRED",
        "L2_RQSTS:LD_HIT",
        "L2_RQSTS:LD_MISS",
        "LLC_REFERENCES",
        "MEM_LOAD_RETIRED:LLC_MISS",
        "PERF_COUNT_SW_CPU_MIGRATIONS",
        "MEM_UNCORE_RETIRED:LOCAL_DRAM_AND_REMOTE_CACHE_HIT",
        "MEM_UNCORE_RETIRED:REMOTE_DRAM"
    };
 
    private static String[] EVENTS_NAME = {
        "Cycles",
        "Instructions",
        "L2 hits",
        "L2 misses",
        "LLC hits",
        "LLC misses",
        "CPU migrations",
        "Local DRAM",
        "Remote DRAM"
    };
 
    private static long[] results = new long[EVENTS.length];
    private static OverHpc oHpc = OverHpc.getInstance();
 
    static int counter;
    static AtomicInteger atomicCounter = new AtomicInteger();
 
    static void stdIncrement()
    {
        counter++;
    }
 
    static void atomicIncrement()
    {
        atomicCounter.incrementAndGet();
    }
 
    static void benchStdIncrement(int loopCount)
    {
        for (int i = 0; i < loopCount; i++)
        {
            stdIncrement();
        }
    }
 
    static void benchAtomicIncrement(int loopCount)
    {
        for (int i = 0; i < loopCount; i++)
        }
    }
 
    static void benchAtomicIncrement(int loopCount)
    {
        for (int i = 0; i < loopCount; i++)
        {
            atomicIncrement();
        }
    }
 
    public static void main(String[] args) throws Exception
    {
        boolean std = args.length > 0 && args[0].equals("std");
        StringBuilder sb  = new StringBuilder();
        for (int i = 0; i < EVENTS.length; i++)
        {
            if (i > 0)
            {
                sb.append(",");
            }
            sb.append(EVENTS[i]);
        }
        oHpc.initEvents(sb.toString());
        // warmup
        if (std)
        {
            benchStdIncrement(10000);
            benchStdIncrement(10000);
        }
        else
        {
            benchAtomicIncrement(10000);
            benchAtomicIncrement(10000);
        }
        Thread.sleep(1000);
        System.out.println("warmup done");
 
        int tid = oHpc.getThreadId();
        oHpc.bindEventsToThread(tid);
        // bench
        if (std)
            benchStdIncrement(5*1000*1000);
        else
            benchAtomicIncrement(5*1000*1000);
 
        for (int i = 0; i < EVENTS.length; i++)
        {
            results[i] = oHpc.getEventFromThread(tid, i);
        }
        for (int i = 0; i < EVENTS.length; i++)
        {
            System.out.println(EVENTS_NAME[i] + ": " + String.format("%,d", results[i]));
        }
        OverHpc.shutdown();
   }
}
```
Depending on the argument I call 2 times the method performing the loop to make sure everything is compiled before starting the counters. With `-XX:+PrintCompilation` I verify this.

So now we can run it and compare the raw results for 3 runs each:
```
[root@archi-srv myths]# java -cp .:overseer.jar Increment atomic
warmup done
Cycles: 150,054,450
Instructions: 59,961,219
L2 hits: 245
L2 misses: 61
LLC hits: 164
LLC misses: 0
CPU migrations: 0
Local DRAM: 10
Remote DRAM: 0
[root@archi-srv myths]# java -cp .:overseer.jar Increment atomic
warmup done
Cycles: 149,907,417
Instructions: 59,968,058
L2 hits: 54
L2 misses: 38
LLC hits: 105
LLC misses: 9
CPU migrations: 0
Local DRAM: 11
Remote DRAM: 0
[root@archi-srv myths]# java -cp .:overseer.jar Increment atomic
warmup done
Cycles: 149,890,503
Instructions: 59,964,047
L2 hits: 372
L2 misses: 30
LLC hits: 85
LLC misses: 0
CPU migrations: 0
Local DRAM: 7
Remote DRAM: 0
 
[root@archi-srv myths]# java -cp .:overseer.jar Increment std
warmup done
Cycles: 1,217,065
Instructions: 3,155,346
L2 hits: 288
L2 misses: 518
LLC hits: 1,452
LLC misses: 0
CPU migrations: 0
Local DRAM: 0
Remote DRAM: 0
[root@archi-srv myths]# java -cp .:overseer.jar Increment std
warmup done
Cycles: 1,367,222
Instructions: 3,155,345
L2 hits: 409
L2 misses: 387
LLC hits: 1,153
LLC misses: 0
CPU migrations: 0
Local DRAM: 0
Remote DRAM: 0
[root@archi-srv myths]# java -cp .:overseer.jar Increment std
warmup done
Cycles: 1,366,928
Instructions: 3,155,345
L2 hits: 374
L2 misses: 423
LLC hits: 1,183
LLC misses: 0
CPU migrations: 0
Local DRAM: 0
Remote DRAM: 0
```
As you can see, they are differences:
* more cycles for atomic version
* more instructions for atomic versions
* a little bit more L2 misses for std versions
* more LLC hits for std versions
* Local DRAM accesses for atomic versions
We can also noticed than for atomic version we have roughly 3 cycles per instruction and for std version 3 instructions for 1 cycle...

Still, conclusion remains the same. we spend more instructions & more cycles with atomic version. We also have more accesses from DRAM which have significant more latency that caches.
But now, we are sure that those measurements are more accurate than including the warmup phase.

Stay tuned for another adventure in HPC land...
