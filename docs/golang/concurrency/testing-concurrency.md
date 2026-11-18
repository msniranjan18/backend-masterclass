# Testing Concurrency

Testing concurrent code is notoriously difficult because bugs like data races and deadlocks depend entirely on timing. If a goroutine finishes a millisecond slower on your machine than in production, the bug stays hidden.

Go solves this with two major tools: the Race Detector (built into the compiler) and the new `testing/synctest` package (which virtualizes time so you don't have to use `time.Sleep` in tests).

### 1. Catching Data Races: The `-race` Flag

A data race happens when two goroutines access the same memory location concurrently, and at least one of the accesses is a write.

Instead of guessing if your code is thread-safe, you run your tests with the race detector enabled.

#### The Buggy Code (`counter.go`)

This code looks innocent but has a massive data race because `count++` is not an atomic operation.

Go

```go
package counter

type RaceCounter struct {
	count int
}

func (c *RaceCounter) Inc() {
	c.count++ // BUG: No mutex protection!
}

func (c *RaceCounter) Value() int {
	return c.count
}
```

#### The Test File (`counter_test.go`)

We spin up concurrent workers to hammer the counter.

Go

```go
package counter

import (
	"sync"
	"testing"
)

func TestCounterConcurrent(t *testing.T) {
	c := &RaceCounter{}
	var wg sync.WaitGroup

	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			c.Inc()
		}()
	}
	wg.Wait()
}
```

#### How to Run It

If you run `go test`, it might pass. To actually catch the bug, run it with the `-race` flag:

Bash

```
go test -race ./...
```

#### The Output

The Go runtime will instantly halt the test and spit out a detailed report showing exactly which lines of code competed for memory:

Plaintext

```
==================
WARNING: DATA RACE
Write at 0x00c0000141b8 by goroutine 8:
  counter.(*RaceCounter).Inc()
      /counter.go:9 +0x40
...
Previous read at 0x00c0000141b8 by goroutine 7:
  counter.(*RaceCounter).Inc()
      /counter.go:9 +0x24
==================
--- FAIL: TestCounterConcurrent (0.04s)
```

### 2. Testing Time-Based Concurrency: `testing/synctest`

Testing code that handles timeouts or delays usually requires adding `time.Sleep()` to your test files. This makes tests slow and flaky.

The `testing/synctest` package lets you wrap your test in a virtual time bubble. Inside this bubble, when your code calls `time.Sleep(10 * time.Hour)`, the testing package instantly advances the fake clock forward without actually making your machine wait.

#### The Code under Test (`worker.go`)

A simple background worker that waits 5 seconds before sending a message down a channel.

Go

```
package worker

import "time"

func DelayMessage(ch chan string) {
	time.Sleep(5 * time.Second) // Takes 5 seconds in the real world
	ch <- "hello from the past"
}
```

#### The Test File (`worker_test.go`)

Using `synctest.Run`, we isolate the goroutine. The runtime notices when all goroutines inside the bubble are blocked/sleeping, advances the virtual clock automatically, wakes them up, and finishes the test in less than a millisecond.

Go

```go
package worker

import (
	"testing"
	"testing/synctest" // Introduced for isolated concurrent time testing
)

func TestDelayMessage(t *testing.T) {
	// synctest.Run creates a virtual time environment
	synctest.Run(func() {
		ch := make(chan string, 1)

		go DelayMessage(ch)

		// In a normal test, reading from this channel would block for 5 seconds.
		// Here, synctest detects the deadlock/sleep state, instantly advances
		// the fake clock by 5 seconds, and delivers the message.
		msg := <-ch

		if msg != "hello from the past" {
			t.Errorf("got %q, want 'hello from the past'", msg)
		}
	})
}
```

### Summary Checklist for Testing Concurrency

1. Always use `-race`: Make `go test -race ./...` a default step in your local workflow and CI/CD pipelines.
2. Never use real `time.Sleep` to fix test flakes: If a concurrent test fails intermittently, adding `time.Sleep(50 * time.Millisecond)` just hides the issue. Fix it using proper channel signaling or standard sync primitives.
3. Leverage `synctest` for timers: If your application logic relies heavily on `time.After`, background tickers, or context timeouts, isolate them inside a `synctest.Run` block to keep your test suites fast and deterministic.
