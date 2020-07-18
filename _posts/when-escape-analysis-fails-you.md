---
layout: default
title: "When Escape Analysis fails you"
date:
---
# When Escape Analysis fails you

## Escape Analysis

## Objects.hashCode
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
