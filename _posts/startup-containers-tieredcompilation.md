---
title: "Startup, containers & Tiered Compilation"
layout: default
---
# Startup, containers & Tiered Compilation
## Introduction
Since the introduction of containers, the way we are running JVM applications has changed a lot. In the same time, microservices architecture rises and the usage of container orchestration like Mesos or Kubernetes transform our approach of deployment. With all of that, I end up encounter people setuping a container with less than 1 core. Yes with docker, for example, we can specify than the container will run with a fraction of a core (called [millicores in Kubernetes](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#resource-units-in-kubernetes)). Kernel is dealing with that using cpu scheduling quota (see CFS bandwidth). If it could make sense to put 1.5 or 2.5 on an already multi-core machine, having those scheduling quota in mind, putting less than a core has different implications...

## Tiered Compilation
Since JDK 8, Tiered Compilation was introduced and made by default. It means that the 2 JIT compilers (C1 & C2 are used in combination). Before that, There was those 2 JITs but you had to choose between the 2. C1 was made for client application, with desktop machine (at that time), with only one core. C1 is focused on startup time and reaching native performance quickly. C2 was made for server application, with at least 2 cores (server-class machine), focused on reaching peak performance but with less time constraint.

Nowadays, with Tiered Compilation we can benefit from both JITs a the same time. So first, JVM JIT compiles with C1 to reach quickly native code, but with profiling and at some point C2 kicks-in and it recompiles for more aggressive and time consuming optimizations. 

Here a description of the different levels of compilation:
![](/assets/2020/05/TieredCompilation_1.png)

The nominal path for most of the methods are level 0 (Interpretation), level 3 (C1 + Full Profiling) and level 4 (C2)
But things chan change like:
- If too many methods are queued for level 4 compilation, some are sent to level 2.
- If methods are trivial (small, nothing to profile) they are directly compiled at level 1 and stop here.

But what you need to keep in mind regarding this, the process is dynamic (levels are also adjusted on the fly).

Also note JIT use some threads in background to perform the compilation. The option [`CICompilerCount`](https://chriswhocodes.com/hotspot_options_jdk8.html?search=CICompilerCount) allow to specifiy this number, but by default in TieredCompilation mode there is 2 threads (1 per JIT Compiler):

```
$ java -XX:+PrintFlagsFinal -version | grep CICompilerCount
     intx CICompilerCount                          := 2                                   {product}
     bool CICompilerCountPerCPU                     = true                                {product}
```

## Measuring startup time

Most of the compilations happen at startup time, let's take a classic application like [Spring PetClinic](https://github.com/spring-projects/spring-petclinic).
Here the `Dockerfile`:
```
FROM azul/zulu-openjdk:8
RUN apt-get update && apt-get install -y git
RUN git clone https://github.com/spring-projects/spring-petclinic.git
RUN cd spring-petclinic && ./mvnw package
CMD ["java", "-jar", "/spring-petclinic/target/spring-petclinic-2.3.0.BUILD-SNAPSHOT.jar"]
```
Build the image `spring-petclinic`:
```
$ docker build --tag spring-petclinic .
```

Let's run it with 4 cores of my intel i7-8569U. The application at the end of the starup displays
```
Started PetClinicApplication in 12.059 seconds (JVM running for 12.918)
```

Now let's run it with different cpus from docker:
```
docker run --cpus=<n> -ti spring-petclinic
```

| Cpus | JVM startup time (s) |
| --- | --- |
| 4 | 12.918 |
| 2 | 14.444 |
| 1 | 35.795 |
| 0.8 | 50.151 |
| 0.4 | 118.462 |
| 0.2 | 297.698 |

From 1 cpu, starup time begins to suffer dramatically.

Adding this will help us quantify the JIT time spent for our startup:
```
System.out.println("Total Compilation time: " + ManagementFactory.getCompilationMXBean().getTotalCompilationTime() + "ms");
```

Results with 4 cpus and therefore with C1 + C2:
```
Total Compilation time: 17718ms
```

Let's measure with only C1 by using `-XX:TieredStopAtLevel=1`

```
Total Compilation time: 1261ms
```

More than 10x difference! But why such difference? How this compilation time is ditributed? 
I have used Azul Zulu 8 distribution for one reason: it includes JDK Flight Recorder (JFR), so we can record compilation events to have more information about the JIT.
Each JDK distribution including JFR, provides by default 2 settings: default & profiling. Those settings can be found in 
`<base_dir>/lib/jfr` directory.
I have duplicated the default.jfc file and edited as follow:



I have used Azul Zulu 8 distribution for one reason: it includes JDK Flight Recorder (JFR), so we can record compilation events to have more information about the JIT.

## References
Article recommending TierdStopAtLevel=1
https://phauer.com/2017/increase-jvm-development-productivity/

TieredCompilation in depth: 
- https://www.slideshare.net/maddocig/tiered
- https://slideplayer.com/slide/12376325/

https://docs.oracle.com/javase/8/docs/technotes/tools/windows/java.html

JIT Threads 
==========================

If you happen to start an app in a constrained container ( <1 core)
JIT threads needs room to work well

take sprint petclinic app, and start with docker restraint core
C1 => 1.2s cputime
C2 => 20s cpu time
Measure more precisely with JFR Compile events

