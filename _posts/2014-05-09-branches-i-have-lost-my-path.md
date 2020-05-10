---
title: "Branches: I have lost my path!"
layout: default
date: 2014-05-09
---
# Branches: I have lost my path!

At Devoxx France 2014, I made a talk about [Hardware Performance Counters](https://jpbempel.github.io/2013/08/02/hardware-performance-counters.html), including examples I had already blogged about previously ([first](https://jpbempel.github.io/2013/10/30/hardware-performance-counters-atomic-vs-standard-incrementation.html) & [second](https://jpbempel.github.io/2013/12/17/arraylist-vs-linkedlist.html)). But I have also included a third one showing branch mispredictions measurement and effects.

For this example, I was inspired by this [question](http://stackoverflow.com/questions/11227809/why-is-processing-a-sorted-array-faster-than-an-unsorted-array) on stackoverflow. There are very good explanations of this phenomenon in answers, I encourage you to read them.

I am more interested in measuring this effect to be able to pinpoint this kind of issues in the future in my code. I have rewritten the code of the example as follow:

```java
import java.util.Random;
import java.util.Arrays;
 
public class CondLoop
{
    final static int COUNT = 64*1024;
    static Random random = new Random(System.currentTimeMillis());
 
    private static int[] createData(int count, boolean warmup, boolean predict)
    {
        int[] data = new int[count];
        for (int i = 0; i < count; i++)
        {
            data[i] = warmup ? random.nextInt(2)
                             : (predict ? 1 : random.nextInt(2));
        }
        return data;
    }
     
    private static int benchCondLoop(int[] data)
    {
        long ms = System.currentTimeMillis();
        HWCounters.start();
        int sum = 0;
        for (int i = 0; i < data.length; i++)
        {
            if (data[i] == 1)
         sum += i;
        }
        HWCounters.stop();
        return sum;
    }
 
    public static void main(String[] args) throws Exception
    {
        boolean predictable = Boolean.parseBoolean(args[0]);
        HWCounters.init();
        int count = 0;
        for (int i = 0; i < 10000; i++)
        {
            int[] data = createData(1024, true, predictable);
            count += benchCondLoop(data);
        }
        System.out.println("warmup done");
        Thread.sleep(1000);
        int[] data = createData(512*1024, false, predictable);
        count += benchCondLoop(data);
        HWCounters.printResults();
        System.out.println(count);
        HWCounters.shutdown();
    }
}
```

I have 2 modes: one is completely predictable with only 1s into the array, and the other is unpredictable with array filled with 0s and 1s randomly.
When I run my code with HPC including branch mispredictions counter on a 2 Xeon X5680 (Westmere) machine I get the following results:
```
[root@archi-srv condloop]# java -cp overseer.jar:. CondLoop true
warmup done
Cycles: 2,039,751
branch mispredicted: 20
-1676149632

[root@archi-srv condloop]# java -cp overseer.jar:. CondLoop false
warmup done
Cycles: 2,042,371
branch mispredicted: 20
-1558729579
```
We can see there is no difference between the 2 modes. In fact there is caveat in my example: It is too simple and the JIT compiler is able to perform an optimization I was not aware of at this time. To understand what's going on with my example, I made a tour with my old friend [PrintAssembly](https://jpbempel.github.io/2012/10/16/how-to-print-disassembly-from-JIT-code.html) as usual! (Note: I am using the intel syntax with the help of `-XX:PrintAssemblyOptions=intel` because well I am running on x86_64 CPU so let's use their syntax!)
```
# {method} 'benchCondLoop' '([I)I' in 'CondLoop'
[...]
0x00007fe45105fcc9: cmp    ebp,ecx
0x00007fe45105fccb: jae    0x00007fe45105fe27  ;*iaload
                                       ; - CondLoop::benchCondLoop@15 (line 28)
0x00007fe45105fcd1: mov    r8d,DWORD PTR [rbx+rbp*4+0x10]
0x00007fe45105fcd6: mov    edx,ebp
0x00007fe45105fcd8: add    edx,r13d
0x00007fe45105fcdb: cmp    r8d,0x1
0x00007fe45105fcdf: cmovne edx,r13d
0x00007fe45105fce3: inc    ebp                ;*iinc
                                       ; - CondLoop::benchCondLoop@24 (line 26)
0x00007fe45105fce5: cmp    ebp,r10d
0x00007fe45105fce8: jge    0x00007fe45105fcef  ;*if_icmpge
                                       ; - CondLoop::benchCondLoop@10 (line 26)
0x00007fe45105fcea: mov    r13d,edx
0x00007fe45105fced: jmp    0x00007fe45105fcc9
[...]
``` 

The output shows a special instruction that I was not familiar with: `cmovne`. But it reminds me a [thread](https://groups.google.com/d/msg/mechanical-sympathy/xNpCA8yjItI/5hEMV0lWv0wJ) in mechanical sympathy forum about this instruction (That's why it is important to read this forum!).
It seems this instruction is used specifically to avoid branch mispredictions.
Then, let's rewrite my condition with a more complex one:
```java
private static int benchCondLoop(int[] data)
{
    long ms = System.currentTimeMillis();
    HWCounters.start();
    int sum = 0;
    for (int i = 0; i < data.length; i++)
    {
        if (i+ms > 0 && data[i] == 1)
     sum += i;
    }
    HWCounters.stop();
    return sum;
}
``` 
Here are now the results:
``` 
[root@archi-srv condloop]# java -cp overseer.jar:. CondLoop true
warmup done
Cycles: 2,114,347
branch mispredicted: 21
-1677344554

[root@archi-srv condloop]# java -cp overseer.jar:. CondLoop false
warmup done
Cycles: 7,471,464
branch mispredicted: 261,988
-1541838686
```
See, number of cycles jump off the roof: more than 3x cycles! Remember that a misprediction for CPU means a flush of the pipeline to decode instructions from the new address and it causes a Stop-Of-The-World during this time. Depending on the CPU it lasts 10 to 20 cycles.

In stackoverflow question, sorting the array improved a lot the test. Let's do the same:
```java
int[] data = createData(512*1024, false, predictable);
Arrays.sort(data);
count += benchCondLoop(data);
```
```
[root@archi-srv condloop]# java -cp overseer.jar:. CondLoop false
warmup done
Cycles: 2,112,265
branch mispredicted: 34
-1659649448
```
This is indeed very efficient, we are now more predictable.

You can find the code of this example on my [github](https://github.com/jpbempel/presentations/blob/master/hpc/condloop/CondLoop.java)
