---
date: '2026-08-18T17:46:41+07:00'
draft: true
title: 'Go Netpoll'
---
How Go’s Network Poller Turns Blocking I/O into Goroutine Parking


Operating system already got thread out of the box, they do they bother to create their own goroutine and scheduler?

## Prerequisites
Some prerequisite knowledge to know to understand the decision of Go

### Syscall
- Syscall is a set of linux kernel functions that an user space application call call to get some high privilege services.
- These syscall supports doing some task like: file management (open, read, write, close), process control (fork, getpid, exit), memory management (brk, sbrk, mmap), network & interprocess communication (socket, connect, accept, pipe), time & multiplexing (nanosleep, epoll_wait, epoll)
- When encounter an syscall, the thread will change from user mode to kernel mode to execute syscall's code

### Blocking IO syscall
- When the thread doing operations like open, read, write,... on the file, if the data is not ready, it may be put into sleep, the os then pick a different thread to run.
- When the request data is ready, the thread will be wake up and get in the queue to be run
- The word "blocking" here mean the thread get sleep and reschedule later

A blocking IO on a empty socket:
user space -> kernel space -> sleep, off CPU -> kernel space -> user space

### Non-blocking IO syscall (`O_NONBLOCK`)
- In Linux, some file types like pipe, socket,... support non-blocking
- When the thread doing operations with non-blocking mode, the kernel always return either the data or the error if data is not ready.
- Non-blocking IO alone is quite useless, it is paired with a notifier like `select`, `poll` and `epoll` to know when the file is ready for operation

A non-blocking io on a socket (regardless empty or not):
user space -> kernel space -> user space

### concurrency and parallel
- concurrency is like, you have a bunch of tasks but only have one cpu core, so you divides time into small slices and give it to those tasks. since the cpu is so fast, it give users an illusion of all tasks are executing simultaneously.
- parallel are like having multiple cpu cores to execute multiple tasks. For example you have four tasks and four cpu cores, each task is executed in a core, this is a real parallel. Task are really executing simultaneously.

### IO bound vs CPU bound
- IO bound tasks are those task that spent most of its time waiting to read/write from an IO devices like disk or networking. For web development, the dominant tasks usually IO bound as the application spent most of its time waiting for database queries or external services communication. To increase throughput for this kind of task, we can create more thread to handle more request, because once the thread enter a blocking IO, it will be put sleep, the CPU now is free to pick up another thread to run. But we can't scale forever by just adding more thread, after the CPU is full utilization, we need to add more CPU.

- CPU bound tasks are those task that spent most of its time on CPU, involving complicated computation like encryption, decryption. To increase the throughput for this kind of task, we need more cpu (more parallel). Adding more thread doesn't help in this case, because the CPU is already busy all the time, adding more thread only make it worser by incur the context switch cost.

## Why is using OS thread and blocking IO bad at scale? 
Blocking IO indeed put the thread into sleep, but it isn't making the CPU to stay idle and waste cycles then why is it bad?
For a web server, if we utilize os thread and create a thread for each incoming request, we then will create a lot of thread.

- First of all, OS thread is expensive, it costs a task_struct, 8-16 KB stack, a reserved user stack (8MB virtual by default) and a slot in scheduler's data.
- Even thought it's very costly to create those threads, but most of the time it will just sleep, occupy the resources and do nothing.
- More thread meaning more context switch, and context switch (thread) is costly: update the current thread scheduling state, search for the next thread, save the current thread register state, load the next thread's saved register,...

Beside the direct cost, context switch has indirect consequences:
- Context switch leads to cache thrashing: L1(data/instruction)/L2/LLC/TLB pollution every switch. Cache thrashing make Cpu waste of cycle to refetch data from RAM to its cache line. The more threads, the more severity of cache thrashing.
- CPU migration is when the thread initially running cpu core A, but after wake up, it get scheduled to core B, losing all of data in cache L1/L2. 
- Cache thrashing and cpu migration can lead to performance degradation.


## Go: Leveraging thread pool and non-blocking IO
- Go is designed to for web development, cloud & network services, IO bound is the dominant tasks
- We now know that open a new os thread for every request is expensive and performance degradation. -> The solution is to create a fixed number of threads and reuse it.
- But blocking IO is also expensive, if we use thread pool with blocking IO, our thread will be put to sleep, lead to no active thread in the pool to handle the work load. -> The solution is to use non-blocking IO, when a request is waiting on an IO, the thread now is free to handling other requests.

Combine the thread pool with non-blocking IO, we can now use a small number os thread to serve a large amount of request. Go scheduler is the one do the job to multiplex goroutines into the thread pool.

The nuance: 
- Go definitely can handle CPU bound tasks as well, but the whole point of Goroutine is optimization for IO bound tasks.
- Go can't prevent you from initiating all type of syscall. When a thread blocked by a syscall, Go will create new thread to replace that.
