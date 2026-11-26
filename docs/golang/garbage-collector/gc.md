# Garbage Collector

**Garbage Collector Internals & High-Performance Memory Tuning**

## Table of Contents

1. **Chapter 1: Stack vs. Heap & Escape Analysis (Stopping Garbage Before It Starts)**
   * How Go Decides Where Memory Lives
   * The 5 Common Reasons Variables "Escape" to the Heap
   * Reading Compiler Escape Analysis Output with Real Go Examples
2. **Chapter 2: How Go's Garbage Collector Actually Works Under the Hood**
   * Why Go Chose a Concurrent, Non-Compacting Tri-Color Mark-and-Sweep GC
   * The Tri-Color Marking Algorithm (White, Grey, and Black Sets)
   * The Write Barrier & Stop-The-World (STW) Phases Explained Simply
   * Mark Assist: Why Heavy Allocators Suddenly Slow Down
3. **Chapter 3: The GC Pacer (`GOGC` and `GOMEMLIMIT`)**
   * How the GC Pacer Decides *When* to Run
   * Understanding `GOGC` (The Growth Percentage Knob)
   * Understanding `GOMEMLIMIT` (Preventing OOM Crashes in Containers)
   * Avoiding the "GC Death Spiral" (Thrashing)
4. **Chapter 4: Memory Layout, False Sharing & Pointer-Free Optimization**
   * Why Pointers Make the Garbage Collector Work 10x Harder
   * Pointer-Free Maps and Slices (`map[int]int` vs `map[string]*User`)
   * Struct Field Alignment & Cache Line Padding
5. **Chapter 5: Advanced Zero-Allocation & Low-GC Patterns in Go**
   * `sync.Pool` Done Right (and the Memory Leak Trap to Avoid)
   * Slice Reuse (`s[:0]`) and Pre-Allocation
   * `strings.Builder` vs. `bytes.Buffer` vs. `strconv.Append*`
   * Weak Pointers (`weak.Make`) & `runtime.AddCleanup` (Modern Go Memory Management)
6. **Chapter 6: Live Production Debugging with `pprof`, `runtime/metrics`, and Execution Traces**
   * `inuse_space` vs. `alloc_space` (Finding CPU-Burning Garbage)
   * Reading `GODEBUG=gctrace=1` Line by Line
   * Exporting Live GC Metrics with `runtime/metrics`

---

## Chapter 1: Stack vs. Heap & Escape Analysis (Stopping Garbage Before It Starts)

The fastest garbage collection in Go is **memory that the Garbage Collector never has to touch**. To understand how to write zero-allocation Go code, you first need to understand the two places Go stores variables: the **Stack** and the **Heap**.

### 1.1 The Stack vs. The Heap (A Simple Mental Model)

* **The Stack (Fast, Free, Automatic):** Every goroutine gets its own tiny private stack (starting at just 2 KB). When a function runs, its local variables are pushed onto the stack. When the function returns, the stack pointer simply moves back down. **Zero Garbage Collector involvement, zero CPU overhead, and zero locks.**
* **The Heap (Shared, Tracked by GC):** If a variable needs to outlive the function that created it, or if the compiler cannot prove its lifetime at compile time, Go moves it to the shared Heap. Every object on the heap must eventually be scanned, marked, and swept by Go's Garbage Collector.

### 1.2 What is Escape Analysis?

During compilation (`go build`), the Go compiler inspects every variable to answer one question: *"Can I prove 100% that nothing will access this variable after its function returns?"*

* If **YES**: It stays on the **Stack**.
* If **NO**: It **"escapes to the heap."**

#### The 5 Most Common Reasons Variables Escape to the Heap:

1. **Returning a Pointer (`return &user`):** The function ends, so its stack frame disappears, yet the caller still holds a pointer to `user`. It *must* live on the heap.
2. **Sending Pointers Over Channels (`ch <- &event`):** The compiler cannot know which goroutine will read from the channel or when, so the data escapes.
3. **Storing Values in `any` / `interface{}`:** Calling `fmt.Println(x)` or `json.Marshal(x)` boxes `x` into an `any` interface, which almost always forces `x` onto the heap.
4. **Slices/Maps Whose Size is Unknown at Compile Time:** `make([]byte, n)` where `n` comes from a variable (like an HTTP header) escapes to the heap because stack frames need predictable sizes.
5. **Closures Capturing Mutated Variables:** If a background goroutine closure modifies a variable from an outer function, that variable moves to the heap so both functions share the same memory address.

### 1.3 Inspecting Escape Analysis in Real Code

You can ask the Go compiler to show you every single heap escape using `-gcflags="-m"`.

```go
package main

import "fmt"

type Order struct {
	ID     int64
	Amount int64
}

// 1. Stays on the STACK: returns a copy by value (small struct, no pointers)
func createOrderOnStack(id, amount int64) Order {
	o := Order{ID: id, Amount: amount}
	return o
}

// 2. Escapes to the HEAP: returns a pointer outliving the function frame
func createOrderOnHeap(id, amount int64) *Order {
	o := Order{ID: id, Amount: amount} // compiler output: "moved to heap: o"
	return &o
}

// 3. Surprise Heap Escape: passing a stack variable to fmt.Println (takes ...any)
func logOrder(o Order) {
	// Boxing 'o' into an interface{} forces a heap allocation!
	fmt.Println(o.ID) // compiler output: "o.ID escapes to heap"
}

func main() {
	sOrder := createOrderOnStack(1, 500)
	hOrder := createOrderOnHeap(2, 900)
	logOrder(sOrder)
	_ = hOrder
}
```

Run `go build -gcflags="-m" main.go` to see the compiler explain its decisions:
```text
./main.go:17:2: moved to heap: o
./main.go:24:15: o.ID escapes to heap
```

> **Production Rule of Thumb:** Passing small structs (under ~64–128 bytes) **by value** instead of by pointer is often *faster* in Go because copying a few bytes on the CPU cache is dramatically cheaper than allocating on the heap and forcing the GC to track pointers.

---

## Chapter 2: How Go's Garbage Collector Actually Works Under the Hood

Many developers coming from Java or C# assume Go's GC freezes the entire application for hundreds of milliseconds while moving objects around in memory. **Go's GC does neither.**

### 2.1 Go's GC Design Philosophy

Go uses a **Concurrent, Non-Compacting, Tri-Color Mark-and-Sweep Garbage Collector**:

1. **Concurrent:** The GC runs *at the exact same time* as your application goroutines on background CPU cores (typically using ~25% of available CPU while active).
2. **Non-Compacting (Does Not Move Memory):** Go never relocates objects in memory to defragment them. Because objects never move, Go doesn't have to pause your program to rewrite pointers everywhere. Instead, Go avoids memory fragmentation using a `tcmalloc`-style **size-class allocator** (pre-dividing memory pages into fixed buckets of 8B, 16B, 32B, 64B, etc.).
3. **Sub-Millisecond Pauses:** Stop-The-World (STW) pauses in modern Go are typically **under 100 microseconds (0.1ms)**—even on heaps with tens of gigabytes of RAM.

### 2.2 The Tri-Color Marking Algorithm (White, Grey, and Black)

Imagine all objects on your heap as a giant spiderweb of pointers starting from **GC Roots** (global variables and active goroutine stacks). Go paints every object one of three colors to figure out what is still alive:

* **White Set (Candidates for Deletion):** At the start of a GC cycle, *every* object on the heap is painted **White**. If an object is still White when marking finishes, it is unreachable garbage and its memory will be reclaimed.
* **Grey Set (Work Queue — Visited, Not Finished):** Objects reachable from a root, *whose outgoing pointers have not been scanned yet*.
* **Black Set (Confirmed Alive — Done):** Objects reachable from a root, *and all of their outgoing pointers have already been scanned*. **Rule: A Black object is NEVER allowed to point directly to a White object.**

#### Step-by-Step Cycle:
1. **Start:** Roots (active stack variables, globals) are painted **Grey**. Everything else is **White**.
2. **Scan:** The GC picks a **Grey** object, looks at every pointer inside it, paints those target objects **Grey**, and paints the current object **Black**.
3. **Repeat:** This continues until the **Grey Set is completely empty**.
4. **Sweep:** Every remaining **White** object is unlinked and returned to Go's free memory pools for reuse.

### 2.3 Stop-The-World (STW) & The Write Barrier

Because your application goroutines are still running and modifying pointers *while* the GC is painting objects, a sneaky race condition could happen:
1. Your goroutine takes a pointer to a **White** object `C` and writes it into a **Black** object `A`.
2. Then your goroutine deletes the original pointer to `C` from a **Grey** object `B`.
3. Because `A` is already Black, the GC won't scan `A` again—so `C` stays White and gets deleted even though `A` still points to it!

**How Go Prevents This (The Hybrid Write Barrier):**
* **STW #1 (~50 microseconds):** Go briefly pauses goroutines to turn on the **Write Barrier**.
* **While Marking Runs:** Whenever your code executes a pointer assignment (`a.child = c`), the compiler silently runs a tiny 3-instruction check (the Write Barrier) that immediately paints *both* the old target and the new target **Grey**. That way, no live object can ever slip through the cracks!
* **STW #2 (~50 microseconds):** Once the Grey set is empty, Go briefly pauses to flush thread-local caches and turn off the Write Barrier.
* **Concurrent Sweep:** Background workers reclaim White memory lazily as your program allocates new objects.

### 2.4 Mark Assist (The Hidden Cause of Latency Spikes)

What happens if your application goroutines allocate new heap memory **faster** than the background GC workers can mark existing memory?

If Go let your code keep allocating unchecked, your process would run out of RAM and crash with an Out-Of-Memory (OOM) kill before the GC finished! To prevent this, Go triggers **Mark Assist**:
* Any goroutine trying to allocate heap memory is **temporarily drafted to do GC marking work first** before its `new()` or `make()` call is allowed to complete.
* **In Production:** If you see random p99 latency spikes on a fast endpoint, check if that handler is allocating massive bursts of heap memory and getting throttled by Mark Assist!

---

## Chapter 3: The GC Pacer (`GOGC` and `GOMEMLIMIT`)

How does Go know *when* to start a new Garbage Collection cycle? If it runs too often, your CPU gets wasted scanning memory. If it waits too long, your RAM usage balloons. Go solves this with a feedback controller called the **GC Pacer**.

### 3.1 `GOGC`: Controlling CPU vs. Memory Trade-off

`GOGC` (default: `100`) sets the **target heap growth percentage** relative to the size of **live memory** left over after the previous GC cycle:

$$\text{Target Heap Size} = \text{Live Heap} \times \left(1 + \frac{\text{GOGC}}{100}\right)$$

* Suppose after a GC sweep, your app has **100 MB** of genuinely live data in RAM.
* With `GOGC=100` (default): Go lets the heap grow by another 100% until it hits **200 MB**, then triggers the next GC cycle.
* With `GOGC=200`: Go lets the heap grow by 200% up to **300 MB**. You use 50% more RAM, but the GC runs **half as often**, saving CPU!
* With `GOGC=50`: Go triggers GC at **150 MB**. You save RAM, but the GC runs twice as often.

### 3.2 `GOMEMLIMIT`: Preventing Container OOM Kills

`GOGC` alone has a major weakness in Docker and Kubernetes: **it has no idea how much RAM your container actually has!**
If your Kubernetes pod has a `1 GiB` memory limit, and your live heap suddenly spikes to `600 MB`, `GOGC=100` will try to let the heap grow to `1.2 GiB`—causing the Linux kernel to immediately kill your pod (`OOMKilled`, Exit Code 137).

**`GOMEMLIMIT` (introduced in Go 1.19) solves this completely:**
* You set `GOMEMLIMIT` to roughly **80%–90% of your container's hard memory limit** (e.g., `900MiB` for a `1GiB` container).
* You can even set `GOGC=off` alongside `GOMEMLIMIT=900MiB`! When you do this, Go skips unnecessary GC cycles completely while memory is plentiful, and smoothly runs GC just in time to keep total memory under `900MiB`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-service
spec:
  containers:
  - name: app
    image: my-service:latest
    resources:
      limits:
        memory: "1Gi"  # Hard limit
    env:
    # Set GOMEMLIMIT to 900MiB (90% of 1Gi)
    - name: GOMEMLIMIT
      value: "900MiB"
```

**Best Practice:**
The text suggests GOGC=off, but in production, keep GOGC at its default (100) unless your memory is extremely stable and low-volume.

Why not GOGC=off? GOGC=off disables the growth-based pacing. If you have a memory spike or a slow leak, the GC will never run until it hits GOMEMLIMIT. This can lead to massive, infrequent pauses.

Let GOMEMLIMIT act as the hard "ceiling" and GOGC act as the "smooth operator" that runs frequently to keep memory low below that ceiling.

### 3.3 Avoiding the "GC Death Spiral" (Thrashing)

What if your application genuinely holds `895 MiB` of live, active data in RAM, and `GOMEMLIMIT` is `900 MiB`?
Without protection, Go would run GC non-stop trying to free 5 MiB, burning 100% of your CPU while making zero forward progress (known as **GC Thrashing**).

To prevent a total freeze, Go's runtime enforces a hard **50% CPU ceiling** on the Garbage Collector:
* Over a sliding time window, Go guarantees the GC will never consume more than 50% of total CPU time.
* If memory still exceeds `GOMEMLIMIT` even with the GC working at 50% CPU capacity, Go intentionally allows the memory limit to be breached (so the OS OOM-killer cleanly restarts the unhealthy container rather than leaving a zombie pod hung at 100% CPU).

```go
package main

import (
	"log/slog"
	"os"
	"runtime/debug"
)

// ConfigureRuntimeMemory sets a soft memory ceiling programmatically if not
// already provided via the GOMEMLIMIT environment variable.
func ConfigureRuntimeMemory(containerLimitBytes int64) {
	// Reserve ~15% headroom for OS overhead, goroutine stacks, and cgo/network buffers
	softLimit := int64(float64(containerLimitBytes) * 0.85)

	if os.Getenv("GOMEMLIMIT") == "" {
		prev := debug.SetMemoryLimit(softLimit)
		slog.Info("configured runtime memory limit",
			slog.Int64("previous_limit", prev),
			slog.Int64("new_limit_bytes", softLimit),
		)
	}
}
```

---

## Chapter 4: Memory Layout, False Sharing & Pointer-Free Optimization

### 4.1 Why Pointers Make the GC Work 10x Harder

During the **Mark** phase, the Garbage Collector only has to follow **pointers**.
* If you allocate a `[]byte` slice of **100 MB**, the GC looks at the slice header, sees that `byte` contains **zero pointers**, and **skips scanning all 100 MB completely!** Scanning time: ~1 nanosecond.
* If you allocate a `[]*User` slice with **1 million pointers**, the GC must dereference and inspect **every single one of those 1 million pointers** on every GC cycle. Scanning time: tens of milliseconds.

#### Production Optimization: Pointer-Free Maps and Slices
Go's runtime marks any type without pointers as **`noscan`**. Look at how changing a map type eliminates GC scanning overhead entirely:

```go
package cache

// BAD FOR GC: Both string (contains a data pointer) and *Session (pointer)
// force the GC to scan every single key and value in the map on every cycle.
type PointerHeavyCache struct {
	entries map[string]*Session
}

type Session struct {
	UserID    int64
	ExpiresAt int64
	RoleID    int32
}

// GOOD FOR GC: Neither uint64 nor Session contains any pointers!
// Go marks the entire map bucket memory as "noscan" — the GC skips it completely,
// even if the map holds 10 million sessions!
type PointerFreeCache struct {
	entries map[uint64]Session
}
```

> **Pro Tip for String Keys:** If you have a huge in-memory lookup table with millions of string keys (like a dictionary or routing table), packing all strings into one single giant `string` (or `[]byte`) and storing integer offsets `(start, length)` reduces millions of GC-scanned string pointers down to **one single pointer**!

### 4.2 Struct Field Alignment (Shrinking Object Footprints)

On 64-bit CPUs, Go aligns struct fields to multiples of their size (`int64` aligns to 8 bytes, `int32` to 4 bytes). Ordering fields poorly inserts invisible empty padding bytes into every struct instance—wasting RAM and triggering GC sooner.

```go
package layout

// UnoptimizedStruct takes 24 bytes per instance due to alignment padding!
type UnoptimizedStruct struct {
	Active bool   // 1 byte + 7 bytes empty padding
	Count  int64  // 8 bytes
	Code   uint8  // 1 byte + 7 bytes empty padding
}

// OptimizedStruct stores the exact same data in 16 bytes (33% smaller)!
// Rule: Order fields from largest alignment (8 bytes) down to smallest (1 byte).
type OptimizedStruct struct {
	Count  int64  // 8 bytes
	Active bool   // 1 byte
	Code   uint8  // 1 byte + 6 bytes padding
}
```

#### How Go lays this out in memory:
random order fields (`UnoptimizedStruct`)

- `Active` (bool) takes 1 byte.
- Padding: To keep Count aligned to 8 bytes, Go inserts 7 empty bytes after Active.
- `Count` (int64) takes 8 bytes.
- Padding: To keep Code aligned (though Code is small, the struct itself often aligns to the largest field), and because Code is only 1 byte, Go adds 7 more bytes of padding at the end to make the total struct size a multiple of 8.

**Total Size:** 1 (bool) + 7 (padding) + 8 (int64) + 1 (uint8) + 7 (padding) = 24 bytes.

You are paying for 14 bytes of empty space that holds no data.

The Solution: Sort Fields by Size (`OptimizedStruct`)
If you reorder the fields from largest to smallest, you can pack them tighter:
- Count (int64) takes 8 bytes.
- Active (bool) takes 1 byte.
- Code (uint8) takes 1 byte.
- Padding: Only 6 bytes of padding at the end to align the total struct size to 8 bytes.

**Total Size:** 8 + 1 + 1 + 6 = 16 bytes.

**Result:** You saved 8 bytes per instance (33% reduction).

#### Why Does This Matter in Production?
- **RAM Savings: If you have 1 million instances of this struct, the unoptimized version uses 24 MB. The optimized version uses 16 MB. That’s 8 MB of wasted RAM just on padding.
- **GC Pressure:** The Garbage Collector has to scan more memory. Even though it doesn’t scan the padding bytes, it still has to track the memory pages. Less memory = faster GC cycles.
- **Cache Efficiency:** CPU caches are limited. If your struct is smaller, more instances fit into the same cache line. This means fewer cache misses when your code loops through data.

#### The Golden Rule for Structs
Always order struct fields from largest alignment to smallest.

`int64`, `uint64`, `float64`, `pointer`, `slice`, `map`, `struct` (8 bytes)
`int32`, `uint32`, `float32` (4 bytes)
`int16`, `uint16` (2 bytes)
`int8`, `uint8`, `bool` (1 byte)

#### When Can You Ignore This?
- Small structs (< 16 bytes): The difference is negligible.
- Memory is cheap: If you aren’t running a high-scale service, it doesn’t matter.
- You use sync.Pool or arena allocators: These often handle alignment internally.

But for production-grade Go, especially in performance-critical paths (like network packets, database rows, or cache entries), always sort your fields. It’s a free optimization.

---

## Chapter 5: Advanced Zero-Allocation & Low-GC Patterns in Go

### 5.1 `sync.Pool` Done Right (And the Memory Leak Trap)

`sync.Pool` lets goroutines borrow and return temporary objects (like buffers or encoders) so you don't allocate new heap memory on every HTTP/gRPC request. When a GC cycle runs, Go automatically clears idle items from `sync.Pool` so unused memory isn't hoarded forever.

Imagine a busy restaurant kitchen.
- **Without `sync.Pool`:** Every chef buys a new knife for every single dish, then throws it away. (Expensive, wasteful).
- **With `sync.Pool`:** Chefs borrow a knife from a shared shelf, use it, and put it back. Next chef grabs it from the shelf. No new knives needed.

```go
var bufferPool = sync.Pool{
    New: func() interface{} {
        // If the pool is empty, create a new 1KB buffer
        return make([]byte, 1024)
    },
}

func handleRequest() {
    // 1. Borrow a buffer (if available) or create a new one
    buf := bufferPool.Get().([]byte)
    
    // 2. Use the buffer
    fmt.Println("Processing request with buffer...")
    
    // 3. Reset the buffer for the next user
    buf = buf[:0] 
    
    // 4. Return it to the pool (do NOT nil it out!)
    bufferPool.Put(buf)
}
```

#### The #1 `sync.Pool` Production Bug: Buffer Bloat
Suppose 99.9% of your requests need a **4 KB** buffer, so your pool holds 4 KB buffers. Suddenly, one user uploads a **50 MB** payload. Your buffer grows to 50 MB, and when the request finishes, `defer pool.Put(buf)` puts that **50 MB buffer back into the pool**! Now, the pool holds a 50 MB buffer. All future requests borrow this huge buffer, wasting memory even if they only need 4 KB.
Result: Memory usage stays high, and the GC can’t free it because the pool is holding it.

**The Fix:** Always check capacity before returning a buffer to the pool, and reset its length buf = buf[:0] (`buf.Reset()`).


```go
package mempool

import (
	"bytes"
	"sync"
)

const (
	defaultBufSize = 4 * 1024  // 4 KB
	maxPooledSize  = 64 * 1024 // 64 KB cap to prevent memory bloat
)

var bufferPool = sync.Pool{
	New: func() any {
		return bytes.NewBuffer(make([]byte, 0, defaultBufSize))
	},
}

func AcquireBuffer() *bytes.Buffer {
	buf := bufferPool.Get().(*bytes.Buffer)
	buf.Reset()
	return buf
}

func ReleaseBuffer(buf *bytes.Buffer) {
	if buf == nil {
		return
	}
	// Drop abnormally large buffers so the GC can reclaim them
	if buf.Cap() > maxPooledSize {
		return
	}
	buf.Reset()
	bufferPool.Put(buf)
}
```

### 5.2 Reusing Slices (`s[:0]`) & `strconv.Append*`

Inside tight loops (like batch processors or workers), clearing a slice with `s = nil` throws away its backing array and forces a fresh heap allocation on the next iteration. Reslicing with `s = s[:0]` keeps the allocated capacity ready for reuse.

Similarly, instead of `fmt.Sprintf("user:%d", id)` (which parses a format string at runtime and allocates a new string on the heap), use `strconv.AppendInt` into a stack-allocated byte buffer:

```go
package fastfmt

import "strconv"

// FormatCacheKey builds "tenant:12345:user:67890" with ZERO heap allocations
// until the final string conversion (or zero allocations if used as []byte).
func FormatCacheKey(tenantID, userID int64) []byte {
	// Stack-allocated 64-byte scratchpad (never touches the heap!)
	var scratch [64]byte
	b := scratch[:0]

	b = append(b, "tenant:"...)
	b = strconv.AppendInt(b, tenantID, 10)
	b = append(b, ":user:"...)
	b = strconv.AppendInt(b, userID, 10)

	// Return a copy if it needs to outlive this function, or pass 'b' directly
	// to hash functions (like xxhash.Sum64(b)) for 0 allocations total!
	out := make([]byte, len(b))
	copy(out, b)
	return out
}
```

### 5.3 Weak Pointers (`weak.Make`) for Memory-Safe Caches

If you build an in-memory cache using a normal `map[string]*BigObject`, **the Garbage Collector will NEVER delete those objects**, even if no active request is using them anymore—because the map itself holds a strong pointer!

In modern Go (`weak` package, introduced in Go 1.24), **`weak.Pointer[T]`** lets a cache reference an object *without* preventing the Garbage Collector from reclaiming it when memory is needed:

```go
package weakcache

import (
	"sync"
	"weak"
)

type Document struct {
	ID      string
	Content []byte
}

// WeakCache automatically allows the Go Garbage Collector to reclaim unused
// documents under memory pressure without manual eviction timers.
type WeakCache struct {
	mu    sync.RWMutex
	items map[string]weak.Pointer[Document]
}

func NewWeakCache() *WeakCache {
	return &WeakCache{
		items: make(map[string]weak.Pointer[Document]),
	}
}

func (c *WeakCache) Set(id string, doc *Document) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.items[id] = weak.Make(doc)
}

func (c *WeakCache) Get(id string) (*Document, bool) {
	c.mu.RLock()
	wp, exists := c.items[id]
	c.mu.RUnlock()

	if !exists {
		return nil, false
	}

	// Value() returns nil if the GC has already reclaimed the Document!
	doc := wp.Value()
	if doc == nil {
		// Clean up the dead map entry lazily
		c.mu.Lock()
		delete(c.items, id)
		c.mu.Unlock()
		return nil, false
	}

	return doc, true
}
```

---

## Chapter 6: Live Production Debugging with `pprof`, `runtime/metrics`, and Execution Traces

When tuning GC in production, never guess—measure using Go's built-in runtime diagnostics.

### 6.1 `inuse_space` vs. `alloc_space` (The Most Important Profiling Distinction)

When you open a Go heap profile (`go tool pprof`), there are two completely different views:

1. **`-inuse_space` (Default):** Shows what is sitting in RAM **right now**. Use this to debug **Memory Leaks / OOM crashes** (e.g., an unbounded map or leaked goroutine holding onto RAM forever).
2. **`-alloc_space`:** Shows **cumulative bytes allocated since the process started**, even if the GC already freed them 1 millisecond later! Use this to debug **High CPU Usage and Tail-Latency Spikes** caused by short-lived garbage forcing the GC to run constantly.

```bash
# 1. Capture a 30-second delta allocation profile to see what is burning GC CPU right now:
go tool pprof -alloc_space http://localhost:6060/debug/pprof/heap?seconds=30

# 2. Capture a live snapshot to find memory leaks:
go tool pprof -inuse_space http://localhost:6060/debug/pprof/heap
```

### 6.2 Reading `GODEBUG=gctrace=1` Line by Line

Running your binary with `GODEBUG=gctrace=1` prints a single summary line to `stderr` every time a GC cycle completes:

```text
gc 14 @4.210s 2%: 0.042+1.85+0.031 ms clock, 0.33+0.92/3.40/0.00+0.24 ms cpu, 128->132->64 MB, 128 MB goal, 8 P
```

Here is how to read every number in that line:
* **`gc 14`**: This is the 14th GC cycle since startup.
* **`@4.210s`**: Occurred 4.21 seconds after the program started.
* **`2%`**: Total percentage of CPU time spent on GC since startup (anything under 5% is very healthy; >20% means your app is spending too much time cleaning up garbage).
* **`0.042+1.85+0.031 ms clock`**:
  * `0.042 ms`: Stop-The-World pause #1 (turn on write barrier).
  * `1.85 ms`: Concurrent marking time (ran alongside your app).
  * `0.031 ms`: Stop-The-World pause #2 (turn off write barrier).
* **`0.92/3.40/0.00 ms cpu`**:
  * `0.92 ms`: CPU spent in **Mark Assist** (application goroutines forced to help mark). If this number is large, your handlers are allocating too fast!
  * `3.40 ms`: CPU spent by dedicated background GC workers.
  * `0.00 ms`: Idle GC worker CPU.
* **`128->132->64 MB`**:
  * `128 MB`: Heap size when GC started.
  * `132 MB`: Heap size when GC finished marking.
  * `64 MB`: **Live heap size** remaining after sweeping garbage.
* **`128 MB goal`**: The target heap size calculated by the GC Pacer.
* **`8 P`**: Number of logical CPU cores (`GOMAXPROCS`) used.

### 6.3 Exporting Live GC Metrics to Prometheus/OpenTelemetry (`runtime/metrics`)

Avoid calling `runtime.ReadMemStats(&m)` in a hot metrics loop—it triggers a brief Stop-The-World pause to gather stats. Instead, use Go's modern, lock-free **`runtime/metrics`** package:

```go
package telemetry

import (
	"runtime/metrics"
	"time"
)

type GCMetricsSnapshot struct {
	LiveHeapBytes      uint64
	MemoryLimitBytes   uint64
	TotalGCCycles      uint64
	LastGCPauseSeconds float64
	GCAssistCPUSeconds float64
}

// ReadGCMetrics reads internal Go runtime GC telemetry without triggering
// any Stop-The-World pauses.
func ReadGCMetrics() GCMetricsSnapshot {
	samples := []metrics.Sample{
		{Name: "/memory/classes/heap/objects:bytes"},
		{Name: "/gc/gomemlimit:bytes"},
		{Name: "/gc/cycles/total:gc-cycles"},
		{Name: "/sched/pauses/total/gc:seconds"},
		{Name: "/cpu/classes/gc/mark/assist:cpu-seconds"},
	}

	metrics.Read(samples)

	var snap GCMetricsSnapshot
	if samples[0].Value.Kind() == metrics.KindUint64 {
		snap.LiveHeapBytes = samples[0].Value.Uint64()
	}
	if samples[1].Value.Kind() == metrics.KindUint64 {
		snap.MemoryLimitBytes = samples[1].Value.Uint64()
	}
	if samples[2].Value.Kind() == metrics.KindUint64 {
		snap.TotalGCCycles = samples[2].Value.Uint64()
	}
	if samples[3].Value.Kind() == metrics.KindFloat64Histogram {
		hist := samples[3].Value.Float64Histogram()
		if len(hist.Buckets) > 0 {
			snap.LastGCPauseSeconds = hist.Buckets[len(hist.Buckets)-1]
		}
	}
	if samples[4].Value.Kind() == metrics.KindFloat64 {
		snap.GCAssistCPUSeconds = samples[4].Value.Float64()
	}

	_ = time.Second
	return snap
}
```