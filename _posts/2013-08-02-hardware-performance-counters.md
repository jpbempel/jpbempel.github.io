---
title: "Hardware performance counters"
layout: default
---

# Hardware performance counters
## Introduction
Hardware Performance Counters (HPC) are counters directly provided by CPU thanks to a dedicated unit: Performance Monitoring Unit (PMU).
Depending on the CPU arch/vendor you can have different kind of counter available. This is highly hardware dependent, even across a same vendor, each CPU generation has its own implementation.

Those counters include cycles, number of instructions, cache hits/misses, branch mispredictions, etc.

This can be valuable if they are correctly used.

## Perf
On linux there is a command line tool which provide access to those counters: perf
for example we can capture statsitcs about command execution like `ls`:

```
perf stat -d ls -lR
```
```
 Performance counter stats for 'ls -lR':

    1725.533251 task-clock:HG             #    0.119 CPUs utilized
          3,421 context-switches:HG       #    0.002 M/sec
             39 CPU-migrations:HG         #    0.023 K/sec
            590 page-faults:HG            #    0.342 K/sec
  5,696,827,092 cycles:HG                 #    3.301 GHz                     [39.30%]
  3,395,404,627 stalled-cycles-frontend:HG #   59.60% frontend cycles idle    [39.23%]
  1,837,081,737 stalled-cycles-backend:HG #   32.25% backend  cycles idle    [39.84%]
  4,539,875,605 instructions:HG           #    0.80  insns per cycle
                                          #    0.75  stalled cycles per insn [49.95%]
    942,557,723 branches:HG               #  546.241 M/sec                   [50.11%]
     16,070,006 branch-misses:HG          #    1.70% of all branches         [50.57%]
  1,271,339,179 L1-dcache-loads:HG        #  736.780 M/sec                   [50.94%]
    145,209,839 L1-dcache-load-misses:HG  #   11.42% of all L1-dcache hits   [50.73%]
     83,147,216 LLC-loads:HG              #   48.186 M/sec                   [40.04%]
        525,580 LLC-load-misses:HG        #    0.63% of all LL-cache hits    [39.54%]

    14.489543284 seconds time elapsed
```

Through this command you can also list more events available:

```
perf list
```

```
List of pre-defined events (to be used in -e):
  cpu-cycles OR cycles                               [Hardware event]
  instructions                                       [Hardware event]
  cache-references                                   [Hardware event]
  cache-misses                                       [Hardware event]
  branch-instructions OR branches                    [Hardware event]
  branch-misses                                      [Hardware event]
  bus-cycles                                         [Hardware event]
  stalled-cycles-frontend OR idle-cycles-frontend    [Hardware event]
  stalled-cycles-backend OR idle-cycles-backend      [Hardware event]

  cpu-clock                                          [Software event]
  task-clock                                         [Software event]
  page-faults OR faults                              [Software event]
  context-switches OR cs                             [Software event]
  cpu-migrations OR migrations                       [Software event]
  minor-faults                                       [Software event]
  major-faults                                       [Software event]
  alignment-faults                                   [Software event]
  emulation-faults                                   [Software event]

  L1-dcache-loads                                    [Hardware cache event]
  L1-dcache-load-misses                              [Hardware cache event]
  L1-dcache-stores                                   [Hardware cache event]
  L1-dcache-store-misses                             [Hardware cache event]
  L1-dcache-prefetches                               [Hardware cache event]
  L1-dcache-prefetch-misses                          [Hardware cache event]
  L1-icache-loads                                    [Hardware cache event]
  L1-icache-load-misses                              [Hardware cache event]
  L1-icache-prefetches                               [Hardware cache event]
  L1-icache-prefetch-misses                          [Hardware cache event]
  LLC-loads                                          [Hardware cache event]
  LLC-load-misses                                    [Hardware cache event]
  LLC-stores                                         [Hardware cache event]
  LLC-store-misses                                   [Hardware cache event]
  LLC-prefetches                                     [Hardware cache event]
  LLC-prefetch-misses                                [Hardware cache event]
  dTLB-loads                                         [Hardware cache event]
  dTLB-load-misses                                   [Hardware cache event]
  dTLB-stores                                        [Hardware cache event]
  dTLB-store-misses                                  [Hardware cache event]
  dTLB-prefetches                                    [Hardware cache event]
  dTLB-prefetch-misses                               [Hardware cache event]
[...]
```
So you can for example specify one of those events during executing your command:

`perf stat -e dTLB-load-misses ls -lR`
```
 Performance counter stats for 'ls -lR':

         7,198,657 dTLB-misses


      13.225589146 seconds time elapsed
```
You can also specify specific and processor dependent counter from the Intel Software Developper's manual Volume 3B chapter 19.

But the major flaw of this approach is that you can only monitor program executed from command line and exiting in order to get results. It is limited to small program, namely micro-benchmark.

How to use hardware counters to monitor real life application ?

## libpfm
Enter libpfm which provides an api to access hardware counters. This enables more flexibility to integrate those hardware counters into applications.

## overseer
For Java applications, I have found the overseer library which provides a Java API through JNI wrapper to libpfm.
With the help of this library it is now possible to integrate hardware counters directly inside any application.

Here an example how to use it:
```java
import ch.usi.overseer.OverHpc;
 
public class HardwareCounters
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
 
    public static void main(String[] args) throws Exception
    {
        OverHpc oHpc = OverHpc.getInstance();
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
        int tid = oHpc.getThreadId();
        oHpc.bindEventsToThread(tid);
         
        // process to monitor
         
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
## Conclusion
Hardware performance counters provided by CPU can be valuable to understand why you have some spikes into your application process. Thanks to overseer library you can inject those counters deep inside your application to fine-grained monitor your process.

In the next posts we will use this technique for various examples.

## References
Overseer library: http://www.peternier.com/projects/overseer/overseer.php
libpfm: http://perfmon2.sourceforge.net/docs_v4.html
Intel Software Developper's manual Volume 3B: http://download.intel.com/products/processor/manual/253669.pdf
