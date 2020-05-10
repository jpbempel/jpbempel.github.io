---
title: "Whitebox API"
layout: default
date: 2015-07-07
---
# WhiteBox API

I have already seen this in [JCStress](http://openjdk.java.net/projects/code-tools/jcstress/) but this is in a [post from Rémi Forax](https://groups.google.com/d/msg/mechanical-sympathy/CnZNbtsxyqw/c2kBFp8llbcJ) on mechanical sympathy forum that brings attention to me when I saw what is possible to do with it. Here is a summary:

```java
public class WhiteBox {
  // Get the maximum heap size supporting COOPs
  public native long getCompressedOopsMaxHeapSize();
  // Arguments
  public native void printHeapSizes();

  // Memory
  public native long getObjectAddress(Object o);
  public native int  getHeapOopSize();
  public native int  getVMPageSize();
  public native long getVMLargePageSize();

  public native boolean isObjectInOldGen(Object o);
  public native long getObjectSize(Object o);

  // Runtime
  // Make sure class name is in the correct format
  public boolean isClassAlive(String name);
  public native boolean isMonitorInflated(Object obj);
  public native void forceSafepoint();

  // Resource/Class Lookup Cache
  public native boolean classKnownToNotExist(ClassLoader loader, String name);
  public native URL[] getLookupCacheURLs(ClassLoader loader);
  public native int[] getLookupCacheMatches(ClassLoader loader, String name);

  // JVMTI
  public native void addToBootstrapClassLoaderSearch(String segment);
  public native void addToSystemClassLoaderSearch(String segment);

  // G1
  public native boolean g1InConcurrentMark();
  public native boolean g1IsHumongous(Object o);
  public native long    g1NumMaxRegions();
  public native long    g1NumFreeRegions();
  public native int     g1RegionSize();
  public native MemoryUsage g1AuxiliaryMemoryUsage();
  public native Object[]    parseCommandLine(String commandline, DiagnosticCommand[] args);

  // NMT
  public native long NMTMalloc(long size);
  public native void NMTFree(long mem);
  public native long NMTReserveMemory(long size);
  public native void NMTCommitMemory(long addr, long size);
  public native void NMTUncommitMemory(long addr, long size);
  public native void NMTReleaseMemory(long addr, long size);
  public native long NMTMallocWithPseudoStack(long size, int index);
  public native boolean NMTIsDetailSupported();
  public native boolean NMTChangeTrackingLevel();
  public native int NMTGetHashSize();

  // Compiler
  public native void    deoptimizeAll();
  public        boolean isMethodCompiled(Executable method) {
    return isMethodCompiled(method, false /*not osr*/);
  }
  public native boolean isMethodCompiled(Executable method, boolean isOsr);
  public        boolean isMethodCompilable(Executable method) {
    return isMethodCompilable(method, -1 /*any*/);
  }
  public        boolean isMethodCompilable(Executable method, int compLevel) {
    return isMethodCompilable(method, compLevel, false /*not osr*/);
  }
  public native boolean isMethodCompilable(Executable method, int compLevel, boolean isOsr);
  public native boolean isMethodQueuedForCompilation(Executable method);
  public        int     deoptimizeMethod(Executable method) {
    return deoptimizeMethod(method, false /*not osr*/);
  }
  public native int     deoptimizeMethod(Executable method, boolean isOsr);
  public        void    makeMethodNotCompilable(Executable method) {
    makeMethodNotCompilable(method, -1 /*any*/);
  }
  public        void    makeMethodNotCompilable(Executable method, int compLevel) {
    makeMethodNotCompilable(method, compLevel, false /*not osr*/);
  }
  public native void    makeMethodNotCompilable(Executable method, int compLevel, boolean isOsr);
  public        int     getMethodCompilationLevel(Executable method) {
    return getMethodCompilationLevel(method, false /*not ost*/);
  }
  public native int     getMethodCompilationLevel(Executable method, boolean isOsr);
  public native boolean testSetDontInlineMethod(Executable method, boolean value);
  public        int     getCompileQueuesSize() {
    return getCompileQueueSize(-1 /*any*/);
  }
  public native int     getCompileQueueSize(int compLevel);
  public native boolean testSetForceInlineMethod(Executable method, boolean value);
  public        boolean enqueueMethodForCompilation(Executable method, int compLevel) {
    return enqueueMethodForCompilation(method, compLevel, -1 /*InvocationEntryBci*/);
  }
  public native boolean enqueueMethodForCompilation(Executable method, int compLevel, int entry_bci);
  public native void    clearMethodState(Executable method);
  public native int     getMethodEntryBci(Executable method);
  public native Object[] getNMethod(Executable method, boolean isOsr);

  // Intered strings
  public native boolean isInStringTable(String str);

  // Memory
  public native void readReservedMemory();
  public native long allocateMetaspace(ClassLoader classLoader, long size);
  public native void freeMetaspace(ClassLoader classLoader, long addr, long size);
  public native long incMetaspaceCapacityUntilGC(long increment);
  public native long metaspaceCapacityUntilGC();

  // force Young GC
  public native void youngGC();

  // force Full GC
  public native void fullGC();

  // Tests on ReservedSpace/VirtualSpace classes
  public native int stressVirtualSpaceResize(long reservedSpaceSize, long magnitude, long iterations);
  public native void runMemoryUnitTests();
  public native void readFromNoaccessArea();
  public native long getThreadStackSize();
  public native long getThreadRemainingStackSize();

  // CPU features
  public native String getCPUFeatures();

  // Native extensions
  public native long getHeapUsageForContext(int context);
  public native long getHeapRegionCountForContext(int context);
  public native int getContextForObject(Object obj);
  public native void printRegionInfo(int context);

  // VM flags
  public native void    setBooleanVMFlag(String name, boolean value);
  public native void    setIntxVMFlag(String name, long value);
  public native void    setUintxVMFlag(String name, long value);
  public native void    setUint64VMFlag(String name, long value);
  public native void    setStringVMFlag(String name, String value);
  public native void    setDoubleVMFlag(String name, double value);
  public native Boolean getBooleanVMFlag(String name);
  public native Long    getIntxVMFlag(String name);
  public native Long    getUintxVMFlag(String name);
  public native Long    getUint64VMFlag(String name);
  public native String  getStringVMFlag(String name);
  public native Double  getDoubleVMFlag(String name);
  private final List<Function<String,Object>> flagsGetters = Arrays.asList(
    this::getBooleanVMFlag, this::getIntxVMFlag, this::getUintxVMFlag,
    this::getUint64VMFlag, this::getStringVMFlag, this::getDoubleVMFlag);

  public Object getVMFlag(String name) {
    return flagsGetters.stream()
                       .map(f -> f.apply(name))
                       .filter(x -> x != null)
                       .findAny()
                       .orElse(null);
  }
  public native int getOffsetForName0(String name);
  public int getOffsetForName(String name) throws Exception {
    int offset = getOffsetForName0(name);
    if (offset == -1) {
      throw new RuntimeException(name + " not found");
    }
    return offset;
  }
}
```

This API is usable since JDK8 and there is some new additions in JDK9.

But how to use it ?
This API is not part of the standard API but in the test library from OpenJDK. You can find it [here](http://hg.openjdk.java.net/jdk8u/jdk8u/hotspot/file/0e4094950cd3/test/testlibrary/whitebox/sun/hotspot/WhiteBox.java).
Download the source of OpenJDK then either you build it entirely and grab the wb.jar or

1. go to `test/testlibrary/whitebox` directory
2. `javac -sourcepath . -d . sun\hotspot\**.java`
3. `jar cf wb.jar .`

Place you wb.jar next to your application and launch it with:

```
java -Xbootclasspath/a:wb.jar -XX:+UnlockDiagnosticVMOptions -XX:+WhiteBoxAPI ...
```

Here is an examble you can run with WhiteBox jar:
```java
import sun.hotspot.WhiteBox;

public static class GCYoungTest {
  static WhiteBox wb = WhiteBox.getWhiteBox();
  public static Object obj;

  public static void main(String args[]) {
    obj = new Object();
    System.out.println(wb.isObjectInOldGen(obj));
    wb.youngGC();
    wb.youngGC();
    // 2 young GC is needed to promote object into OldGen
    System.out.println(wb.isObjectInOldGen(obj));
  }
}
```

For this example you need to add `-XX:MaxTenuringThreshold=1` to make it work as expected.
Now you have an API to trigger minor GC and test if an object resides in Old generation, pretty awesome!

You can also trigger JIT compilation on demand for some methods and change VM flags on the fly:

```java
public class WhiteBoxTest
{
    static WhiteBox wb = WhiteBox.getWhiteBox();

    private void m()
    {
        System.out.println("foo");
    }

    public static void main(String[] args) throws Exception
    {
        wb.setBooleanVMFlag("PrintCompilation", true);
        wb.setBooleanVMFlag("BackgroundCompilation", false);
        wb.enqueueMethodForCompilation(WhiteBoxTest.class.getDeclaredMethod("m", null), 4);
    }
}
```

Unlike unsafe, this API seems difficult to use in production environment, but at least you can have fun in labs or adding this like the OpenJDK for your low level tests. Enjoy!
