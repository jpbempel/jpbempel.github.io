---
title:  "Is it really how JVM optimizes the code?"
layout: default
---
# Is it really how JVM optimizes the code?
This post is shorter than the previous ones. It is just to give you another example of the usage of my lovely JVM option PrintAssembly.

This example is inspired from Brian Goetz talk from Devoxx 2009: [Towards A Universal VM](http://parleys.com/play/514892260364bc17fc56be1d/chapter0/about).

Here is the code sample used in this presentation:
```java
    public interface Holder<t> {
        T get();
      }
     
    public class MyHolder<t> implements Holder<t> {
        private final T content;
        public MyHolder(T someContent) {
            content = someContent;
        }
        @Override public T get() {
            return content;
        }
    }
     
    public String getFrom(Holder<string> h) {
        if(h == null)
            throw new IllegalArgumentException("h cannot be null");
        else
            return h.get();
    }
     
    public String doSomething() {
        MyHolder<string> holder = new MyHolder<string>("Hello World");
        return getFrom(holder);
    }
```

Mr Goetz explains in its talk several steps performed by the JIT in order to optimize the code. I let you see the talk for the details. At the end of the optimization process, we get the following equivalent in Java code:
```java
public String doSomething() {
    return "Hello World";
}
```
Yeah, right! Really?
Let's add another level and put that code in to our Test class:

```java
public class TestJIT
{
    // previous code here
 
    public void bench()
    {
        if (doSomething().length() == 0)
        {
            throw new NullPointerException();
        }
    }
 
    public static void main(String[] args) throws Exception
    {
        TestJIT testJIT = new TestJIT();
        for (int i = 0; i < 10000; i++)
        {           
            testJIT.bench();
        }
        Thread.sleep(1000);
    }
}
```
With `PrintAssembly` option, here is the native code of bench method:
```
  # {method} 'bench' '()V' in 'com/bempel/sandbox/TestJIT'
  #           [sp+0x10]  (sp of caller)
  0x02487340: cmp    eax,DWORD PTR [ecx+0x4]
  0x02487343: jne    0x0246ce40         ;   {runtime_call}
  0x02487349: xchg   ax,ax
[Verified Entry Point]
  0x0248734c: mov    DWORD PTR [esp-0x3000],eax
  0x02487353: push   ebp
  0x02487354: sub    esp,0x8            ;*synchronization entry
                                        ; - com.bempel.sandbox.TestJIT::doSomething@-1 (line 40)
                                        ; - com.bempel.sandbox.TestJIT::bench@1 (line 46)
  0x0248735a: mov    ebx,0x56ad530      ;   {oop("Hello World")}
  0x0248735f: mov    ecx,DWORD PTR [ebx+0x10]
  0x02487362: test   ecx,ecx
  0x02487364: je     0x02487371         ;*ifne
                                        ; - com.bempel.sandbox.TestJIT::bench@7 (line 46)
  0x02487366: add    esp,0x8
  0x02487369: pop    ebp
  0x0248736a: test   DWORD PTR ds:0x180000,eax
                                        ;   {poll_return}
  0x02487370: ret   
```
It seems that the doSomething method was inlined and replaced by the `"Hello World"` string instead. The java code equivalent is actually:
```java
public void bench()
{
    if ("Hello World".length() == 0)
    {
        throw new NullPointerException();
    }
}
```
Well not so good, JIT could compute the constant length of Hello World and eliminate the whole code from the method. It happens if you change `doSomething().length == 0` to `doSomething() == null`

Anyway point taken, JVM does the optimizations explained by Mr Goetz!
