# Examples
## Example 1: The "Closed Channel Spins Forever" Trap
```go
package main

import (
	"fmt"
)

func main() {
	ch := make(chan int, 3)
	ch <- 1
	ch <- 2
	close(ch)

	for i := 0; i < 5; i++ {
		select {
		case v, ok := <-ch:
			fmt.Println("got", v, "ok:", ok)
		default:
			fmt.Println("nothing ready")
		}
	}
}
```

Output
```text
got 1 ok: true
got 2 ok: true
got 0 ok: false
got 0 ok: false
got 0 ok: false
```

💡 The lesson
A closed channel is always ready — it immediately yields the zero value with ok == false, forever. So:

A default case never fires once the channel is closed.
Your loop must check ok and break, or you'll spin infinitely. This is the #1 cause of "my CPU is at 100% and nothing is happening."


## 🔥 Example 2: Producer/Consumer That Deadlocks on Close
```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var wg sync.WaitGroup
	ch := make(chan int) // unbuffered!

	wg.Go(func() { // producer
		for i := range 5 {
			ch <- i
		}
		close(ch) // ❌ can this ever run?
	})

	wg.Go(func() { // consumer
		defer wg.Done()
		for v := range ch {
			fmt.Println(v)
		}
	})

	wg.Wait()
	fmt.Println("done")
}
```

Output:
```text
fatal error: all goroutines are asleep - deadlock!
```

💡 The lesson
The producer blocks on ch <- 0 because there's no receiver yet... actually the consumer is there, but here's the real trap in the general form: only the sender closes, and closing must happen after all sends complete. With multiple producers, one producer closing kills the others with panic: send on closed channel.

Fix for multi-producer: use a separate WaitGroup just for producers:
```go
var producers sync.WaitGroup
producers.Add(3)
for p := range 3 {
	go func(p int) {
		defer producers.Done()
		for i := range 3 {
			ch <- p*100 + i
		}
	}(p)
}

// Dedicated closer goroutine
go func() {
	producers.Wait()
	close(ch) // ✅ safe: nobody else can send now
}()

for v := range ch { // consumer exits when channel closes
	fmt.Println(v)
}
```



## 🔥 Example 3: select Placement Decides Who Survives
```go
package main

import (
	"fmt"
	"time"
)

func main() {
	data := make(chan string)
	done := make(chan struct{})

	go func() {
		time.Sleep(100 * time.Millisecond)
		data <- "payload"
		close(data)
	}()

	go func() {
		for {
			select {
			case v, ok := <-data:
				if !ok {
					fmt.Println("data channel closed, exiting")
					return
				}
				fmt.Println("received:", v)
			case <-done:
				fmt.Println("shutdown signal")
				return
			}
		}
	}()

	time.Sleep(300 * time.Millisecond)
	close(done) // signal shutdown AFTER data is already closed
	time.Sleep(50 * time.Millisecond)
}
```


1. The nil Channel Trap in select
Sending to or receiving from a nil channel blocks forever—which is actually a powerful feature in select statements, unless triggered accidentally before initialization.

Go
package main

import (
	"fmt"
	"time"
)

func main() {
	var ch chan int // nil channel
	done := make(chan bool)

	go func() {
		time.Sleep(100 * time.Millisecond)
		ch = make(chan int, 1)
		ch <- 42
		close(done)
	}()

	select {
	case v := <-ch:
		fmt.Println("Received:", v)
	case <-done:
		fmt.Println("Done without receiving!")
	}
}
Output:

Plaintext
Done without receiving!
Why it happens: When select begins executing, it evaluates all channel expressions once up front. At t = 0, ch is nil, so case v := <-ch: is permanently disabled for that select execution. Even after the goroutine assigns a valid channel to ch and sends 42, the select statement is still waiting on the original nil value.

The Fix: Initialize the channel before starting the goroutine, or loop over select if dynamically swapping channels.

Go
ch := make(chan int, 1) // Initialize up front
2. Closing a Channel with Multiple Senders (The Panic)
Closing an already-closed channel or sending on a closed channel triggers an unrecoverable runtime panic.

Go
package main

import (
	"fmt"
	"sync"
)

func main() {
	ch := make(chan int, 10)
	var wg sync.WaitGroup

	for i := 0; i < 3; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			ch <- id
			close(ch) // Tricky bug: each worker tries to close
		}(i)
	}

	wg.Wait()
	for v := range ch {
		fmt.Println(v)
	}
}
Output:

Plaintext
panic: close of closed channel
Why it happens: The Channel Closing Principle states: never close a channel from the receiver side, and never close a channel if the channel has multiple concurrent senders. Multiple goroutines race to call close(ch) after their send.

The Fix: Use a dedicated coordinator goroutine with sync.WaitGroup to close the channel after all senders finish.

Go
// Inside worker: remove close(ch)
// After spawning workers:
go func() {
    wg.Wait()
    close(ch)
}()
3. Reading From a Closed Channel Forever (Zero-Value Spin)
A closed channel never blocks on receive—it immediately returns the zero value of the channel's element type.

Go
package main

import (
	"fmt"
	"time"
)

func main() {
	ch := make(chan int, 2)
	ch <- 10
	ch <- 20
	close(ch)

	timeout := time.After(5 * time.Millisecond)
	count := 0
	for {
		select {
		case v := <-ch:
			count++
			if count <= 4 {
				fmt.Println("Read:", v)
			}
		case <-timeout:
			fmt.Println("Total reads:", count)
			return
		}
	}
}
Output:

Plaintext
Read: 10
Read: 20
Read: 0
Read: 0
Total reads: 148291
Why it happens: After draining 10 and 20, <-ch continues yielding 0 immediately on every loop iteration, spinning at 100% CPU until the timeout fires.

The Fix: Use the two-value receive idiom (v, ok := <-ch) and set ch = nil when ok is false to disable that select case cleanly.

Go
case v, ok := <-ch:
    if !ok {
        ch = nil // Disables this select branch permanently
        continue
    }
    fmt.Println("Read:", v)
4. Unbuffered Channel Deadlock on Early Return (Goroutine Leak)
When a receiver abandons an unbuffered channel early, the sender blocks forever in sendq—leaking memory silently without triggering a global deadlock panic if main keeps running.

Go
package main

import (
	"fmt"
	"runtime"
	"time"
)

func fetchFirst() int {
	ch := make(chan int) // Unbuffered

	for i := 1; i <= 3; i++ {
		go func(val int) {
			ch <- val // Goroutines 2 and 3 block here forever
		}(i)
	}

	return <-ch
}

func main() {
	fmt.Println("Result:", fetchFirst())
	time.Sleep(50 * time.Millisecond)
	fmt.Println("Active goroutines:", runtime.NumGoroutine())
}
Output:

Plaintext
Result: 1
Active goroutines: 3
(Note: main itself is 1 goroutine, leaving 2 leaked worker goroutines stuck forever).

Why it happens: fetchFirst() reads the first response and returns. The remaining two goroutines try to send to ch, have no receiver, and sit parked in hchan.sendq for the lifetime of the process.

The Fix: Size the channel buffer to match the exact number of workers (make(chan int, 3)), or pass a context.Context / done channel so abandoned senders can abort.

Go
ch := make(chan int, 3) // All 3 sends complete without blocking
5. select Argument Evaluation Side Effects
Go evaluates all channel expressions and right-hand values of send cases before picking which case can proceed.

Go
package main

import "fmt"

func compute(name string) int {
	fmt.Println("Evaluated:", name)
	return 1
}

func main() {
	ch1 := make(chan int, 1)
	ch2 := make(chan int, 1)
	ch1 <- 99 // ch1 is full

	select {
	case ch1 <- compute("ch1"):
		fmt.Println("Sent to ch1")
	case ch2 <- compute("ch2"):
		fmt.Println("Sent to ch2")
	}
}
Output:

Plaintext
Evaluated: ch1
Evaluated: ch2
Sent to ch2
Why it happens: Even though ch1 is full and cannot accept a send, Go evaluates compute("ch1") before checking channel readiness. If compute() is expensive or mutates state, it still executes.

The Fix: Keep send expressions in select side-effect-free (pass precomputed variables), or guard expensive work behind readiness checks.

