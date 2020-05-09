---
title: "Volatile and memory barriers"
layout: default
---
# Volatile and memory barriers

I have already [blogged](https://jpbempel.github.io/2012/10/09/volatile.html) on the effect of a volatile variable on optimization performed by the JIT. But what is the really difference between a regular variable ? And what are the impacts in terms of performance ?

## Semantic

Volatile has a well defined semantic in Java Memory Model (JMM), but to summarize, it has the following:
* Cannot be reordered
* Ensure data visibility to other threads

## Visibility

Doug Lea [refers](http://www.cs.umd.edu/~pugh/java/memoryModel/jsr-133-faq.html#volatile) thread visibility to flushing caches, however, as pointed out by Martin Thompson in his [post](http://mechanical-sympathy.blogspot.fr/2013/02/cpu-cache-flushing-fallacy.html), volatile access does not flush the cache for visibility (as in writing data to memory to make it visible for all cores).
[ccNUMA](https://en.wikipedia.org/wiki/CcNUMA#Cache_coherent_NUMA_.28ccNUMA.29) architecture means that all data in cache subsystem are in fact coherent with main memory. So the semantic of volatile apply through the Load/Store buffers (or Memory Ordering Buffers) placed between registers and L1 cache.

Depending on CPU architecture/instructions set, generated instructions to ensure those properties can vary. Let's focus on x86 which is the most wide spread.

## Reordering

For reordering there is 2 kinds:
* Compiler
* Hardware/CPU
Compiler is able to reorder instructions during [instruction scheduling](http://en.wikipedia.org/wiki/Instruction_scheduling) to match cost of loading or storing data with the CPU specifications.
It could be interesting to trigger 2 loads without dependencies between each other in order to optimize the time spent by the CPU to wait for those data and perform other operations in the mean time.

In the following example I have put 6 non-volatile fields :

```java
public class TestJIT
{
    private static int field1;
    private static int field2;
    private static int field3;
    private static int field4;
    private static int field5;
    private static int field6;
     
    private static void assign(int i)
    {
        field1 = i << 1;
        field2 = i << 2;
        field3 = i << 3;
        field4 = i << 4;
        field5 = i << 5;
        field6 = i << 6;
    }
 
    public static void main(String[] args) throws Exception
    {
        for (int i = 0; i < 10000; i++)
        {
            assign(i);
        }
        Thread.sleep(1000);
    }
}
```

Let's examine what is [generated](https://jpbempel.github.io/2012/10/16/how-to-print-disassembly-from-JIT-code.html) by the JIT for the method `assign`:
```
# {method} 'assign' '(I)V' in 'com/bempel/sandbox/TestJIT'
# parm0:    ecx       = int
#           [sp+0x10]  (sp of caller)
0x02438800: push   ebp
0x02438801: sub    esp,0x8            ;*synchronization entry
                                      ; - com.bempel.sandbox.TestJIT::assign@-1 (line 26)
0x02438807: mov    ebx,ecx
0x02438809: shl    ebx,1
0x0243880b: mov    edx,ecx
0x0243880d: shl    edx,0x2
0x02438810: mov    eax,ecx
0x02438812: shl    eax,0x3
0x02438815: mov    esi,ecx
0x02438817: shl    esi,0x4
0x0243881a: mov    ebp,ecx
0x0243881c: shl    ebp,0x5
0x0243881f: shl    ecx,0x6
0x02438822: mov    edi,0x160
0x02438827: mov    DWORD PTR [edi+0x565c3b0],ebp
                                      ;*putstatic field5
                                      ; - com.bempel.sandbox.TestJIT::assign@27 (line 30)
                                      ;   {oop('com/bempel/sandbox/TestJIT')}
0x0243882d: mov    ebp,0x164
0x02438832: mov    DWORD PTR [ebp+0x565c3b0],ecx
                                      ;*putstatic field6
                                      ; - com.bempel.sandbox.TestJIT::assign@34 (line 31)
                                      ;   {oop('com/bempel/sandbox/TestJIT')}
0x02438838: mov    ebp,0x150
0x0243883d: mov    DWORD PTR [ebp+0x565c3b0],ebx
                                      ;*putstatic field1
                                      ; - com.bempel.sandbox.TestJIT::assign@3 (line 26)
                                      ;   {oop('com/bempel/sandbox/TestJIT')}
0x02438843: mov    ecx,0x154
0x02438848: mov    DWORD PTR [ecx+0x565c3b0],edx
                                      ;*putstatic field2
                                      ; - com.bempel.sandbox.TestJIT::assign@9 (line 27)
                                      ;   {oop('com/bempel/sandbox/TestJIT')}
0x0243884e: mov    ebx,0x158
0x02438853: mov    DWORD PTR [ebx+0x565c3b0],eax
                                      ;*putstatic field3
                                      ; - com.bempel.sandbox.TestJIT::assign@15 (line 28)
                                      ;   {oop('com/bempel/sandbox/TestJIT')}
0x02438859: mov    ecx,0x15c
0x0243885e: mov    DWORD PTR [ecx+0x565c3b0],esi
                                      ;*putstatic field4
                                      ; - com.bempel.sandbox.TestJIT::assign@21 (line 29)
                                      ;   {oop('com/bempel/sandbox/TestJIT')}
0x02438864: add    esp,0x8
0x02438867: pop    ebp
0x02438868: test   DWORD PTR ds:0x190000,eax
                                      ;   {poll_return}
0x0243886e: ret   
```

As you can see in the comments, the order of field assignation is the following: `field5`, `field6`, `field1`, `field2`, `field3`.
We now add volatile modifier for `field1` & `field6`:
```
# {method} 'assign' '(I)V' in 'com/bempel/sandbox/TestJIT'
# parm0:    ecx       = int
#           [sp+0x10]  (sp of caller)
0x024c8800: push   ebp
0x024c8801: sub    esp,0x8
0x024c8807: mov    ebp,ecx
0x024c8809: shl    ebp,1
0x024c880b: mov    edx,ecx
0x024c880d: shl    edx,0x2
0x024c8810: mov    esi,ecx
0x024c8812: shl    esi,0x3
0x024c8815: mov    eax,ecx
0x024c8817: shl    eax,0x4
0x024c881a: mov    ebx,ecx
0x024c881c: shl    ebx,0x5
0x024c881f: shl    ecx,0x6
0x024c8822: mov    edi,0x150
0x024c8827: mov    DWORD PTR [edi+0x562c3b0],ebp
                                      ;*putstatic field1
                                      ; - com.bempel.sandbox.TestJIT::assign@3 (line 26)
                                      ;   {oop('com/bempel/sandbox/TestJIT')}
0x024c882d: mov    ebp,0x160
0x024c8832: mov    DWORD PTR [ebp+0x562c3b0],ebx
                                      ;*putstatic field5
                                      ; - com.bempel.sandbox.TestJIT::assign@27 (line 30)
                                      ;   {oop('com/bempel/sandbox/TestJIT')}
0x024c8838: mov    ebx,0x154
0x024c883d: mov    DWORD PTR [ebx+0x562c3b0],edx
                                      ;*putstatic field2
                                      ; - com.bempel.sandbox.TestJIT::assign@9 (line 27)
                                      ;   {oop('com/bempel/sandbox/TestJIT')}
0x024c8843: mov    ebp,0x158
0x024c8848: mov    DWORD PTR [ebp+0x562c3b0],esi
                                      ;*putstatic field3
                                      ; - com.bempel.sandbox.TestJIT::assign@15 (line 28)
                                      ;   {oop('com/bempel/sandbox/TestJIT')}
0x024c884e: mov    ebx,0x15c
0x024c8853: mov    DWORD PTR [ebx+0x562c3b0],eax
                                      ;   {oop('com/bempel/sandbox/TestJIT')}
0x024c8859: mov    ebp,0x164
0x024c885e: mov    DWORD PTR [ebp+0x562c3b0],ecx
                                      ;   {oop('com/bempel/sandbox/TestJIT')}
0x024c8864: lock add DWORD PTR [esp],0x0  ;*putstatic field1
                                      ; - com.bempel.sandbox.TestJIT::assign@3 (line 26)
0x024c8869: add    esp,0x8
0x024c886c: pop    ebp
0x024c886d: test   DWORD PTR ds:0x180000,eax
                                      ;   {poll_return}
0x024c8873: ret   
```

Now, `field1` is really the first field assigned and `field6` is last one. But in the middle we have the following order: `field5`, `field2`, `field3`, `field4`.

Beside that, CPU is able to reorder the instruction flow in certain circumstances to also optimize the efficiency of instruction execution. Those properties are well summarized in the Intel white paper [Memory Ordering](http://www.cs.cmu.edu/~410-f10/doc/Intel_Reordering_318147.pdf).

## Write access
From previous example which is only volatile writes you can notice there is a special instruction: `lock add`. This is a bit weird at first, we add 0 to the memory pointed by SP (Stack Pointer) register. So we are not changing the data at this memory location but with `lock` prefix, memory related instructions are processed specially to act as a *memory barrier*, similar to `mfence` instruction. Dave Dice explains in his [blog](https://blogs.oracle.com/dave/entry/instruction_selection_for_volatile_fences) that they benchmarked the 2 kinds of barrier, and the lock add seems the most efficient one on today's architecture.

So this barrier ensures that there is no reordering before and after this instruction and also drains all instructions pending into Store Buffer. After executing these instructions, all writes are visible to all other threads through cache subsystem or main memory. This costs some latency to wait for this drain.

## LazySet
Some time we can relax the constraint on immediate visibility but still keep the ordering guarantee. For this reason, Doug Lea introduces `lazySet` method for Atomic* objects. Let's use `AtomicInteger` in replacement of `volatile int`:

```java
public class TestJIT
{
    private static AtomicInteger field1 = new AtomicInteger(0);
    private static int field2;
    private static int field3;
    private static int field4;
    private static int field5;
    private static AtomicInteger field6 = new AtomicInteger(0);
     
    public static void assign(int i)
    {
        field1.lazySet(i << 1);
        field2 = i << 2;
        field3 = i << 3;       
        field4 = i << 4;
        field5 = i << 5;
        field6.lazySet(i << 6);
    }
     
    public static void main(String[] args) throws Throwable
    {
        for (int i = 0; i < 10000; i++)
        {
            assign(i);
        }
        Thread.sleep(1000);
    }
}
```

We have the following output for [PrintAssembly](https://jpbempel.github.io/2012/10/16/how-to-print-disassembly-from-JIT-code.html):
```
  # {method} 'assign' '(I)V' in 'com/bempel/sandbox/TestJIT'
  # this:     ecx       = 'com/bempel/sandbox/TestJIT'
  # parm0:    edx       = int
  #           [sp+0x10]  (sp of caller)
  0x024c7f40: cmp    eax,DWORD PTR [ecx+0x4]
  0x024c7f43: jne    0x024ace40         ;   {runtime_call}
  0x024c7f49: xchg   ax,ax
[Verified Entry Point]
  0x024c7f4c: mov    DWORD PTR [esp-0x3000],eax
  0x024c7f53: push   ebp
  0x024c7f54: sub    esp,0x8            ;*synchronization entry
                                        ; - com.bempel.sandbox.TestJIT::assign@-1 (line 30)
  0x024c7f5a: mov    ebp,edx
  0x024c7f5c: shl    ebp,1              ;*ishl
                                        ; - com.bempel.sandbox.TestJIT::assign@5 (line 30)
  0x024c7f5e: mov    ebx,0x150
  0x024c7f63: mov    ecx,DWORD PTR [ebx+0x56ec4b8]
                                        ;*getstatic field1
                                        ; - com.bempel.sandbox.TestJIT::assign@0 (line 30)
                                        ;   {oop('com/bempel/sandbox/TestJIT')}
  0x024c7f69: test   ecx,ecx
  0x024c7f6b: je     0x024c7fd0
  0x024c7f6d: mov    DWORD PTR [ecx+0x8],ebp  ;*invokevirtual putOrderedInt
                                        ; - java.util.concurrent.atomic.AtomicInteger::lazySet@8 (line 80)
                                        ; - com.bempel.sandbox.TestJIT::assign@6 (line 30)
  0x024c7f70: mov    edi,edx
  0x024c7f72: shl    edi,0x2
  0x024c7f75: mov    ebp,edx
  0x024c7f77: shl    ebp,0x3
  0x024c7f7a: mov    esi,edx
  0x024c7f7c: shl    esi,0x4
  0x024c7f7f: mov    eax,edx
  0x024c7f81: shl    eax,0x5
  0x024c7f84: shl    edx,0x6            ;*ishl
                                        ; - com.bempel.sandbox.TestJIT::assign@39 (line 35)
  0x024c7f87: mov    ebx,0x154
  0x024c7f8c: mov    ebx,DWORD PTR [ebx+0x56ec4b8]
                                        ;*getstatic field6
                                        ; - com.bempel.sandbox.TestJIT::assign@33 (line 35)
                                        ;   {oop('com/bempel/sandbox/TestJIT')}
  0x024c7f92: mov    ecx,0x158
  0x024c7f97: mov    DWORD PTR [ecx+0x56ec4b8],edi
                                        ;*putstatic field2
                                        ; - com.bempel.sandbox.TestJIT::assign@12 (line 31)
                                        ;   {oop('com/bempel/sandbox/TestJIT')}
  0x024c7f9d: mov    ecx,0x164
  0x024c7fa2: mov    DWORD PTR [ecx+0x56ec4b8],eax
                                        ;*putstatic field5
                                        ; - com.bempel.sandbox.TestJIT::assign@30 (line 34)
                                        ;   {oop('com/bempel/sandbox/TestJIT')}
  0x024c7fa8: mov    edi,0x15c
  0x024c7fad: mov    DWORD PTR [edi+0x56ec4b8],ebp
                                        ;*putstatic field3
                                        ; - com.bempel.sandbox.TestJIT::assign@18 (line 32)
                                        ;   {oop('com/bempel/sandbox/TestJIT')}
  0x024c7fb3: mov    ecx,0x160
  0x024c7fb8: mov    DWORD PTR [ecx+0x56ec4b8],esi
                                        ;*putstatic field4
                                        ; - com.bempel.sandbox.TestJIT::assign@24 (line 33)
                                        ;   {oop('com/bempel/sandbox/TestJIT')}
  0x024c7fbe: test   ebx,ebx
  0x024c7fc0: je     0x024c7fdd
  0x024c7fc2: mov    DWORD PTR [ebx+0x8],edx  ;*invokevirtual putOrderedInt
                                        ; - java.util.concurrent.atomic.AtomicInteger::lazySet@8 (line 80)
                                        ; - com.bempel.sandbox.TestJIT::assign@40 (line 35)
  0x024c7fc5: add    esp,0x8
  0x024c7fc8: pop    ebp
  0x024c7fc9: test   DWORD PTR ds:0x180000,eax
                                        ;   {poll_return}
  0x024c7fcf: ret   
```
So now no trace of memory barriers whatsoever: (no `mfence`, no `lock add` instruction). But we have the order of `field1` and `field6` remain (first and last) then `field2`, `field5`, `field3` and `field4`.
In fact, `lazySet` method call `putOrderedInt` from `Unsafe` object which do not emit hardware memory barrier but guarantee no reordering by the JIT.

### Read Access
We will now examine what is the cost of a volatile read with this example:
```java
public class TestJIT
{
    private static volatile int field1 = 42;
     
    public static void testField1(int i)
    {
        if (field1 < 0)
        {
            System.out.println("field value: " + field1);
        }
    }
     
    public static void main(String[] args) throws Throwable
    {
        for (int i = 0; i < 10000; i++)
        {
            testField1(i);
        }
        Thread.sleep(1000);
    }
}
```

The [PrintAssembly](https://jpbempel.github.io/2012/10/16/how-to-print-disassembly-from-JIT-code.html) output looks like:
```
# {method} 'testField1' '(I)V' in 'com/bempel/sandbox/TestJIT'
# parm0:    ecx       = int
#           [sp+0x10]  (sp of caller)
0x024f8800: mov    DWORD PTR [esp-0x3000],eax
0x024f8807: push   ebp
0x024f8808: sub    esp,0x8            ;*synchronization entry
                                      ; - com.bempel.sandbox.TestJIT::testField1@-1 (line 22)
0x024f880e: mov    ebx,0x150
0x024f8813: mov    ecx,DWORD PTR [ebx+0x571c418]
                                      ;*getstatic field1
                                      ; - com.bempel.sandbox.TestJIT::testField1@0 (line 22)
                                      ;   {oop('com/bempel/sandbox/TestJIT')}
0x024f8819: test   ecx,ecx
0x024f881b: jl     0x024f8828         ;*ifge
                                      ; - com.bempel.sandbox.TestJIT::testField1@3 (line 22)
0x024f881d: add    esp,0x8
0x024f8820: pop    ebp
0x024f8821: test   DWORD PTR ds:0x190000,eax
                                      ;   {poll_return}
0x024f8827: ret   
```
No memory barrier. Only a load from memory address to a register. This is what is required for a volatile read. JIT cannot optimize to cache it into a register. But with cache subsystem, latency for volatile read is very similar compared to regular variable read. If you test the same Java code without volatile modifier the result for this test is in fact the same.
Volatile reads put also some constraint on reordering.

## Summary
Volatile access prevents instructions reordering either at compiler level or at hardware level.
It ensures visibility, for write with the price of memory barriers, for read with no register caching or reordering possibilities.

## References

* [CPU Cache Flushing Fallacy ](http://mechanical-sympathy.blogspot.fr/2013/02/cpu-cache-flushing-fallacy.html)
* [8.1.2.2 in Volume 3 of Intel® 64 and IA-32 Architectures Software Developer’s Manuals]()
* [Intel® 64 Architecture Memory Ordering](http://www.cs.cmu.edu/~410-f10/doc/Intel_Reordering_318147.pdf) 
* [The Semantics of x86 Multiprocessor Programs](http://www.cl.cam.ac.uk/~pes20/weakmemory/index3.html)
* [Memory Barriers and JVM Concurrency](http://www.infoq.com/articles/memory_barriers_jvm_concurrency)
* [JSR 133 (Java Memory Model) FAQ](http://www.cs.umd.edu/~pugh/java/memoryModel/jsr-133-faq.html#volatile)
* [Instruction selection for volatile fences : MFENCE vs LOCK:ADD](https://blogs.oracle.com/dave/entry/instruction_selection_for_volatile_fences)
* [ccNUMA](https://en.wikipedia.org/wiki/CcNUMA#Cache_coherent_NUMA_.28ccNUMA.29)
* [Instruction scheduling](http://en.wikipedia.org/wiki/Instruction_scheduling)
