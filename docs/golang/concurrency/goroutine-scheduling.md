# Goroutine Scheduling

Goroutines are lightweight threads because Goroutines are not managed by the operating system. They are managed by the Go runtime scheduler.

Understanding this scheduler is the key to understanding why Go performs so well for backend and distributed systems.

***

### The Core Problem

Imagine a server handling 200,000 concurrent HTTP or gRPC requests.

If every request required a traditional OS thread:

* memory usage would explode
* kernel scheduling overhead would become enormous
* context switching would dominate CPU time

Traditional threads are expensive because the operating system manages them directly.

Go solves this differently.

Instead of:

```
1 task -> 1 OS thread
```

Go uses:

```
many goroutines -> small number of OS threads
```

This is called **M:N scheduling**.

The Go runtime acts almost like a miniature operating system inside your process.

It decides:

* which goroutine should run
* where it should run
* when it should pause
* when it should resume

***

## The G-M-P Model

Modern Go scheduling revolves around three internal concepts:

```
G = GoroutineM = Machine (OS Thread)P = Processor (scheduler context)
```

These names come directly from the runtime source code.

***

### G — Goroutine

A goroutine is not just a function.

Internally it contains:

* stack
* execution state
* program counter
* scheduling metadata

When you write:

```
go worker()
```

the runtime creates a new `G` object.

Initially, this goroutine is only _runnable_.\
It is not executing yet.

That distinction matters.

***

### M — Machine

An `M` represents an actual operating system thread.

This is the entity the kernel schedules onto CPU cores.

The runtime does not execute goroutines directly on CPUs.

Instead:

```
goroutine -> OS thread -> CPU
```

But there is another important layer between them.

***

### P — Processor

This is the most misunderstood part of Go scheduling.

A `P` is a logical processor owned by the Go scheduler.

You can think of it as:

* a scheduling token
* a local scheduler state
* a run queue owner

A goroutine can only execute if:

1. an OS thread (`M`) exists
2. that thread owns a processor (`P`)

So execution actually looks like:

```
G -> P -> M -> CPU
```

***

## Why the P Exists

Without the `P` abstraction, all goroutines would compete for one global queue.

That would create massive lock contention on multicore systems.

Instead, every `P` maintains its own local run queue.

Example:

```
P1 -> [G1 G2 G3]P2 -> [G4 G5]P3 -> [G6 G7]
```

This dramatically improves scalability because most scheduling decisions happen locally instead of globally.

This is one of the major reasons Go scales well on multicore systems.

***

## GOMAXPROCS

The number of `P`s is controlled by:

```
runtime.GOMAXPROCS(n)
```

Usually this defaults to the number of CPU cores.

For example:

```
8 CPU cores -> 8 Ps
```

That means at most 8 goroutines can execute simultaneously.

You may have:

* 500,000 goroutines
* but only 8 running at the same instant

The others are:

* waiting
* blocked
* or runnable

***

## Runnable vs Running

This distinction is extremely important.

A goroutine being “alive” does NOT mean it is executing.

A goroutine can be:

| State    | Meaning                     |
| -------- | --------------------------- |
| Runnable | Ready but waiting for CPU   |
| Running  | Currently executing         |
| Waiting  | Blocked on IO/channel/mutex |
| Dead     | Finished                    |

Many performance issues happen because applications create huge numbers of _runnable_ goroutines.

That creates scheduler pressure.

***

## Local Run Queues

Each `P` owns a local queue of runnable goroutines.

When a goroutine is created:

the runtime usually pushes it into the local queue of the current `P`.

The scheduler repeatedly does:

```
take goroutine from queue
run it
repeat
```

This keeps scheduling fast and cache-friendly.

***

## Work Stealing

Now imagine this situation:

```
P1 queue -> empty
P2 queue -> 10,000 goroutines
```

Without balancing:

* one CPU becomes idle
* another overloaded

Go solves this using **work stealing**.

An idle `P` can steal half the runnable goroutines from another `P`.

Example:

```
Before:P1 -> []
P2 -> [100 goroutines]
After stealing:
P1 -> [50]
P2 -> [50]
```

This allows excellent CPU utilization across cores.

Work stealing is one of the most important optimizations in Go’s scheduler.

***

## Goroutine Switching

Traditional thread context switching is expensive because it requires kernel involvement.

Goroutine switching is much cheaper because it happens mostly in user space.

The runtime mainly saves:

* stack pointer
* registers
* execution state

and switches to another goroutine.

This lightweight switching is why Go can handle massive concurrency efficiently.

***

## Blocking Operations

One of the smartest things Go does is handling blocked goroutines efficiently.

Suppose a goroutine waits on:

```
<-ch
```

If no value is available:

* the goroutine is parked
* it stops consuming CPU
* the scheduler runs something else

The same idea applies to:

* mutex waits
* network IO
* timers
* sleeps

This is critical.

Go concurrency works well not because goroutines are magical, but because blocked goroutines are cheap.

***

## Network Poller (Netpoller)

This is one of the most powerful parts of the runtime.

Go networking is not:

```
1 connection -> 1 thread
```

Instead, Go uses OS event systems:

* epoll (Linux)
* kqueue (macOS/BSD)
* IOCP (Windows)

The runtime monitors sockets efficiently using an internal netpoller.

When network IO completes:

* the corresponding goroutine becomes runnable again

This is why Go servers can handle enormous numbers of concurrent connections without exploding thread counts.

***

## Syscalls and Thread Blocking

Now consider a blocking syscall:

```
syscall.Read(...)
```

An OS thread may block inside the kernel.

If Go were naive, that blocked thread would also block scheduling progress.

Instead, the runtime detaches the `P` from the blocked thread and attaches it to another thread.

Conceptually:

```
Before:
M1 + P1 running

Thread blocks in syscall

After:
M1 blocked
P1 moved to M2
```

The scheduler keeps moving.

This is a huge reason Go remains efficient under heavy IO workloads.

***

## Preemption

Older versions of Go had a major problem.

A goroutine running a tight CPU loop could monopolize execution.

Example:

```
for {}
```

The scheduler could struggle to interrupt it.

Modern Go introduced asynchronous preemption.

Now the runtime can interrupt long-running goroutines at safe points:

Examples:

* function calls
* loop checks
* stack growth checks

This improved:

* fairness
* latency
* GC responsiveness
* scheduler stability

significantly.

***

## Scheduler and Garbage Collection

The scheduler and garbage collector are deeply connected.

GC must:

* stop goroutines briefly
* scan stacks
* coordinate memory visibility

At large scale, scheduler behavior and GC behavior heavily influence:

* throughput
* tail latency
* CPU efficiency

This is why production Go performance engineering often involves:

* scheduler tracing
* GC tuning
* goroutine profiling

together.

***

## The Most Important Production Insight

The biggest misconception is:

> “goroutines are free”

They are cheap.

Very cheap.

But not free.

Every goroutine still consumes:

* stack memory
* scheduler metadata
* GC scanning cost
* queue management overhead

The real production problem is usually not goroutine creation itself.

It is:

```
too many runnable goroutines
```

That creates:

* scheduler contention
* latency spikes
* excessive context switching
* CPU thrashing

This is why mature Go systems use:

* worker pools
* bounded concurrency
* backpressure
* cancellation
* structured concurrency

instead of unlimited `go func()` everywhere.

***

## CPU Bound Workloads

Example:

* compression
* encryption
* image processing

Too many goroutines:

* increases contention
* no performance gain

Need bounded parallelism.

***

## IO Bound Workloads

Example:

* HTTP
* gRPC
* DB calls

Can support huge goroutine counts efficiently.



## **Why Go Uses Dynamic Stack Growth**

Go uses **dynamic stack growth** despite the overhead of memory allocation and copying because the trade-off heavily favors **scalability** and **memory efficiency**.

* **The Problem with Fixed Stacks:** Traditional OS threads require large, fixed stacks (e.g., 2MB). Running millions of these would exhaust RAM immediately, limiting concurrency to thousands.
* **The Go Solution:** Goroutines start tiny (\~2KB) and only grow when needed. This allows millions of concurrent tasks to exist in the same memory footprint.
* **Minimized Overhead:**
  * **Exponential Growth:** Stacks double in size (2KB → 4KB → 8KB), meaning reallocations happen rarely. The cost is "amortized" over thousands of function calls.
  * **Speed:** Copying small amounts of data (a few KB) is extremely fast on modern CPUs.
  * **Frequency:** Most I/O-bound goroutines never grow beyond the initial size.
* **The Verdict:** The tiny CPU cost of copying a stack is negligible compared to the massive advantage of supporting **high concurrency** without running out of memory. In cloud-native and distributed systems, this scalability is critical.

## Final Mental Model

The best way to think about the scheduler is this:

```
Goroutines = lightweight tasksP = execution slotsM = worker threadsScheduler = traffic controllerNetpoller = async IO engine
```

And the runtime continuously tries to answer:

```
Which goroutine should run next,on which thread,using which CPU,while minimizing blocking,contention,and idle time?
```

That entire orchestration is what makes Go concurrency powerful.\
<br>

\
If you deeply understand Go scheduler internals, you start understanding:
-------------------------------------------------------------------------

* why Go scales well
* why goroutines are cheap
* why some systems suddenly degrade
* why CPU usage spikes
* why latency happens
* why goroutine leaks become catastrophic
* why lock contention hurts throughput
* how runtime actually executes your code

For high-scale backend systems, distributed systems, gRPC servers, telecom systems, Kafka consumers, Kubernetes controllers, etc., scheduler knowledge becomes extremely valuable.

***

## 1. What Problem Does the Scheduler Solve?

When you write:

```
go processRequest()
```

Questions arise:

* Where does this goroutine go?
* Which thread executes it?
* How is it paused?
* How does runtime resume it?
* How does Go handle millions of goroutines?
* How does Go avoid creating millions of OS threads?

This is the scheduler’s job.

***

## 2. Important Foundation

### Goroutine != OS Thread

This is the MOST IMPORTANT understanding.

#### Traditional Languages

In many systems:

```
1 thread = 1 OS thread
```

OS kernel handles:

* scheduling
* context switching
* blocking
* wakeup

This becomes expensive.

***

### Go Model

Go uses:

```
Many goroutines -> few OS threads
```

This is called:

```
M:N Scheduling
```

Meaning:

* M goroutines
* N threads

mapped dynamically.

***

## 3. Why Traditional Threads Are Expensive

OS thread costs:

| Component         | Cost        |
| ----------------- | ----------- |
| Stack memory      | Large (MBs) |
| Kernel scheduling | Expensive   |
| Context switch    | Expensive   |
| Syscall overhead  | High        |
| Kernel structures | Heavy       |

Creating 1 million threads:\
→ impossible/practical disaster.

***

## 4. Why Goroutines Are Cheap

Goroutines:

* tiny stack (\~2 KB initially)
* user-space scheduling
* managed by runtime
* fast switching
* cheap creation

You can realistically run:

* 100K+
* 500K+
* even millions

depending on workload.

***

## 5. Go Runtime Scheduler

The scheduler is part of:

```
Go Runtime
```

It decides:

* which goroutine runs
* where it runs
* when it pauses
* when it resumes
* thread management
* CPU utilization

***

## 6. Evolution of Go Scheduler

### Old Model (before Go 1.1)

Earlier:

* simpler scheduler
* global lock bottleneck
* poor scalability

***

### Modern Scheduler

Go now uses:

```
G-M-P Scheduler
```

This is the core architecture.

***

## 7. G-M-P Model

Critical topic.

***

### G = Goroutine

Represents:

* your function execution
* stack
* metadata
* state

Example:

```
go worker()
```

creates a `G`.

***

### M = Machine

Represents:

* actual OS thread

Executes goroutines.

***

### P = Processor

This is the MOST misunderstood component.

P is:

* logical processor
* scheduler context
* local run queue owner

A goroutine CANNOT run unless:

* an M owns a P.

***

## 8. Visual Model

```
G = GoroutineM = OS ThreadP = Logical Processor
```

Execution flow:

```
G -> P -> M -> CPU
```

***

## 9. Why P Exists

Without P:

* global scheduler lock contention huge
* scalability terrible

P provides:

* local run queue
* cache locality
* reduced contention

This was a massive improvement.

***

## 10. Number of P

Controlled by:

```
runtime.GOMAXPROCS(n)
```

Usually:

* defaults to CPU core count

Example:

* 8-core machine\
  → default P count = 8

Meaning:

* at most 8 goroutines execute simultaneously.

Important:

* many more goroutines may exist.

***

## 11. Goroutine States

A goroutine can be:

| State    | Meaning      |
| -------- | ------------ |
| Running  | Executing    |
| Runnable | Ready to run |
| Waiting  | Blocked      |
| Dead     | Finished     |

***

## 12. Runnable vs Running

Critical distinction.

### Runnable

Ready to execute but waiting for CPU.

### Running

Currently executing on thread.

***

## 13. Goroutine Lifecycle

```
Created   ↓Runnable   ↓Running   ↓Blocked/Waiting   ↓Runnable   ↓Running   ↓Dead
```

***

## 14. Local Run Queue

Each P has:

```
Local Run Queue
```

Stores runnable goroutines.

This is VERY important for scalability.

***

## 15. Why Local Queues Matter

Without local queues:

```
All threads compete for one global queue
```

Problems:

* lock contention
* cache misses
* scheduler bottleneck

Local queues solve this.

***

## 16. Global Run Queue

Go also has:

* global queue

Used when:

* local queues overflow
* work balancing needed

***

## 17. Scheduler Flow

Typical execution:

```
Create goroutine      ↓Place into local queue      ↓P selects goroutine      ↓M executes it      ↓Blocks/yields/completes
```

***

## 18. Goroutine Creation

When you do:

```
go task()
```

Runtime:

1. creates G structure
2. allocates initial stack
3. places G into queue

Creation is extremely fast.

***

## 19. Work Stealing

One of Go scheduler's most important optimizations.

***

### Problem

Suppose:

```
P1 queue = emptyP2 queue = huge
```

Without balancing:

* CPU idle
* poor utilization

***

### Solution

Idle processor steals work.

```
P1 steals half from P2
```

This is called:

```
Work Stealing
```

***

## 20. Work Stealing Visualization

```
P1: []P2: [G1 G2 G3 G4 G5 G6]↓ stealP1: [G4 G5 G6]P2: [G1 G2 G3]
```

***

## 21. Why Work Stealing Is Powerful

Benefits:

* excellent CPU utilization
* scalable multicore performance
* load balancing
* reduced central contention

This is one reason Go scales well.

***

## 22. Context Switching

Scheduler switches goroutines.

***

### OS Thread Switch

Traditional thread switch:

* kernel involvement
* expensive

***

### Goroutine Switch

Go runtime switch:

* user-space
* much cheaper

This is a major performance advantage.

***

## 23. Goroutine Context

A goroutine context contains:

* stack pointer
* registers
* execution state
* metadata

Switching is lightweight.

***

## 24. Cooperative Scheduling (Old Go)

Older Go versions used:

```
Cooperative Scheduling
```

Meaning:

* goroutine voluntarily yields.

Problem:

* long loops could starve scheduler.

***

## 25. Starvation Problem

Example:

```
func main() {	go func() {		for {		}	}()	fmt.Println("hello")}
```

Earlier Go versions:

* infinite loop could monopolize CPU.

***

## 26. Preemptive Scheduling

Modern Go supports:

```
Asynchronous Preemption
```

Runtime can interrupt goroutines.

Huge improvement.

***

## 27. Why Preemption Matters

Without preemption:

* latency spikes
* starvation
* GC delays
* poor fairness

With preemption:

* scheduler fairness improves.

***

## 28. Safe Points

Runtime cannot interrupt anywhere.

Preemption occurs at:

* safe points

Examples:

* function calls
* loop checks
* stack growth checks

***

## 29. Stack Management

Each goroutine has stack.

***

### Traditional Thread

Fixed large stack:

* wasteful

***

### Goroutine Stack

Starts tiny:

* dynamically grows

This is called:

```
Segmented/Dynamic Stack Growth
```

***

## 30. Stack Growth Process

If stack becomes full:

Runtime:

1. allocates larger stack
2. copies existing stack
3. updates pointers

Transparent to user.

***

## 31. Stack Shrinking

Go can also shrink stacks.

Important for:

* memory efficiency

***

## 32. Scheduler and Blocking Syscalls

Critical topic.

***

### Problem

Suppose goroutine performs:

```
syscall.Read(...)
```

Thread may block.

If runtime not smart:

* entire scheduler stalls.

***

## 33. Runtime Handling

When thread blocks in syscall:

Runtime:

1. detaches P
2. assigns P to another M
3. keeps scheduler moving

Very important optimization.

***

## 34. Visualization

```
M1 + P1 running G1G1 performs blocking syscall↓ runtime detachesM1 blockedP1 moved to M2
```

System continues efficiently.

***

## 35. Network Poller (Netpoller)

One of Go runtime’s most powerful systems.

Go networking is NOT:

* one thread per socket

Instead:

* event-driven polling

Uses:

* epoll (Linux)
* kqueue (BSD/macOS)
* IOCP (Windows)

***

## 36. Why Netpoller Matters

Without it:

* huge thread explosion

With netpoller:

* small thread count handles massive connections

This is why Go handles:

* massive HTTP servers
* gRPC
* Kafka consumers
* websocket systems

efficiently.

***

## 37. Netpoller Flow

```
Socket waits for IO      ↓Netpoller monitors      ↓Event occurs      ↓Goroutine becomes runnable
```

***

## 38. Parking Goroutines

Blocked goroutines are:

```
Parked
```

Meaning:

* not consuming CPU

Examples:

* channel receive
* mutex wait
* sleep
* IO wait

***

## 39. Wakeup Process

When event completes:

* goroutine becomes runnable again
* scheduler queues it

***

## 40. Channel Blocking and Scheduler

Example:

```
<-ch
```

If no sender:

* goroutine parked
* thread free for other work

This is very important.

***

## 41. Mutex Blocking

Similarly:

```
mu.Lock()
```

If lock unavailable:

* goroutine parks

NOT necessarily OS thread blocking.

***

## 42. Scheduler Fairness

Scheduler tries fairness but:

* not perfect strict fairness

Potential issues:

* starvation edge cases
* lock contention imbalance

***

## 43. Scheduler Tick

Runtime periodically checks:

* preemption
* global queue
* GC coordination

***

## 44. Spinning Threads

Sometimes runtime keeps threads spinning briefly.

Why?

* reduce wakeup latency

Tradeoff:

* slightly higher CPU
* lower scheduling latency

***

## 45. Thread Parking

Idle threads eventually:

* sleep/park

to reduce CPU waste.

***

## 46. Scheduler and GC

Critical advanced topic.

Garbage collector heavily interacts with scheduler.

GC requires:

* coordination with goroutines
* safe points
* stack scanning

***

## 47. Stop-The-World (STW)

Some GC phases require:

```
Stop The World
```

Runtime pauses goroutines briefly.

Modern Go minimizes pause duration heavily.

***

## 48. Scheduler Assist for GC

Goroutines may assist GC work.

This prevents:

* dedicated GC bottlenecks

***

## 49. Scheduler Tracing

Useful runtime diagnostics:

```
GODEBUG=schedtrace=1000
```

Prints scheduler info every second.

***

## 50. Example schedtrace Output

```
SCHED 1000ms: gomaxprocs=8 idleprocs=0 threads=12 spinningthreads=1 runqueue=5
```

***

## 51. Understanding schedtrace

| Field           | Meaning                  |
| --------------- | ------------------------ |
| gomaxprocs      | Number of P              |
| idleprocs       | Idle processors          |
| threads         | OS threads               |
| spinningthreads | Active searching threads |
| runqueue        | Runnable goroutines      |

***

## 52. High Runqueue Meaning

Large runqueue may indicate:

* CPU saturation
* excessive goroutines
* scheduler pressure

***

## 53. Too Many Goroutines Problem

Creating millions blindly:

```
for {	go work()}
```

can cause:

* memory explosion
* scheduler overload
* latency spikes

***

## 54. Scheduler Scalability Limits

Goroutines are cheap:\
BUT NOT FREE.

Costs still exist:

* stack memory
* scheduler metadata
* queue management
* GC scanning

***

## 55. Production Lesson

The real problem is often:

```
unbounded concurrency
```

NOT goroutine overhead alone.

***

## 56. Scheduler Latency

Runnable goroutine may wait:

* before execution

Causes:

* CPU saturation
* long queues
* lock contention

***

## 57. Tail Latency Impact

Bad scheduling can increase:

* p99 latency
* p999 latency

Very important in:

* APIs
* telecom systems
* trading systems

***

## 58. Locality Benefits

Keeping goroutine on same P helps:

* cache locality
* reduced memory movement
* performance

***

## 59. False Sharing and Scheduler

Advanced topic.

Different goroutines modifying nearby memory:

* CPU cache invalidation
* performance degradation

Scheduler interactions can amplify this.

***

## 60. cgo and Scheduler

Very important.

When Go calls C:

* runtime loses control

Blocking C calls can:

* hurt scheduler efficiency

***

## 61. Locked Threads

```
runtime.LockOSThread()
```

Pins goroutine to thread.

Used rarely:

* GUI systems
* thread-local APIs

Dangerous if misused.

***

## 62. Scheduler and Containers

In Kubernetes/container environments:

Incorrect CPU limits may:

* distort GOMAXPROCS
* reduce performance

Modern Go versions improved container awareness.

***

## 63. CPU Bound Workloads

Example:

* compression
* encryption
* image processing

Too many goroutines:

* increases contention
* no performance gain

Need bounded parallelism.

***

## 64. IO Bound Workloads

Example:

* HTTP
* gRPC
* DB calls

Can support huge goroutine counts efficiently.

***

## 65. Scheduler and High Throughput Systems

In large systems:\
scheduler performance affects:

* throughput
* latency
* tail latency
* CPU efficiency

***

## 66. Scheduler Anti-Patterns

### Bad

```
go func() {	for {	}}()
```

Busy loops waste CPU.

***

### Bad

```
for req := range requests {	go handle(req)}
```

Unbounded fan-out.

***

### Bad

Huge channel buffers hiding overload.

***

## 67. Good Production Practices

### Use:

* worker pools
* bounded concurrency
* context cancellation
* profiling
* backpressure

***

## 68. Scheduler Observability Tools

Very important.

***

### pprof

```
go tool pprof
```

***

### trace

```
go tool trace
```

Extremely valuable.

Shows:

* goroutine states
* blocking
* scheduling
* network waits
* GC pauses

***

## 69. Runtime Metrics

```
runtime.NumGoroutine()
```

Useful but insufficient alone.

***

## 70. Understanding Scheduler Philosophically

Go runtime acts like:

```
mini operating system
```

inside your process.

It manages:

* scheduling
* memory
* synchronization
* network polling
* timers
* GC

***

## 71. One Critical Insight

The scheduler is optimized for:

```
massive IO concurrency
```

NOT infinite CPU parallelism.

This is VERY important.

Go excels at:

* servers
* networking
* distributed systems

***

## 72. Advanced Internal Components

Internally runtime contains:

* run queues
* timers
* netpoller
* GC workers
* scavenger
* sysmon thread

***

## 73. sysmon Thread

Special monitoring thread.

Handles:

* preemption
* deadlock detection
* timer management
* scheduler monitoring

Very important runtime component.

***

## 74. Scheduler and Timers

Functions like:

```
time.Sleep()
```

use runtime timer system.

Sleeping goroutine:

* parked efficiently
* awakened later

***

## 75. Final Expert-Level Mental Model

Think of scheduler as:

```
Goroutines = tasksP = scheduling slotsM = worker threadsScheduler = traffic controllerNetpoller = IO event engineGC = memory coordinator
```

***

## 76. MOST IMPORTANT Production Understanding

At scale, concurrency problems are usually:

```
scheduler pressure problems
```

caused by:

* excessive runnable goroutines
* lock contention
* unbounded fan-out
* goroutine leaks
* CPU saturation
* blocking operations

***

## 77. What You Should Learn Next

Now that scheduler fundamentals are clear, next best deep dives are:

1. Netpoller Internals
2. Goroutine Parking/Unparking
3. Async Preemption
4. Go Memory Model
5. Channel Internals
6. Mutex Internals
7. Garbage Collector Internals
8. Scheduler Tracing with `go tool trace`
9. Tail Latency Analysis
10. High-throughput concurrency design
