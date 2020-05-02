In my previous post, I have shown you some disassembly from JIT native code. How to do the same at home ?

I found relevant informations in this [wiki page](https://wiki.openjdk.java.net/display/HotSpot/PrintAssembly).

First, you need to get the dissasembler named
* [hsdis-i386.dll](http://www.ssw.jku.at/General/Staff/LS/hsdis-i386.zip) for windows 32 bits JVM
* [hsdis-amd64.dll](http://jpbempel.blogspot.com/2013/07/how-to-build-hsdis-amd64dll.html) for windows 64 bits JVM
* [linux-libhsdis-i386.so](http://kenai.com/projects/base-hsdis/downloads/download/linux-hsdis-i386.so) for linux 32 bits JVM
* [linux-libhsdis-amd64.so](http://kenai.com/projects/base-hsdis/downloads/download/linux-hsdis-amd64.so) for linux 64 JVM
* [solaris-libhsdis-i386.so](http://kenai.com/projects/base-hsdis/downloads/download/solaris-hsdis-i386.so) for solaris 32 bits JVM
* [solaris-libhsdis-amd64.so](http://kenai.com/projects/base-hsdis/downloads/download/solaris-hsdis-amd64.so) for solaris 64 bits JVM
Put this disassembler into the bin directory of your JRE: `C:\Program Files\Java\jre6\bin`
Or for your JDK: `C:\Program Files\Java\jdk1.6.0_35\jre\bin`

Then launch your java command with the following options:
`-XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly`

Then you will get:
```
"C:\Program Files (x86)\Java\jdk1.6.0_23\jre\bin\java" -server
-XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly com.bempel.sandbox.TestJIT
Java HotSpot(TM) Server VM warning: PrintAssembly is enabled; turning on
DebugNonSafepoints to gain additional output
Loaded disassembler from hsdis-i386.dll
Decoding compiled method 0x02548708:
Code:
[Disassembling for mach='i386']
[Entry Point]
[Verified Entry Point]
  # {method} 'run' '()V' in 'com/bempel/sandbox/TestJIT$C'
[...]
```
Enjoy !

**Updated**: To get intel syntax over AT&T, use -XX:PrintAssemblyOptions=intel
