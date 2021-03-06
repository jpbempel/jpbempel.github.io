---
title:  "The useful JVM options"
layout: default
date: 2012-11-15
---
# The useful JVM options
*This post relates to JDK6 at the time of writing* 

HotSpot JVM has a lot of options available. Maybe too much. Sometimes we are looking for the right options, or the "magic" one that can give you a serious boost on your application. Unfortunately, I think it does not exist ! However some can help you for monitoring your application or for tuning some parts.

To find the complete list of options you will find in the [globals.hpp](https://github.com/openjdk/jdk/blob/master/src/hotspot/share/runtime/globals.hpp) file from OpenJDK sources.
[VM Options Explorer](https://chriswhocodes.com/vm-options-explorer.html) however can help you to navigate through the list.

I have summed up  here, in my humble opinion, some of the most useful JVM options.

## Heap sizing
You perfectly know `-Xms` & `-Xms` options (that can also abbreviate by `-ms` `-mx`), but some parts of the Java heap and non-heap can also be sized:

* `-XX:NewSize=n` Defines the initial size of the Young generation, including Eden, & Survivors. 
* `-XX:MaxNewSize=n` Defines the maximum size of the Young generation, including Eden & Survivors.
* `-XX:SurvivorRatio=n` Ratio between Eden Size and one of the 2 survivors

`n`  without unit is expressed in bytes, you can also used `k`, `K`, `m`, `M`, `g` & `G` to expressed respectively kilobytes, megabytes & gigabytes.
If `NewSize` < `MaxNewSize`, young generation size can be adjust during application life. However resizing does require a FullGC. To avoid this, set the same value for both options.

* `-XX:PermSize=n` Defines the initial size of the Permanent generation
* `-XX:MaxPermSize=n` Defines the maximum size of the Permanent generation

`n` without unit is expressed in bytes, you can also used `k`, `K`, `m`, `M`, `g` & `G` to expressed respectively kilobytes, megabytes & gigabytes.
If `PermSize` < `MaxPermSize`, permanent generation size can be adjust during application life. However resizing does require a FullGC. To avoid this, set the same value for both options.


* `-XX:InitialCodeCacheSize=n` Defines the initial size of the Code Cache.
* `-XX:ReservedCodeCacheSize=n` Defines the maximum size of the Code Cache.

`n` without unit is expressed in bytes, you can also used `k`, `K`, `m`, `M`, `g` & `G` to expressed respectively kilobytes, megabytes & gigabytes.
Code Cache stores the JITed code. This is an off-heap space, so GC does not reclaim it. If you reached the limit of the `ReservedCodeCacheSize`, the JIT compiler stops to compile more methods since it cannot store them. So if you have a lot of classes/methods that are needed to be compiled, be aware of these options.
When you reach the limit, a warning is emitted on the standard output:
```
Java HotSpot(TM) Server VM warning: CodeCache is full. Compiler has been disabled"
```
with -XX:+PrintCompilation you will get also:
```
7383 COMPILE SKIPPED: code cache is full
```

## GC Settings/Tuning
For configuring GC, there is tons of options, specially for the CMS! But at the end few are really usefull. I will not enter into fine tuning of CMS.

* `-XX:+UseSerialGC` Activates classic monothreaded young GC
* `-XX:+UseParallelGC` Activates multi-threaded young GC
* `-XX:+UseParNewGC` Activates another multi-threaded young GC but required for CMS.
* `-XX:+UseParalelOldGC` Activates multi-threaded Old GC.
* `-XX:+UseConcMarkSweepGC` Activates Concurrent GC
* `-XX:+UseG1GC Activates` Garbage First GC
* `-XX:ParallelGCThreads=n` Fixes number of threads for ParallelGC Usefull for multiple JVM on same machine to avoid GC threads disturb other JVM threads
* `-XX:CMSInitiatingOccupancyFraction=n` Fixes fraction in percentage of the old when CMS start. Usefull when allocation rate is not very predictable.
* `-XX:+UseCMSInitiatingOccupancyOnly` CMS only start based on the previous parameter and not on the internal heuristic.
* `-XX:+CMSClassUnloadingEnabled` Enables to unload classes from PermGen during CMS GC. Otherwise you need to wait the FullGC.

## GC Monitoring 
For an application, it is absolutely required to log GC activity in a separate file in order to diagnostic any abnormal behavior of memory management. Also note that activating this info has virtually no impact on your application, so there is no excuse to not activate these options.

* `-Xloggc:file` Indicates where the gc log file will be written

With only this option you will get the following content:
```
8.567: [GC 16448K->168K(62848K), 0.0014256 secs]
16.979: [GC 16616K->152K(62848K), 0.0007727 secs]
25.157: [GC 16600K->152K(62848K), 0.0007124 secs]
33.881: [GC 16600K->136K(62848K), 0.0003721 secs]
43.032: [GC 16584K->152K(59968K), 0.0004123 secs]
50.795: [GC 16216K->152K(59584K), 0.0009683 secs]
58.584: [GC 15832K->148K(59136K), 0.0008669 secs]
66.430: [GC 15508K->148K(58816K), 0.0003436 secs]
73.766: [GC 15188K->148K(58688K), 0.0002917 secs]
81.082: [GC 14868K->148K(58176K), 0.0003157 secs]
88.851: [GC 14548K->148K(58048K), 0.0003148 secs]
96.379: [GC 14228K->148K(57536K), 0.0003129 secs]
101.618: [GC 13908K->704K(57472K), 0.0032532 secs]
102.684: [GC 3206K->744K(57280K), 0.0034611 secs]
102.687: [Full GC 744K->723K(57280K), 0.0525762 secs]
109.313: [GC 14227K->755K(57344K), 0.0005165 secs]
115.905: [GC 14003K->779K(56768K), 0.0006688 secs] 
```

* `-XX:+PrintGCDetails` Activates more details in gc logs

With this option you will get the following content:

```
9.053: [GC [PSYoungGen: 16448K->168K(19136K)] 16448K->168K(62848K), 0.0014471 secs] [Times: user=0.03 sys=0.00, real=0.00 secs]
16.991: [GC [PSYoungGen: 16616K->152K(19136K)] 16616K->152K(62848K), 0.0008244 secs] [Times: user=0.02 sys=0.00, real=0.00 secs]
24.965: [GC [PSYoungGen: 16600K->152K(19136K)] 16600K->152K(62848K), 0.0006850 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
32.915: [GC [PSYoungGen: 16600K->152K(19136K)] 16600K->152K(62848K), 0.0007040 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
40.857: [GC [PSYoungGen: 16600K->152K(16256K)] 16600K->152K(59968K), 0.0007618 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
48.653: [GC [PSYoungGen: 16216K->152K(15872K)] 16216K->152K(59584K), 0.0007632 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
53.896: [GC [PSYoungGen: 15832K->192K(15552K)] 15832K->620K(59264K), 0.0131835 secs] [Times: user=0.03 sys=0.00, real=0.01 secs]
55.252: [GC [PSYoungGen: 3888K->64K(15424K)] 4317K->716K(59136K), 0.0025925 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
55.255: [Full GC (System) [PSYoungGen: 64K->0K(15424K)] [PSOldGen: 652K->715K(43712K)] 716K->715K(59136K) [PSPermGen: 5319K->5319K(16384K)], 0.0269456 secs] [Times: user=0.03 sys=0.00, real=0.03 secs]
62.544: [GC [PSYoungGen: 15360K->32K(15488K)] 16075K->747K(59200K), 0.0006252 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
69.502: [GC [PSYoungGen: 15072K->32K(14784K)] 15787K->747K(58496K), 0.0003713 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
```

As you can see there is a lot more and valuable info. You can now know the type of GC configured for Young & Old (`PSYoungGen` & `PSOlgGen` = ParallelScavenge, ParallelGC), Full GC is triggered by an explicit call to `System.GC()` and you have the detail of memory usage of young gen & old gen so you can deduce promotion between the two. Also with CMS you have all the CMS phases that are logged with Initial Mark pause time & Remark pause time. This is a MUST HAVE option: No JVM should start without this option.

* `-XX:+PrintGCDateStamps` Prints date & time along with second elapsed since JVM's start.

With this option you will get the following content:
```
2012-11-08T22:31:12.310+0100: 8.250: [GC [PSYoungGen: 16448K->168K(19136K)] 16448K->168K(62848K), 0.0007445 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2012-11-08T22:31:20.526+0100: 16.466: [GC [PSYoungGen: 16616K->136K(19136K)] 16616K->136K(62848K), 0.0008255 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2012-11-08T22:31:28.503+0100: 24.444: [GC [PSYoungGen: 16584K->152K(19136K)] 16584K->152K(62848K), 0.0007196 secs] [Times: user=0.02 sys=0.00, real=0.00 secs]
2012-11-08T22:31:36.497+0100: 32.439: [GC [PSYoungGen: 16600K->152K(19136K)] 16600K->152K(62848K), 0.0007185 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2012-11-08T22:31:44.565+0100: 40.507: [GC [PSYoungGen: 16600K->152K(16256K)] 16600K->152K(59968K), 0.0007928 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2012-11-08T22:31:52.694+0100: 48.637: [GC [PSYoungGen: 16216K->152K(15872K)] 16216K->152K(59584K), 0.0007599 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2012-11-08T22:31:58.204+0100: 54.147: [GC [PSYoungGen: 15832K->192K(15552K)] 15832K->620K(59264K), 0.0072012 secs] [Times: user=0.03 sys=0.00, real=0.01 secs]
2012-11-08T22:31:59.999+0100: 55.942: [GC [PSYoungGen: 4932K->64K(15424K)] 5360K->716K(59136K), 0.0015449 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2012-11-08T22:32:00.000+0100: 55.944: [Full GC (System) [PSYoungGen: 64K->0K(15424K)] [PSOldGen: 652K->716K(43712K)] 716K->716K(59136K) [PSPermGen: 5311K->5311K(16384K)], 0.0273445 secs] [Times: user=0.03 sys=0.00, real=0.03 secs]
2012-11-08T22:32:07.453+0100: 63.397: [GC [PSYoungGen: 15360K->32K(15488K)] 16076K->748K(59200K), 0.0003439 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2012-11-08T22:32:14.695+0100: 70.640: [GC [PSYoungGen: 15072K->32K(14784K)] 15788K->748K(58496K), 0.0007333 secs]
```

To remove elapsed time since JVM's start you can use `-XX:-PrintGCTimeStamps` which is activated by default with `-Xloggc`

* `-XX:+PrintReferenceGC` Prints timing for special reference processing

With this option you will get the following content:
```
8.494: [GC8.496: [SoftReference, 0 refs, 0.0000084 secs]8.496: [WeakReference, 3 refs, 0.0000061 secs]8.496: [FinalReference, 6 refs, 0.0000117 secs]8.496: [PhantomReference, 0 refs, 0.0000042 secs]8.496: [JNI Weak Reference, 0.0000034 secs] [PSYoungGen: 16448K->168K(19136K)] 16448K->168K(62848K), 0.0017036 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
16.874: [GC16.874: [SoftReference, 0 refs, 0.0000050 secs]16.874: [WeakReference, 1 refs, 0.0000034 secs]16.874: [FinalReference, 4 refs, 0.0000036 secs]16.874: [PhantomReference, 0 refs, 0.0000028 secs]16.874: [JNI Weak Reference, 0.0000028 secs] [PSYoungGen: 16616K->152K(19136K)] 16616K->152K(62848K), 0.0005199 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
24.889: [GC24.890: [SoftReference, 0 refs, 0.0000078 secs]24.890: [WeakReference, 1 refs, 0.0000053 secs]24.890: [FinalReference, 4 refs, 0.0000056 secs]24.890: [PhantomReference, 0 refs, 0.0000039 secs]24.890: [JNI Weak Reference, 0.0000036 secs] [PSYoungGen: 16600K->152K(19136K)] 16600K->152K(62848K), 0.0008915 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
32.829: [GC32.830: [SoftReference, 0 refs, 0.0000084 secs]32.830: [WeakReference, 1 refs, 0.0000050 secs]32.830: [FinalReference, 4 refs, 0.0000056 secs]32.830: [PhantomReference, 0 refs, 0.0000045 secs]32.830: [JNI Weak Reference, 0.0000045 secs] [PSYoungGen: 16600K->152K(19136K)] 16600K->152K(62848K), 0.0009845 secs]
```
Helps to troubleshoot issues with special references like Weak, Soft, Phantom & JNI.

* `-XX:+PrintTenuringDistribution` Prints object age distribution in Survivors

With this option you will get the following content:

```
8.654: [GC 8.655: [ParNew
Desired survivor size 1114112 bytes, new threshold 15 (max 15)
- age   1:     187568 bytes,     187568 total
: 17472K->190K(19648K), 0.0022117 secs] 17472K->190K(63360K), 0.0023006 secs] [Times: user=0.03 sys=0.00, real=0.00 secs]
17.087: [GC 17.087: [ParNew
Desired survivor size 1114112 bytes, new threshold 15 (max 15)
- age   1:      80752 bytes,      80752 total
- age   2:     166752 bytes,     247504 total
: 17662K->286K(19648K), 0.0036440 secs] 17662K->286K(63360K), 0.0037619 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
25.491: [GC 25.491: [ParNew
Desired survivor size 1114112 bytes, new threshold 15 (max 15)
- age   1:     104008 bytes,     104008 total
- age   2:      34256 bytes,     138264 total
- age   3:     166752 bytes,     305016 total
: 17758K->321K(19648K), 0.0035331 secs] 17758K->321K(63360K), 0.0036424 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
34.745: [GC 34.746: [ParNew
Desired survivor size 1114112 bytes, new threshold 15 (max 15)
- age   1:      34400 bytes,      34400 total
- age   2:     104008 bytes,     138408 total
- age   3:      34256 bytes,     172664 total
- age   4:     166752 bytes,     339416 total
: 17793K->444K(19648K), 0.0040891 secs] 17793K->444K(63360K), 0.0042000 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
43.215: [GC 43.215: [ParNew
Desired survivor size 1114112 bytes, new threshold 15 (max 15)
- age   1:     138808 bytes,     138808 total
- age   2:      34400 bytes,     173208 total
- age   3:      34264 bytes,     207472 total
- age   4:      34256 bytes,     241728 total
- age   5:     166752 bytes,     408480 total
: 17916K->586K(19648K), 0.0048752 secs] 17916K->586K(63360K), 0.0049889 secs]
```

This is the age distribution of the object related to the number of time they are survived from a minor GC (contained in Survivor spaces)

* `-XX:+PrintHeapAtGC` Prints usage for each kind of spaces.
With this option you will get the following content:
```
{Heap before GC invocations=1 (full 0):
 PSYoungGen      total 19136K, used 16448K [0x33f20000, 0x35470000, 0x49470000)
  eden space 16448K, 100% used [0x33f20000,0x34f30000,0x34f30000)
  from space 2688K, 0% used [0x351d0000,0x351d0000,0x35470000)
  to   space 2688K, 0% used [0x34f30000,0x34f30000,0x351d0000)
 PSOldGen        total 43712K, used 0K [0x09470000, 0x0bf20000, 0x33f20000)
  object space 43712K, 0% used [0x09470000,0x09470000,0x0bf20000)
 PSPermGen       total 16384K, used 1715K [0x05470000, 0x06470000, 0x09470000)
  object space 16384K, 10% used [0x05470000,0x0561ccc8,0x06470000)
8.304: [GC [PSYoungGen: 16448K->228K(19136K)] 16448K->228K(62848K), 0.0028543 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
Heap after GC invocations=1 (full 0):
 PSYoungGen      total 19136K, used 228K [0x33f20000, 0x35470000, 0x49470000)
  eden space 16448K, 0% used [0x33f20000,0x33f20000,0x34f30000)
  from space 2688K, 8% used [0x34f30000,0x34f69100,0x351d0000)
  to   space 2688K, 0% used [0x351d0000,0x351d0000,0x35470000)
 PSOldGen        total 43712K, used 0K [0x09470000, 0x0bf20000, 0x33f20000)
  object space 43712K, 0% used [0x09470000,0x09470000,0x0bf20000)
 PSPermGen       total 16384K, used 1715K [0x05470000, 0x06470000, 0x09470000)
  object space 16384K, 10% used [0x05470000,0x0561ccc8,0x06470000)
}
{Heap before GC invocations=2 (full 0):
 PSYoungGen      total 19136K, used 16676K [0x33f20000, 0x35470000, 0x49470000)
  eden space 16448K, 100% used [0x33f20000,0x34f30000,0x34f30000)
  from space 2688K, 8% used [0x34f30000,0x34f69100,0x351d0000)
  to   space 2688K, 0% used [0x351d0000,0x351d0000,0x35470000)
 PSOldGen        total 43712K, used 0K [0x09470000, 0x0bf20000, 0x33f20000)
  object space 43712K, 0% used [0x09470000,0x09470000,0x0bf20000)
 PSPermGen       total 16384K, used 1721K [0x05470000, 0x06470000, 0x09470000)
  object space 16384K, 10% used [0x05470000,0x0561e620,0x06470000)
16.577: [GC [PSYoungGen: 16676K->261K(19136K)] 16676K->261K(62848K), 0.0021589 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
Heap after GC invocations=2 (full 0):
 PSYoungGen      total 19136K, used 261K [0x33f20000, 0x35470000, 0x49470000)
  eden space 16448K, 0% used [0x33f20000,0x33f20000,0x34f30000)
  from space 2688K, 9% used [0x351d0000,0x352115f0,0x35470000)
  to   space 2688K, 0% used [0x34f30000,0x34f30000,0x351d0000)
 PSOldGen        total 43712K, used 0K [0x09470000, 0x0bf20000, 0x33f20000)
  object space 43712K, 0% used [0x09470000,0x09470000,0x0bf20000)
 PSPermGen       total 16384K, used 1721K [0x05470000, 0x06470000, 0x09470000)
  object space 16384K, 10% used [0x05470000,0x0561e620,0x06470000)
}
```
You see before & after each minor or Full GC the usage of each spaces (eden, from & to survivors, old & perm)

* `-XX:+PrintSafepointStatistics` Prints safepoint statistics

With this option you will get the following content when application terminates on stdout:
```
         vmop                      [threads: total initially_running wait_to_block] [time: spin block sync cleanup vmop] page_trap_count
4.162: EnableBiasedLocking         [       8          0              0]      [0     0     0     0     0]  0   
8.466: ParallelGCFailedAllocation  [       8          0              0]      [0     0     0     0     2]  0   
23.927: ParallelGCFailedAllocation [       8          0              0]      [0     0     0     0     2]  0   
31.039: RevokeBias                 [       9          0              1]      [0     0     0     0     0]  0   
31.377: RevokeBias                 [      10          0              1]      [0     0     0     0     0]  0   
31.388: RevokeBias                 [      11          1              1]      [2     0     2     0     0]  0   
31.390: ParallelGCFailedAllocation [      11          0              0]      [0     0     0     0     4]  0   
31.452: RevokeBias                 [      12          0              1]      [0     0     0     0     0]  0   
31.487: BulkRevokeBias             [      12          0              1]      [0     0     0     0     0]  0   
31.515: RevokeBias                 [      12          0              1]      [0     0     0     0     0]  0   
31.550: BulkRevokeBias             [      12          0              0]      [0     0     0     0     0]  0   
32.572: ThreadDump                 [      12          0              1]      [0     0     0     0     0]  0   
38.572: ThreadDump                 [      13          0              2]      [0     0     0     0     0]  0   
39.042: ParallelGCFailedAllocation [      13          0              0]      [0     0     0     0     6]  0   
39.575: ThreadDump                 [      13          0              1]      [0     0     0     0     0]  0   
39.775: ParallelGCSystemGC         [      13          0              0]      [0     0     0     0    35]  0   
40.574: ThreadDump                 [      13          0              1]      [0     0     0     0     0]  0   
45.519: HeapDumper                 [      13          0              0]      [0     0     0     0    93]  0   
45.626: ThreadDump                 [      13          0              1]      [0     0     0     0     0]  0   
50.575: no vm operation            [      13          0              0]      [0     0     0     0     0]  0   
50.576: ThreadDump                 [      13          0              1]      [0     0     0     0     0]  0   
50.875: ThreadDump                 [      13          0              1]      [0     0     0     0     0]  0   
50.881: GC_HeapInspection          [      13          0              1]      [0     0     0     0     9]  0   
51.577: ThreadDump                 [      13          0              0]      [0     0     0     0     0]  0   
82.281: ParallelGCFailedAllocation [      13          0              0]      [0     0     0     0     4]  0   
82.578: ThreadDump                 [      13          0              1]      [0     0     0     0     0]  0   
88.578: ThreadDump                 [      13          0              0]      [0     0     0     0     0]  0   
88.999: ParallelGCFailedAllocation [      13          0              0]      [0     0     0     0     4]  0   
94.580: ThreadDump                 [      13          0              0]      [0     0     0     0     0]  0   
95.439: ParallelGCFailedAllocation [      13          0              0]      [0     0     0     0     2]  0   
95.579: ThreadDump                 [      13          0              0]      [0     0     0     0     0]  0   
101.579: ThreadDump                [      13          0              0]      [0     0     0     0     0]  0   
102.276: ParallelGCFailedAllocation[      13          0              0]      [0     0     0     0     5]  0   
102.579: ThreadDump                [      13          0              0]      [0     0     0     0     0]  0   
133.776: no vm operation           [      12          0              1]      [0     0     0     0   332]  0   

Polling page always armed
ThreadDump                       103
HeapDumper                         1
GC_HeapInspection                  1
ParallelGCFailedAllocation        16
ParallelGCSystemGC                 1
EnableBiasedLocking                1
RevokeBias                        57
BulkRevokeBias                     2
    0 VM operations coalesced during safepoint
Maximum sync time      2 ms
Maximum vm operation time (except for Exit VM operation)     93 ms
```
These stats indicate which vm operation was performed with time taken. It means that during vm operation all threads are stopped (like for STW GC)

* `-XX:+PrintGCApplicationStoppedTime` Prints time were all application threads are stopped.
With this option you will get the following content:
``` 
Total time for which application threads were stopped: 0.0000257 seconds
6.253: [GC [PSYoungGen: 16448K->565K(19136K)] 16448K->565K(62848K), 0.0042058 secs] [Times: user=0.03 sys=0.00, real=0.00 secs]
Total time for which application threads were stopped: 0.0045092 seconds
Total time for which application threads were stopped: 0.0000916 seconds
Total time for which application threads were stopped: 0.0047405 seconds
Total time for which application threads were stopped: 0.0000869 seconds
Total time for which application threads were stopped: 0.0000285 seconds
Total time for which application threads were stopped: 0.0002229 seconds
Total time for which application threads were stopped: 0.0002159 seconds
13.310: [GC [PSYoungGen: 17013K->798K(19136K)] 17013K->798K(62848K), 0.0058890 secs] [Times: user=0.00 sys=0.00, real=0.01 secs]
Total time for which application threads were stopped: 0.0064142 seconds
26.148: [GC [PSYoungGen: 11951K->908K(19136K)] 11951K->908K(62848K), 0.0060538 secs] [Times: user=0.03 sys=0.00, real=0.01 secs]
26.154: [Full GC (System) [PSYoungGen: 908K->0K(19136K)] [PSOldGen: 0K->876K(43712K)] 908K->876K(62848K) [PSPermGen: 5363K->5363K(16384K)], 0.0385560 secs] [Times: user=0.03 sys=0.00, real=0.04 secs]
Total time for which application threads were stopped: 0.0450568 seconds
39.332: [GC [PSYoungGen: 16448K->150K(16256K)] 17349K->1051K(33664K), 0.0017511 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
Total time for which application threads were stopped: 0.0020282 seconds
``` 
This output is very usefull since we have the actual time during all threads are stopped by VM operations like GC, thread dump, heap dump, revoke bias, deoptimization, etc.
As you can see GCs may stop threads a little more than what is printed for just the GC processing. The time needed for a thread to reach a safepoint may add overhead.

* `-XX:+HeapDumpBeforeFullGC` Generates a Heap Dump just after a Full GC. If you do not expect a Full GC to occur during a certain period of time (like during day) and if a Full GC occurred, you can take time to generate a heap dump to understand why it happens.
* `-XX:+PrintClassHistogramBeforeFullGC` Prints a class histogram (number of instances per class with bytes allocated) in GC log. Light version of heap dump.
* `-XX:+HeapDumpOnOutOfMemoryError` Generates a heap dump when OutOfMemory occurs. Help to determine which classes/objects generates this OOME
* `-XX:HeapDumpPath=path` Specifies where to put the heap dump (java_pid<pid>.hprof)

## Monitoring 
Some options are usefull for monitoring the JVM behavior and that are not related to GC.

* `-XX:+PrintCompilation` Prints each method that is compiled by the JIT compiler.

With this option you will get the following content on stdout:
```
  1       java.lang.Object::<init> (1 bytes)
---   n   java.lang.System::currentTimeMillis (static)
  2       java.util.ArrayList::add (29 bytes)
  3       java.util.ArrayList::ensureCapacity (58 bytes)
---   n   java.lang.Thread::sleep (static)
  1%      com.bempel.sandbox.TestJIT::allocate @ 7 (84 bytes)
  4       com.bempel.sandbox.TestJIT::call (10 bytes)
  5       java.util.ArrayList::contains (14 bytes)
  6       java.util.ArrayList::indexOf (67 bytes)
  4   made not entrant  (2)  com.bempel.sandbox.TestJIT::call (10 bytes)
  7       com.bempel.sandbox.TestJIT::call (10 bytes)
  8       java.util.concurrent.CopyOnWriteArrayList::contains (22 bytes)
  9       java.util.concurrent.CopyOnWriteArrayList::indexOf (63 bytes)
  2%      com.bempel.sandbox.TestJIT::main @ 60 (83 bytes)
```

* `-XX:+LogCompilation` (required `-XX:+UnlockDiagnosticVMOptions`) Logs JIT compiler informations into hotspot.log into the current directory. Content is in xml:
```
<task_queued compile_id='4' method='com/bempel/sandbox/TestJIT call (Ljava/util/List;)V' bytes='10' count='5000' backedge_count='1' iicount='10000' stamp='20.696' comment='count' hot_count='10000'/>
<task_queued compile_id='5' method='java/util/ArrayList contains (Ljava/lang/Object;)Z' bytes='14' count='5000' backedge_count='1' iicount='10000' stamp='20.696' comment='count' hot_count='10000'/>
<task_queued compile_id='6' method='java/util/ArrayList indexOf (Ljava/lang/Object;)I' bytes='67' count='5000' backedge_count='1' iicount='10000' stamp='20.696' comment='count' hot_count='10000'/>
<writer thread='2508'/>
<nmethod compile_id='5' compiler='C2' level='2' entry='0x024c8a00' size='788' address='0x024c8908' relocation_offset='208' code_offset='248' stub_offset='472' consts_offset='487' scopes_data_offset='496' scopes_pcs_offset='636' dependencies_offset='748' handler_table_offset='752' nul_chk_table_offset='776' oops_offset='488' method='java/util/ArrayList contains (Ljava/lang/Object;)Z' bytes='14' count='7000' backedge_count='1' iicount='12000' stamp='20.702'/>
<writer thread='1392'/>
<nmethod compile_id='4' compiler='C2' level='2' entry='0x024c7a40' size='776' address='0x024c7948' relocation_offset='208' code_offset='248' stub_offset='568' consts_offset='583' scopes_data_offset='604' scopes_pcs_offset='668' dependencies_offset='748' nul_chk_table_offset='756' oops_offset='584' method='com/bempel/sandbox/TestJIT call (Ljava/util/List;)V' bytes='10' count='7000' backedge_count='1' iicount='12000' stamp='20.703'/>
<writer thread='2508'/>
<nmethod compile_id='6' compiler='C2' level='2' entry='0x024c9700' size='728' address='0x024c9608' relocation_offset='208' code_offset='248' stub_offset='472' consts_offset='487' scopes_data_offset='492' scopes_pcs_offset='576' dependencies_offset='688' handler_table_offset='692' nul_chk_table_offset='716' oops_offset='488' method='java/util/ArrayList indexOf (Ljava/lang/Object;)I' bytes='67' count='7000' backedge_count='1' iicount='12000' stamp='20.707'/>
<writer thread='3508'/>
<uncommon_trap thread='3508' reason='class_check' action='maybe_recompile' compile_id='4' compiler='C2' level='2' stamp='21.700'>
<jvms bci='3' method='com/bempel/sandbox/TestJIT call (Ljava/util/List;)V' bytes='10' count='7000' backedge_count='1' iicount='12000'/>
</uncommon_trap>
```
Can be usefull to understand why some method are deoptimized.

* `-XX:+LogVMOutput` (requires `-XX:+UnlockDiagnosticVMOptions`) Logs all the vm output (like `PrintCompilation` to the default `hotspot.log` file in current directory.

* `-XX:LogFile=file` (requires `-XX:+UnlockDiagnosticVMOptions`) Specifies the path and name for `hotpot.log` file
* `-XX:+PrintAssembly` (requires `-XX:+UnlockDiagnosticVMOptions`) Prints disassembly of method compiled by JIT compiler. See my [post](https://jpbempel.github.io/2012/10/16/how-to-print-disassembly-from-JIT-code.html).
* `-XX:ErrorFile=file` Specifies where the `hs_err_<pid>.log` file will be generated in case of a JVM crash.
* `-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.port=1234  -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false` Activates JMX remote connections to be able to connect with VisualVM.

## JIT Compiler
These options control the JIT compiler behavior:
* `-XX:CICompilerCount=n` Specifies number of JIT compiler threads used to compile methods
* `-XX:CompileThreshold=n` Specifies number of call a method is considered as a HotSpot and JIT compiler compiles it.
* `-Xcomp` Compiles method first time they are executed. (`CompileThreshold=1`)
* `-Xbatch` Compiles methods in foreground, no execution is stopped until compilation is done.
* `-Xint` Interpreted mode only. JIT compiler deactivated. (`CompileThreshold=0`)

## Optimizations 
Some options enables or disables certain kind of JIT optimizations

* `-XX:+UseCompressedStrings` Handles String objects with byte array instead of char array. Can reduce the size of consumed Heap. Beware of side effects! Disabled by default.
* `-XX:+UseBiasedLocking` Enables Biased Locking. Allow to execute code including synchronized block with virtually no overhead if there is no contention on it (only one thread execution the block). However if another thread revoke the Bias, this can be costly (Stop-The-World/safepoint during revoking). Enabled by default.
-XX:+TraceBiasedLocking Gives extra information about BiasedLocking optimization. Disabled by default
* `-XX:+PrintBiasedLockingStatistics` Gives extra information about BiasedLocking optimization. Disabled by default
* `-XX:+Inline` Enables call inlining. Could be disabled for inspecting disassembly and virtual call. Enabled by default.
* `-XX:+AggressiveOpts` Enables some new optimizations that are not enabled by default. If optimizations are good they will be enabled in a next release of JDK.
