If you read my post on [How to print disassembly from JIT code?](http://jpbempel.blogspot.com/2012/10/how-to-print-dissasembly-from-jit-code.html), you will find that one file is missing to download: hsdis-am64.dll. It is required to disassemble JIT code with windows 64 bits version of JDK.
Until now, I have always shown you disassemble code from 32bits (i386) because I was not able to find a pre-built version or recompile it my self.

I have finally managed to build it. But according to this [page](http://dropzone.nfshost.com/hsdis.htm), it seems there is incompatible licenses between binutils & OpenJDK.
I am not an expert in licensing, so in the doubt I will not publish my binary, but I will share you my step by step procedure to be able to reproduce it at home.

My different attempts with cygwin environments were unsuccessful (I am, by the way, not a big fan of cygwin, this may be explain that...). So I try a different approach with MinGW and MSYS.

Based on this [tutorial](http://ingar.satgnu.net/devenv/mingw32/base.html), here what I did to build this famous dll:

1. Download the MingGW-Get installer: mingw-get-inst-20120426.exe
2. Run the installer. In the third wizard screen, choose the pre-packaged repository catalogues. In the fifth wizard screen, you can change the installation directory. The default C:\mingw is usually approriate.
3. When asked what components to install, select the following:
 * MSYS Basic System
 * MSYS Developer Toolkit
4. Do NOT install the MinGW Compiler Suite, make sure you deselect the C Compiler.
5. Once the installation has finished, start the MinGW Shell. You can find it in Start -> Programs -> MSYS -> MinGW Shell. We will use the installer's command line interface to install additional packages.
You can also start the shell from C:\MinGW\msys\1.0\msys.bat
6. MinGW is a port of the GCC compiler to the win32 platform. Instead of the official release, we install a build from the mingw-w64 project:
x86_64-w64-mingw32-gcc-4.7.0-release-win64_rubenvb.7z
7. Unzip the packages into the C:\MinGW directory. You should and up with the new subdirectory:  C:\MinGW\mingw64. 
8. 
```
mount 'C:\MinGW\mingw64\' /mingw64
mkdir /c/mingw/opt
mkdir /c/mingw/build64 /c/mingw/local64
mount 'C:\MinGW\opt\' /opt
mount 'C:\MinGW\local64\' /local64
mount 'C:\MinGW\build64\' /build64
mkdir /opt/bin /local64/{bin,etc,include,lib,share}
mkdir /local64/lib/pkgconfig
```
9. Create /local64/etc/profile.local:
```
cat > /local64/etc/profile.local << "EOF"
#
# /local64/etc/profile.local
#

alias dir='ls -la --color=auto'
alias ls='ls --color=auto'

PKG_CONFIG_PATH="/local64/lib/pkgconfig"
CPPFLAGS="-I/local64/include"
CFLAGS="-I/local64/include -mms-bitfields -mthreads"
CXXFLAGS="-I/local64/include -mms-bitfields -mthreads"
LDFLAGS="-L/local64/lib"
export PKG_CONFIG_PATH CPPFLAGS CFLAGS CXXFLAGS LDFLAGS

PATH=".:/local64/bin:/mingw64/bin:/mingw/bin:/bin:/opt/bin"
PS1='\[\033[32m\]\u@\h \[\033[33m\w\033[0m\]$ '
export PATH PS1

# package build directory
LOCALBUILDDIR=/build64
# package installation prefix
LOCALDESTDIR=/local64
export LOCALBUILDDIR LOCALDESTDIR

EOF
```
10. `source /local64/etc/profile.local`
11. I encountered an issue with ar tool, so you need to copy & rename the following file
`C:\MinGW\mingw64\bin\ar.exe`
to
`C:\MinGW\mingw64\bin\x86_64-w64-mingw32-ar.exe`
12. Download binutils sources (works great with 2.22) here
13. Unzip the package and go into the directory & type:
`./configure --host=x86_64-w64-mingw32`
14. Then 
`make`
15. Go to your OpenJDK sources in hsdis directory:
`hotspot/src/share/tools/hsdis`
16. edit you hsdis.c to add the line
`#include <stdint.h>`
which is missing for this version of the compiler.
17. finally build hsdis with:
`make BINUTILS=/c/binutils-2.22 ARCH=amd64`
You should now get the hsdis-amd64.dll in `hotspot/src/share/tools/hsdis/bin/win`

Happy disassembling !
