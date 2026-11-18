# Sync and Context

The `sync` package provides low-level primitives for memory access and coordination, while the `context` package handles high-level signaling like cancellations, timeouts, and tracking request-scoped data.

### 1. The `sync` Package

Goroutines share the same memory space. Without coordination, multiple goroutines trying to read and write to the same variable at the same time cause data races and undefined behavior.

Here are the primary tools in the `sync` package you will use daily.

#### `sync.Mutex` (Mutual Exclusion)

A Mutex ensures that only one goroutine can access a critical section of code at a time. If a second goroutine tries to lock an already locked mutex, it blocks until the first one unlocks it.

Go

{% code lineNumbers="true" expandable="true" %}
```go
package main

import (
	"fmt"
	"sync"
)

type SafeCounter struct {
	mu    sync.Mutex
	value int
}

func (c *SafeCounter) Increment() {
	c.mu.Lock()         // Acquire the lock; other goroutines must wait
	defer c.mu.Unlock() // Always ensure it unlocks when the function exits
	c.value++
}

func (c *SafeCounter) Value() int {
	c.mu.Lock()
	defer c.mu.Unlock()
	return c.value
}

func main() {
	counter := &SafeCounter{}
	var wg sync.WaitGroup

	// Spawn 1000 goroutines to increment concurrently
	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			counter.Increment()
		}()
	}

	wg.Wait()
	fmt.Printf("Final Counter Value: %d\n", counter.Value()) // Guaranteed to be 1000
}
```
{% endcode %}

#### `sync.RWMutex` (Reader/Writer Mutex)

If you have a resource that is read frequently but written to rarely, a standard `Mutex` creates a bottleneck. `RWMutex` allows multiple readers to look at the data simultaneously, but only one writer can modify it at a time, blocking all readers.

Go

{% code lineNumbers="true" expandable="true" %}
```go
package main

import (
	"fmt"
	"sync"
	"time"
)

type Cache struct {
	mu   sync.RWMutex
	data map[string]string
}

func (c *Cache) Get(key string) string {
	c.mu.RLock()         // Multiple readers can hold an RLock at once
	defer c.mu.RUnlock()
	return c.data[key]
}

func (c *Cache) Set(key, value string) {
	c.mu.Lock()         // Only one writer can hold a Lock, blocking readers too
	defer c.mu.Unlock()
	c.data[key] = value
}
```
{% endcode %}

#### `sync.WaitGroup`

A `WaitGroup` blocks execution until a collection of goroutines has finished running.

* `Add(n)` increases the counter.
* `Done()` decreases the counter (typically used with `defer`).
* `Wait()` blocks until the counter hits 0.

> Crucial Rule: Always pass a `WaitGroup` to a function as a pointer (`*sync.WaitGroup`). If you pass it by value, Go copies the internal state, and the `Wait()` call in your main function will never see the `Done()` signals, causing a deadlock.

Go

{% code lineNumbers="true" expandable="true" %}
```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func worker(id int, wg *sync.WaitGroup) {
	defer wg.Done() // Signal completion when this function exits
	fmt.Printf("Worker %d starting...\n", id)
	time.Sleep(time.Millisecond * 500)
	fmt.Printf("Worker %d finished!\n", id)
}

func main() {
	var wg sync.WaitGroup

	for i := 1; i <= 3; i++ {
		wg.Add(1) // Register 1 goroutine
		go worker(i, &wg)
	}

	wg.Wait() // Block here until the counter reaches 0
	fmt.Println("All workers finished execution.")
}
```
{% endcode %}

Go 1.25 introduced a major quality-of-life upgrade specifically designed to eliminate the boilerplate and accidental bugs that come with managing traditional `WaitGroup` counters.

The package added a new method: `sync.WaitGroup.Go()`.

### What Changed: The `wg.Go()` Method

Historically, you had to manually coordinate three distinct lines of code: `wg.Add(1)`, `go func()`, and `defer wg.Done()`. If you forgot `Add`, you had a data race; if you forgot `Done`, you had a deadlock.

The new `wg.Go()` method acts as a native "error-proofing" (poka-yoke) wrapper. It abstracts those three steps into a single method call:

1. It automatically calls `wg.Add(1)`.
2. It safely spawns the goroutine under the hood.
3. It automatically executes a deferred `wg.Done()` when your function returns.

#### The Modern Go 1.25+ Way

You no longer explicitly write the `go` keyword or manually tweak the counters when spinning up standard worker pools.

Go

{% code lineNumbers="true" expandable="true" %}
```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func main() {
	var wg sync.WaitGroup

	// Let's spawn 3 concurrent workers using the new API
	for i := 1; i <= 3; i++ {
		// Capture loop variable for safety (pre-Go 1.22 semantics)
		workerID := i 

		// wg.Go handles the Add, the goroutine creation, and the Done signal automatically
		wg.Go(func() {
			fmt.Printf("Worker %d starting...\n", workerID)
			time.Sleep(500 * time.Millisecond)
			fmt.Printf("Worker %d finished!\n", workerID)
		})
	}

	// This remains exactly the same—it blocks until all 3 finish
	wg.Wait()
	fmt.Println("All workers finished execution.")
}
```
{% endcode %}

### Crucial Rules to Avoid Bugs with `wg.Go()`

Because this abstraction handles the state updates for you, mixing old patterns inside the new method will break things:

* Do NOT use the `go` keyword inside `wg.Go()`: Writing `wg.Go(func() { go doWork() })` will spawn a _nested_ goroutine that exits instantly. The WaitGroup will decrement its counter immediately, completely defeating the purpose of waiting.
* Do NOT call `wg.Done()` inside the function: Because `wg.Go()` defers a `Done()` call natively, adding your own manual `wg.Done()` inside the closure will cause the counter to decrement twice, triggering a `panic: negative WaitGroup counter`.

### When should you still use the traditional `Add/Done`?

While `wg.Go()` is excellent for spinning up standard, independent concurrent tasks, you still need traditional `Add()` and `Done()` if:

1. You need to pre-allocate or increment the counter by a dynamic batch number upfront (e.g., `wg.Add(len(tasks))`).
2. You have complex conditional branches where a counter should only be incremented _if_ a certain runtime criteria is met.



## 2. The `context` Package

While `sync` focuses on raw coordination, `context` is all about lifecycle propagation. It allows you to signal deep down an execution stack (across API calls, database drivers, or background tasks) that the operation should be aborted.

Every context interface implements a `Done()` method, which returns a channel that closes when the context is cancelled or timed out.

#### `context.WithCancel`

Used when you want to explicitly stop background goroutines based on a specific event (like a shutdown signal or an error in another goroutine).

Go

{% code lineNumbers="true" expandable="true" %}
```go
package main

import (
	"context"
	"fmt"
	"time"
)

func streamData(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
			// The context was cancelled from the outside
			fmt.Println("Stop streaming: Context cancelled.")
			return
		default:
			fmt.Println("Streaming log chunk...")
			time.Sleep(200 * time.Millisecond)
		}
	}
}

func main() {
	// Create a background root context that can be cancelled
	ctx, cancel := context.WithCancel(context.Background())

	go streamData(ctx)

	// Let it run for a bit
	time.Sleep(700 * time.Millisecond)

	fmt.Println("Main: Triggering cancellation...")
	cancel() // Closes ctx.Done() channel

	// Give the goroutine a brief moment to log its cleanup
	time.Sleep(100 * time.Millisecond)
}
```
{% endcode %}

#### `context.WithTimeout`

Essential for handling network calls, slow disk I/O, or database queries. If the operation takes longer than the duration specified, it cleanly aborts rather than hogging resources.

Go

{% code lineNumbers="true" expandable="true" %}
```go
package main

import (
	"context"
	"fmt"
	"time"
)

func fetchFromDB(ctx context.Context) (string, error) {
	// Simulate a query that takes 1 second
	dbResultChan := make(chan string, 1)

	go func() {
		time.Sleep(1 * time.Second)
		dbResultChan <- "Row Data"
	}()

	select {
	case <-ctx.Done():
		// Timeout reached before database responded
		return "", ctx.Err()
	case res := <-dbResultChan:
		return res, nil
	}
}

func main() {
	// Context will automatically cancel after 400 milliseconds
	ctx, cancel := context.WithTimeout(context.Background(), 400*time.Millisecond)
	defer cancel() // Always release context resources even if it finishes early

	data, err := fetchFromDB(ctx)
	if err != nil {
		fmt.Printf("Failed: %v\n", err) // Output: Failed: context deadline exceeded
		return
	}
	fmt.Printf("Success: %s\n", data)
}
```
{% endcode %}

#### `context.WithValue`

Used strictly for passing request-scoped variables down the pipeline—such as a tracing Request ID, user authentication tokens, or a logger instance attached to a specific HTTP request.

> Warning: Do not use `WithValue` to pass optional arguments to functions or application state. It bypasses type-safety and obscures dependencies.

Go

{% code lineNumbers="true" expandable="true" %}
```go
package main

import (
	"context"
	"fmt"
)

// Define a custom unexported type for the key to prevent naming collisions
type contextKey string

const requestIDKey contextKey = "request_id"

func processRequest(ctx context.Context) {
	// Extract value from context. It returns type interface{}, so we cast it.
	if reqID, ok := ctx.Value(requestIDKey).(string); ok {
		fmt.Printf("[ReqID: %s] Processing database transaction...\n", reqID)
	} else {
		fmt.Println("No request ID found in context.")
	}
}

func main() {
	rootCtx := context.Background()

	// Inject the value into the context tree
	ctx := context.WithValue(rootCtx, requestIDKey, "tx_99823a")

	processRequest(ctx)
}
```
{% endcode %}

###
