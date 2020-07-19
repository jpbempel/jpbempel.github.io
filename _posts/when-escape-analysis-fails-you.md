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
### jdkObjectsHash C2

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
  0x00000296bf5481d0:   mov    DWORD PTR [rsp-0x7000],eax
  0x00000296bf5481d7:   push   rbp
  0x00000296bf5481d8:   sub    rsp,0x30
  0x00000296bf5481dc:   mov    rbp,rdx
  0x00000296bf5481df:   mov    DWORD PTR [rsp+0x14],r8d
  0x00000296bf5481e4:   mov    r9,QWORD PTR [r15+0x120]
  0x00000296bf5481eb:   mov    r10,r9
  0x00000296bf5481ee:   add    r10,0x18
  0x00000296bf5481f2:   cmp    r10,QWORD PTR [r15+0x130]
  0x00000296bf5481f9:   jae    0x00000296bf548301
  0x00000296bf5481ff:   mov    QWORD PTR [r15+0x120],r10
  0x00000296bf548206:   prefetchw BYTE PTR [r10+0xc0]
  0x00000296bf54820e:   mov    QWORD PTR [r9],0x1
  0x00000296bf548215:   prefetchw BYTE PTR [r10+0x100]
  0x00000296bf54821d:   mov    DWORD PTR [r9+0x8],0x2115    ;   {metadata('java/lang/Object'[])}
  0x00000296bf548225:   prefetchw BYTE PTR [r10+0x140]
  0x00000296bf54822d:   mov    DWORD PTR [r9+0xc],0x2
  0x00000296bf548235:   prefetchw BYTE PTR [r10+0x180]
  0x00000296bf54823d:   mov    r11d,DWORD PTR [rbp+0x10]
  0x00000296bf548241:   mov    r10d,DWORD PTR [rbp+0xc]
  0x00000296bf548245:   mov    DWORD PTR [r9+0x10],r10d
  0x00000296bf548249:   mov    DWORD PTR [r9+0x14],r11d
  0x00000296bf54824d:   mov    ebp,0x1f
  0x00000296bf548252:   xor    r8d,r8d
  0x00000296bf548255:   jmp    0x00000296bf548267
  0x00000296bf548257:   nop    WORD PTR [rax+rax*1+0x0]
  0x00000296bf548260:   mov    ebp,ecx
  0x00000296bf548262:   shl    ebp,0x5
  0x00000296bf548265:   sub    ebp,ecx
  0x00000296bf548267:   mov    r10d,DWORD PTR [r9+r8*4+0x10]
  0x00000296bf54826c:   mov    ecx,DWORD PTR [r10+0x8]      ; implicit exception: dispatches to 0x00000296bf548320
  0x00000296bf548270:   movabs r11,0x800000000
  0x00000296bf54827a:   lea    r11,[r11+rcx*8]
  0x00000296bf54827e:   mov    r11,QWORD PTR [r11+0x1f8]
  0x00000296bf548285:   movabs rcx,0x8000107d8              ;   {metadata({method} {0x00000008000107e0} 'hashCode' '()I' in 'java/lang/Object')}
  0x00000296bf54828f:   cmp    r11,rcx
  0x00000296bf548292:   jne    0x00000296bf5482d6
  0x00000296bf548294:   mov    r11,QWORD PTR [r10]
  0x00000296bf548297:   mov    rcx,r11
  0x00000296bf54829a:   and    rcx,0x7
  0x00000296bf54829e:   cmp    rcx,0x1
  0x00000296bf5482a2:   jne    0x00000296bf5482d6
  0x00000296bf5482a4:   shr    r11,0x8
  0x00000296bf5482a8:   mov    ecx,r11d
  0x00000296bf5482ab:   and    ecx,0x7fffffff
  0x00000296bf5482b1:   test   ecx,ecx
  0x00000296bf5482b3:   je     0x00000296bf5482d6
  0x00000296bf5482b5:   add    ecx,ebp
  0x00000296bf5482b7:   inc    r8d
  0x00000296bf5482ba:   cmp    r8d,0x2
  0x00000296bf5482be:   jl     0x00000296bf548260
  0x00000296bf5482c0:   mov    eax,DWORD PTR [rsp+0x14]
  0x00000296bf5482c4:   add    eax,ecx
  0x00000296bf5482c6:   add    rsp,0x30
  0x00000296bf5482ca:   pop    rbp
  0x00000296bf5482cb:   mov    r10,QWORD PTR [r15+0x110]
  0x00000296bf5482d2:   test   DWORD PTR [r10],eax          ;   {poll_return}
  0x00000296bf5482d5:   ret    
  0x00000296bf5482d6:   mov    DWORD PTR [rsp+0x8],r8d
  0x00000296bf5482db:   mov    QWORD PTR [rsp],r9
  0x00000296bf5482df:   mov    rdx,r10
  0x00000296bf5482e2:   data32 xchg ax,ax
  0x00000296bf5482e5:   movabs rax,0xffffffffffffffff
  0x00000296bf5482ef:   call   0x00000296bf50aa00           ; ImmutableOopMap {[0]=Oop }
                                                            ;*invokevirtual hashCode {reexecute=0 rethrow=0 return_oop=0}
                                                            ; - java.util.Arrays::hashCode@43 (line 4498)
                                                            ; - java.util.Objects::hash@1 (line 147)
                                                            ; - ObjectsHashJIT::jdkObjectsHash@18 (line 26)
                                                            ; - ObjectsHashJIT::bench@1 (line 65)
                                                            ;   {virtual_call}
  0x00000296bf5482f4:   mov    r9,QWORD PTR [rsp]
  0x00000296bf5482f8:   mov    r8d,DWORD PTR [rsp+0x8]
  0x00000296bf5482fd:   mov    ecx,eax
  0x00000296bf5482ff:   jmp    0x00000296bf5482b5
```
Without to explain every line or instructions, we can see the first part which allocates in TLAB the varargs array (around the `prefetchw` instruction) and if you want more information on how to interpret it, read the post about [TLAB allocation](https://shipilev.net/jvm/anatomy-quarks/4-tlab-allocation/). Then some instructions to perform the hash code computation iterating over the array. Inlining decision shows us that we have a virtual call on `Object#hashCode` which is confirmed by the assembly output, at the end we have `call` instruction and comment `{virtual_call}`.

### jdkObjectsHash Graal
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
Here the code is more compact, no allocation, no loop, no virtual call.

### rawObjectsHash
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
Compared to the `jdkObjectsHash` method, we are able to devirtualize and inline down to Integer::hashCode an Double::hashCode, where the first experiment shows us `@ 43   java.lang.Object::hashCode (0 bytes)   (intrinsic, virtual)`! Iterating on the array makes the call to `Object.hashCode` megamorphic while in this particular case I was expecting a [bimorphic call](https://shipilev.net/blog/2015/black-magic-method-dispatch/#_discussion_ii) because we have only 2 target types.

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
We have now something that is very similar to the Graal output. I don't know if the performance is similar or not, this is not the point of this post. At least this is the code I expected to have when calling `Objects.hash`.

### noVarArgsObjectsHash
Let's try to avoid the allocation of varargs array. The method `noVarArgsObjectsHash` calls `Arrays.hashCode` with pre-allocated array.
Inlining decision:
```
  @ 1   ObjectsHashJIT::noVarArgsObjectsHash (8 bytes)   inline (hot)
    @ 4   java.util.Arrays::hashCode (56 bytes)   inline (hot)
      @ 43   java.lang.Object::hashCode (0 bytes)   (intrinsic, virtual)
```
Again, we stopped the inlining at `Object#hashCode`.

Assembly output:
``` 
  0x0000017f39105370:   mov    DWORD PTR [rsp-0x7000],eax
  0x0000017f39105377:   push   rbp
  0x0000017f39105378:   sub    rsp,0x30
  0x0000017f3910537c:   mov    DWORD PTR [rsp+0x10],r8d
  0x0000017f39105381:   mov    ecx,DWORD PTR [rdx+0x14]
  0x0000017f39105384:   mov    r9d,DWORD PTR [rcx+0xc]      ; implicit exception: dispatches to 0x0000017f391054d0
  0x0000017f39105388:   test   r9d,r9d
  0x0000017f3910538b:   ja     0x0000017f39105397
  0x0000017f3910538d:   mov    eax,0x1
  0x0000017f39105392:   jmp    0x0000017f39105413
  0x0000017f39105397:   mov    r11d,r9d
  0x0000017f3910539a:   dec    r11d
  0x0000017f3910539d:   cmp    r11d,r9d
  0x0000017f391053a0:   jae    0x0000017f39105464
  0x0000017f391053a6:   xor    r8d,r8d
  0x0000017f391053a9:   mov    r11d,0x1f
  0x0000017f391053af:   jmp    0x0000017f391053bb
  0x0000017f391053b1:   mov    r11d,eax
  0x0000017f391053b4:   shl    r11d,0x5
  0x0000017f391053b8:   sub    r11d,eax
  0x0000017f391053bb:   mov    r10d,DWORD PTR [rcx+r8*4+0x10]
  0x0000017f391053c0:   mov    edi,DWORD PTR [r10+0x8]      ; implicit exception: dispatches to 0x0000017f39105494
  0x0000017f391053c4:   movabs rbx,0x800000000
  0x0000017f391053ce:   lea    rbx,[rbx+rdi*8]
  0x0000017f391053d2:   mov    rbx,QWORD PTR [rbx+0x1f8]
  0x0000017f391053d9:   movabs rdi,0x8000107d8              ;   {metadata({method} {0x00000008000107e0} 'hashCode' '()I' in 'java/lang/Object')}
  0x0000017f391053e3:   cmp    rbx,rdi
  0x0000017f391053e6:   jne    0x0000017f39105427
  0x0000017f391053e8:   mov    rbx,QWORD PTR [r10]
  0x0000017f391053eb:   mov    rdi,rbx
  0x0000017f391053ee:   and    rdi,0x7
  0x0000017f391053f2:   cmp    rdi,0x1
  0x0000017f391053f6:   jne    0x0000017f39105427
  0x0000017f391053f8:   shr    rbx,0x8
  0x0000017f391053fc:   mov    eax,ebx
  0x0000017f391053fe:   and    eax,0x7fffffff
  0x0000017f39105404:   test   eax,eax
  0x0000017f39105406:   je     0x0000017f39105427
  0x0000017f39105408:   add    eax,r11d
  0x0000017f3910540b:   inc    r8d
  0x0000017f3910540e:   cmp    r8d,r9d
  0x0000017f39105411:   jl     0x0000017f391053b1
  0x0000017f39105413:   add    eax,DWORD PTR [rsp+0x10]
  0x0000017f39105417:   add    rsp,0x30
  0x0000017f3910541b:   pop    rbp
  0x0000017f3910541c:   mov    r10,QWORD PTR [r15+0x110]
  0x0000017f39105423:   test   DWORD PTR [r10],eax          ;   {poll_return}
  0x0000017f39105426:   ret    
  0x0000017f39105427:   mov    DWORD PTR [rsp+0xc],r11d
  0x0000017f3910542c:   mov    DWORD PTR [rsp+0x8],r8d
  0x0000017f39105431:   mov    DWORD PTR [rsp+0x4],r9d
  0x0000017f39105436:   mov    rdx,r10
  0x0000017f39105439:   mov    rbp,rcx
  0x0000017f3910543c:   mov    DWORD PTR [rsp],ecx
  0x0000017f3910543f:   xchg   ax,ax
  0x0000017f39105441:   movabs rax,0xffffffffffffffff
  0x0000017f3910544b:   call   0x0000017f390caa00           ; ImmutableOopMap {rbp=Oop [0]=NarrowOop }
                                                            ;*invokevirtual hashCode {reexecute=0 rethrow=0 return_oop=0}
                                                            ; - java.util.Arrays::hashCode@43 (line 4498)
                                                            ; - ObjectsHashJIT::noVarArgsObjectsHash@4 (line 29)
                                                            ; - ObjectsHashJIT::bench@1 (line 66)
                                                            ;   {virtual_call}
  0x0000017f39105450:   mov    ecx,DWORD PTR [rsp]
  0x0000017f39105453:   mov    r9d,DWORD PTR [rsp+0x4]
  0x0000017f39105458:   mov    r8d,DWORD PTR [rsp+0x8]
  0x0000017f3910545d:   mov    r11d,DWORD PTR [rsp+0xc]
  0x0000017f39105462:   jmp    0x0000017f39105408
``` 
We don't have the allocation, but still the virtual call.

### rawVarArgsObjectHash
With the method `rawVarArgsObjectHash` we keep the varargs array but we don't iterate on it but unrolling manually the loop with distinct calls.

Inlining decision:
```
  @ 1   ObjectsHashJIT::rawVarArgsObjectHash_noargs (22 bytes)   inline (hot)
    @ 18   ObjectsHashJIT::rawVarArgsObjectsHash (32 bytes)   inline (hot)
      @ 10   java.lang.Integer::hashCode (8 bytes)   inline (hot)
        @ 4   java.lang.Integer::hashCode (2 bytes)   inline (hot)
      @ 24   java.lang.Double::hashCode (8 bytes)   inline (hot)
        @ 4   java.lang.Double::hashCode (13 bytes)   inline (hot)
          @ 1   java.lang.Double::doubleToLongBits (16 bytes)   (intrinsic)
``` 
As expected full inlining.

```
  0x0000017aafa23950:   mov    DWORD PTR [rsp-0x7000],eax
  0x0000017aafa23957:   push   rbp
  0x0000017aafa23958:   sub    rsp,0x10
  0x0000017aafa2395c:   mov    r10d,DWORD PTR [rdx+0xc]
  0x0000017aafa23960:   mov    r11d,DWORD PTR [r10+0xc]     ; implicit exception: dispatches to 0x0000017aafa239bb
  0x0000017aafa23964:   mov    r10d,DWORD PTR [rdx+0x10]
  0x0000017aafa23968:   vmovsd xmm0,QWORD PTR [r10+0x10]    ; implicit exception: dispatches to 0x0000017aafa239e4
  0x0000017aafa2396e:   vucomisd xmm0,xmm0
  0x0000017aafa23972:   jp     0x0000017aafa23976
  0x0000017aafa23974:   je     0x0000017aafa239a7
  0x0000017aafa23976:   mov    r10d,0x7ff80000
  0x0000017aafa2397c:   mov    ecx,r11d
  0x0000017aafa2397f:   shl    ecx,0x5
  0x0000017aafa23982:   sub    ecx,r11d
  0x0000017aafa23985:   add    ecx,r10d
  0x0000017aafa23988:   add    ecx,r11d
  0x0000017aafa2398b:   add    r8d,ecx
  0x0000017aafa2398e:   mov    eax,r8d
  0x0000017aafa23991:   add    eax,0x400
  0x0000017aafa23997:   add    rsp,0x10
  0x0000017aafa2399b:   pop    rbp
  0x0000017aafa2399c:   mov    r10,QWORD PTR [r15+0x110]
  0x0000017aafa239a3:   test   DWORD PTR [r10],eax          ;   {poll_return}
  0x0000017aafa239a6:   ret    
```
No varargs allocation! We have something similar to `rawObjectsHash` but using the varargs notation.

Can we diagnostic the EA decision? Looking at [VM Option Explorer](https://chriswhocodes.com/hotspot_options_jdk14.html), you can found 2 options: `-XX:+PrintEscapeAnalysis` and `-XX:+PrintEliminateAllocations` but both are `notproduct` meaning you cannot use it with a release build of OpenJDK and requires a [fastdebug one](https://builds.shipilev.net/openjdk-jdk14/).

EA report:
``` 
======== Connection graph for  ObjectsHashJIT::bench
JavaObject NoEscape(NoEscape) [ 266F 263F [ 63 ]]   51	AllocateArray	===  5  6  7  8  1 ( 49  34  39  33  1  11  1  10 ) [[ 52  53  54  61  62  63 ]]  rawptr:NotNull ( int:>=0, java/lang/Object:NotNull *, bool, int ) ObjectsHashJIT::rawVarArgsObjectHash_noargs @ bci:1 ObjectsHashJIT::bench @ bci:1  Type:{0:control, 1:abIO, 2:memory, 3:rawptr:BotPTR, 4:return_address, 5:rawptr:NotNull} !jvms: ObjectsHashJIT::rawVarArgsObjectHash_noargs @ bci:1 ObjectsHashJIT::bench @ bci:1
LocalVar NoEscape(NoEscape) [ 51P [ 266b 263b ]]   63	Proj	===  51  [[ 64  266  263 ]] #5  Type:rawptr:NotNull !jvms: ObjectsHashJIT::rawVarArgsObjectHash_noargs @ bci:1 ObjectsHashJIT::bench @ bci:1

Scalar  51	AllocateArray	===  5  6  7  8  1 ( 49  34  39  33  1  11  1  10 ) [[ 52  53  54  61  62  63 ]]  rawptr:NotNull ( int:>=0, java/lang/Object:NotNull *, bool, int ) ObjectsHashJIT::rawVarArgsObjectHash_noargs @ bci:1 ObjectsHashJIT::bench @ bci:1  Type:{0:control, 1:abIO, 2:memory, 3:rawptr:BotPTR, 4:return_address, 5:rawptr:NotNull} !jvms: ObjectsHashJIT::rawVarArgsObjectHash_noargs @ bci:1 ObjectsHashJIT::bench @ bci:1
++++ Eliminated: 51 AllocateArray

``` 
The allocation of an array was indeed eliminated (`Eliminated: 51 AllocateArray`)

### iterVarArgsObjectsHash



## References
 - [Abstractions Without Regret with GraalVM by Thomas Wuerthinger @ Devoxx BE 2019](https://youtu.be/noX2uHA2Udo?t=1532)
 - [Escape Analysis (Hotspot Wiki)](https://wiki.openjdk.java.net/display/HotSpot/EscapeAnalysis)
 - [Anatomy Quarks #4: TLAB allocation](https://shipilev.net/jvm/anatomy-quarks/4-tlab-allocation/)
 - [Anatomy Quarks #18: Scalar replacement](https://shipilev.net/jvm/anatomy-quarks/18-scalar-replacement/)
 - [Black magic method dispath](https://shipilev.net/blog/2015/black-magic-method-dispatch/)
 - [Stack allocation prototype for C2 by Charlie Gracie](https://mail.openjdk.java.net/pipermail/hotspot-compiler-dev/2020-January/036835.html)
 - [patch for stack allocation for C2](https://mail.openjdk.java.net/pipermail/hotspot-compiler-dev/2020-June/038779.html)
 - [Stack Allocation JEP proposal](https://github.com/microsoft/openjdk-proposals/blob/master/stack_allocation/Stack_Allocation_JEP.md)
 - https://twitter.com/HansWurst315/status/1246003165478166528
 - [Compiler Control](https://openjdk.java.net/jeps/165)
