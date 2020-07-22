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
		  @ 43   java.lang.Integer::hashCode (8 bytes)   inline (hot)                 
		  @ 43   java.lang.Double::hashCode (8 bytes)   inline (hot)                  
		   \-> TypeProfile (4427/8854 counts) = java/lang/Double                      
		   \-> TypeProfile (4427/8854 counts) = java/lang/Integer                     
			@ 4   java.lang.Double::hashCode (13 bytes)   inline (hot)                
			  @ 1   java.lang.Double::doubleToLongBits (16 bytes)   (intrinsic)       
			@ 4   java.lang.Integer::hashCode (2 bytes)   inline (hot)                
```
Then the assembly output (yes, with the [intel syntax](https://jpbempel.github.io/2012/10/16/how-to-print-disassembly-from-JIT-code.html)):
```
  0x000002774d3c9bd0:   mov    DWORD PTR [rsp-0x7000],eax
  0x000002774d3c9bd7:   push   rbp
  0x000002774d3c9bd8:   sub    rsp,0x30
  0x000002774d3c9bdc:   mov    QWORD PTR [rsp],rdx
  0x000002774d3c9be0:   mov    ebp,r8d
  0x000002774d3c9be3:   mov    rax,QWORD PTR [r15+0x120]
  0x000002774d3c9bea:   mov    r10,rax
  0x000002774d3c9bed:   add    r10,0x18
  0x000002774d3c9bf1:   cmp    r10,QWORD PTR [r15+0x130]
  0x000002774d3c9bf8:   jae    0x000002774d3c9ca8
  0x000002774d3c9bfe:   mov    QWORD PTR [r15+0x120],r10
  0x000002774d3c9c05:   prefetchw BYTE PTR [r10+0xc0]
  0x000002774d3c9c0d:   mov    QWORD PTR [rax],0x1
  0x000002774d3c9c14:   prefetchw BYTE PTR [r10+0x100]
  0x000002774d3c9c1c:   mov    DWORD PTR [rax+0x8],0x2115   ;   {metadata('java/lang/Object'[])}
  0x000002774d3c9c23:   prefetchw BYTE PTR [r10+0x140]
  0x000002774d3c9c2b:   mov    DWORD PTR [rax+0xc],0x2
  0x000002774d3c9c32:   prefetchw BYTE PTR [r10+0x180]
  0x000002774d3c9c3a:   mov    r10,QWORD PTR [rsp]
  0x000002774d3c9c3e:   mov    r11d,DWORD PTR [r10+0x10]
  0x000002774d3c9c42:   mov    r8d,DWORD PTR [r10+0xc]
  0x000002774d3c9c46:   mov    DWORD PTR [rax+0x10],r8d
  0x000002774d3c9c4a:   mov    DWORD PTR [rax+0x14],r11d
  0x000002774d3c9c4e:   test   r8d,r8d
  0x000002774d3c9c51:   je     0x000002774d3c9cc5
  0x000002774d3c9c53:   mov    r9d,DWORD PTR [r8+0xc]
  0x000002774d3c9c57:   mov    r10d,r9d
  0x000002774d3c9c5a:   shl    r10d,0x5
  0x000002774d3c9c5e:   sub    r10d,r9d
  0x000002774d3c9c61:   test   r11d,r11d
  0x000002774d3c9c64:   je     0x000002774d3c9cd2
  0x000002774d3c9c66:   vmovsd xmm0,QWORD PTR [r11+0x10]
  0x000002774d3c9c6c:   vucomisd xmm0,xmm0
  0x000002774d3c9c70:   jp     0x000002774d3c9c74
  0x000002774d3c9c72:   je     0x000002774d3c9c94
  0x000002774d3c9c74:   mov    eax,0x7ff80000
  0x000002774d3c9c79:   add    eax,r10d
  0x000002774d3c9c7c:   add    eax,ebp
  0x000002774d3c9c7e:   add    eax,0x3c1
  0x000002774d3c9c84:   add    rsp,0x30
  0x000002774d3c9c88:   pop    rbp
  0x000002774d3c9c89:   mov    r10,QWORD PTR [r15+0x110]
  0x000002774d3c9c90:   test   DWORD PTR [r10],eax          ;   {poll_return}
  0x000002774d3c9c93:   ret
```
Without to explain every line or instructions, we can see the first part which allocates in TLAB the varargs array (around the `prefetchw` instruction) and if you want more information on how to interpret it, read the post about [TLAB allocation](https://shipilev.net/jvm/anatomy-quarks/4-tlab-allocation/). Then some instructions to perform the hash code computation, loop is unrolled. 

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
Here the code is more compact, no allocation.

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
		@ 43   java.lang.Integer::hashCode (8 bytes)   inline (hot)                
		@ 43   java.lang.Double::hashCode (8 bytes)   inline (hot)                 
		 \-> TypeProfile (4467/8934 counts) = java/lang/Double                     
		 \-> TypeProfile (4467/8934 counts) = java/lang/Integer                    
		  @ 4   java.lang.Double::hashCode (13 bytes)   inline (hot)               
			@ 1   java.lang.Double::doubleToLongBits (16 bytes)   (intrinsic)      
		  @ 4   java.lang.Integer::hashCode (2 bytes)   inline (hot)
```

Assembly output:
``` 
  0x0000016cc78d9bf0:   mov    DWORD PTR [rsp-0x7000],eax                                                                  
  0x0000016cc78d9bf7:   push   rbp                                                                                         
  0x0000016cc78d9bf8:   sub    rsp,0x30                                                                                    
  0x0000016cc78d9bfc:   mov    edi,r8d                                                                                     
  0x0000016cc78d9bff:   mov    esi,DWORD PTR [rdx+0x14]                                                                    
  0x0000016cc78d9c02:   mov    ebx,DWORD PTR [rsi+0xc]      ; implicit exception: dispatches to 0x0000016cc78d9d6c         
  0x0000016cc78d9c05:   test   ebx,ebx                                                                                     
  0x0000016cc78d9c07:   ja     0x0000016cc78d9c20                                                                          
  0x0000016cc78d9c09:   mov    eax,0x1                                                                                     
  0x0000016cc78d9c0e:   add    eax,edi                                                                                     
  0x0000016cc78d9c10:   add    rsp,0x30                                                                                    
  0x0000016cc78d9c14:   pop    rbp                                                                                         
  0x0000016cc78d9c15:   mov    r10,QWORD PTR [r15+0x110]                                                                   
  0x0000016cc78d9c1c:   test   DWORD PTR [r10],eax          ;   {poll_return}                                              
  0x0000016cc78d9c1f:   ret                                                                                                
  0x0000016cc78d9c20:   mov    r11d,ebx                                                                                    
  0x0000016cc78d9c23:   dec    r11d                                                                                        
  0x0000016cc78d9c26:   cmp    r11d,ebx                                                                                    
  0x0000016cc78d9c29:   jae    0x0000016cc78d9cf8                                                                          
  0x0000016cc78d9c2f:   mov    rbp,rsi                                                                                     
  0x0000016cc78d9c32:   xor    ecx,ecx                                                                                     
  0x0000016cc78d9c34:   mov    r9d,0x1f                                                                                    
  0x0000016cc78d9c3a:   mov    r10d,0x3e8                                                                                  
  0x0000016cc78d9c40:   jmp    0x0000016cc78d9c67                                                                          
  0x0000016cc78d9c42:   mov    eax,DWORD PTR [rdx+0xc]                                                                     
  0x0000016cc78d9c45:   add    eax,r9d                                                                                     
  0x0000016cc78d9c48:   mov    r9d,eax                                                                                     
  0x0000016cc78d9c4b:   shl    r9d,0x5                                                                                     
  0x0000016cc78d9c4f:   sub    r9d,eax                                                                                     
  0x0000016cc78d9c52:   inc    ecx                                                                                         
  0x0000016cc78d9c54:   cmp    ecx,r8d                                                                                     
  0x0000016cc78d9c57:   jl     0x0000016cc78d9c77                                                                          
  0x0000016cc78d9c59:   mov    r11,QWORD PTR [r15+0x110]    ; ImmutableOopMap {rsi=NarrowOop rbp=Oop }                     
                                                            ;*goto {reexecute=1 rethrow=0 return_oop=0}                    
                                                            ; - (reexecute) java.util.Arrays::hashCode@51 (line 4497)      
                                                            ; - ObjectsHashJIT::noVarArgsObjectsHash@4 (line 29)           
                                                            ; - ObjectsHashJIT::bench@1 (line 73)                          
  0x0000016cc78d9c60:   test   DWORD PTR [r11],eax          ;   {poll}                                                     
  0x0000016cc78d9c63:   cmp    ecx,ebx                                                                                     
  0x0000016cc78d9c65:   jge    0x0000016cc78d9c0e                                                                          
  0x0000016cc78d9c67:   mov    r8d,ebx                                                                                     
  0x0000016cc78d9c6a:   sub    r8d,ecx                                                                                     
  0x0000016cc78d9c6d:   cmp    r8d,r10d                                                                                    
  0x0000016cc78d9c70:   cmovg  r8d,r10d                                                                                    
  0x0000016cc78d9c74:   add    r8d,ecx                                                                                     
  0x0000016cc78d9c77:   mov    r11d,DWORD PTR [rsi+rcx*4+0x10]                                                             
  0x0000016cc78d9c7c:   mov    eax,DWORD PTR [r11+0x8]      ; implicit exception: dispatches to 0x0000016cc78d9d2c         
  0x0000016cc78d9c80:   mov    rdx,r11                                                                                     
  0x0000016cc78d9c83:   cmp    eax,0x5d33c                  ;   {metadata('java/lang/Integer')}                            
  0x0000016cc78d9c89:   je     0x0000016cc78d9c42                                                                          
  0x0000016cc78d9c8b:   cmp    eax,0x697fa                  ;   {metadata('java/lang/Double')}                             
  0x0000016cc78d9c91:   jne    0x0000016cc78d9cba                                                                          
  0x0000016cc78d9c93:   vmovsd xmm0,QWORD PTR [rdx+0x10]                                                                   
  0x0000016cc78d9c98:   vucomisd xmm0,xmm0                                                                                 
  0x0000016cc78d9c9c:   jp     0x0000016cc78d9ca0                                                                          
  0x0000016cc78d9c9e:   je     0x0000016cc78d9ca7                                                                          
  0x0000016cc78d9ca0:   mov    eax,0x7ff80000                                                                              
  0x0000016cc78d9ca5:   jmp    0x0000016cc78d9c45                                                                          
  0x0000016cc78d9ca7:   vmovq  r11,xmm0                                                                                    
  0x0000016cc78d9cac:   mov    rdx,r11                                                                                     
  0x0000016cc78d9caf:   shr    rdx,0x20                                                                                    
  0x0000016cc78d9cb3:   xor    rdx,r11                                                                                     
  0x0000016cc78d9cb6:   mov    eax,edx                                                                                     
  0x0000016cc78d9cb8:   jmp    0x0000016cc78d9c45                                                                          
``` 
We don't have the allocation, but still generated code is very convoluted.

### rawVarArgsObjectHash
With the method `rawVarArgsObjectHash` we keep the varargs array but we don't iterate on it, unrolling manually the loop with distinct calls.

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
Assembly output:
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

Can we diagnostic the EA decision? Looking at [VM Option Explorer](https://chriswhocodes.com/hotspot_options_jdk14.html), you can found 2 options: `-XX:+PrintEscapeAnalysis` and `-XX:+PrintEliminateAllocations` but both are `notproduct` meaning you cannot use it with a release build of OpenJDK and requires a [fastdebug one](https://builds.shipilev.net/openjdk-jdk14/). We are also using `-XX:+Verbose` to have more information about eliminated allocations.

EA report:
``` 
======== Connection graph for  ObjectsHashJIT::bench                                                                                                                    
JavaObject NoEscape(NoEscape) [ 266F 263F [ 63 ]]   51  AllocateArray   ===  5  6  7  8  1 ( 49  34  39  33  1  11  1  10 ) [[ 52  53  54  61  62  63 ]]  rawptr:NotNull
 ( int:>=0, java/lang/Object:NotNull *, bool, int ) ObjectsHashJIT::rawVarArgsObjectHash_noargs @ bci:1 ObjectsHashJIT::bench @ bci:1 !jvms: ObjectsHashJIT::rawVarArgsO
bjectHash_noargs @ bci:1 ObjectsHashJIT::bench @ bci:1                                                                                                                  
LocalVar [ 51P [ 266b 263b ]]   63      Proj    ===  51  [[ 64  266  263 ]] #5 !jvms: ObjectsHashJIT::rawVarArgsObjectHash_noargs @ bci:1 ObjectsHashJIT::bench @ bci:1 
                                                                                                                                                                        
Scalar  51      AllocateArray   ===  5  6  7  8  1 ( 49  34  39  33  1  11  1  10 ) [[ 52  53  54  61  62  63 ]]  rawptr:NotNull ( int:>=0, java/lang/Object:NotNull *, 
bool, int ) ObjectsHashJIT::rawVarArgsObjectHash_noargs @ bci:1 ObjectsHashJIT::bench @ bci:1 !jvms: ObjectsHashJIT::rawVarArgsObjectHash_noargs @ bci:1 ObjectsHashJIT:
:bench @ bci:1                                                                                                                                                          
++++ Eliminated: 51 AllocateArray                                                                                                                                       
``` 
The allocation of an array was indeed eliminated (`Eliminated: 51 AllocateArray`)

### iterVarArgsObjectsHash
we have kept the varargs but reintroduced the iteration over the varargs array in a form very similar to `Arrays.hashCode`.

Inlining decision:
```
	@ 1   ObjectsHashJIT::iterVarArgsObjectsHash_noargs (22 bytes)   inline (hot)
	  @ 18   ObjectsHashJIT::iterVarArgsObjectsHash (5 bytes)   inline (hot)
		@ 1   java.util.Arrays::hashCode (56 bytes)   inline (hot)
		  @ 43   java.lang.Integer::hashCode (8 bytes)   inline (hot)
		  @ 43   java.lang.Double::hashCode (8 bytes)   inline (hot)
		   \-> TypeProfile (4290/8580 counts) = java/lang/Double
		   \-> TypeProfile (4290/8580 counts) = java/lang/Integer
			@ 4   java.lang.Double::hashCode (13 bytes)   inline (hot)
			  @ 1   java.lang.Double::doubleToLongBits (16 bytes)   (intrinsic)
			@ 4   java.lang.Integer::hashCode (2 bytes)   inline (hot)
```
Assembly output:
``` 
  0x000001d2ef949bd0:   mov    DWORD PTR [rsp-0x7000],eax
  0x000001d2ef949bd7:   push   rbp
  0x000001d2ef949bd8:   sub    rsp,0x30
  0x000001d2ef949bdc:   mov    QWORD PTR [rsp],rdx
  0x000001d2ef949be0:   mov    ebp,r8d
  0x000001d2ef949be3:   mov    rax,QWORD PTR [r15+0x120]
  0x000001d2ef949bea:   mov    r10,rax
  0x000001d2ef949bed:   add    r10,0x18
  0x000001d2ef949bf1:   cmp    r10,QWORD PTR [r15+0x130]
  0x000001d2ef949bf8:   jae    0x000001d2ef949ca8
  0x000001d2ef949bfe:   mov    QWORD PTR [r15+0x120],r10
  0x000001d2ef949c05:   prefetchw BYTE PTR [r10+0xc0]
  0x000001d2ef949c0d:   mov    QWORD PTR [rax],0x1
  0x000001d2ef949c14:   prefetchw BYTE PTR [r10+0x100]
  0x000001d2ef949c1c:   mov    DWORD PTR [rax+0x8],0x2115   ;   {metadata('java/lang/Object'[])}
  0x000001d2ef949c23:   prefetchw BYTE PTR [r10+0x140]
  0x000001d2ef949c2b:   mov    DWORD PTR [rax+0xc],0x2
  0x000001d2ef949c32:   prefetchw BYTE PTR [r10+0x180]
  0x000001d2ef949c3a:   mov    r10,QWORD PTR [rsp]
  0x000001d2ef949c3e:   mov    r11d,DWORD PTR [r10+0x10]
  0x000001d2ef949c42:   mov    r8d,DWORD PTR [r10+0xc]
  0x000001d2ef949c46:   mov    DWORD PTR [rax+0x10],r8d
  0x000001d2ef949c4a:   mov    DWORD PTR [rax+0x14],r11d
  0x000001d2ef949c4e:   test   r8d,r8d
  0x000001d2ef949c51:   je     0x000001d2ef949cc5
  0x000001d2ef949c53:   mov    r9d,DWORD PTR [r8+0xc]
  0x000001d2ef949c57:   mov    r10d,r9d
  0x000001d2ef949c5a:   shl    r10d,0x5
  0x000001d2ef949c5e:   sub    r10d,r9d
  0x000001d2ef949c61:   test   r11d,r11d
  0x000001d2ef949c64:   je     0x000001d2ef949cd2
  0x000001d2ef949c66:   vmovsd xmm0,QWORD PTR [r11+0x10]
  0x000001d2ef949c6c:   vucomisd xmm0,xmm0
  0x000001d2ef949c70:   jp     0x000001d2ef949c74
  0x000001d2ef949c72:   je     0x000001d2ef949c94
  0x000001d2ef949c74:   mov    eax,0x7ff80000
  0x000001d2ef949c79:   add    eax,r10d
  0x000001d2ef949c7c:   add    eax,ebp
  0x000001d2ef949c7e:   add    eax,0x3c1
  0x000001d2ef949c84:   add    rsp,0x30
  0x000001d2ef949c88:   pop    rbp
  0x000001d2ef949c89:   mov    r10,QWORD PTR [r15+0x110]
  0x000001d2ef949c90:   test   DWORD PTR [r10],eax          ;   {poll_return}
  0x000001d2ef949c93:   ret
``` 
Ouch, now we have lost the elimination of the varargs array allocation just by iterating on it!

Confirmed with EA report:
```
======== Connection graph for  ObjectsHashJIT::bench                                                                                                                    
JavaObject NoEscape(NoEscape) NSR [ 390F 393F 213F 214F [ 63 68 ]]   51 AllocateArray   ===  5  6  7  8  1 ( 49  34  39  33  1  11  1  10 ) [[ 52  53  54  61  62  63 ]]
  rawptr:NotNull ( int:>=0, java/lang/Object:NotNull *, bool, int ) ObjectsHashJIT::iterVarArgsObjectsHash_noargs @ bci:1 ObjectsHashJIT::bench @ bci:1  Type:{0:control
, 1:abIO, 2:memory, 3:rawptr:BotPTR, 4:return_address, 5:rawptr:NotNull} !jvms: ObjectsHashJIT::iterVarArgsObjectsHash_noargs @ bci:1 ObjectsHashJIT::bench @ bci:1     
LocalVar NoEscape(NoEscape) [ 51P [ 68 390b 393b ]]   63        Proj    ===  51  [[ 64  68  390  393 ]] #5  Type:rawptr:NotNull !jvms: ObjectsHashJIT::iterVarArgsObject
sHash_noargs @ bci:1 ObjectsHashJIT::bench @ bci:1                                                                                                                      
LocalVar NoEscape(NoEscape) [ 63 51P [ 213b 214b ]]   68        CheckCastPP     ===  65  63  [[ 406  261  157  144  214  213  231  170  204  204  214 ]]   Type:narrowoo
p: java/lang/Object:BotPTR *[int:2]:NotNull:exact * !jvms: ObjectsHashJIT::iterVarArgsObjectsHash_noargs @ bci:1 ObjectsHashJIT::bench @ bci:1                          
                                                                                                                                                                        
=== No allocations eliminated for  ObjectsHashJIT::bench since there are no scalar replaceable candidates ===                                                           
```
Despite the fact that all objects are `NoEscape`!

## Conclusion
Escape Analysis seems to fail for no ovious reason to elimnate the varargs array allocation which prevents to use freely `Objects.hashCode` method. Is it something that could be fixed easily?

## References
 - [Abstractions Without Regret with GraalVM by Thomas Wuerthinger @ Devoxx BE 2019](https://youtu.be/noX2uHA2Udo?t=1532)
 - [Escape Analysis (Hotspot Wiki)](https://wiki.openjdk.java.net/display/HotSpot/EscapeAnalysis)
 - [Anatomy Quarks #4: TLAB allocation](https://shipilev.net/jvm/anatomy-quarks/4-tlab-allocation/)
 - [Anatomy Quarks #18: Scalar replacement](https://shipilev.net/jvm/anatomy-quarks/18-scalar-replacement/)
 - [Stack Allocation JEP proposal](https://github.com/microsoft/openjdk-proposals/blob/master/stack_allocation/Stack_Allocation_JEP.md)
 - https://twitter.com/HansWurst315/status/1246003165478166528
 - [Compiler Control](https://openjdk.java.net/jeps/165)
