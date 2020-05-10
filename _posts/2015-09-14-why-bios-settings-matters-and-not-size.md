---
title: "Why BIOS settings matters (and not size)!"
layout: default
date: 2015-09-14
---
# Why BIOS settings matters (and not size)!
Recently, we have received new server machines based on Xeon E5v3 (Haswell). I have heard from different people that this CPU generation is very good, and figures are really impressive.
So I was pretty excited to test those new beasts on our standard benchmark to compare to the previous E5v2 (Ivy Bridge) we have.

Let's go for the first run:

![](/assets/2015/09/Bios_1.png)

Honestly, this is pretty baaaaad compared to what we've got with E5v2. This is so bad that I emailed to my Production System Manager to find out what's going wrong. Usually when we receive new machines we apply a procedure to configure them correctly for our requirements. As we work in low latency space we need to avoid many pit falls like power management, NUMA, isolcpus, in order to get the best performance.
When I checked at the OS level, I noticed, for example, cpuidle was active, which is not expected. With a well configured/tuned machine, cpuidle is not enabled. My suspicions went to a misconfigured BIOS. My PSM asked me to check with a command line the BIOS (which pretty handy, not need to reboot the machine).
The usual features that we change are the following:
* C states: disabled
* C1E state: disabled
* Power management profile: Maximum performance
* Collaborative CPU performance: disabled

Bingo, BIOS was not reconfigured following our standards! I applied them and re-run our benchmark:

![](/assets/2015/09/Bios_2.png)

Latencies divided by 2 (or more), that's really better! but still slower than E5v2. Let's recheck those BIOS settings one more time.

Why power management features are so bad for low latency? The thing is for a component to be woken up, it takes time, can be hundreds of microseconds. For a process that we are measuring under 100 microseconds, it is huge.
For CPU there is C states that are different sleep modes. On Linux with C states enabled at BIOS level, you can see the latency associated:
```
cat /sys/devices/system/cpu/cpu0/cpuidle/state3/latency
200
```
It means here that to wake up a core from C3 to C0 (running) it takes 200 microseconds!

With new server generation comes new features and options, maybe there is one that make the difference.
I identified one that sounds pretty "bad" from low latency POV: "uncore frequency"=dynamic
Available options: dynamic/maximum.
Let's set it to maximum and run our benchmark:

![](/assets/2015/09/Bios_3.png)

Now we are talking! Results are better than E5v2 (roughly +30%), which is REALLY impressive!
We have tested on a [E5 2697 v2 @ 2.7Ghz](http://ark.intel.com/products/75283/Intel-Xeon-Processor-E5-2697-v2-30M-Cache-2_70-GHz) and on a [E5 2697 v3 @ 2.6Ghz](http://ark.intel.com/products/81059/Intel-Xeon-Processor-E5-2697-v3-35M-Cache-2_60-GHz). There is only 100Mhz less on the v3 but still 30% better on our benchmark.

Finally, there is some fun features we can play in BIOS settings: we can enabled Turbo Boost and make the turbo frequency static by reducing the number of cores available into the CPU.
The E5v3 has 14 cores, let's cut this to only 2 cores and make the frequency permanently to 3.6GHz (yeah this is overclocking for servers!):

![](/assets/2015/09/Bios_4.png)

Compared to the default setup we divided our latency by 4! Just by well tuning the BIOS!
My PSM asked me to email those results to make sure everybody in his team is aware of the importance to apply BIOS settings correctly on production servers.
