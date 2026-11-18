# Channel Implementation

#### 🧱 Core Channel Concepts

1. **Explain the difference between buffered and unbuffered channels.**
   * When to use each?
   * What happens if you send to a full buffered channel?
2. **What happens if you send to a closed channel? What if you receive from a closed channel?**
3. **How do you safely close a channel? Who should close it and why?**
4. **What is the purpose of the `ok` idiom (`v, ok := <-ch`)?**
5. **What happens if you send to or receive from a `nil` channel?**

***

#### 🔄 Concurrency Patterns

6. **Implement a worker pool with graceful shutdown.**
   * Use channels + `sync.WaitGroup`
   * Ensure all in-flight tasks finish before exit.
7. **Implement a work queue that respects a `context.Context` timeout.**
   * Workers should stop accepting or processing tasks when the context is done.
8. **Implement a pipeline with multiple stages (e.g., reader → processor → writer).**
   * Show how to chain channels and clean up resources.
9. **Implement a fan-out/fan-in pattern.**
   * One sender → multiple workers → one aggregator.
10. **Implement a rate limiter using a token bucket or time-based approach with channels.**
11. **Implement a "first-one-wins" pattern:**
    * Launch multiple goroutines, take the first result, cancel the rest.
12. **Implement a simple pub/sub or broadcast system.**
    * One sender, many listeners, handle disconnects cleanly.
13. **Implement a producer-consumer system with backpressure.**
    * Use buffered channels to show slowdown when consumer is busy.

***

#### 🧠 Advanced / Tricky Scenarios

14. **Detect and prevent a deadlock in a channel-based system.**
    * Given a snippet, identify the issue and fix it.
15. **How do you implement a timeout using channels?**
    * Show `select` with `time.After`.
16. **How do you cancel a long-running goroutine using channels?**
    * Compare with `context.Context`.
17. **How would you implement a semaphore using channels?**
18. **Write a function that reads from a channel but times out if no data arrives.**
19. **How do you safely stop all workers in a pool when an error occurs?**
20. **Implement a task scheduler that processes jobs in priority order (hint: use a sorted channel or heap).**

***

#### 🧪 Real-World / System Design

21. **Design a concurrent log aggregator using channels.**
    * Multiple writers, one collector, graceful shutdown.
22. **Implement a concurrent HTTP request dispatcher with a limit on concurrent requests.**
23. **Build a simple chat server with multiple clients using channels.**
24. **Implement a job queue with retry logic and dead-letter channel.**
25. **How would you implement a distributed task queue using Go channels (hint: simulate with local channels)?**

***

#### 💡 Bonus: Pattern Recognition

26. **Given a buggy channel code snippet, identify and fix:**
    * Goroutine leak
    * Deadlock
    * Race condition
    * Improper channel close
27. **Compare and contrast:**
    * `sync.WaitGroup` vs. channel-based signaling
    * `context.Context` vs. custom stop channels
28. **How would you refactor a mutex-heavy concurrent system to use channels instead?**

<br>

Here are the **clean, production-ready Go code snippets** for each of the interview questions listed above. These are designed to be concise enough to write on a whiteboard but robust enough to pass a technical interview.

***

#### 🧱 Core Concepts

**1. Buffered vs. Unbuffered Channels**

{% code lineNumbers="true" expandable="true" %}
```go
// Unbuffered: Blocks sender until receiver is ready (Rendezvous)
func unbufferedDemo() {
    ch := make(chan int) // capacity 0
    go func() {
        ch <- 10 // Blocks here until receiver arrives
    }()
    <-ch // Receiver waits for sender
}

// Buffered: Sender blocks only when buffer is full
func bufferedDemo() {
    ch := make(chan int, 2) // capacity 2
    ch <- 10 // Does not block
    ch <- 20 // Does not block
    ch <- 30 // Blocks here (buffer full)
}
```
{% endcode %}

**2. Sending to Closed Channel**

{% code lineNumbers="true" expandable="true" %}
```go
func closedChannelDemo(ch chan int) {
    close(ch)
    // ch <- 10 // PANIC: send on closed channel
    // Safe receive:
    v, ok := <-ch // Returns zero value (0) and ok=false
    if !ok {
        fmt.Println("Channel is closed")
    }
}
```
{% endcode %}

**3. Who Closes the Channel?**

_Rule: **The sender** closes the channel. Never the receiver._

{% code lineNumbers="true" expandable="true" %}
```go
func producer(ch chan<- int) {
    for i := 0; i < 5; i++ {
        ch <- i
    }
    close(ch) // Sender closes when done
}

func consumer(ch <-chan int) {
    for v := range ch { // Automatically stops when closed
        fmt.Println(v)
    }
}
```
{% endcode %}

**4. The `ok` Idiom**

```go
func okIdiom(ch chan int) {
    go func() {
        ch <- 42
        close(ch)
    }()

    for {
        v, ok := <-ch
        if !ok {
            fmt.Println("Done receiving")
            break
        }
        fmt.Println(v)
    }
}
```

**5. Nil Channel Behavior**

{% code lineNumbers="true" expandable="true" %}
```go
func nilChannelDemo() {
    var ch chan int // nil channel
    // ch <- 10 // Blocks forever
    // v := <-ch // Blocks forever
    // close(ch) // Panics

    // Safe usage in select:
    select {
    case v := <-ch:
        fmt.Println(v)
    default:
        fmt.Println("No operation on nil channel")
    }
}
```
{% endcode %}

***





## Concurrency Patterns

&#x20;**Worker Pool with Graceful Shutdown**

Implement a pool of background workers that processes tasks from a queue and shuts down cleanly only after processing all remaining items once the queue is stopped.

{% code lineNumbers="true" expandable="true" %}
```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func worker(id int, queue <-chan int, wg *sync.WaitGroup) {
	defer wg.Done()
	// Loop terminates naturally ONLY when queue is closed AND empty
	for task := range queue {
		fmt.Printf("Worker %d processing task %d\n", id, task)
		time.Sleep(100 * time.Millisecond) 
	}
	fmt.Printf("Worker %d shut down cleanly.\n", id)
}

func main() {
	queue := make(chan int, 10)
	var wg sync.WaitGroup

	// Start 2 workers
	for i := 1; i <= 2; i++ {
		wg.Add(1)
		go worker(i, queue, &wg)
	}

	// Add data
	for i := 1; i <= 5; i++ {
		queue <- i
	}

	close(queue) // Step 1: Signal no more work is coming
	wg.Wait()    // Step 2: Wait for workers to drain remaining tasks
	fmt.Println("All workers finished. Main exiting.")
}
```
{% endcode %}



**Work Queue with Context Timeout**

Implement a work pipeline where workers instantly abandon processing and drop tasks if a global timeout context expires.

{% code lineNumbers="true" expandable="true" %}
```go
package main

import (
	"context"
	"fmt"
	"sync"
	"time"
)

func process(ctx context.Context, id int, queue <-chan int, wg *sync.WaitGroup) {
	defer wg.Done()
	for {
		select {
		case task, ok := <-queue:
			if !ok {
				return // Queue closed
			}
			time.Sleep(200 * time.Millisecond) // Simulate work
			fmt.Printf("Worker %d finished task %d\n", id, task)
		case <-ctx.Done():
			fmt.Printf("Worker %d stopped: %v\n", id, ctx.Err())
			return // Context timed out, drop everything and exit
		}
	}
}

func main() {
	// Global cancellation fires automatically after 250ms
	ctx, cancel := context.WithTimeout(context.Background(), 250*time.Millisecond)
	defer cancel()

	queue := make(chan int, 10)
	var wg sync.WaitGroup

	wg.Add(1)
	go process(ctx, 1, queue, &wg)

	for i := 1; i <= 5; i++ {
		queue <- i
	}

	wg.Wait()
	fmt.Println("Main exiting.")
}
```
{% endcode %}



**Alternating Output (The "Ping-Pong" Pattern)**

_Use two goroutines to alternate printing numbers or text sequentially (e.g., Goroutine A prints "Ping", Goroutine B prints "Pong")._

{% code lineNumbers="true" expandable="true" %}
```go
package main

import (
	"fmt"
	"sync"
)

func player(name string, myTurn chan bool, nextTurn chan bool, wg *sync.WaitGroup) {
	defer wg.Done()
	for i := 1; i <= 3; i++ {
		<-myTurn // Wait for my turn token
		fmt.Println(name)
		nextTurn <- true // Pass the turn token to the next player
	}
}

func main() {
	pingTurn := make(chan bool, 1)
	pongTurn := make(chan bool)
	var wg sync.WaitGroup

	wg.Add(2)
	go player("Ping", pingTurn, pongTurn, &wg)
	go player("Pong", pongTurn, pingTurn, &wg)

	pingTurn <- true // Toss the initial token to kickstart the game

	wg.Wait()
	close(pingTurn)
	close(pongTurn)
}
```
{% endcode %}

**"Ping-Pong" with timeout:**

{% code lineNumbers="true" expandable="true" %}
```go
// You can edit this code!
// Click here and start typing.
package main

import (
	"context"
	"fmt"
	"sync"
	"time"
)

func PingPong(ctx context.Context, str string, myTurn, nextTurn chan bool, wg *sync.WaitGroup) {
	defer wg.Done()
	for {
		select {
		case <-myTurn:
			fmt.Println(str)
			time.Sleep(200 * time.Millisecond)
			select {
			case <-ctx.Done():
				return
			case nextTurn <- true:
			}

		case <-ctx.Done():
			return
		}
	}

}

func main() {
	ping := make(chan bool)
	pong := make(chan bool)
	ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
	defer cancel()

	var wg sync.WaitGroup

	//spawn go routine
	wg.Add(2)
	go PingPong(ctx, "Ping", ping, pong, &wg)
	go PingPong(ctx, "Pong", pong, ping, &wg)

	ping <- true

	wg.Wait()
	close(ping)
	close(pong)
  fmt.Println("Game Over!!")
}

```
{% endcode %}





**Rate Limiter (Token Bucket Pattern)**\
The Question: _Implement a function that processes incoming requests but restricts throughput to a strict maximum execution rate (e.g., max 5 requests per second)._

{% code lineNumbers="true" expandable="true" %}
```go
package main

import (
	"fmt"
	"time"
)

func main() {
	// Ticker acts as our rate manager, firing a token every 200ms (5 per second)
	limiter := time.NewTicker(200 * time.Millisecond)
	defer limiter.Stop()

	requests := make(chan int, 5)
	for i := 1; i <= 5; i++ {
		requests <- i
	}
	close(requests)

	for req := range requests {
		<-limiter.C // Blocks and waits for the next tick interval
		fmt.Printf("Processing request %d at %v\n", req, time.Now().Format("15:04:05.000"))
	}
}
```
{% endcode %}



**First-One-Wins (Fan-In Racing)**\
_&#x51;uery multiple mirrors or backend services simultaneously and return only the fastest response, discarding the slower ones without leaking memory._<br>

This pattern is explicitly a Fan-In mechanism. You are taking three entirely separate, pre-existing independent source streams (`server1`, `server2`, `server3`) and manually forcing them into a single local execution point using Go's built-in `select` statement.

```
server1 ───┐
server2 ───┼─> select { case ... } (Hardcoded bottleneck point)
server3 ───┘
```

{% code lineNumbers="true" expandable="true" %}
```go
package main

import (
	"fmt"
	"time"
)

func queryMirror(id int, delay time.Duration) <-chan string {
	ch := make(chan string, 1) // MUST be buffered to prevent slower goroutines from leaking!
	go func() {
		time.Sleep(delay)
		ch <- fmt.Sprintf("Response from server %d", id)
	}()
	return ch
}

func main() {
	// Start three concurrent queries with different response speeds
	server1 := queryMirror(1, 300*time.Millisecond)
	server2 := queryMirror(2, 100*time.Millisecond) // The winner
	server3 := queryMirror(3, 500*time.Millisecond)

	var fastestResponse string

	select {
	case fastestResponse = <-server1:
	case fastestResponse = <-server2:
	case fastestResponse = <-server3:
	}

	fmt.Println("Result:", fastestResponse)
}
```
{% endcode %}



**First-One-Wins (Fan-Out Racing)**

This pattern utilizes both Fan-Out and Fan-In sequentially to manage an unpredictable workload layout dynamically.

1. Fan-Out: The main routine takes a dynamic slice (`[]chan int`) of unknown size at runtime. The `for _, t := range tasks` loop fans out by spawning a custom background goroutine _per slice element_.
2. Fan-In: All of these newly generated background workers race against each other to write into a single, centralized multiplexed `results` buffer channel.

```
               ┌──> Goroutine 1 ──┐
Tasks Slice ───┼──> Goroutine 2 ──┼─> [ results buffer chan ] ──> Consumer
               └──> Goroutine 3 ──┘
```

{% code lineNumbers="true" expandable="true" %}
```go
package main

import (
	"context"
	"fmt"
	"time"
)

func firstWinner(ctx context.Context, cancel context.CancelFunc, tasks []chan int) <-chan int {
	results := make(chan int, len(tasks)) 
	
	for _, t := range tasks {
		go func(taskChan chan int) {
			v, ok := <-taskChan
			if !ok {
				return
			}

			// FIXED: Use select on SEND to catch the context cancellation
			select {
			case results <- v:
				// Successfully sent
			case <-ctx.Done():
				// Main or Orchestrator closed up shop, exit quietly
				return
			}
		}(t)
	}

	go func() {
		_, ok := <-results 
		if ok {
			cancel()       // FIXED: Explicitly trigger context cancellation to save slow workers
			close(results) 
		}
	}()

	return results
}

func main() {
	// Pass cancel down so the orchestrator can trigger it early
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	t1 := make(chan int, 1)
	t2 := make(chan int, 1)
	t3 := make(chan int, 1)

	go func() { time.Sleep(300 * time.Millisecond); t1 <- 10 }()
	go func() { time.Sleep(100 * time.Millisecond); t2 <- 20 }() 
	go func() { time.Sleep(500 * time.Millisecond); t3 <- 30 }()

	tasks := []chan int{t1, t2, t3}

	winnerChan := firstWinner(ctx, cancel, tasks)
	fmt.Println("Fastest result received:", <-winnerChan)
	
	time.Sleep(1 * time.Second) // Safe now! No panics will occur.
}
```
{% endcode %}



\
**Multi-Stage Pipeline**\
Design a concurrent, three-stage data pipeline in Go using channels and goroutines:

1. Stage 1 (Producer): Generates integers `0` to `4` in the background.
2. Stage 2 (Transformer): Concurrently consumes those integers, doubles them, and passes them on.
3. Stage 3 (Consumer): Reads and prints the final values on the main thread.

Constraint: The system must process data as a stream and shut down sequentially from Stage 1 to Stage 3 without causing deadlocks or leaking memory.

{% code lineNumbers="true" expandable="true" %}
```go
package main

import "fmt"

func main() {
	ch1 := make(chan int)
	ch2 := make(chan int)

	// Stage 1: Producer
	go func() {
		defer close(ch1)
		for i := 0; i < 5; i++ {
			ch1 <- i
		}
	}()

	// Stage 2: Transformer (Double)
	go func() {
		defer close(ch2)
		for n := range ch1 {
			ch2 <- n * 2 
		}
	}()

	// Stage 3: Consumer (Runs on main goroutine)
	for v := range ch2 {
		fmt.Println(v)
	}
}
```
{% endcode %}

&#x20;



**Fan-Out / Fan-In**\
Design a highly concurrent system in Go using Fan-Out/Fan-In architecture to parallelize intensive data mutations:

1. Fan-Out: Concurrently distribute workload from a single input stream (`0` to `4`) across a dynamic pool of `N` background workers. Each worker multiplies its assigned values by `2`.
2. Fan-In: Consolidate the output from all parallel background workers back into a single aggregate result channel.

Constraint: The application must orchestrate completion smoothly—ensuring all workers finish their tasks and the combined output channel closes cleanly without deadlocks or resource leaks.

{% code lineNumbers="true" expandable="true" %}
```go
package main

import (
	"fmt"
	"sync"
)

func fanOut(in <-chan int, numWorkers int) <-chan int {
	out := make(chan int)
	var wg sync.WaitGroup

	// 1. Fan-Out: Launch multiple workers reading from the SAME input channel
	for i := 0; i < numWorkers; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for v := range in {
				out <- v * 2 // Independent parallel transformation
			}
		}()
	}

	// 2. Fan-In Orchestrator: Wait for all workers, then close the aggregate output
	go func() {
		wg.Wait()
		close(out)
	}()

	return out
}

func main() {
	in := make(chan int)

	// Producer
	go func() {
		for i := 0; i < 5; i++ {
			in <- i
		}
		close(in)
	}()

	// Distribute work to 3 parallel workers
	out := fanOut(in, 3)

	// Consumer
	for v := range out {
		fmt.Println(v)
	}
}
```
{% endcode %}





**Pub/Sub (Broadcast)**\
Design a basic message broadcast system in Go utilizing the Pub/Sub (Publisher/Subscriber) concurrency pattern:

1. The Broker: Implement an orchestration function (`pubSub`) that accepts a variable slice of subscriber channels and dynamically spins up an isolated background consumer routine for each channel.
2. The Subscribers: Each background routine must range over its individual channel, printing incoming string messages concurrently.

Constraint: The application must prevent early termination leaks by introducing a `sync.WaitGroup` framework, ensuring that the `main` loop blocks safely until all subscribers have completely drained their queues and shut down.<br>

{% code lineNumbers="true" expandable="true" %}
```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// pubSub launches background consumer routines for each subscriber channel
func pubSub(topic string, subs []chan string, wg *sync.WaitGroup) {
	for _, sub := range subs {
		wg.Add(1) // Register each subscriber routine with the main WaitGroup
		go func(s chan string) {
			defer wg.Done()
			for msg := range s {
				fmt.Printf("[%s]: %s\n", topic, msg)
			}
		}(sub)
	}
}

func main() {
	var wg sync.WaitGroup

	// Set up 2 dedicated subscriber channels
	sub1 := make(chan string, 5)
	sub2 := make(chan string, 5)
	subs := []chan string{sub1, sub2}

	// 1. Initialize the Pub/Sub worker routines
	pubSub("TechNews", subs, &wg)

	// 2. Publisher: Broadcast messages to all channels
	messages := []string{"Go Concurrency is fun!", "Always protect your main routine!"}
	for _, msg := range messages {
		for _, sub := range subs {
			sub <- msg
		}
		time.Sleep(10 * time.Millisecond) // Tiny gap to keep printing readable
	}

	// 3. Close the channels to tell the subscribers "no more data is coming"
	for _, sub := range subs {
		close(sub)
	}

	// 4. Wait for the background subscribers to finish printing and exit cleanly
	wg.Wait()
	fmt.Println("Main: All subscribers exited. System shut down safely.")
}
```
{% endcode %}





**Pub/Sub (Multi-Topic Broadcast)**

Design a multi-topic routing system in Go using the Pub/Sub (Publisher/Subscriber) architecture:

1. The Multi-Topic Broker: Implement an orchestration function that processes a topic-to-subscriber mapping (`map[string][]chan string`). It must spawn isolated background workers to listen concurrently to specific message feeds.
2. Targeted Broadcasting: The system must allow a publisher to selectively route messages to all subscribers of a specific topic without leaking data to unrelated streams.

Constraint: The architecture must utilize a `sync.WaitGroup` so the application blocks safely until every channel across all topics is fully drained and closed without deadlocks or leaked goroutines.

```go
package main

import (
	"fmt"
	"sync"
)

// pubSub launches background consumers grouped by their chosen topic
func pubSub(broker map[string][]chan string, wg *sync.WaitGroup) {
	for topic, subs := range broker {
		for _, sub := range subs {
			wg.Add(1)
			go func(t string, s chan string) {
				defer wg.Done()
				// This loop runs until this specific subscriber channel is closed
				for msg := range s {
					fmt.Printf("[%s Subscriber]: %s\n", t, msg)
				}
			}(topic, sub)
		}
	}
}

func main() {
	var wg sync.WaitGroup

	// 1. Initialize Subscriber Channels
	techSub1 := make(chan string, 2)
	techSub2 := make(chan string, 2)
	foodSub1 := make(chan string, 2)

	// 2. Build the Broker Map (Grouping subscribers by multiple topics)
	broker := map[string][]chan string{
		"tech": {techSub1, techSub2},
		"food": {foodSub1},
	}

	// 3. Start all background subscriber consumers
	pubSub(broker, &wg)

	// 4. Publisher: Broadcast messages only to relevant topic channels
	broker["tech"][0] <- "Go 1.35 introduces new compiler optimizations!"
	broker["tech"][1] <- "Go 1.35 introduces new compiler optimizations!"
	
	broker["food"][0] <- "Secret to a perfect sourdough crust."

	// 5. Graceful Shutdown: Close all channels to break the background 'for range' loops
	for _, subs := range broker {
		for _, sub := range subs {
			close(sub)
		}
	}

	// 6. Block until all subscribers process their remaining messages and exit
	wg.Wait()
	fmt.Println("Main: All topic subscribers exited cleanly.")
}
```



**Backpressure (Buffered Channel)**

{% code lineNumbers="true" expandable="true" %}
```go
package main

import (
	"fmt"
	"time"
)

func main() {
	// 1. Create a bounded channel with a small buffer of 2
	ch := make(chan int, 2) 

	// Fast Producer
	go func() {
		for i := 1; i <= 5; i++ {
			fmt.Printf("[Producer]: Sending item %d...\n", i)
			ch <- i // This blocks automatically when the buffer is full (Backpressure)
			fmt.Printf("[Producer]: Item %d sent successfully!\n", i)
		}
		close(ch)
	}()

	// Slow Consumer (Runs on the main goroutine)
	for v := range ch {
		fmt.Printf("[Consumer]: Processing item %d\n", v)
		time.Sleep(200 * time.Millisecond) // Simulated heavy bottleneck work
	}
	
	fmt.Println("Main: Pipeline drained cleanly.")
}
```
{% endcode %}

***



