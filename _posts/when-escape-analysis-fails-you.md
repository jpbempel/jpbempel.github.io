---
layout: default
title: "When Escape Analysis fails you?"
date:
---
# When Escape Analysis fails you?

## Context
In november 2019, I was attending a presentation by [Thomas Wuerthinger](https://twitter.com/thomaswue) about [Abstractions Without Regret with GraalVM](https://youtu.be/noX2uHA2Udo?t=1532) at Devoxx in Antwerp. He took a simple example using `Objects#hash()` that was significantly faster with Graal Compiler than C2. I was deeply surprised by that. it was obvious that the allocation of the varargs was the culprit: It was eliminated in the case of Graal but not in the C2 one. Why the Escape Analysis failed here?

## Escape Analysis
This post is not intended to introduce you what is Escape Analysis (EA) there are plenty of other articles for that (see References). 
Though I will just say that EA is not an optimization per se, but an analysis phase (hence the name ;-)) that gather information to ba able to apply some optimizations like lock elision, [scalar replacement](https://shipilev.net/jvm/anatomy-quarks/18-scalar-replacement/) or even [stack allocation](https://github.com/microsoft/openjdk-proposals/blob/master/stack_allocation/Stack_Allocation_JEP.md).

## Objects#hashCode
The question is why in this case, that seems simple, C2's EA fails?

Let's reproduce the issue, not with a JMH benchmark but with a simple case where we can analyze the JITed code.
But first let's analyze Java code:
```java
    public static int hash(Object... values) {
        return Arrays.hashCode(values);
    }
```
`Objects#hash` calls `Arrays#hashCode`
```java
    public static int hashCode(Object a[]) {
        if (a == null)
            return 0;

        int result = 1;

        for (Object element : a)
            result = 31 * result + (element == null ? 0 : element.hashCode());

        return result;
    }
``` 
Nothing fancy, `Arrays#hashcode` should be inlined and it just iterating on the varargs array to compute hash code on each element with multiplication with a prime number.

Here is my class to reproduce the case and to play with it:
```java
import java.util.Arrays;
import java.util.Objects;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.LockSupport;

public class ObjectsHashJIT {

    Integer integerField = new Integer(42);
    Double doubleField = new Double(3.14);
    Object[] fields = {integerField, doubleField};

    public static void main(String[] args) {
        ObjectsHashJIT objectsHashJIT = new ObjectsHashJIT();
        int res = 0;
        int iters = Integer.getInteger("iters", 20_000);
        for (int i = 0; i < iters; i ++) {
            res += objectsHashJIT.bench(i);
        }
        System.out.println(res);
        long endSleep = Long.getLong("endSleep", 1);
        LockSupport.parkNanos(TimeUnit.SECONDS.toNanos(endSleep));
    }

    private int jdkObjectsHash() {
        return Objects.hash(integerField, doubleField);
    }

    private int noVarArgsObjectsHash() {
        return Arrays.hashCode(fields);
    }

    private int rawVarArgsObjectHash_noargs() {
        return rawVarArgsObjectsHash(integerField, doubleField);
    }

    private int iterVarArgsObjectsHash_noargs() {
        return iterVarArgsObjectsHash(integerField, doubleField);
    }

    private int rawObjectsHash() {
        int res = 1;
        res += 31 * res + integerField.hashCode();
        res += 31 * res + doubleField.hashCode();
        return  res;
    }

    private static int rawVarArgsObjectsHash(Object... elements) {
        int res = 1;
        res += 31 * res + elements[0].hashCode();
        res += 31 * res + elements[1].hashCode();
        return res;
    }

    private static int iterVarArgsObjectsHash(Object... elements) {
        if (elements == null)
            return 0;
        int result = 1;
        for (Object element : elements)
            result += 31 * result + (element == null ? 0 : element.hashCode());
        return result;
    }

    private int bench(int i) {
        int res = jdkObjectsHash();
        //int res = rawObjectsHash();
        //int res = noVarArgsObjectsHash();
        //int res = rawVarArgsObjectHash_noargs();
        //int res = iterVarArgsObjectsHash_noargs();
        return res + i;
    }
}
```

I run the first method `jdkObjectsHash` in `bench` method which exercised the code to be JITed and to analyze the output. I am using OpenJDK 14, and to print assembly I am using the new [Compiler Control](https://openjdk.java.net/jeps/165) feature instead of [`CompileCommand` JVM options](https://jpbempel.github.io/2016/03/16/compilecommand-jvm-option.html) to just output the `bench` method.
```json
[
  {
    match: "ObjectsHashJIT::bench",
    PrintAssembly: true,
    PrintInlining: true
  }
]
```
First the inlining decision:
```
  @ 1   ObjectsHashJIT::jdkObjectsHash (22 bytes)   inline (hot)
    @ 18   java.util.Objects::hash (5 bytes)   inline (hot)
      @ 1   java.util.Arrays::hashCode (56 bytes)   inline (hot)
        @ 43   java.lang.Object::hashCode (0 bytes)   (intrinsic, virtual)
```
Then the assembly output (yes, with the [intel syntax](https://jpbempel.github.io/2012/10/16/how-to-print-disassembly-from-JIT-code.html)):
```
  0x000002b2ef8b6350:   mov    DWORD PTR [rsp-0x7000],eax
  0x000002b2ef8b6357:   push   rbp
  0x000002b2ef8b6358:   sub    rsp,0x30
  0x000002b2ef8b635c:   mov    rbp,rdx
  0x000002b2ef8b635f:   mov    DWORD PTR [rsp],r8d
  0x000002b2ef8b6363:   mov    r8,QWORD PTR [r15+0x120]
  0x000002b2ef8b636a:   mov    r10,r8
  0x000002b2ef8b636d:   add    r10,0x18
  0x000002b2ef8b6371:   cmp    r10,QWORD PTR [r15+0x130]
  0x000002b2ef8b6378:   jae    0x000002b2ef8b6493
  0x000002b2ef8b637e:   mov    QWORD PTR [r15+0x120],r10
  0x000002b2ef8b6385:   prefetchw BYTE PTR [r10+0xc0]
  0x000002b2ef8b638d:   mov    QWORD PTR [r8],0x1
  0x000002b2ef8b6394:   prefetchw BYTE PTR [r10+0x100]
  0x000002b2ef8b639c:   mov    DWORD PTR [r8+0x8],0x2115    ;   {metadata('java/lang/Object'[])}
  0x000002b2ef8b63a4:   prefetchw BYTE PTR [r10+0x140]
  0x000002b2ef8b63ac:   mov    DWORD PTR [r8+0xc],0x2
  0x000002b2ef8b63b4:   prefetchw BYTE PTR [r10+0x180]
  0x000002b2ef8b63bc:   mov    r11d,DWORD PTR [rbp+0x10]
  0x000002b2ef8b63c0:   mov    r10d,DWORD PTR [rbp+0xc]
  0x000002b2ef8b63c4:   mov    DWORD PTR [r8+0x10],r10d
  0x000002b2ef8b63c8:   mov    DWORD PTR [r8+0x14],r11d
  0x000002b2ef8b63cc:   mov    r9d,0x1f
  0x000002b2ef8b63d2:   xor    ecx,ecx
  0x000002b2ef8b63d4:   jmp    0x000002b2ef8b63ea
  0x000002b2ef8b63d6:   data32 nop WORD PTR [rax+rax*1+0x0]
  0x000002b2ef8b63e0:   mov    r9d,r11d
  0x000002b2ef8b63e3:   shl    r9d,0x5
  0x000002b2ef8b63e7:   sub    r9d,r11d
  0x000002b2ef8b63ea:   mov    ebp,DWORD PTR [r8+rcx*4+0x10]
  0x000002b2ef8b63ef:   mov    r11d,DWORD PTR [rbp+0x8]     ; implicit exception: dispatches to 0x000002b2ef8b64b0
  0x000002b2ef8b63f3:   movabs r10,0x800000000
  0x000002b2ef8b63fd:   lea    r10,[r10+r11*8]
  0x000002b2ef8b6401:   mov    r10,QWORD PTR [r10+0x1f8]
  0x000002b2ef8b6408:   movabs r11,0x8000107d8              ;   {metadata({method} {0x00000008000107e0} 'hashCode' '()I' in 'java/lang/Object')}
  0x000002b2ef8b6412:   cmp    r10,r11
  0x000002b2ef8b6415:   jne    0x000002b2ef8b645b
  0x000002b2ef8b6417:   mov    r10,QWORD PTR [rbp+0x0]
  0x000002b2ef8b641b:   mov    r11,r10
  0x000002b2ef8b641e:   and    r11,0x7
  0x000002b2ef8b6422:   cmp    r11,0x1
  0x000002b2ef8b6426:   jne    0x000002b2ef8b645b
  0x000002b2ef8b6428:   shr    r10,0x8
  0x000002b2ef8b642c:   mov    r11d,r10d
  0x000002b2ef8b642f:   and    r11d,0x7fffffff
  0x000002b2ef8b6436:   test   r11d,r11d
  0x000002b2ef8b6439:   je     0x000002b2ef8b645b
  0x000002b2ef8b643b:   add    r11d,r9d
  0x000002b2ef8b643e:   inc    ecx
  0x000002b2ef8b6440:   cmp    ecx,0x2
  0x000002b2ef8b6443:   jl     0x000002b2ef8b63e0
  0x000002b2ef8b6445:   mov    eax,DWORD PTR [rsp]
  0x000002b2ef8b6448:   add    eax,r11d
  0x000002b2ef8b644b:   add    rsp,0x30
  0x000002b2ef8b644f:   pop    rbp
  0x000002b2ef8b6450:   mov    r10,QWORD PTR [r15+0x110]
  0x000002b2ef8b6457:   test   DWORD PTR [r10],eax          ;   {poll_return}
  0x000002b2ef8b645a:   ret    
```
Without to explain every line or instructions, we can see the first part which allocates in TLAB the varargs array (around the `prefetchw` instruction) and if you want more information on how to interpret it, read the post about [TLAB allocation](https://shipilev.net/jvm/anatomy-quarks/4-tlab-allocation/). Then some instructions to perform the hash code computation iterating over the array. it *seems* everything is fine regardng the inline as we don't see any `call` instructions

Now, I run with Graal Compiler to see the difference:
```
  0x000002590a605640:   nop    DWORD PTR [rax+rax*1+0x0]                                    
  0x000002590a605645:   mov    eax,DWORD PTR [rdx+0xc]                                      
  0x000002590a605648:   test   eax,eax                                                      
  0x000002590a60564a:   je     0x000002590a6056b9                                           
  0x000002590a605650:   mov    eax,DWORD PTR [rax*1+0xc]                                    
  0x000002590a605657:   mov    r10d,DWORD PTR [rdx+0x10]                                    
  0x000002590a60565b:   test   r10d,r10d                                                    
  0x000002590a60565e:   je     0x000002590a6056c0                                           
  0x000002590a605664:   vmovsd xmm0,QWORD PTR [r10*1+0x10]                                  
  0x000002590a60566e:   vmovq  r10,xmm0                                                     
  0x000002590a605673:   movabs r11,0x7ff8000000000000                                       
  0x000002590a60567d:   vucomisd xmm0,xmm0                                                  
  0x000002590a605681:   mov    r9,r11                                                       
  0x000002590a605684:   cmove  r9,r10                                                       
  0x000002590a605688:   cmovp  r9,r11                                                       
  0x000002590a60568c:   mov    r10,r9                                                       
  0x000002590a60568f:   shr    r10,0x20                                                     
  0x000002590a605693:   xor    r9,r10                                                       
  0x000002590a605696:   mov    r10d,r9d                                                     
  0x000002590a605699:   lea    eax,[rax+0x1f]                                               
  0x000002590a60569c:   mov    r11d,eax                                                     
  0x000002590a60569f:   shl    r11d,0x5                                                     
  0x000002590a6056a3:   sub    r11d,eax                                                     
  0x000002590a6056a6:   add    r10d,r11d                                                    
  0x000002590a6056a9:   add    r8d,r10d                                                     
  0x000002590a6056ac:   mov    eax,r8d                                                      
  0x000002590a6056af:   mov    rcx,QWORD PTR [r15+0x110]                                    
  0x000002590a6056b6:   test   DWORD PTR [rcx],eax          ;   {poll_return}               
  0x000002590a6056b8:   ret                                                                 
```
Here the code is more compact, no allocation, no loop.

Let's see with my `rawObjectsHash` method which manually unroll and inline the computation of the hash of the 2 fields.
Inlining decision:
```
  @ 1   ObjectsHashJIT::rawObjectsHash (34 bytes)   inline (hot)
    @ 11   java.lang.Integer::hashCode (8 bytes)   inline (hot)
      @ 4   java.lang.Integer::hashCode (2 bytes)   inline (hot)
    @ 26   java.lang.Double::hashCode (8 bytes)   inline (hot)
      @ 4   java.lang.Double::hashCode (13 bytes)   inline (hot)
        @ 1   java.lang.Double::doubleToLongBits (16 bytes)   (intrinsic)
``` 
Interesting. Compared to the `jdkObjectsHash` method, we are able to devirtualize and inline down to Integer::hashCode an Double::hashCode, where the first experiment shows us `@ 43   java.lang.Object::hashCode (0 bytes)   (intrinsic, virtual)`!

Assembly output:
```
  0x000001d2abe5ded0:   mov    DWORD PTR [rsp-0x7000],eax
  0x000001d2abe5ded7:   push   rbp
  0x000001d2abe5ded8:   sub    rsp,0x10
  0x000001d2abe5dedc:   mov    r10d,DWORD PTR [rdx+0xc]
  0x000001d2abe5dee0:   mov    r11d,DWORD PTR [r10+0xc]     ; implicit exception: dispatches to 0x000001d2abe5df3b
  0x000001d2abe5dee4:   mov    r10d,DWORD PTR [rdx+0x10]
  0x000001d2abe5dee8:   vmovsd xmm0,QWORD PTR [r10+0x10]    ; implicit exception: dispatches to 0x000001d2abe5df64
  0x000001d2abe5deee:   vucomisd xmm0,xmm0
  0x000001d2abe5def2:   jp     0x000001d2abe5def6
  0x000001d2abe5def4:   je     0x000001d2abe5df27
  0x000001d2abe5def6:   mov    r10d,0x7ff80000
  0x000001d2abe5defc:   mov    ecx,r11d
  0x000001d2abe5deff:   shl    ecx,0x5
  0x000001d2abe5df02:   sub    ecx,r11d
  0x000001d2abe5df05:   add    ecx,r10d
  0x000001d2abe5df08:   add    ecx,r11d
  0x000001d2abe5df0b:   add    r8d,ecx
  0x000001d2abe5df0e:   mov    eax,r8d
  0x000001d2abe5df11:   add    eax,0x400
  0x000001d2abe5df17:   add    rsp,0x10
  0x000001d2abe5df1b:   pop    rbp
  0x000001d2abe5df1c:   mov    r10,QWORD PTR [r15+0x110]
  0x000001d2abe5df23:   test   DWORD PTR [r10],eax          ;   {poll_return}
  0x000001d2abe5df26:   ret    
```
We have now something that is very similar to the Graal output. I don't know isf the performance is similar or not, this is not the point of this post. At least this is the code I expected to have when calling `Objects.hash`.

Let's focus on the allocation of varargs array. 

analysis of JITed code

try on JDK8, JDK 11 and JDK14-15 ref C Gracie improvements?

deeper analysis with fastdebug build (shipilev) for PrintEliminateAllocations VM options
 => works for rawVarArgs version

## References
 - [Abstractions Without Regret with GraalVM by Thomas Wuerthinger @ Devoxx BE 2019](https://youtu.be/noX2uHA2Udo?t=1532)
 - [Escape Analysis (Hotspot Wiki)](https://wiki.openjdk.java.net/display/HotSpot/EscapeAnalysis)
 - [Anatomy Quarks #4: TLAB allocation](https://shipilev.net/jvm/anatomy-quarks/4-tlab-allocation/)
 - [Anatomy Quarks #18: Scalar replacement](https://shipilev.net/jvm/anatomy-quarks/18-scalar-replacement/)
 - [Stack allocation prototype for C2 by Charlie Gracie](https://mail.openjdk.java.net/pipermail/hotspot-compiler-dev/2020-January/036835.html)
 - [patch for stack allocation for C2](https://mail.openjdk.java.net/pipermail/hotspot-compiler-dev/2020-June/038779.html)
 - [Stack Allocation JEP proposal](https://github.com/microsoft/openjdk-proposals/blob/master/stack_allocation/Stack_Allocation_JEP.md)
 - https://twitter.com/HansWurst315/status/1246003165478166528
 - [Compiler Control](https://openjdk.java.net/jeps/165)
