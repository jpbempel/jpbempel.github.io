---
layout: default
title: "Chasing a Metaspace leak"
date: 2023-12-?
---
# Chasing a Metaspace leak

## Introduction

I have recently chased a bug into [OpenJDK](https://github.com/openjdk/jdk) that was here since at least JDK 8, here the story of my journey to identify and fix it.

## Context

At Datadog, I am working on Dynamic Instrumentation product. As the name suggest instrument any part of the code dynamically for inserting code that would capture execution context and state (stack traces, fields, arguments local variables), generate logs, metrics, span or span tags.
On the JVM we are, of course, relying heavily on the instrumentation API for doing this, which gives us the advantage and guartanee that it is a supported API that is used since many years (introduced since JDK 1.5).
We have in place, along our backend, a demo application (based on Spring [PetClinic](https://github.com/spring-projects/spring-petclinic)) that we are using to continuously verify that the basic features are working as expected.
One day, we noted that the pod that was running this demo, was periodically restarted. Our journey started!

## Troubleshooting

First thing is to understand for which reason a pod is restarting. Looking at pod description (OOMkill) we could see that the culprit is the memory used by the pod exceeeds the limit of the container.
Maybe we overlooked the sizing of the container and what is required by the demo to run correctly. The Java heap will not go beyond what we configured as the `Xmx` but for the native part it's another story: Metaspace, code cache, GC structs, VM structs, there are plenty of reasons the spaces can grow. But increasing the container memory was not sufficient.
So a deeper analysis was required. Looking at all native spaces, Metaspace was the one that correlates the increase of RSS memory of the process:

![](/assets/2023/12/DebuggerDemoMetaspaceRSS.png)

Did we have a class/classloader leak?
Checking with the command `jcmd <pid> VM.metaspace` or looking at JFR recordings, the number of classes were still stable.

By default, Metaspace is unbounded: It can increase its size until there is no more memory on the host/container. But we can cap it with `-XX:MaxMetaspaceSize` JVM argument. Let's put somehting reasonable and avoid this OOMkill.

It turned out that we reached the maximum of the Metaspace and `OutOfMemoryError` (OOME) exception was thrown but leaving the process alive in a bad state. This was worse than the OOMkill in a container environement which will restart automatically the container.
I was not sure if the Metaspace triggers a GC to reclaim unused classes in those circumstances, so I verified it into the OpenJDK code.
The allocation for Metaspace happens using this [`Metaspace::allocate`](https://github.com/openjdk/jdk/blob/0678253bffca91775d29d2942f48c806ab4d2cab/src/hotspot/share/memory/metaspace.cpp#L890)  method that attempt an allocation and if no allocation is possible because the Metaspace is full, it calls [`CollectedHeap::satisfy_failed_metadata_allocation`](https://github.com/openjdk/jdk/blob/0678253bffca91775d29d2942f48c806ab4d2cab/src/hotspot/share/gc/shared/collectedHeap.cpp#L317)  method that will trigger a special VM operation: [`VM_CollectForMetadataAllocation`](https://github.com/openjdk/jdk/blob/0678253bffca91775d29d2942f48c806ab4d2cab/src/hotspot/share/gc/shared/gcVMOperations.cpp#L231) at safepoint and will trigger a GC (for G1 a concurrent cycle otherwise a Full GC) with cause: `Metadata Threshold`. So if an OOME is thrown it means we have really not enough space or a leak is happening.

The goal of our demo application is to verify that our instrumentation is working as expected. This instrumentation is done through a [`java.lang.instrumentation.ClassFileTransformer`]() registered through Instrumentation API. When a class is loaded, the transformer checks if we need to instrument or not this class. But what if the class is already loaded and we need to instrument it? In this case, we use the [`Instrumentation::retransformClasses`]() method. For our demos we are continuously (every minute) trying to retransform the same class to inject code.

Let's speed up the process by tight looping on this retransformation: Instantly, we were filling up the Metaspace and got OOME (when capped).

Now, we have the source of this increase. But is it really a leak, or a problem with Metaspace fragmentation? (Link to Metapsace blog post form Thomas stuefe)

## Reproducer

Even if the source is found, coming up with a PetClinic app saying there is a leak to an OpenJDK mailing list sounds not like a good idea. We needed to find a minimal reproducer that everybody can play with and try some small modifications to understand the root cause.
Our demo is a modified version of the original Petclinic. So I tried to clone again the original version and try to reproduce our issue. But I failed. The original code of the class we are targeting to retransform was not including the thing that triggers our leak. Therefore the strategy then was to bisect the code to diff the trigger. Our modification is not huge so I could proceed step by step. I commented some large portions of code and tried if the issue was still there. After several iterations I found the portion of code that seems to be the culprit:

```
  private void syntheticSpan() {
    Tracer tracer = GlobalTracer.get();
    Span synSpan = tracer.buildSpan("synthetic").withTag(Tags.SPAN_KIND, Tags.SPAN_KIND_CLIENT).start();
    try (Scope scope = tracer.activateSpan(synSpan)) {
      LockSupport.parkNanos(Duration.ofMillis(VETS_SYNTHETIC_SPAN_SLEEP_MS).toNanos());
    }
    finally {
      synSpan.finish();
    }
  }
```

For our demo, I was generating synthetic spans with the above code. Commenting the full method was not generating any leak. But the moment I add the try-with-resources statement, the issue is triggered. Let's verify with a small reproducer:

```
class MyClass {
    private static void writeFile() {
      try (TWR twr = new TWR()) {
        twr.process();
      }
    }

    static class TWR implements AutoCloseable {
        public void process() {}
        @Override
        public void close() {}
    }
}
```

the code for the retransforation is added as a javaagent:
```
public class Agent {
    public static void premain(String arg, Instrumentation inst) {
        new Thread(() -> retransformLoop(inst, arg)).start();
    }

    private static void retransformLoop(Instrumentation instrumentation, String className) {
        Class<?> classToRetransform = null;
        while (true) {
            if (classToRetransform == null) {
                for (Class<?> clazz : instrumentation.getAllLoadedClasses()) {
                    if (clazz.getName().equals(className)) {
                        System.out.println("found class: " + className);
                        classToRetransform = clazz;
                        break;
                    }
                }
            }
            if (classToRetransform != null) {
                try {
                    instrumentation.retransformClasses(classToRetransform);
                    //Thread.sleep(1);
                } catch (Exception e) {
                    throw new RuntimeException(e);
                }
            }
        }
    }
}
```

and run with :
`java -javaagent:agent.jar=MyClass -XX:MaxMetaspaceSize=128M -cp . RetransformLeak`

Note: No `ClassFileTransformer` is registered here, we are just invoking the retransform operation while no modification of the bytecode is required.

With this setup, I can now reproduce easily the leak and I can confidently report this problem to [OpenJDK mailing list](https://mail.openjdk.org/pipermail/hotspot-dev/2023-May/074576.html): 

## Finding the root cause

With that reproducer I was now confident that someone will find the exact root cause of this leak and a patch will be made for fixing it. My curiosity was too strong and I continue
to investigate the leak to understand it. Let's focus on the try-with-resources: This is only a syntactic sugar, let's use a decompiler to get basic structure of an equivalent:

```
  TWR var0 = new TWR();
  try {
      var0.process();
  } catch (Throwable var4) {
      try {
          var0.close();
      } catch (Throwable var3) {
          var4.addSuppressed(var3);
      }
      throw var4;
  }
  var0.close();
```

Playing now with this code, I noticed when I comment the line `var4.addSuppressed(var3);` it resolves the leak. Sounds like there is something fishy with that catch block.
Using `catch` with `Throwable` is not something you do everyday, usually you are using a `catch (Exception ex)`.
let's try to focus more on this:

```
class MyClass {
    private static void m() {
        try {
            int i = 42;
        } catch (Throwable var4) {
            var4.printStackTrace();
        }
    }
}
```

By just adding a `catch (Throwable t)` and using the throwable variable by calling a method I am able to leak the Metaspace during the retransformation! Here is my final minimal reproducer. Sounds stupidly silly!
But what happens If I replace by the usual `Exception` instead of `Throwable`? No more leak!


With this minimal reproducer it was then easier to understand what's happening during the retransform operation. Though, I have no clue where to look into the JVM code to start digging. One idea was to use the JVM flag `-XX:+CrashOnOutOfMemoryError` that will give me more context on the OutOfmemoryError raised by the Metaspace.

Here the stacktrace:
```
Stack: [0x00007f8d9c400000,0x00007f8d9c500000],
sp=0x00007f8d9c4fdb80, free space=1014k
Native frames: (J=compiled Java code, j=interpreted, Vv=VM code, C=native code)
V [libjvm.so+0xf9f4e7] VMError::report_and_die(int, char const*, char const*, __va_list_tag*, Thread*, unsigned char*, void*, void*, char const*, int, unsigned long)+0x197 debug.cpp:368)
V [libjvm.so+0x672eab] report_fatal(VMErrorType, char const*, int, char const*, ...)+0x12b
V [libjvm.so+0x67315d] report_java_out_of_memory(char const*)+0xed
V [libjvm.so+0xc238d0] Metaspace::report_metadata_oome(ClassLoaderData*, unsigned long, MetaspaceObj::Type, Metaspace::MetadataType, JavaThread*)+0x260
V [libjvm.so+0xc23b33] Metaspace::allocate(ClassLoaderData*,unsigned long, MetaspaceObj::Type, JavaThread*)+0x133
V [libjvm.so+0x650e8b] ConstantPool::allocate(ClassLoaderData*, int, JavaThread*)+0x7b
V [libjvm.so+0xae518c] VM_RedefineClasses::merge_cp_and_rewrite(InstanceKlass*, InstanceKlass*, JavaThread*)+0x4c
V [libjvm.so+0xae6e6d] VM_RedefineClasses::load_new_class_versions()
[clone .part.0]+0x34d
V [libjvm.so+0xae794f] VM_RedefineClasses::doit_prologue()+0x17f
V [libjvm.so+0xfa8ee0] VMThread::execute(VM_Operation*)+0x40
V [libjvm.so+0xab416f] JvmtiEnv::RetransformClasses(int, _jclass*const*)+0x2bf
V [libjvm.so+0xa5f04c] jvmti_RetransformClasses+0xfc
C [libinstrument.so+0x4e86] retransformClasses+0x1b6
J 182 sun.instrument.InstrumentationImpl.retransformClasses0(J[Ljava/lang/Class;)V java.instrument@20.0.1 (0 bytes) @ 0x00007f8d886d8077 [0x00007f8d886d7fa0+0x00000000000000d7]
J 181 c1 sun.instrument.InstrumentationImpl.retransformClasses([Ljava/lang/Class;)V java.instrument@20.0.1 (33 bytes) @ 0x00007f8d80c29a5c [0x00007f8d80c29860+0x00000000000001fc]
j Agent.retransformLoop(Ljava/lang/instrument/Instrumentation;Ljava/lang/String;)V+82
j Agent.lambda$premain$0(Ljava/lang/instrument/Instrumentation;Ljava/lang/String;)V+2
j Agent$$Lambda$14+0x0000000801002c00.run()V+8
j java.lang.Thread.runWith(Ljava/lang/Object;Ljava/lang/Runnable;)V+5 java.base@20.0.1
j java.lang.Thread.run()V+19 java.base@20.0.1
v ~StubRoutines::call_stub 0x00007f8d88138cc6
V [libjvm.so+0x8c9c85] JavaCalls::call_helper(JavaValue*, methodHandle const&, JavaCallArguments*, JavaThread*)+0x315
V [libjvm.so+0x8cb5f2] JavaCalls::call_virtual(JavaValue*, Handle, Klass*, Symbol*, Symbol*, JavaThread*)+0x1d2
V [libjvm.so+0x99debe] thread_entry(JavaThread*, JavaThread*)+0x8e
V [libjvm.so+0x8e1578] JavaThread::thread_main_inner() [clone .part.0]+0xb8
V [libjvm.so+0xf18736] Thread::call_run()+0xa6
V [libjvm.so+0xcae8d8] thread_native_entry(Thread*)+0xd8
Java frames: (J=compiled Java code, j=interpreted, Vv=VM code)
```

the lines `VM_RedefineClasses::merge_cp_and_rewrite` and `ConstantPool::allocate` give us a direction toward constant pools management during the `retransform` operation.

Discussions with OpenJDK community helps me also to diagnostic what's going on at the JVM level. Using logging of the JVM provides good insight.
Running with `-Xlog:redefine*` and here the end of the output just before the OOME:

```
[2.565s][info][redefine,class,constantpool] old_cp_len=3871, scratch_cp_len=3871
[2.566s][info][redefine,class,constantpool] merge_cp_len=3873, index_map_len=2
[2.566s][info][redefine,class,load ] redefined name=MyClass, count=1920 (avail_mem=2193668K)
[2.566s][info][redefine,class,update ] adjust: name=MyClass
[2.567s][info][redefine,class,timer ] vm_op: all=1 prologue=1 doit=0
[2.567s][info][redefine,class,timer ] redefine_single_class: phase1=0 phase2=0
[2.567s][info][redefine,class,constantpool] old_cp_len=3873, scratch_cp_len=3873
[2.568s][info][redefine,class,constantpool] merge_cp_len=3875, index_map_len=2
[2.575s][info][redefine,class,load ] redefined name=MyClass, count=1921 (avail_mem=2192912K)
[2.575s][info][redefine,class,update ] adjust: name=MyClass
[2.576s][info][redefine,class,timer ] vm_op: all=8 prologue=7 doit=1
[2.576s][info][redefine,class,timer ] redefine_single_class: phase1=0 phase2=0
[2.585s][info][redefine,class,load,exceptions] merge_cp_and_rewrite exception: 'java/lang/OutOfMemoryError'
```

We can notice that the size of the constant pool (cp) used for retransforming the class is increasing linearly by 2 entries for each operation until... boom!

To help me navigate through the JVM code, I even profile the reproducer to have grasp on what's going on here:

[profile_leak.html]

With this profile it seems more obvious that the `retransform` operation is done at safepoint and under Stop-The-World of application threads.
With the logs and the profile we can correlate to find which method is doing the allocation that can leak to Metaspace.

## The bug

To really understand the bug, I need to explain a few things first:

### What is a constant pool?

Each classfile contains a dictionary called constant pool that refers classe, method, field references and strings that are used by the bytecode.

Example:

```
Constant pool:
   #1 = Methodref          #4.#21         // java/lang/Object."<init>":()V
   #2 = Methodref          #22.#23        // java/lang/Integer.parseInt:(Ljava/lang/String;)I
   #3 = Class              #24            // CapturedSnapshot01
   #4 = Class              #25            // java/lang/Object
   #5 = Utf8               <init>
   #6 = Utf8               ()V
   #7 = Utf8               Code
   #8 = Utf8               LineNumberTable
   #9 = Utf8               LocalVariableTable
  #10 = Utf8               this
  #11 = Utf8               LCapturedSnapshot01;
  #12 = Utf8               main
  #13 = Utf8               (Ljava/lang/String;)I
  #14 = Utf8               arg
  #15 = Utf8               Ljava/lang/String;
  #16 = Utf8               var1
  #17 = Utf8               I
  #18 = Utf8               StackMapTable
  #19 = Utf8               SourceFile
  #20 = Utf8               CapturedSnapshot01.java
  #21 = NameAndType        #5:#6          // "<init>":()V
  #22 = Class              #26            // java/lang/Integer
  #23 = NameAndType        #27:#13        // parseInt:(Ljava/lang/String;)I
  #24 = Utf8               CapturedSnapshot01
  #25 = Utf8               java/lang/Object
  #26 = Utf8               java/lang/Integer
  #27 = Utf8               parseInt
```

When you want to load a constant string into the stack to call a method, bytecode contains an instruction `ldc` followed  by the index inside the constant pool:
`ldc #7`

It allows to share this strings with other bytecode instruction and having a compact representation of the bytecode.

### Merging constant pools

When retransforming a class, bytecode may have changed so constant may have changed as well. We need to adjust entries: create additionals, remove unnecessaries. This is done by copying the original then merging. As entries of constant pool can reference each other, loading it involved also a pass for resolution. in the example above, entry #1
`#1 = Methodref          #4.#21`
reference entry #4 and #21, so we need to resolve it to get all information needed when a bytecode refernce this #1 entry like invoking this method.

For its internal representation the JVM is using some specific values to indicate the state of the constant pool entry resolved/unresolved.

If we look at the stacktrace from our Metaspace OOM happenswe can see 

```
...
V [libjvm.so+0xae518c] VM_RedefineClasses::merge_cp_and_rewrite(InstanceKlass*, InstanceKlass*, JavaThread*)+0x4c
...
```

`cp` stands for constant pool. Looks like a good entry point to navigate into the code and understands the process involved. 
To understand the process of merging the constant pools you just have read the following comment found [here](https://github.com/openjdk/jdk/blob/a4bd9e4d0bca0218f27a405b8154425441c10f3f/src/hotspot/share/prims/jvmtiRedefineClasses.hpp#L102-L292). This is like blog post:):

```
// Constant Pool Details:
//
// When the_class is redefined, we cannot just replace the constant
// pool in the_class with the constant pool from scratch_class because
// that could confuse obsolete methods that may still be running.
// Instead, the constant pool from the_class, old_cp, is merged with
// the constant pool from scratch_class, scratch_cp. The resulting
// constant pool, merge_cp, replaces old_cp in the_class.
//
// The key part of any merging algorithm is the entry comparison
// function so we have to know the types of entries in a constant pool
// in order to merge two of them together. Constant pools can contain
// up to 12 different kinds of entries; the JVM_CONSTANT_Unicode entry
// is not presently used so we only have to worry about the other 11
// entry types. For the purposes of constant pool merging, it is
// helpful to know that the 11 entry types fall into 3 different
// subtypes: "direct", "indirect" and "double-indirect".
//
// Direct CP entries contain data and do not contain references to
// other CP entries. The following are direct CP entries:
//     JVM_CONSTANT_{Double,Float,Integer,Long,Utf8}
//
// Indirect CP entries contain 1 or 2 references to a direct CP entry
// and no other data. The following are indirect CP entries:
//     JVM_CONSTANT_{Class,NameAndType,String}
//
// Double-indirect CP entries contain two references to indirect CP
// entries and no other data. The following are double-indirect CP
// entries:
//     JVM_CONSTANT_{Fieldref,InterfaceMethodref,Methodref}
//
// When comparing entries between two constant pools, the entry types
// are compared first and if they match, then further comparisons are
// made depending on the entry subtype. Comparing direct CP entries is
// simply a matter of comparing the data associated with each entry.
// Comparing both indirect and double-indirect CP entries requires
// recursion.
//
// Fortunately, the recursive combinations are limited because indirect
// CP entries can only refer to direct CP entries and double-indirect
// CP entries can only refer to indirect CP entries. The following is
// an example illustration of the deepest set of indirections needed to
// access the data associated with a JVM_CONSTANT_Fieldref entry:
//
//     JVM_CONSTANT_Fieldref {
//         class_index => JVM_CONSTANT_Class {
//             name_index => JVM_CONSTANT_Utf8 {
//                 <data-1>
//             }
//         }
//         name_and_type_index => JVM_CONSTANT_NameAndType {
//             name_index => JVM_CONSTANT_Utf8 {
//                 <data-2>
//             }
//             descriptor_index => JVM_CONSTANT_Utf8 {
//                 <data-3>
//             }
//         }
//     }
//
// The above illustration is not a data structure definition for any
// computer language. The curly braces ('{' and '}') are meant to
// delimit the context of the "fields" in the CP entry types shown.
// Each indirection from the JVM_CONSTANT_Fieldref entry is shown via
// "=>", e.g., the class_index is used to indirectly reference a
// JVM_CONSTANT_Class entry where the name_index is used to indirectly
// reference a JVM_CONSTANT_Utf8 entry which contains the interesting
// <data-1>. In order to understand a JVM_CONSTANT_Fieldref entry, we
// have to do a total of 5 indirections just to get to the CP entries
// that contain the interesting pieces of data and then we have to
// fetch the three pieces of data. This means we have to do a total of
// (5 + 3) * 2 == 16 dereferences to compare two JVM_CONSTANT_Fieldref
// entries.
//
// Here is the indirection, data and dereference count for each entry
// type:
//
//    JVM_CONSTANT_Class               1 indir, 1 data, 2 derefs
//    JVM_CONSTANT_Double              0 indir, 1 data, 1 deref
//    JVM_CONSTANT_Fieldref            2 indir, 3 data, 8 derefs
//    JVM_CONSTANT_Float               0 indir, 1 data, 1 deref
//    JVM_CONSTANT_Integer             0 indir, 1 data, 1 deref
//    JVM_CONSTANT_InterfaceMethodref  2 indir, 3 data, 8 derefs
//    JVM_CONSTANT_Long                0 indir, 1 data, 1 deref
//    JVM_CONSTANT_Methodref           2 indir, 3 data, 8 derefs
//    JVM_CONSTANT_NameAndType         1 indir, 2 data, 4 derefs
//    JVM_CONSTANT_String              1 indir, 1 data, 2 derefs
//    JVM_CONSTANT_Utf8                0 indir, 1 data, 1 deref
//
// So different subtypes of CP entries require different amounts of
// work for a proper comparison.
//
// Now that we've talked about the different entry types and how to
// compare them we need to get back to merging. This is not a merge in
// the "sort -u" sense or even in the "sort" sense. When we merge two
// constant pools, we copy all the entries from old_cp to merge_cp,
// preserving entry order. Next we append all the unique entries from
// scratch_cp to merge_cp and we track the index changes from the
// location in scratch_cp to the possibly new location in merge_cp.
// When we are done, any obsolete code that is still running that
// uses old_cp should not be able to observe any difference if it
// were to use merge_cp. As for the new code in scratch_class, it is
// modified to use the appropriate index values in merge_cp before it
// is used to replace the code in the_class.
//
// There is one small complication in copying the entries from old_cp
// to merge_cp. Two of the CP entry types are special in that they are
// lazily resolved. Before explaining the copying complication, we need
// to digress into CP entry resolution.
//
// JVM_CONSTANT_Class entries are present in the class file, but are not
// stored in memory as such until they are resolved. The entries are not
// resolved unless they are used because resolution is expensive. During class
// file parsing the entries are initially stored in memory as
// JVM_CONSTANT_ClassIndex and JVM_CONSTANT_StringIndex entries. These special
// CP entry types indicate that the JVM_CONSTANT_Class and JVM_CONSTANT_String
// entries have been parsed, but the index values in the entries have not been
// validated. After the entire constant pool has been parsed, the index
// values can be validated and then the entries are converted into
// JVM_CONSTANT_UnresolvedClass and JVM_CONSTANT_String
// entries. During this conversion process, the UTF8 values that are
// indirectly referenced by the JVM_CONSTANT_ClassIndex and
// JVM_CONSTANT_StringIndex entries are changed into Symbol*s and the
// entries are modified to refer to the Symbol*s. This optimization
// eliminates one level of indirection for those two CP entry types and
// gets the entries ready for verification.  Verification expects to
// find JVM_CONSTANT_UnresolvedClass but not JVM_CONSTANT_Class entries.
//
// Now we can get back to the copying complication. When we copy
// entries from old_cp to merge_cp, we have to revert any
// JVM_CONSTANT_Class entries to JVM_CONSTANT_UnresolvedClass entries
// or verification will fail.
//
// It is important to explicitly state that the merging algorithm
// effectively unresolves JVM_CONSTANT_Class entries that were in the
// old_cp when they are changed into JVM_CONSTANT_UnresolvedClass
// entries in the merge_cp. This is done both to make verification
// happy and to avoid adding more brittleness between RedefineClasses
// and the constant pool cache. By allowing the constant pool cache
// implementation to (re)resolve JVM_CONSTANT_UnresolvedClass entries
// into JVM_CONSTANT_Class entries, we avoid having to embed knowledge
// about those algorithms in RedefineClasses.
//
// Appending unique entries from scratch_cp to merge_cp is straight
// forward for direct CP entries and most indirect CP entries. For the
// indirect CP entry type JVM_CONSTANT_NameAndType and for the double-
// indirect CP entry types, the presence of more than one piece of
// interesting data makes appending the entries more complicated.
//
// For the JVM_CONSTANT_{Double,Float,Integer,Long,Utf8} entry types,
// the entry is simply copied from scratch_cp to the end of merge_cp.
// If the index in scratch_cp is different than the destination index
// in merge_cp, then the change in index value is tracked.
//
// Note: the above discussion for the direct CP entries also applies
// to the JVM_CONSTANT_UnresolvedClass entry types.
//
// For the JVM_CONSTANT_Class entry types, since there is only
// one data element at the end of the recursion, we know that we have
// either one or two unique entries. If the JVM_CONSTANT_Utf8 entry is
// unique then it is appended to merge_cp before the current entry.
// If the JVM_CONSTANT_Utf8 entry is not unique, then the current entry
// is updated to refer to the duplicate entry in merge_cp before it is
// appended to merge_cp. Again, any changes in index values are tracked
// as needed.
//
// Note: the above discussion for JVM_CONSTANT_Class entry
// types is theoretical. Since those entry types have already been
// optimized into JVM_CONSTANT_UnresolvedClass entry types,
// they are handled as direct CP entries.
//
// For the JVM_CONSTANT_NameAndType entry type, since there are two
// data elements at the end of the recursions, we know that we have
// between one and three unique entries. Any unique JVM_CONSTANT_Utf8
// entries are appended to merge_cp before the current entry. For any
// JVM_CONSTANT_Utf8 entries that are not unique, the current entry is
// updated to refer to the duplicate entry in merge_cp before it is
// appended to merge_cp. Again, any changes in index values are tracked
// as needed.
//
// For the JVM_CONSTANT_{Fieldref,InterfaceMethodref,Methodref} entry
// types, since there are two indirect CP entries and three data
// elements at the end of the recursions, we know that we have between
// one and six unique entries. See the JVM_CONSTANT_Fieldref diagram
// above for an example of all six entries. The uniqueness algorithm
// for the JVM_CONSTANT_Class and JVM_CONSTANT_NameAndType entries is
// covered above. Any unique entries are appended to merge_cp before
// the current entry. For any entries that are not unique, the current
// entry is updated to refer to the duplicate entry in merge_cp before
// it is appended to merge_cp. Again, any changes in index values are
// tracked as needed.
```

In our case we are referencing with a `Methodref` the `Throwable` class, and David Holmes pointed us to the class verifier, in ClassVerifier::verify_exception_handler_table:

```
      VerificationType throwable =
        VerificationType::reference_type(vmSymbols::java_lang_Throwable());
      // If the catch type is Throwable pre-resolve it now as the assignable check won't
      // do that, and we need to avoid a runtime resolution in case we are trying to
      // catch OutOfMemoryError.
      if (cp->klass_name_at(catch_type_index) == vmSymbols::java_lang_Throwable()) {
        cp->klass_at(catch_type_index, CHECK);
      }
```
Basically the verifier will pre-resolve a Throwable class in the constant pool in case of a caught `OutOfMemoryError`. In that case we don't want to do
a resolution that could fail because of new potential allocations...

Throawble is therefore the only class resolved into the constant pool, but with the process described above it means the entry compaison will fail and generate new entries in the
merged constant pool, allocting new buffers, and each time for every retransformation... here the leak!

### The fix

After some discussions, specially with Coleen Philimore about the bug and attempts to fix it, she directs me toward a simple fix:
Make sure all entries in the constant pool that are related to Class are marked as unresolved even if this is a Throwable class during the comparison/merge
you can find the PR merged here.

## Conclusion

This long standing memory leak was a interesting journey for all phases: diagnostic, reproduciblity, understanding and fix. The OpenJDK team was also very helpful to understand
and prepare the fix.

## References

OpenJDK mailing list: https://mail.openjdk.org/pipermail/hotspot-dev/2023-May/074576.html
ticket in JBS: https://bugs.openjdk.org/browse/JDK-8308762
PR: https://github.com/openjdk/jdk/pull/14780
Redefine/Retranform process, big comment in source code: https://github.com/openjdk/jdk/blob/a4bd9e4d0bca0218f27a405b8154425441c10f3f/src/hotspot/share/prims/jvmtiRedefineClasses.hpp#L35-L324
