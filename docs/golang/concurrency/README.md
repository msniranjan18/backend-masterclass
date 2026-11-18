# Concurrency

Go was designed for scalable backend systems, distributed services, network servers, streaming systems, telecom systems, and cloud-native applications — exactly the kind of systems where concurrency matters.

We'll focus on below in this chapter:

* Scheduler internals
* Memory model
* Race conditions
* Context propagation
* Structured concurrency
* Worker pools
* Backpressure
* Cancellation
* Production-grade patterns
* Debugging deadlocks and leaks
* Performance implications

***

## 1. What is Concurrency?

Concurrency means:

> Multiple tasks making progress independently.

It does **NOT** necessarily mean parallel execution.

***

## 2. Concurrency vs Parallelism

### Concurrency

* Tasks are interleaved
* One CPU core can still run concurrent tasks
* Focus: structure

Example:

* Chat server
* HTTP requests
* Background jobs

***

### Parallelism

* Multiple tasks literally execute simultaneously
* Requires multiple CPU cores

Example:

* Matrix multiplication
* Image processing
* ML workloads

***

## 3. Why Go Concurrency is Powerful

Go provides:

| Feature    | Purpose                       |
| ---------- | ----------------------------- |
| Goroutines | Lightweight threads           |
| Channels   | Safe communication            |
| Scheduler  | Efficient execution           |
| CSP Model  | Share memory by communicating |
| Select     | Multiplex communication       |
| Context    | Cancellation and deadlines    |

Go concurrency is based heavily on:

Tony Hoare's CSP (Communicating Sequential Processes).

***

## 4. OS Thread vs Goroutine

### Traditional Thread

OS thread:

* Heavyweight
* MBs of stack
* Expensive context switching
* Managed by kernel

***

### Goroutine

Goroutine:

* Extremely lightweight
* Starts with tiny stack (\~2 KB)
* Managed by Go runtime
* Cheap to create

You can run:

* thousands
* hundreds of thousands
* even millions

of goroutines.

***

## 5. First Goroutine

{% code lineNumbers="true" expandable="true" %}
```go
package main

import (
	"fmt"
	"time"
)

func hello() {
	fmt.Println("Hello from goroutine")
}

func main() {
	go hello()

	time.Sleep(time.Second)
}
```
{% endcode %}

***

## 6. Important: Main Goroutine

`main()` itself runs inside a goroutine.

If main exits:

* all goroutines die immediately.

***

## 7. Goroutine Scheduling

Go runtime contains a scheduler.

### G-M-P Model

Core internal architecture:

| Component | Meaning                       |
| --------- | ----------------------------- |
| G         | Goroutine                     |
| M         | OS Thread                     |
| P         | Processor (logical scheduler) |

***

### Flow

```
Goroutine (G)
   ↓
Processor Queue (P)
   ↓
OS Thread (M)
   ↓
CPU
```

***

## 8. Work Stealing Scheduler

Each processor (P) has:

* local run queue

If one processor becomes idle:

* it steals work from another processor.

This improves:

* CPU utilization
* scalability

***

## 9. Cooperative + Preemptive Scheduling

Older Go:

* cooperative scheduling

Modern Go:

* asynchronous preemption

Meaning:

* runtime can interrupt long-running goroutines.

Very important for:

* fairness
* latency

***

## 10. Stack Growth

Goroutine stack:

* starts tiny
* grows dynamically

Unlike C:

* no fixed huge stack

This is one reason Go can support huge concurrency.

***

## 11. WaitGroup

Used to wait for goroutines.

```
package mainimport (	"fmt"	"sync")func worker(id int, wg *sync.WaitGroup) {	defer wg.Done()	fmt.Println("Worker", id)}func main() {	var wg sync.WaitGroup	for i := 0; i < 5; i++ {		wg.Add(1)		go worker(i, &wg)	}	wg.Wait()}
```

***

## 12. Common WaitGroup Bug

Wrong:

```
wg.Add(1)go func() {	wg.Done()}()
```

If goroutine exits before Add:\
→ panic possible.

Always:

* call `Add()` BEFORE launching goroutine.

***

## 13. Channels

Channels are typed pipes for communication.

```
ch := make(chan int)
```

***

## 14. Channel Send/Receive

```
ch <- 10v := <-ch
```

***

## 15. Unbuffered Channels

```
ch := make(chan int)
```

Behavior:

* sender blocks until receiver ready
* receiver blocks until sender ready

This creates synchronization.

***

## 16. Buffered Channels

```
ch := make(chan int, 3)
```

Can hold 3 values without blocking.

***

## 17. Channel Internals

A channel internally contains:

* mutex
* wait queue
* ring buffer
* sender queue
* receiver queue

Channels are not "magic".\
They are runtime synchronization structures.

***

## 18. Channel Direction

```
func producer(ch chan<- int)func consumer(ch <-chan int)
```

Good for:

* API safety
* architecture clarity

***

## 19. Closing Channels

```
close(ch)
```

Meaning:

* no more values will come

***

## 20. Reading Closed Channel

```
v, ok := <-ch
```

If closed:

* `ok == false`

***

## 21. Important Rule

ONLY sender should close channel.

Never receiver.

Otherwise:

* panic risk
* race conditions

***

## 22. Range Over Channel

```
for v := range ch {	fmt.Println(v)}
```

Ends when:

* channel closed

***

## 23. Select Statement

Used to wait on multiple channel operations.

```
select {case msg := <-ch1:	fmt.Println(msg)case ch2 <- 10:	fmt.Println("sent")default:	fmt.Println("no activity")}
```

***

## 24. Select Internals

Runtime:

* checks channel states
* parks goroutine if needed
* wakes later

Select enables:

* multiplexing
* fan-in
* cancellation
* timeout

***

## 25. Timeout Pattern

```
select {case res := <-ch:	fmt.Println(res)case <-time.After(time.Second):	fmt.Println("timeout")}
```

***

## 26. Context Package

One of the MOST IMPORTANT concurrency tools.

Package:

```
context
```

Used for:

* cancellation
* deadlines
* request scope
* propagation

***

## 27. Why Context Matters

Without cancellation:

* goroutine leaks
* memory leaks
* resource exhaustion

In production:\
this becomes catastrophic.

***

## 28. Context Example

```
ctx, cancel := context.WithCancel(context.Background())go worker(ctx)cancel()
```

***

## 29. Context Cancellation Pattern

```
func worker(ctx context.Context) {	for {		select {		case <-ctx.Done():			fmt.Println("stopped")			return		default:			// work		}	}}
```

***

## 30. Context Tree

Contexts form hierarchy.

```
Request Context   ├── DB Context   ├── Cache Context   └── gRPC Context
```

Cancel parent:\
→ all children cancel.

***

## 31. Context Deadline

```
ctx, cancel := context.WithTimeout(	context.Background(),	2*time.Second,)
```

***

## 32. Fan-Out Pattern

One request spawning multiple workers.

```
Request ├── Worker1 ├── Worker2 └── Worker3
```

Common in:

* microservices
* aggregators
* search systems

***

## 33. Fan-In Pattern

Multiple goroutines send into one channel.

```
Worker1 ─┐Worker2 ─┼──> Result ChannelWorker3 ─┘
```

***

## 34. Worker Pool Pattern

Very important production pattern.

Limits concurrency.

***

### Example

```
package mainimport (	"fmt"	"sync")func worker(id int, jobs <-chan int, wg *sync.WaitGroup) {	defer wg.Done()	for job := range jobs {		fmt.Printf("worker=%d job=%d\n", id, job)	}}func main() {	jobs := make(chan int, 100)	var wg sync.WaitGroup	for i := 0; i < 3; i++ {		wg.Add(1)		go worker(i, jobs, &wg)	}	for j := 0; j < 10; j++ {		jobs <- j	}	close(jobs)	wg.Wait()}
```

***

## 35. Why Worker Pools Matter

Without limits:

```
for _, req := range requests {	go process(req)}
```

You may create:

* millions of goroutines
* OOM
* scheduler collapse

Worker pools provide:

* bounded concurrency
* backpressure

***

## 36. Semaphore Pattern

Limit concurrency.

```
sem := make(chan struct{}, 5)sem <- struct{}{}go func() {	defer func() {		<-sem	}()	work()}()
```

At most:

* 5 goroutines active

***

## 37. Mutex

Used for shared memory synchronization.

```
var mu sync.Mutex
```

***

## 38. Race Condition

Multiple goroutines:

* read/write same memory
* without synchronization

Example:

```
count++
```

Actually:

```
LOADADDSTORE
```

Not atomic.

***

## 39. Mutex Example

```
var (	count int	mu    sync.Mutex)func inc() {	mu.Lock()	count++	mu.Unlock()}
```

***

## 40. RWMutex

Allows:

* multiple readers
* one writer

```
var mu sync.RWMutex
```

***

## 41. Atomic Operations

For lightweight synchronization.

```
atomic.AddInt64(&counter, 1)
```

Very fast.

***

## 42. Mutex vs Atomic

| Mutex                     | Atomic            |
| ------------------------- | ----------------- |
| General purpose           | Simple operations |
| More overhead             | Extremely fast    |
| Complex critical sections | Counters/flags    |

***

## 43. Go Memory Model

Critical advanced topic.

Defines:

* visibility guarantees
* happens-before relationships

Without synchronization:

* no guarantee another goroutine sees updates.

***

## 44. Happens-Before

Synchronization creates memory visibility.

Examples:

* mutex unlock → lock
* channel send → receive
* atomic ops

***

## 45. Data Race Detection

Run:

```
go run -race main.go
```

Extremely important in production systems.

***

## 46. Deadlock

Classic error:

```
ch := make(chan int)ch <- 10
```

No receiver:\
→ deadlock

***

## 47. Deadlock Detection

Go runtime detects:

* all goroutines asleep

Error:

```
fatal error: all goroutines are asleep - deadlock!
```

***

## 48. Goroutine Leak

One of the MOST IMPORTANT production issues.

Example:

```
func worker(ch chan int) {	<-ch}
```

If nobody sends:

* goroutine leaks forever

***

## 49. Detecting Goroutine Leaks

Use:

* pprof
* runtime.NumGoroutine()
* tracing
* metrics

***

## 50. Context + Leak Prevention

Always:

* propagate context
* support cancellation
* close resources

***

## 51. Pipeline Pattern

```
Stage1 -> Stage2 -> Stage3
```

Each stage:

* goroutine
* channel

***

## 52. Example Pipeline

```
func generator(nums ...int) <-chan int {	out := make(chan int)	go func() {		defer close(out)		for _, n := range nums {			out <- n		}	}()	return out}
```

***

## 53. Backpressure

Very important distributed systems concept.

If producer faster than consumer:

* memory grows
* queues explode

Channels naturally provide backpressure.

***

## 54. Buffered Channel Danger

Huge buffer:

```
make(chan Job, 1000000)
```

Can hide system overload.

Dangerous in production.

***

## 55. Structured Concurrency

Modern concurrency philosophy.

Idea:

* goroutines should have lifecycle ownership

Bad:

```
go backgroundTask()
```

Who owns it?\
Who cancels it?\
Who waits for it?

***

## 56. errgroup Package

Production-grade structured concurrency.

Package:

```
golang.org/x/sync/errgroup
```

***

## 57. errgroup Example

```
g, ctx := errgroup.WithContext(ctx)g.Go(func() error {	return task1(ctx)})g.Go(func() error {	return task2(ctx)})if err := g.Wait(); err != nil {	return err}
```

Benefits:

* cancellation
* error propagation
* lifecycle management

***

## 58. Scheduler Observability

Useful runtime tools:

```
runtime.NumGoroutine()
```

```
go tool trace
```

```
go tool pprof
```

***

## 59. GOMAXPROCS

Controls parallelism.

```
runtime.GOMAXPROCS(4)
```

Usually:

* defaults to CPU count

***

## 60. CPU-Bound vs IO-Bound

### CPU-bound

Examples:

* compression
* encryption
* ML

Need:

* limited goroutines

***

### IO-bound

Examples:

* HTTP
* DB
* gRPC

Can support:

* huge goroutine counts

***

## 61. Channel Ownership Pattern

Producer owns:

* creation
* writing
* closing

Consumer only reads.

This prevents many concurrency bugs.

***

## 62. Select Fairness

Select is pseudo-random.

No strict fairness guarantee.

Can cause:

* starvation edge cases

***

## 63. Nil Channels Trick

Nil channel blocks forever.

Useful in dynamic select.

```
var ch chan intselect {case <-ch:}
```

Never executes.

***

## 64. sync.Once

Ensures one-time execution.

```
var once sync.Onceonce.Do(initDB)
```

***

## 65. sync.Cond

Low-level condition variable.

Rarely needed.\
Usually:

* channels better

But useful for:

* complex signaling

***

## 66. sync.Pool

Object reuse.

Reduces GC pressure.

Useful in:

* high throughput systems
* serialization
* buffers

***

## 67. False Sharing

Advanced performance issue.

Two goroutines modify nearby memory:

* cache line contention occurs

Can severely hurt performance.

***

## 68. Lock Contention

Too many goroutines competing for mutex.

Symptoms:

* high latency
* poor scaling

Solutions:

* sharding
* lock-free structures
* reduce critical section

***

## 69. Channel vs Mutex

Very important interview question.

Use:

* channels for communication/ownership transfer
* mutex for shared state protection

Do NOT force channels everywhere.

***

## 70. Production Concurrency Principles

### Always:

* use context
* bound concurrency
* monitor goroutines
* handle cancellation
* avoid unbounded queues
* use race detector
* measure contention

***

## 71. Common Real Production Patterns

### HTTP Request Fan-Out

```
Client Request   ├── User Service   ├── Payment Service   └── Inventory Service
```

***

### Event Processing

```
Kafka Consumer    ↓Worker Pool    ↓DB Writes
```

***

### Rate Limiting

Using:

* token bucket
* semaphore
* worker pool

***

### Streaming Systems

```
Producer -> Buffer -> Workers -> Sink
```

***

## 72. Important Interview Topics

Must know deeply for senior Go engineer:

* Goroutine scheduler
* Channel internals
* Context propagation
* Memory model
* Happens-before
* Race conditions
* Deadlocks
* Goroutine leaks
* Worker pools
* Backpressure
* errgroup
* Atomic vs mutex
* Channel closing semantics
* Select internals

***

## 73. Recommended Progression

### Beginner

* goroutines
* channels
* select
* mutex
* WaitGroup

***

### Intermediate

* worker pools
* context
* pipelines
* fan-in/fan-out
* race detector

***

### Advanced

* scheduler
* memory model
* atomics
* structured concurrency
* profiling
* tracing

***

### Expert

* lock-free design
* runtime internals
* GC interaction
* cache optimization
* high-throughput architecture

***

## 74. Most Important Advice

In production systems:

> Concurrency bugs are usually lifecycle bugs.

Meaning:

* forgotten cancellation
* leaked goroutines
* blocked channels
* unbounded queues
* uncontrolled fan-out

NOT just mutex issues.

***

## 75. Best Learning Path for You

Given your background in:

* distributed systems
* gRPC
* telecom
* cloud infrastructure
* microservices

You should next study:

1. Go Scheduler Internals
2. Go Memory Model
3. Context Propagation in gRPC
4. High-throughput worker pool design
5. Backpressure systems
6. Kafka consumer concurrency
7. Lock contention optimization
8. pprof + trace
9. Goroutine leak debugging
10. Structured concurrency with errgroup

***

## 76. Recommended Advanced Topics

You should eventually deeply learn:

* Epoll/Kqueue interaction
* Netpoller
* Async preemption
* GC + concurrency interaction
* Memory barriers
* CAS operations
* Lock-free queues
* Actor model
* Reactive systems
* CSP vs Actor architecture

***

## 77. One Important Real-World Rule

This is critical:

```
Concurrency increases complexity non-linearly.
```

A concurrent system:

* is harder to debug
* harder to reason about
* harder to test

So:

* use concurrency only where beneficial
* prefer simplicity
* bound everything

***

## 78. Final Mental Model

Think of Go concurrency as:

```
Goroutines = lightweight tasksChannels = coordinationContext = lifecycle managementScheduler = execution engineMutex/Atomic = memory synchronization
```

And production concurrency engineering is mostly about:

```
controlling lifecycle + controlling load
```

not merely launching goroutines.

<br>
