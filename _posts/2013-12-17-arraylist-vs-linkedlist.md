---
title: "ArrayList vs LinkedList"
layout: default
---

*This post was originally posted on [Java Advent Calendar](http://www.javaadvent.com/2013/12/arraylist-vs-linkedlist.html)*

I must confess the title of this post is a little bit catchy. I have recently read this blog post and this is a good summary of  discussions & debates about this subject.
But this time I would like to try a different approach to compare those 2 well known data structures: using [Hardware Performance Counters](https://jpbempel.github.io/2013/08/02/hardware-performance-counters.html).

I will not perform a micro-benchmark, well not directly. I will not time using `System.nanoTime()`, but rather using HPCs like cache hits/misses.

No need to present those data structures, everybody knows what they are using for and how they are implemented. I am focusing my study on list iteration because, beside adding an element, this is the most common task for a list. And also because the memory access pattern for a list is a good example of CPU cache interaction.


Here my code for measuring list iteration for `LinkedList` & `ArrayList`:
```java
import java.util.ArrayList;
import java.util.LinkedList;
import java.util.List;
 
import ch.usi.overseer.OverHpc;
 
public class ListIteration
{
    private static List<String> arrayList = new ArrayList<>();
    private static List<String> linkedList = new LinkedList<>();
 
    public static void initializeList(List<String> list, int bufferSize)
    {
        for (int i = 0; i < 50000; i++)
        {
            byte[] buffer = null;
            if (bufferSize > 0)
            {
                buffer = new byte[bufferSize];
            }
            String s = String.valueOf(i);
            list.add(s);
            // avoid buffer to be optimized away
            if (System.currentTimeMillis() == 0)
            {
                System.out.println(buffer);
            }
        }
    }
 
    public static void bench(List<String> list)
    {
        if (list.contains("bar"))
        {
            System.out.println("bar found");
        }
    }
 
    public static void main(String[] args) throws Exception
    {
        if (args.length != 2) return;
        List<String> benchList = "array".equals(args[0]) ? arrayList : linkedList;
        int bufferSize = Integer.parseInt(args[1]);
        initializeList(benchList, bufferSize);
        HWCounters.init();
        System.out.println("init done");
        // warmup
        for (int i = 0; i < 10000; i++)
        {
            bench(benchList);
        }
        Thread.sleep(1000);
        System.out.println("warmup done");
 
        HWCounters.start();
        for (int i = 0; i < 1000; i++)
        {
            bench(benchList);
        }
        HWCounters.stop();
        HWCounters.printResults();
        HWCounters.shutdown();
    }
}
``` 
To measure, I am using a class called HWCounters based on [overseer library](http://www.peternier.com/projects/overseer/overseer.php) to get Hardware Performance Counters. You can find the code of this class [here](https://github.com/jpbempel/snippets/blob/master/HPC/HWCounters.java).

The program take 2 parameters: the first one to choose between `ArrayList` implementation or `LinkedList` one, the second one to take a buffer size used in `initializeList` method. This method fills a list implementation with 50K strings. Each string is newly created just before add it to the list. We may also allocate a buffer based on our second parameter of the program. if 0, no buffer is allocated.
`bench` method performs a search of a constant string which is not contained into the list, so we fully traverse the list.
Finally, `main` method, perform initialization of the list, warmups the `bench` method and measure 1000 runs of this method. Then, we print results from HPCs.

Let's run our program with no buffer allocation on Linux with 2 Xeon X5680:
```
[root@archi-srv]# java -cp .:overseer.jar com.ullink.perf.myths.ListIteration array 0
init done
warmup done
Cycles: 428,711,720
Instructions: 776,215,597
L2 hits: 5,302,792
L2 misses: 23,702,079
LLC hits: 42,933,789
LLC misses: 73
CPU migrations: 0
Local DRAM: 0
Remote DRAM: 0

[root@archi-srv]# /java -cp .:overseer.jar com.ullink.perf.myths.ListIteration linked 0
init done
warmup done
Cycles: 767,019,336
Instructions: 874,081,196
L2 hits: 61,489,499
L2 misses: 2,499,227
LLC hits: 3,788,468
LLC misses: 0
CPU migrations: 0
Local DRAM: 0
Remote DRAM: 0
```
First run is on the `ArrayList` implementation, second with `LinkedList`.

* Number of cycles is the number of CPU cycle spent on executing our code. Clearly `LinkedList` spent much more cycles than `ArrayList`.
* Instructions is little higher for `LinkedList`. But it is not so significant here.
* For L2 cache accesses we have a clear difference: `ArrayList` has significant more L2 misses compared to `LinkedList`.
* Mechanically, LLC hits are very important for `ArrayList`.

The conclusion on this comparison is that most of the data accessed during list iteration is located into L2 for `LinkedList` but into L3 for `ArrayList`.
My explanation for this is that strings added to the list are created right before. For `LinkedList` it means that it is local the `Node` entry that is created when adding the element. We have more locality with the node.

But let's re-run the comparison with intermediary buffer allocated for each new `String` added.

```
[root@archi-srv]# java -cp .:overseer.jar com.ullink.perf.myths.ListIteration array 256
init done
warmup done
Cycles: 584,965,201
Instructions: 774,373,285
L2 hits: 952,193
L2 misses: 62,840,804
LLC hits: 63,126,049
LLC misses: 4,416
CPU migrations: 0
Local DRAM: 824
Remote DRAM: 0

[root@archi-srv]# java -cp .:overseer.jar com.ullink.perf.myths.ListIteration linked 256
init done
warmup done
Cycles: 5,289,317,879
Instructions: 874,350,022
L2 hits: 1,487,037
L2 misses: 75,500,984
LLC hits: 81,881,688
LLC misses: 5,826,435
CPU migrations: 0
Local DRAM: 1,645,436
Remote DRAM: 1,042
```
Here the results are quite different:

* Cycles are 10 times more important.
* Instructions stay the same as previously
* For cache accesses, `ArrayList` have more L2 misses/LLC hits, than previous run, but still in the same magnitude order. `LinkedList` on the contrary have a lot more L2 misses/LLC hits, but moreover a significant number of LLC misses/DRAM accesses. And the difference is here.

With the intermediary buffer, we are pushing away entries and strings, which generate more cache misses and the end also DRAM accesses which is much more slower than hitting caches.
`ArrayList` is more predictable here since we keep locality of element from each other.

The memory access pattern here is crucial for list iteration performance. `ArrayList` is more stable than `LinkedList` in the way that whatever you are doing between each element adding, you are keeping your data  much more local than the `LinkedList`.
Remember also that, iterating through an array is much more efficient for CPU since it can trigger Hardware Prefetching because access pattern is very predictable.
