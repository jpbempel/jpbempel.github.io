---
title: "Startup, containers & Tiered Compilation"
layout: default
---
# Startup, containers & Tiered Compilation
## Introduction
Since the introduction of containers, the way we are running JVM applications has changed a lot. In the same time, microservices architecture rises and the usage of container orchestration like Mesos or Kubernetes transform our approach of deployment. With all of that, I end up encounter people setuping a container with less than 1 core. Yes with docker, for example, we can specify than the container will run with a fraction of a core (called millicores in Kubernetes). Kernel is dealing with that using cpu scheduling quota (see CFS bandwidth). If it could make sense to put 1.5 or 2.5 on an already multi-core machine, having those scheduling quota in mind, putting less than a core has different implications...

## Tiered Compilation
Since JDK 8, Tiered Compilation was introduced and made by default. It means that the 2 JIT compilers (C1 & C2 are used in combination). Before that, There was those 2 JITs but you had to choose between the 2. C1 was made for client application, with desktop machine (at that time), with only one core. C1 is focused on startup time and reaching native performance quickly. C2 was made for server application, with at least 2 cores (server-class machine), focused on reaching peak performance but with less time constraint.

Nowadays, with Tiered Compilation we can benefit from both JITs a the same time. So first, JVM JIT compiles with C1 to reach quickly native code, but with profiling and at some point C2 kicks-in and it recompiles for more aggressive and time consuming optimizations. 

![](/assets/2020/05/TieredCompilation_1.png)

Article recommending TierdStopAtLevel=1
https://phauer.com/2017/increase-jvm-development-productivity/

TieredCompilation in depth: 
- https://www.slideshare.net/maddocig/tiered
- https://slideplayer.com/slide/12376325/


JIT Threads 
==========================

If you happen to start an app in a constrained container ( <1 core)
JIT threads needs room to work well

take sprint petclinic app, and start with docker restraint core
C1 => 1.2s cputime
C2 => 20s cpu time
Measure more precisely with JFR Compile events

