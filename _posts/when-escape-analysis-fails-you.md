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
Though I will just say that EA is not an optimization per se, but an analysis phase (hence the name ;-)) that gather information to ba able to apply some optimizations like lock elision, scalar replacement or even [stack allocation](https://github.com/microsoft/openjdk-proposals/blob/master/stack_allocation/Stack_Allocation_JEP.md).

## Objects.hashCode
The question is why in this case, that seems simple, C2's EA fails?

Let's reproduce the issue, not with a JMH benchmark but with a simple case where we can analyze the JITed code.





devoxx, jmh benchmark

analysis of JITed code

try on JDK8, JDK 11 and JDK14-15 ref C Gracie improvements?

deeper analysis with fastdebug build (shipilev) for PrintEliminateAllocations VM options
 => works for rawVarArgs version

## References
 - [Abstractions Without Regret with GraalVM by Thomas Wuerthinger @ Devoxx BE 2019](https://youtu.be/noX2uHA2Udo?t=1532)
 - [Escape Analysis (Hotspot Wiki)](https://wiki.openjdk.java.net/display/HotSpot/EscapeAnalysis)
 - [Anatomy Quarks 18: Scalar replacement](https://shipilev.net/jvm/anatomy-quarks/18-scalar-replacement/)
 - [Stack allocation prototype for C2 by Charlie Gracie](https://mail.openjdk.java.net/pipermail/hotspot-compiler-dev/2020-January/036835.html)
 - [patch for stack allocation for C2](https://mail.openjdk.java.net/pipermail/hotspot-compiler-dev/2020-June/038779.html)
 - [Stack Allocation JEP proposal](https://github.com/microsoft/openjdk-proposals/blob/master/stack_allocation/Stack_Allocation_JEP.md)
 - https://twitter.com/HansWurst315/status/1246003165478166528
