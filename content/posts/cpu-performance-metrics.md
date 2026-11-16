---
date: '2026-10-03T14:02:30+07:00'
draft: false
title: 'What CPU Utilization Really Means on Linux'
---

> TL;DR \
> CPU Time counters in Linux don't necessarily reflect the CPU hardware utilization.

In Linux, how do you know the current CPU utilization on your server?

You can use tools such as `vmstat` (there are other tools like `top` or `htop`) which show CPU time percentages accounting for states such as system, user, idle, io wait, steal, etc. The sum of idle and iowait represents the idle time of the CPU, and `steal` represents the time the CPU was stolen; therefore, the CPU utilization percentage is 100 minus the total idle and steal percentage.

logical CPU busy time percentage = 100% − idle − iowait − steal

But is that really "CPU Utilization"?

First, let's understand how tools such as `vmstat` calculate those percentages for each state.

*A `vmstat` sample:*
```bash
root@ubuntu-c-4-sfo2:~# vmstat 1
procs -----------memory---------- ---swap-- -----io---- -system-- -------cpu-------
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st gu
 2  0      0 7282696  21260 575860    0    0 19878  5806 3347    5 12  6 81  1  0  0
 1  0      0 7282696  21260 577840    0    0   328    12 1527  359 23  2 75  0  0  0
 1  0      0 7282696  21260 578012    0    0     0     8 1220  122 24  1 75  0  0  0
 1  0      0 7283384  21260 578504    0    0     0    32 1114   84 24  1 75  0  0  0
```
`vmstat 1` can show CPU percentages of the current second. Every second it will append a new data row showing stats for that current second; the first row shows summarized stats since boot. The above sample shows stats after three seconds.

The second row shows stats of the first second, from left to right, CPU time percentage for each state: `user` - 23%, `system` - 2%, `idle` - 75%, `iowait` - 0%, `steal` - 0%, `guest` - 0%. The total across states should be 100% - note that `guest` is included in `user`, so we must exclude it from the sum.
Other tools might present some other states such as `nice`.

> **Terminology note:** This post uses *logical core* for a processor the Linux scheduler can see. With Hyper-Threading, two logical cores can share one physical core; each is also called a hardware thread or logical CPU. Inside a virtual machine, Linux sees vCPUs, whose mapping to host hardware depends on the hypervisor.

### How does `vmstat` calculate CPU time percentage?

**The cumulative CPU time counters**

Monitoring tools such as `vmstat` show different CPU time metric presentations to the users, but under the hood they use the same data source, which is `/proc/stat`. `/proc/stat` is a special file in Linux; it gives us the current value of cumulative CPU time counters for different states. These counters are counted by the Linux kernel.

*A `/proc/stat` sample*
```bash
root@ubuntu-s-4vcpu-8gb-sgp1:~# cat /proc/stat
cpu  40805 599 3083 240249 1080 0 160 1975 0 0
cpu0 11580 0 807 58081 366 0 57 543 0 0
cpu1 8557 516 967 61574 292 0 34 439 0 0
cpu2 10566 83 677 60358 166 0 22 462 0 0
cpu3 10100 0 630 60234 254 0 46 529 0 0
intr 748698 88 9 0 0 911 0 0 0 0 0 0 0 15 0 0 0 0 0 0 0 0 0 0 0 0 24857 0 116 0 0 0 0 0 0 256 0 19668 19371 0 712 1723 0 3591 ... (excluded)
ctxt 247923
btime 1792420696
processes 3147
procs_running 2
procs_blocked 0
softirq 259891 1 37040 1 40374 24696 0 82 67503 10 90184
```
What we need to care about the most is the first N+1 line; N is the number of logical cores of the instance. The first line shows counters that are the sum across logical cores in the computer. The next N lines show counters for each logical core.

Each line has 10 counters, which are the cumulative time that the core spent in a state. From left to right, the states are: `user`, `nice`, `system`, `idle`, `iowait`, `irq`, `softirq`, `steal`, `guest`, `guest_nice`. These counters start at zero at boot time.

`procs_running` and `procs_blocked` show the number of tasks currently running or blocked, respectively.

More detail about : [proc_stat(5) — Linux manual page](https://man7.org/linux/man-pages/man5/proc_stat.5.html)

In the above example, there were `240249` units of time across four cores that the cores stayed idle.

These counters are reported in units of `USER_HZ`, which is a fixed kernel constant defining the granularity of time exposed to userspace applications. Usually, the value of `USER_HZ` is 100, meaning there are 100 units per second; 1 unit is 0.01 seconds.

Knowing that, we can calculate the time the CPU spent in any state, which is equal to the raw counter of a state divided by `USER_HZ`
For example, we can calculate the total idle time across CPUs = 240249 / 100 = 2402.49 seconds.

To calculate the utilization percentage for each state for a period, we just need to take the values of the counters at two points in time: t1 and t2. Since the counters are cumulative, take the later counter minus the former one, and we know how many USER_HZ have passed for a state. Divide the USER_HZ for a state by the total USER_HZ across states; we know its percentage. Note that `guest` and `guest_nice` (the last two) are already included in `user` and `nice`, so the general total must avoid counting them twice.

```
cpu t1: 40000 500 3000 240000 1000 0 160 1900 0 0
cpu t2: 41000 800 3500 250000 2000 0 200 2000 0 0

delta = cpu t2 - cpu t1
delta:   1000 300  500  10000 1000 0  40  100 (ignore last two)

delta sum: 12940 USER_HZ

user percentage: 1000/12940*100 = 7.73%

side effect: t2 - t1 = (delta sum / N / USER_HZ) seconds (N is number of vCPU)
```

### Why counters in `/proc/stat` can be misinterpreted?

**`/proc/stat` doesn't account for CPU hardware details**

There are two concepts I want to emphasize before we move on to the later part of this article: CPU time and CPU hardware utilization.

- CPU Time: It is the elapsed time the kernel sees the CPU working or not working. It is counted and maintained by the Linux kernel in `/proc/stat`; therefore, it represents the Linux kernel's view of CPU time accounting.
- CPU hardware utilization: Internally, each CPU core has many hardware components and optimizations that support executing multiple instructions concurrently.

The point is that the Linux kernel's `/proc/stat` can only report elapsed time in each logical CPU state. It can't tell how much hardware resource the core has utilized in that period.
If the counters in `/proc/stat` can't measure the CPU hardware utilization, then the CPU percentages on tools such as `vmstat` also can't.

At the beginning of the post, asking about **CPU Utilization** is also wrong, because there is no single universal percentage that can represent the hardware utilization.

**An analogy**
*Note that the team here represents **a single core**, with each member like an execution port. While the manager represents the Linux kernel.*

Imagine a team of four workers; they were hired by a manager and get paid per hour they work. The manager only records how many hours the team works, without the detail of what each worker does. On the first day, the team serves 12 hours and idles for 12 hours. Does that mean the team is at 50% of its capacity on the first day? Not really; maybe just one member of the team is actually working, and the rest are free. Therefore, the time reported by the manager doesn't tell how fully the team's capacity was used.

### Understand the CPU hardware resources and utilization
This section aims to help the reader understand the basic level of CPU micro-architecture and why that makes CPU hardware utilization hard to measure from the OS.

Let's take a look at a simplified view of an out-of-order CPU core with Hyper-Threading:

```text
         Thread A state          Thread B state
        (PC, registers)         (PC, registers)
               \                     /
                \                   /
          FRONT END: fetch + decode
          (micro-ops belong to A or B;
           thread selection depends on the microarchitecture)
                        |
                        v
          RENAME + ALLOCATE
          (rename registers; reserve scheduler and
           reorder buffer (ROB) entries)
                        |
                        v
          SCHEDULER
          [A][B][A][A][B][B][A][B] ...   (micro-ops waiting)
                        |
       selects ready micro-ops for available execution units
        /       |        |        |        |       \
     ALU0     ALU1     ALU2     Load     Store    FP/Vec
        \       |        |        |        |       /
                        v
          WRITE BACK / COMPLETE
          (make results available; mark ROB entries complete)
                        |
                        v
          RETIRE via the ROB
          (commit completed operations in program order
           within each thread; reclaim resources)
```
A typical Intel or AMD CPU core probably looks like the above diagram. It is a pipeline with many stages, maybe 20 stages in some Intel chip. The above diagram has some stages such as: fetch, decode, rename, allocate, schedule, execute, write back, retire. Each stage could take one or a few CPU clock cycles to finish. There are two logical threads; each has its own register set. The front end can fetch instructions from both logical threads and decode them into micro-ops. A very wide list of execution ports capable of executing micro-ops in parallel. This diagram is missing some other components, such as the hierarchy of caches (L1/L2/L3), TLB, Load/Store buffer, etc.

#### Superscalar Architecture
Superscalar architecture means that the execution ports are wide, with the capacity to execute many micro-ops in parallel.

As the diagram above, there are many execution ports such as ALU0, ALU1, ALU2, Load, Store, FP/Vec. A real CPU core may have more ports than this.

When the `/proc/stat` of the kernel reports that the CPU was working on `system`, for example, that doesn't mean the CPU is at 100% hardware utilization; it may still have other free execution ports with the capacity to do something else.

#### Out-of-Order execution (OoO)
In-order execution means executing the instructions in the order as they appear in the codebase. But with a wide execution port, OoO can help to execute more micro-ops. OoO is an Instruction-Level Parallelism optimization where instructions are executed not based on their position in the codebase, but based on their dependency readiness.

```asm
LOAD R1, [Memory_Address]  
ADD R2, R1, R3             
ADD R4, R3, R5 
```
Take the above code as an example; line 2 depends on the load in line 1. Therefore, line 1 and 3 can be executed in parallel, while line 2 must wait until line 1 finishes.

OoO also makes it hard to reason about the current hardware utilization of the CPU. Because `/proc/stat` may observe that the CPU is working, but it doesn't know if the core is executing micro-ops in parallel, or if it may be stuck on a long chain of dependencies and many ports stay idle. The Linux kernel `/proc/stat` can't show the hardware utilization details.

#### Branch prediction
The fetch and decode stages are at early stages of the pipeline; when it encounters a branching instruction, it needs to make the decision on which instruction to jump to and process next. It may need to do that without the value of the branching condition, because the condition value is still being calculated in the pipeline and not ready yet.

Not often, it makes a wrong prediction, and many in-progress instructions must be discarded, causing CPU cycle waste.

Because of this, the `/proc/stat` may observe that the CPU core is working, but if the core makes a wrong prediction, many speculative works will be discarded.

#### Stall cycles are counted as busy
> “A second goal was to ensure that when one logical processor is stalled, the other logical processor could continue to make forward progress. A logical processor may be temporarily stalled for a variety of reasons, including servicing cache misses, handling branch mispredictions, or waiting for the results of previous instructions.”
> Hyper-Threading Technology Architecture and Microarchitecture, Intel Technology Journal Q1, 2002

A core can be stalled for several reasons such as:
- **Cache miss**: resulting in the core having to access data down in the cache line or in memory. When that happens, the pipeline still continues, but if the cache miss handling is long enough, the scheduler buffer and the reorder buffer will be full, and the pipeline must stop. The CPU core is now stalled.
- **Branch misprediction**: all younger instructions on the wrong path in the pipeline must be discarded. The pipeline will now be doing nothing for many cycles.
- instructions waiting for previous slow instructions.

When the CPU core stalls, clock cycles pass by without any useful work done. But these stalled cycles are still counted as one of those states by the Linux kernel `/proc/stat`. A memory-heavy workload can show high CPU utilization while the cores are stalled most of the time.

#### CPU frequency changes
A CPU running at 4GHz means there are 4 billion cycles per second; each cycle is 1/4 ns. At 3GHz, there are 3 billion cycles per second, and each cycle is 1/3 ns. That means, in the same chip, running at 3GHz, it takes 1/3 ns to finish a step, but at 4GHz it only takes 1/4 ns. 

There are other things that didn't speed up when the clock speed went up. For example, the time to DRAM. The time from CPU to DRAM is constant; at higher clock speed, it takes more cycles than at lower clock speed. So higher CPU clock speed doesn't help if the workload is memory bound.

From the Linux kernel `/proc/stat` view, it can only account for whether or not the CPU core worked in a period; it can't tell what the CPU frequency of the core was during that period.

#### Hyper-Threading
Hyper-threading is Intel's name for simultaneous multi-threading (SMT). The CPU core with this capacity typically has two logical threads, each with its own register set.

Hyper-threading helps the other logical thread make progress while one logical thread is stalled for whatever reason. The frontend fetches and decodes instructions from one of the two logical threads every clock cycle. The execution ports can simply run any decoded instructions (micro-ops). Hyper-threading helps to utilize the compute resource better by offering more queued instructions.

Why does hyper-threading make the CPU time counters misleading?

*Double logical core doesn't mean double performance*
With hyper-threading enabled, a CPU core usually appears as two logical cores to the kernel. The kernel then schedules tasks based on the number of logical cores.

[HP](https://www.hp.com/gb-en/tech-takes/components/explainer/hyper-threading-everything-to-know.html) claimed that hyper-threading could help improve performance by about 30%, but from os perspective, it sees double the core count. The computer user might interpret that as double the performance.

*Sibling logical threads compete for the resource*
Sibling logical threads are those that share the same physical core. When tasks execute on sibling logical threads, each logical thread has its own register set, but still shares the execution engine, cache, and system bus. This can cause contention and degrade performance. Both tasks are seen as runnable but slow each other down.

So the `/proc/stat` time counters don't show whether the sibling threads slow each other down.

### IO wait is unreliable
The `iowait` in `/proc/stat` sounds like the time that the CPU has been waiting on IO. But the CPU never waits for IO. When a thread makes a blocking IO, it goes to sleep until the io finishes. The kernel picks another thread to run on the CPU; if there is nothing else to run, then the CPU simply goes idle.

When the CPU goes idle, in those sleeping threads, if at least one of them is waiting on io, this idle of the CPU is marked as iowait. Otherwise, the idle is still idle.

Therefore, iowait is really just idle with a tag "iowait" under the hood. 

For example, there are two threads running on the same CPU core. One thread spends most of the time waiting for a read from disk; another thread is doing a CPU-intensive task. The iowait report will mostly be zero because iowait is idle under the hood, but in this case the CPU isn't idle at all because it's busy with the CPU-intensive task.

One more thing is that iowait mostly works with block devices, not network sockets.

### What `/proc/stat` can reliably tell?
It can't tell what the hard utilization of the CPU is. It can only tell whether it saw the CPU busy.

- Small idle + iowait + steal percentage is a signal that the logical cores are busy. 10% of idle + wait + steal doesn't mean the CPU has 10% of hardware utilization left. 90% of the time left, the CPU is working in another state at unknown hardware utilization.
- It can tell how much time or what percentage of time the CPU spent in any state such as system, user, nice, etc. It helps us to find the state that consumes the CPU time.

### What other metrics to look for?
So the nature of the CPU makes it hard for the os to know its hardware utilization. There are other os metrics we can use along with `/proc/stat` to have a better view of current CPU capacity.

**Load average**
```bash
root@ubuntu-s-4vcpu-8gb-sgp1:~# cat /proc/loadavg
0.75 0.69 0.38 3/176 3164
```
`/proc/loadavg` is a special file just like `/proc/stat`, so you can `cat` it to see the values. 

There are a total of five values. The first three values are the load averages over the last 1, 5, and 15-minute intervals, respectively. The fourth value is runnable tasks/total tasks. The fifth value is the PID of the most recently created process.

Load average measures the system demand on resources over an interval. Basically, the kernel divides time into intervals; for each interval, it counts how many tasks are running and waiting for block storage I/O.
A short interval, like 1 minute, is sensitive to sudden bursts in demand, while a long interval, like 15 minutes, gives you a smoother chart.

One weakness of this metric is that it doesn't tell which specific resource the tasks are waiting for. We only know the tasks are waiting, but don't know what they are waiting for.

**Runnable Threads**
The number of runnable threads is `procs_running` in `/proc/stat`. This metric shows you how many tasks are running and waiting for the CPU. Compare this metric to the number of logical cores you have; you get a sense of how much the CPU is utilized. If runnable threads > logical cores, this means your CPU is quite saturated.
Since `procs_running` only shows the runnable task count at a moment, sometimes the CPU can experience a short burst, so we need to observe it over a long period and pair it with load average.

**PSI - Pressure Stall Information**
```bash
root@ubuntu-s-4vcpu-8gb-sgp1:~# cat /proc/pressure/cpu
some avg10=0.00 avg60=0.01 avg300=0.28 total=5759273
full avg10=0.00 avg60=0.00 avg300=0.00 total=0
```

It can show demand for each resource over 10-, 60-, and 300-second intervals. It tracks the demand of resources separately for CPU, memory, and IO. PSI value is the percent of time during the interval that tasks wait for the resource, so the PSI value ranges from 0 to 100. It has two modes, `some` and `full`: `full` means the percent of time that all non-idle tasks simultaneously wait for a specific resource; `some` means at least one task waits for a specific resource. Note that CPU `full` is undefined and displayed as zero.

If you care about CPU, you can look at `/proc/pressure/cpu`. For example, `avg10=4.12` means that in the last 10 seconds, 4.12% of the time, at least one task must wait for CPU.

Similar to load average, short intervals like 10s in PSI are much more spiky than 60s or 300s.

**Application metrics**
Application metrics such as latency, throughput, and error rate should be considered along with these metrics. If there is a correlation between application-level metrics and other system metrics, you might suspect that the CPU is overloaded.

**Scheduling pressure metrics can be misleading**
Load average, `procs_running` and PSI show the kernel's scheduling pressure, it doesn't measure the of CPU hardware utilization. On a system without hyper-threading, hight pressure on scheduling metrics might suggest high CPU hardware utilization in some scenarios. However, with hyper-threading on, these metrics are purely showing scheduling pressure.

For example, 4 cores/8 logical cores. Tasks are continuously runnable, all logical cores are available, and the scheduler places tasks on separate cores before using siblings.

|Task count | Metrics | Explain |
|----------|-|-|
| x <= 4 | runnable = 4, PSI ≈ 0 | Each task gets scheduled on a physical core, so no waiting for a core |
| 4 < x <= 8 | runnable > 4 <= 8,  PSI ≈ 0 | some tasks are scheduled on sibling logical threads, and might slow each other down, but PSI and runnable can't show that performance issue |
| x > 8 | runnable > 8, psi > 0 | some tasks waiting for logical cores |

*Note that the numbers in the example are simplified for demonstration purposes. In reality, it doesn't necessarily have to be exactly like that.*

### Linux `perf`
So we learned that `/proc/stat` only shows the elapsed time for each logical core. Scheduling pressure metrics such as PSI or load average only show the task scheduling pressure on the Linux kernel. None of them can show the CPU hardware utilization.

Linux `perf` with top-down analysis can give us a breakdown of pipeline slots and bottlenecks. But I discuss it in a different article.

### References
- https://man7.org/linux/man-pages/man8/vmstat.8.html
- https://man7.org/linux/man-pages/man5/proc_stat.5.html
- https://man7.org/linux/man-pages/man7/time.7.html
- https://www.intel.com/content/dam/www/public/us/en/documents/research/2002-vol06-iss-1-intel-technology-journal.pdf
- https://www.hp.com/gb-en/tech-takes/components/explainer/hyper-threading-everything-to-know.html
