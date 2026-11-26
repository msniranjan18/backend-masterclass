# Pointers vs. Values & Memory Mechanics

## Table of Contents

1. **Chapter 1: How CPUs & Go Actually Handle Memory (The Mental Model)**

   * Stack Copying vs. Heap Allocation: What Really Happens in Hardware

   * CPU Cache Lines, Spatial Locality & Why "Pointer Chasing" Slows Programs Down

   * How Pointers Multiply Garbage Collector Work

2. **Chapter 2: The Golden Rules of When to Use a Pointer (`*T`)**

   * Semantic Rule 1: In-Place Mutation

   * Semantic Rule 2: Synchronization Primitives (`sync.Mutex`, `atomic`)

   * Semantic Rule 3: Optional / Absent State (`nil` vs. Zero Value)

   * Performance Rule 4: Large Structs (`> 1KB` and `64KB–128KB` Buffers)

3. **Chapter 3: Nested Structs, Struct Size Bloat & Memory Layout**

   * Small Nested Structs: Why Embedding by Value Wins

   * Large or Rarely-Populated Nested Structs: When to Split with a Pointer

   * Struct Field Ordering & Alignment Padding

4. **Chapter 4: Slices, Maps & Strings (Where Developers Waste the Most Performance)**

   * Value Slices (`[]T`) vs. Pointer Slices (`[]*T`)

   * Why You Should Almost Never Use `*string` or `*[]byte`

   * Maps (`map[K]V` vs. `map[K]*V`) & The Mutation Trap

5. **Chapter 5: Method Receivers, Interfaces & Escape Analysis**

   * Value Receivers vs. Pointer Receivers (Consistency & Method Sets)

   * How Assigning a Pointer to an Interface Triggers Heap Escapes

   * The Production Decision Matrix & Benchmarking Checklist

---

## Chapter 1: How CPUs & Go Actually Handle Memory (The Mental Model)

When engineers come to Go from C, C++, or Java, their first instinct is often: *"Passing structs by value copies bytes, and copying is slow—so I should pass pointers (`*T`) everywhere."*

In Go, **passing pointers everywhere often makes your program slower, not faster.** To understand why, we have to compare two competing costs: **CPU copying cost** vs. **Heap allocation + Garbage Collector cost**.

### 1.1 Stack Copying vs. Heap Allocation

Every running goroutine has its own private **Stack** (starting at just 2 KB). Meanwhile, your entire Go process shares one large **Heap**.

* **Passing a Small Value (`T`):**
  * Go copies the bytes straight onto the next stack frame.
  * Modern 64-bit CPUs copy contiguous stack memory at **20–40 GB/s** using hardware vector instructions (`AVX2`/`AVX-512`). Copying a 64-byte struct takes **~1–2 nanoseconds**.
  * When the function returns, the stack pointer simply moves back down. **Zero heap allocation, zero locks, and zero Garbage Collector work.**

* **Passing a Pointer (`*T`):**
  * You only copy an **8-byte memory address**.
  * **However**, if the Go compiler cannot prove that `*T` stays strictly inside that function (for instance, if you return `&t`, store it in a map, send it on a channel, or pass it to an interface), Go's Escape Analysis moves `T` off the fast stack and onto the **Heap**.
  * Moving `T` to the heap triggers `runtime.mallocgc` (~15–50+ ns), fragments memory, and forces the Garbage Collector to scan and sweep that object later.

### 1.2 CPU Cache Lines & "Pointer Chasing"

Modern CPUs do not read memory 1 byte at a time from RAM—they fetch **64-byte chunks called Cache Lines** into ultra-fast L1/L2 caches right next to the CPU core:

* **L1 Cache Hit:** ~1 nanosecond
* **L2 Cache Hit:** ~4 nanoseconds
* **L3 Cache Hit:** ~15–20 nanoseconds
* **Main RAM Fetch (Cache Miss):** **~60–100 nanoseconds**

When structs are stored **by value** side by side in memory, the CPU's hardware prefetcher predicts what your loop will read next and loads the next cache line before you even ask for it.

When you link small structs together using **pointers**, each object may live in a completely different area of heap memory. Every time your code dereferences a pointer (`order.Customer.Address.City`), the CPU stalls waiting for RAM—a bottleneck known as **Pointer Chasing**.

### 1.3 How Pointers Multiply Garbage Collector Work

Go's Garbage Collector is **non-compacting** (it never moves objects in memory), and during its Mark phase, **it only follows pointers**:

* If an object or slice contains **zero pointers** (for example, `[]int64` or a struct of integers and fixed byte arrays), Go marks that memory block as **`noscan`**. The GC skips scanning its contents completely—whether it is 100 bytes or 100 MB!
* If you replace value structs with pointers (`[]*User` instead of `[]User`), you turn 1 memory block into 100,001 separate heap allocations and force the GC to chase 100,000 pointers on every single GC cycle.

#### Explanation:
When you have []int64, Go actually has two parts in memory:

1. **The Slice Header (The "Box"):** This is a small struct (24 bytes) that contains:
- Data (Pointer) → Points to the underlying array.
- Len (Integer)
- Cap (Integer)
- This header is scanned by the GC because it contains a pointer.

2. **The Underlying Array (The "Contents"):** This is the actual block of memory holding the int64 values.
- int64 is a primitive type. It has zero pointers.
- This array is marked noscan.

so the GC does check the Slice Header to find the array. But once it finds the array, it looks at the contents of the array.
- If the array contains []*User (pointers), the GC must follow every single one.
- If the array contains []int64 (values), the GC sees no pointers inside and skips scanning the entire 100 MB of data.

---

## Chapter 2: The Golden Rules of When to Use a Pointer (`*T`)

Before thinking about micro-optimizations, let **program correctness (semantics)** decide first, and **object size** decide second.

### 2.1 Semantic Rule 1: In-Place Mutation

In Go, everything is passed by value—even pointers (passing `*T` passes a copy of the 8-byte memory address). If a function or method must modify the caller's struct and keep those changes after returning, you **must** use a pointer.

```go
package accounts

type Wallet struct {
	ID      uint64
	Balance int64
}

// WRONG: Modifies a temporary stack copy; caller's Wallet never changes.
func DepositBroken(w Wallet, amount int64) {
	w.Balance += amount
}

// RIGHT: Dereferences the pointer to mutate the original Wallet in place.
func Deposit(w *Wallet, amount int64) {
	w.Balance += amount
}
```

### 2.2 Semantic Rule 2: Synchronization Primitives (`sync.Mutex`, `atomic`)

If a struct embeds or holds a `sync.Mutex`, `sync.RWMutex`, `sync.WaitGroup`, `sync.Once`, or `atomic.Int64`, **you must NEVER copy that struct**. Copying a mutex duplicates its internal lock state, allowing two goroutines to enter a critical section simultaneously.

```go
package registry

import "sync"

type SafeCounter struct {
	mu    sync.RWMutex
	count int64
}

// Always pass *SafeCounter and use pointer receivers on methods.
func (c *SafeCounter) Increment() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.count++
}
```

### 2.3 Semantic Rule 3: Distinguishing "Absent" (`nil`) from "Zero Value"

Value types in Go always have a default zero value (`0`, `""`, `false`). If your domain logic needs to distinguish between *"user explicitly set discount to 0"* vs. *"user did not provide a discount field at all"* (common in JSON `PATCH` APIs or SQL `NULL` columns), use a pointer (`*int64`).

```go
type UpdateProfileRequest struct {
	DisplayName *string `json:"display_name"` // nil = don't update; &"" = clear display name
	MaxAlerts   *int    `json:"max_alerts"`   // nil = don't update; &0 = disable alerts
}
```

### 2.4 Performance Rule 4: Large Structs (`> 1 KB`, especially `64 KB – 128 KB`)

When a struct grows large—such as a `64 KB` or `128 KB` packet buffer, batch frame, or fixed array—passing it by value **will degrade performance**:

1. **Memory Bandwidth Saturation:** Copying `128 KB` across 5 function calls copies `640 KB` of memory per request.
2. **Goroutine Stack Growth Penalty:** Because Go goroutines start with a **2 KB stack**, passing a `128 KB` value forces the Go runtime to pause the goroutine, allocate a larger stack segment, copy all existing stack frames over, and rewrite stack pointers (`runtime.morestack`).

**Production Pattern for Large Objects:** Pass `*T` (an 8-byte pointer), and if you allocate these large objects frequently on hot paths, pair `*T` with `sync.Pool` so the heap memory is reused without triggering Garbage Collector sweeps.

```go
package pipeline

import "sync"

// LargeFrame is 128 KB—far too big to copy on goroutine stacks.
type LargeFrame struct {
	SequenceID uint64
	Flags      uint32
	Payload    [131072]byte // 128 KB
}

var framePool = sync.Pool{
	New: func() any {
		return new(LargeFrame)
	},
}

func AcquireFrame() *LargeFrame {
	f := framePool.Get().(*LargeFrame)
	f.SequenceID = 0
	f.Flags = 0
	return f
}

func ReleaseFrame(f *LargeFrame) {
	if f != nil {
		framePool.Put(f)
	}
}

// ProcessFrame copies only an 8-byte pointer—zero stack growth, zero GC churn.
func ProcessFrame(f *LargeFrame) uint64 {
	return f.SequenceID
}
```

---

## Chapter 3: Nested Structs, Struct Size Bloat & Memory Layout

A common architectural question is: *"If I embed structs inside another struct without pointers, won't the outer struct become huge and slow to copy?"*

The answer depends on **size and access frequency**:

### 3.1 Small-to-Medium Nested Structs (`< 256 bytes` total): Embed by Value

When sub-structs are small (e.g., 16–64 bytes each), embedding them **by value** gives you:
* **1 allocation** instead of $N$ separate heap allocations.
* **Contiguous CPU cache locality** (the entire parent struct loads into 2–4 adjacent cache lines).
* **Fewer pointers for the GC to trace.**

```go
package orders

import "time"

type Money struct {
	AmountCents int64  // 8 bytes
	Currency    [4]byte // 4 bytes (fixed array avoids string pointer!)
}

type AuditTimestamps struct {
	CreatedAt time.Time // 24 bytes
	UpdatedAt time.Time // 24 bytes
}

// GOOD: Total size is ~88 bytes. Fits in two 64-byte CPU cache lines,
// requires 1 allocation if on heap, and has ZERO internal pointers!
type CompactOrder struct {
	OrderID    uint64          // 8 bytes
	CustomerID uint64          // 8 bytes
	Total      Money           // 16 bytes (aligned)
	Timestamps AuditTimestamps // 48 bytes
	Status     uint8           // 1 byte
}

// BAD: Using pointers for tiny nested structs causes 3 separate heap allocations,
// fragments memory across random RAM addresses, and forces the GC to trace 2 extra pointers.
type FragmentedOrder struct {
	OrderID    uint64
	CustomerID uint64
	Total      *Money           // Extra heap allocation + GC pointer
	Timestamps *AuditTimestamps // Extra heap allocation + GC pointer
	Status     uint8
}
```

### 3.2 Large or Rarely-Used Nested Structs (`> 1 KB`): Use a Pointer

If a compact `120-byte` struct occasionally carries a massive `32 KB` diagnostic dump or raw payload that is only populated 5% of the time, embedding that `32 KB` struct by value forces **100% of instances** to allocate and copy `32 KB` of empty bytes.

In this case, isolate the heavy sub-struct behind a pointer so the common path stays fast and compact:

```go
type HeavyDebugTrace struct {
	RawPacketDump [32768]byte // 32 KB
	StackFrames   [64]uintptr
}

type Transaction struct {
	ID        uint64
	AccountID uint64
	Amount    int64
	// Pointer is nil 95% of the time, keeping Transaction at a compact 32 bytes.
	DebugTrace *HeavyDebugTrace
}
```

### 3.3 Struct Field Ordering (Eliminating Hidden Padding Bloat)

Before reaching for pointers to shrink a struct, check your **field ordering**. On 64-bit CPUs, Go aligns fields to multiples of their byte size (`int64` to 8 bytes, `int32` to 4 bytes). Poor field ordering inserts invisible dead padding bytes into every struct instance:

```go
// Bloated: 32 bytes per struct due to 13 bytes of invisible alignment padding
type UnalignedEvent struct {
	IsRetry   bool   // 1 byte + 7 bytes padding
	Timestamp int64  // 8 bytes
	Priority  uint8  // 1 byte + 3 bytes padding
	TenantID  uint32 // 4 bytes
	ShardID   uint16 // 2 bytes + 6 bytes padding
}

// Compact: 24 bytes per struct (25% smaller!) with identical fields.
// Rule: Order fields from largest alignment (8B) down to smallest (1B).
type AlignedEvent struct {
	Timestamp int64  // 8 bytes
	TenantID  uint32 // 4 bytes
	ShardID   uint16 // 2 bytes
	IsRetry   bool   // 1 byte
	Priority  uint8  // 1 byte (total = 16 bytes—50% smaller than UnalignedEvent!)
}
```

---

## Chapter 4: Slices, Maps & Strings (Where Developers Waste the Most Performance)

### 4.1 Value Slices (`[]T`) vs. Pointer Slices (`[]*T`)

This is the single highest-leverage optimization in production Go services:

* **`users := make([]User, 100_000)` (Value Slice):**
  * Allocates **one contiguous block** of memory on the heap.
  * Iterating through `users` streams linearly through L1/L2 CPU cache at maximum hardware speed.
  * If `User` contains no pointers, the GC marks all 100,000 items in **nanoseconds** without scanning a single element.
* **`users := make([]*User, 100_000)` (Pointer Slice):**
  * Allocates **100,001 separate objects** (`1` slice backing array + `100,000` individual `User` structs).
  * Every loop iteration jumps to a different memory address (cache misses).
  * Every GC cycle must inspect all 100,000 pointers.

**When to choose `[]*T` over `[]T`:**
1. `T` is very large (`> 1 KB`) and you frequently sort, filter, or swap elements in place (swapping 8-byte pointers is faster than swapping 4 KB structs).
2. Multiple data structures share references to the exact same mutable `T` instance.
3. `T` contains a `sync.Mutex`.

### 4.2 Never Pass `*string` or `*[]byte` to "Avoid Copying"

A common misconception is that passing a `string` or `[]byte` by value copies the underlying text or byte array. **It does not.**

Under the hood in the Go runtime, `string` and `slice` are tiny **descriptor headers**:

```go
// What a string actually is in Go runtime (16 bytes total):
type stringHeader struct {
	Data uintptr // 8-byte pointer to underlying immutable bytes
	Len  int     // 8-byte length
}

// What a slice actually is in Go runtime (24 bytes total):
type sliceHeader struct {
	Data uintptr // 8-byte pointer to underlying array
	Len  int     // 8-byte length
	Cap  int     // 8-byte capacity
}
```

Passing a 10 MB `string` or 10 MB `[]byte` by value copies only **16 or 24 bytes**! Passing `*string` adds an unnecessary extra pointer indirection and forces the 16-byte header onto the heap. Only use `*string` when you explicitly need `nil` to represent an unset optional field.

### 4.3 Maps: `map[K]V` vs. `map[K]*V`

Go maps have two specific behaviors that guide whether to store values or pointers:

1. **Map values are not addressable:** In Go, writing `users[id].LoginCount++` on a `map[uint64]User` **will not compile**, because Go's hash table may move buckets in memory during map growth. You must read the copy, modify it, and write it back (`u := users[id]; u.LoginCount++; users[id] = u`).
2. **GC Bucket Scanning:**
   * If `V` is small (`<= 128 bytes`) and mostly read-only (like a configuration or routing lookup table), use **`map[K]V`**. If `K` and `V` have no pointers (e.g., `map[uint64]RouteConfig`), the GC skips scanning the entire map!
   * If `V` is large (`> 256 bytes`) or its fields are updated thousands of times per second in place, use **`map[K]*V`** (protected by a mutex or sharded locks) so you don't rewrite hundreds of bytes on every field update.

---

## Chapter 5: Method Receivers, Interfaces & Escape Analysis

### 5.1 Value Receivers vs. Pointer Receivers

When defining methods on a struct `T`:

* **Use a Pointer Receiver `func (t *T) Method()` if:**
  1. Any method on `T` mutates `T`'s fields.
  2. `T` contains a `sync.Mutex` or other non-copyable field.
  3. `T` is a large struct (`> 256 bytes`).
  4. **Consistency Rule:** If *one* method on `T` requires a pointer receiver, make **all** methods on `T` use pointer receivers. Mixing value and pointer receivers on the same type confuses callers and causes subtle bugs with interface satisfaction (`*T` implements both value and pointer receiver methods, whereas `T` only implements value receiver methods).

* **Use a Value Receiver `func (t T) Method()` if:**
  1. `T` is a small, immutable value type (`time.Time`, `Money`, `Point`, `UUID`, small DTOs).

### 5.2 How Interfaces Trigger Hidden Heap Escapes

Even if you pass a pointer to a function that does not store it anywhere, **converting that pointer into an `interface` (like `any`, `io.Reader`, or a custom repository interface) almost always forces the underlying object onto the heap**.

Why? Because at compile time, the compiler often cannot see which concrete implementation sits behind an interface call, so it conservatively allocates the object on the heap to guarantee memory safety.

```go
package escape

import "io"

type SmallReader struct {
	data [32]byte
	pos  int
}

func (r *SmallReader) Read(p []byte) (int, error) {
	if r.pos >= len(r.data) {
		return 0, io.EOF
	}
	n := copy(p, r.data[r.pos:])
	r.pos += n
	return n, nil
}

// Stays on STACK: Compiler knows exact concrete type *SmallReader and inlines Read().
func ReadConcrete() int {
	var r SmallReader
	var buf [16]byte
	n, _ := r.Read(buf[:])
	return n
}

// Escapes to HEAP: Assigning &r to io.Reader interface forces r onto the heap!
func ReadViaInterface() int {
	var r SmallReader
	var reader io.Reader = &r // compiler: "moved to heap: r"
	var buf [16]byte
	n, _ := reader.Read(buf[:])
	return n
}
```

### 5.3 The Complete Production Decision Matrix

| Use Case | Use Value (`T`) | Use Pointer (`*T`) | Primary Reason |
| :--- | :---: | :---: | :--- |
| **Primitives (`int`, `bool`, `float64`) & `time.Time`** | Yes | No (unless optional `nil`) | Tiny (1–24 bytes); zero heap/GC cost on stack. |
| **`string` and `[]T` slices** | Yes | No (unless optional `nil`) | Already 16/24-byte pointer headers internally. |
| **Small Structs (`<= 128 bytes`)** | Yes | Only if mutated or mutex | Fits in 1–2 cache lines; stack copy (~1 ns) beats heap + GC. |
| **Medium Structs (`128 B – 1 KB`)** | Short-lived local vars | Deep call chains / hot loops | Balance stack copy cost against heap escape cost via benchmarks. |
| **Large Structs (`> 1 KB`, e.g., `64KB–128KB`)** | **Never** | **Always** (+ `sync.Pool`) | Prevents memory bandwidth saturation and goroutine stack growth. |
| **Structs with `sync.Mutex` / `atomic`** | **Never** | **Always** | Copying locks breaks thread safety (`go vet` error). |
| **Slices of Structs** | `[]T` (Default) | `[]*T` (Only if `T > 1KB` or shared) | Contiguous cache locality + $O(1)$ GC scanning when pointer-free. |
| **Nested Structs** | `< 256 bytes` | `> 1 KB` or rarely populated | Avoids multi-allocation fragmentation while preventing struct bloat. |

### 5.4 How to Prove It in Your Own Codebase

Never guess when optimizing a hot path. Verify with Go's built-in escape analyzer and benchmark suite:

```bash
# 1. Inspect which variables stay on the stack vs. escape to the heap:
go build -gcflags="-m=1" ./your/package

# 2. Benchmark latency (ns/op), heap bytes (B/op), and GC allocation count (allocs/op):
go test -bench=. -benchmem ./your/package
```