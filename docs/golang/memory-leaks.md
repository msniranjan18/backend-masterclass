# Memory Leaks



In Go, a “memory leak” usually does **not** mean unreachable memory that can never be freed (like classic C/C++ leaks). Go has a garbage collector (GC), so unreachable objects are reclaimed automatically.

In Go, memory leaks mostly happen when:

* Memory is still **reachable** accidentally.
* Goroutines never terminate.
* Resources are not closed.
* Large backing arrays/maps stay referenced.
* Unbounded caching/buffering occurs.
* GC cannot reclaim memory because references still exist.

***

## 1. Understanding Go Memory Model First

Before learning leaks, understand how Go manages memory.

### Stack vs Heap

#### Stack

* Fast allocation/deallocation
* Function-local variables
* Automatically cleaned when function returns

#### Heap

* Used for objects that outlive function scope
* Managed by Go Garbage Collector

Example:

```go
func foo() *int {
    x := 10
    return &x // escapes to heap
}
```

`x` escapes to heap because it survives function return.

***

## 2. What Is a Memory Leak in Go?

A memory leak means:

> Memory that is no longer useful to the program but is still referenced, preventing GC from reclaiming it.

***

## 3. Common Types of Memory Leaks in Go

## A. Goroutine Leaks (MOST COMMON)

This is the #1 leak in production Go systems.

### Example

```go
func worker(ch chan int) {
    <-ch // waiting forever}func main() {    
    ch := make(chan int)    
    go worker(ch)    // nobody sends data
}
```

The goroutine waits forever.

#### Why This Leaks

* Goroutine stack memory remains allocated
* Scheduler metadata remains
* References held by goroutine remain alive

Thousands of leaked goroutines = huge memory growth.

***

### Real Production Scenario

#### HTTP Call Without Timeout

```go
resp, err := http.Get(url)
```

Without timeout:

* request may hang forever
* goroutine blocked forever

***

### Proper Fix

```go
client := http.Client{
    Timeout: 5 * time.Second,
}
```

Or use context:

```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
```

***

## B. Channel Deadlocks Causing Leaks

### Example

```go
func producer(ch chan int) {
    for i := 0; i < 100; i++ {
            ch <- i    
    }
}

func main() {
    ch := make(chan int)    
    go producer(ch)    
    time.Sleep(time.Hour)
}
```

Producer blocks forever because no receiver exists.

***

### Fix

Always ensure:

* receivers exist
* channels close properly
* cancellation mechanism exists

***

## C. Slice Backing Array Leaks

Very important and commonly asked in interviews.

***

### Problem

Slices internally reference an array.

Even a tiny slice can keep a huge array alive.

### Example

```go
func leak() []byte {
    big := make([]byte, 100<<20) // 100 MB    
    return big[:10]
}
```

Returned slice is only 10 bytes.

BUT:

* underlying 100MB array still referenced
* GC cannot free it

***

### Visualization

```
[100MB backing array] 
^ slice header still points here
```

***

### Proper Fix

Copy required data:

```go
func fixed() []byte {
    big := make([]byte, 100<<20)    
    small := make([]byte, 10)    
    copy(small, big[:10])    
    return small
}
```

Now 100MB array can be GC’d.

***

## D. Substring Memory Leak

Older Go versions had this issue more heavily.

### Example

```go
func leak() string {
    big := readHugeFile()    
    return big[:100]
}
```

Small substring references huge original string data.

***

### Fix

```go
small := string([]byte(big[:100]))
```

Forces new allocation.

***

## E. Map Growth Leaks

Maps do not shrink automatically.

### Example

```go
m := make(map[string][]byte)
for {
    key := randomKey()    
    m[key] = hugeData()
}
```

Memory continuously grows.

***

### Real Problems

* unbounded cache
* session storage
* metrics labels
* user tracking

***

### Fixes

#### 1. Use eviction policy

LRU cache:

* Ristretto
* groupcache

#### 2. Periodic cleanup

#### 3. TTL expiration

***

## F. `time.After()` Leak

Classic Go issue.

### Bad

```go
for {
    select {
        case <-time.After(time.Second):
                fmt.Println("tick")    
    }
}
```

Every iteration allocates a new timer.

Large loops = timer accumulation.

***

### Proper Fix

```go
ticker := time.NewTicker(time.Second)
defer ticker.Stop()
for {
    select {
        case <-ticker.C:
        fmt.Println("tick")
    }
}
```

***

## G. Missing `Close()` Calls

Extremely common in backend services.

***

### File Descriptor Leak

```go
f, _ := os.Open("file.txt")
```

If not closed:

* OS resource leak
* buffers remain
* eventually process crashes

***

### Fix

```go
f, err := os.Open("file.txt")
if err != nil {
    return err
}
defer f.Close()
```

***

### Common Resources Requiring Close

* files
* DB rows
* HTTP response body
* network connections
* Kafka consumers
* gRPC streams

***

## H. HTTP Response Body Leak

Very important for backend engineers.

### Bad

```go
resp, err := http.Get(url)
if err != nil {
    return err
}
```

Without:

```
defer resp.Body.Close()
```

connection stays occupied.

***

### Consequences

* connection pool exhaustion
* goroutine buildup
* memory growth
* socket exhaustion

***

## I. Context Leak

### Bad

```go
ctx, cancel := context.WithTimeout(context.Background(), time.Minute)
// forgot cancel()
```

Timers/resources remain alive.

***

### Proper

```go
ctx, cancel := context.WithTimeout(context.Background(), time.Minute)
defer cancel()
```

***

## J. Global References

### Bad

```go
var cache [][]byte
```

Appending forever:

```go
cache = append(cache, hugeObject)
```

Objects never become unreachable.

***

## K. Closures Holding References

### Example

```go
func main() {
    huge := make([]byte, 100<<20)    
    go func() {
            fmt.Println(len(huge))    
    }()
}
```

Goroutine closure keeps huge object alive.

***

## 4. How Garbage Collector Works

Understanding GC is critical for debugging leaks.

***

## Go GC Basics

Go uses:

* concurrent
* tri-color mark-and-sweep GC

***

## Tri-Color Algorithm

Objects categorized as:

| Color | Meaning                    |
| ----- | -------------------------- |
| White | candidate for collection   |
| Gray  | discovered but not scanned |
| Black | scanned and alive          |

GC traverses reachable objects from:

* stacks
* globals
* registers
* goroutines

Anything reachable survives.

***

## Important Insight

GC only removes:

* unreachable memory

NOT:

* unnecessary but still referenced memory

That is the root of most Go leaks.

***

## 5. Detecting Memory Leaks

## A. Runtime Memory Stats

```go
var m runtime.MemStatsruntime.ReadMemStats(&m)
fmt.Println(m.Alloc)
fmt.Println(m.HeapAlloc)
```

***

## B. pprof (MOST IMPORTANT)

Essential production skill.

***

## Enable pprof

```go
import _ "net/http/pprof"
go func() {
    log.Println(http.ListenAndServe("localhost:6060", nil))
}()
```

***

## Heap Profile

```go
go tool pprof http://localhost:6060/debug/pprof/heap
```

***

## Useful Commands

### Top Allocators

```
top
```

### Allocation Source

```
list functionName
```

### Call Graph

```
web
```

***

## C. Goroutine Leak Detection

```
go tool pprof http://localhost:6060/debug/pprof/goroutine
```

or:

```
curl localhost:6060/debug/pprof/goroutine?debug=2
```

***

## D. Trace Tool

```
go test -trace trace.out
```

Then:

```
go tool trace trace.out
```

Useful for:

* scheduler analysis
* blocking operations
* goroutine lifecycle

***

## E. GODEBUG

Very useful in debugging GC.

### Example

```
GODEBUG=gctrace=1 ./app
```

Shows:

* GC frequency
* heap size
* pause times

***

## 6. Interview-Level Deep Concepts

## Memory Retention vs Memory Leak

Important distinction.

### Leak

Memory unintentionally retained forever.

### Retention

Memory temporarily retained longer than needed.

***

## Heap Fragmentation

Even after GC:

* RSS memory may remain high
* allocator fragmentation possible

Go runtime may not immediately return memory to OS.

***

## Why Memory Doesn't Drop After GC

Very common confusion.

Reasons:

* heap reserved for reuse
* fragmentation
* runtime optimization

Use:

```
debug.FreeOSMemory()
```

only for debugging.

***

## 7. Production Leak Patterns

## A. Worker Pool Without Shutdown

```go
for {
    go process()
}
```

Unbounded goroutine creation.

***

## B. Kafka Consumer Leak

Not closing:

* partitions
* consumers
* channels

***

## C. gRPC Stream Leak

Long-running streams without cancellation.

Especially important in:

* telecom
* 5G core
* cloud infra

(Highly relevant to your background in SMF/cloud systems.)

***

## D. Metrics Cardinality Explosion

Example:

```
prometheus.CounterVec.WithLabelValues(userID)
```

Millions of unique labels = huge memory usage.

***

## E. Log Buffer Leak

Buffered async logging without flush/rotation.

***

## 8. Advanced pprof Analysis

## Heap Profile Types

### inuse\_space

Current live heap memory.

### alloc\_space

Total historical allocations.

***

## Example

```
go tool pprof -inuse_space heap.out
```

***

## Finding Leak Growth

Take multiple snapshots:

```
curl -o heap1.out ...sleep 60curl -o heap2.out ...
```

Compare:

```
go tool pprof -base heap1.out heap2.out
```

***

## 9. Best Practices to Avoid Leaks

## Goroutines

* Always have cancellation
* Use contexts
* Avoid fire-and-forget goroutines

***

## Channels

* Close when appropriate
* Avoid blocked senders/receivers

***

## Slices

* Copy small data from huge arrays

***

## Maps

* Add eviction
* Use bounded cache

***

## Resources

Always close:

* files
* DB rows
* HTTP bodies
* streams

***

## Context

Always call:

```
defer cancel()
```

***

## Timers

Use:

* `NewTicker`
* `Stop()`

instead of repeated `time.After`.

***

## 10. Real Production Debugging Strategy

When memory increases in production:

***

### Step 1

Check:

* RSS
* heap growth
* goroutine count

***

### Step 2

Capture:

* heap profile
* goroutine profile

***

### Step 3

Compare profiles over time.

***

### Step 4

Identify:

* growing object type
* allocation path

***

### Step 5

Check:

* blocked goroutines
* caches
* channels
* unclosed resources

***

## 11. Example: Full Goroutine Leak

### BAD

```
func fetch(url string) {    resp, _ := http.Get(url)    defer resp.Body.Close()}func main() {    for {        go fetch("http://slow-service")    }}
```

Problems:

* unbounded goroutines
* no timeout
* scheduler overload
* memory growth

***

## FIXED

```go
var sem = make(chan struct{}, 100)
func fetch(url string) {
    sem <- struct{}{}    
    defer func() {
     <-sem }()
    client := http.Client{        
          Timeout: 5 * time.Second,    
    }
    
    resp, err := client.Get(url)
    if err != nil {        
          return    
    }    
    
    defer resp.Body.Close()
}
```

Now:

* bounded concurrency
* timeout protection
* stable memory

***

## 12. Key Senior-Level Takeaways

A senior Go engineer should deeply understand:

* GC behavior
* escape analysis
* goroutine lifecycle
* slice backing arrays
* pprof usage
* heap profiling
* resource lifecycle
* context cancellation
* production debugging

***

## 13. Must-Know Interview Questions

### Why can small slices leak huge memory?

Because slices reference backing arrays.

***

### Why are goroutine leaks dangerous?

Each goroutine consumes:

* stack
* scheduler metadata
* retained references

***

### Why doesn't Go automatically solve all memory leaks?

GC only frees unreachable objects.

***

### Difference between memory leak and high allocation rate?

Leak:

* memory continuously retained

High allocation:

* memory frequently allocated/freed

***

### How do you debug Go memory leaks?

Using:

* pprof
* runtime stats
* goroutine dump
* heap comparison

***

## 14. Essential Tools Every Go Backend Engineer Must Know

* pprof
* go tool trace
* Prometheus
* Grafana

***

## 15. Final Mental Model

The most important concept:

> In Go, leaks usually happen not because memory is forgotten,\
> but because memory is accidentally remembered.

That single idea explains most real-world Go memory leaks.
