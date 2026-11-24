# url fetcher 1

```go
package main

import (
	"fmt"
	"net/http"
	"os"
	"time"
)

type result struct {
	url     string
	resp    *http.Response
	err     error
	latency time.Duration
}

func get(url string, ch chan<- result) {
	start := time.Now()

	if resp, err := http.Get(url); err != nil {
		ch <- result{url, nil, err, 0}
	} else {
		t := time.Since(start).Round(time.Millisecond)
		ch <- result{url, resp, nil, t}
		resp.Body.Close()
	}
}

func main() {
	urls := []string{
		"https://amazon.com",
		"https://google.com",
	}

	result := make(chan result)

	for _, url := range urls {
		go get(url, result)
	}

	for range urls {
		r := <-result
		if r.err != nil {
			fmt.Fprintf(os.Stderr, "error in fetching url:[%s]; err:%s\n", r.url, r.err.Error())
		} else {
			fmt.Fprintf(os.Stdout, "successful; url:[%s]; resp:%#v\n", r.url, r.resp.StatusCode)
		}
	}
}
```

```text
$ go run url-fetcher.go 
successful; url:[https://google.com]; resp:200
successful; url:[https://amazon.com]; resp:200
$

```

This Go program concurrently fetches multiple URLs using goroutines and channels, logging the HTTP status code or error for each request as soon as it completes.

## How It Works
* Data Structure (`result` struct): Bundles the outcome of each HTTP request—including the `url`, the `*http.Response`, any `err`, and the measured latency.

* Worker Function (`get`):
 - Records the start time and makes an `http.Get` request.

 - Sends a `result` struct into a send-only channel (`chan<- result`).

 - Closes the response body (`resp.Body.Close()`) upon a successful fetch to prevent resource leaks.

* Concurrency & Synchronization (`main`):

 - Creates an unbuffered channel (`result`) to collect responses.

 - Loops over the `urls` slice and spawns a separate goroutine (`go get(...)`) for each URL so they execute simultaneously rather than waiting for one another.

 - Uses a second loop (`for range urls`) to receive exactly as many results from the channel as there were URLs launched. Because channel receives block until data arrives, this ensures `main` waits for all goroutines to finish before exiting. 

## Key Observations
* Non-Deterministic Order: 
    Notice that amazon.com is listed first in the urls slice, yet google.com printed first in the terminal output. Because the requests run concurrently, results are printed in the order they finish—not the order they were started.

* Deterministic Completion Without `sync.WaitGroup`:
    Instead of importing the sync package and tracking goroutine lifecycles with `wg.Add(1)` and `wg.Done()`, the `main` function relies on a counted receive loop (`for range urls`). Because every spawned goroutine is guaranteed to send exactly one result struct back (whether the HTTP call succeeds or fails), reading len(urls) times from the channel acts as a natural barrier—preventing main from exiting prematurely while avoiding `WaitGroup` overhead.

* No `close(result)` Required: 
    Because the receiver knows the exact number of expected messages upfront (`len(urls)`) rather than using a `for r := range result` channel loop, there is no need to track when all senders are finished just to safely close() the channel. `Leaving the unbuffered channel open is completely safe in Go`; once main exits (or the channel goes out of scope with no active references), the garbage collector reclaims it.

* Unbuffered Synchronous Handshake: 
    make(chan result) creates an unbuffered channel, meaning each worker goroutine blocks on ch <- result{...} until main is ready to receive it. This provides natural backpressure and synchronization without needing locks or mutexes around shared state.

* Direction-Restricted Channel Signature (`chan<- result`): 
    The get function accepts a send-only channel (`chan<- result`) rather than a bidirectional chan result. This enforces at compile time that worker goroutines can only push results and cannot accidentally read from or close the channel.

* Guaranteed One-to-One Send Contract: 
    In get(), both the error branch (if err != nil) and the success branch (else) send a result struct into ch. This strict 1:1 guarantee between goroutines launched and messages sent is what makes the for range urls receive loop deterministic and prevents deadlocks (where main would block forever waiting on a channel receive that never arrives).

* Shadowed Variable Naming (result type vs. result channel): 
    Inside main(), result := make(chan result) reuses the struct type name result as the local channel variable name—valid in Go because types and variables live in separate scopes within the function body, though it prevents referencing the result type again inside main().

## Minor Edge Case:
* In get(), resp.Body.Close() is called after sending resp over the channel. If main() were modified to read the response body rather than just checking StatusCode, this could cause a race condition where the body closes while main() is reading it.

* In the else block, resp.Body.Close() runs immediately after the channel send (ch <- result{...}) completes. Since ch is unbuffered, resp.Body.Close() happens right after main receives the result struct, ensuring the underlying TCP connection isn't leaked while keeping the status code readable.



### 1. Lets understand the Edge case (above last two points):
    In Go's `net/http`, `resp.StatusCode` and `resp.Header` are plain values already parsed into memory when `http.Get()` returns. 
    
    However, `resp.Body` is an `io.ReadCloser` streaming directly from the active TCP socket.

in current code:
```go
    ch <- result{url, resp, nil, t}
    resp.Body.Close()
```

* Why it works right now (Point 2): 
    Because `ch` is unbuffered, the send (`ch <- ...`) is a synchronous handshake—`get()` pauses on that line until `main()` executes `r := <-result`. Once the handoff finishes, `main()` prints `r.resp.StatusCode` (which is already in memory) while `get()` immediately moves to the next line and closes the TCP stream (`resp.Body.Close()`). No connection is leaked, and reading StatusCode doesn't touch Body.

* Where the race condition happens (Point 1): 
    The moment `ch <- result{...}` finishes, both goroutines hold a pointer to the same `http.Response` simultaneously without any synchronization between them:

    *   `get()` immediately executes `resp.Body.Close()`.

    *   `main()` proceeds to the next line in its loop.

    *   If you added `io.ReadAll(r.resp.Body)` inside `main()`, the Go scheduler could run `resp.Body.Close()` in `get()` before or during main()'s read—causing `http: read on closed response body` errors or truncated reads.

### 2. What Happens If `ch` Is Buffered (`make(chan result, len(urls))`)?
if you change `result := make(chan result)` to a buffered channel:
 - No Rendezvous / Handshake: ch <- result{...} no longer waits for `main()` to receive. It drops the struct into the channel's ring buffer (`hchan.buf`) and immediately continues to the next line.

 - Immediate Closure Before `main()` Even Receives: `get()` calls `resp.Body.Close()` before `main()` has even pulled `r` out of the channel.

 - Guaranteed Failure If Reading Body: By the time `main()` executes `r := <-result`, `r.resp.Body` is already closed. Checking `r.resp.StatusCode` still works (because it's just an int on the struct), but any attempt to read `r.resp.Body` in `main()` will almost certainly fail with a closed-body error.

### 3. What Is the Correct Way?
Follow the Go concurrency rule of single ownership: whichever goroutine is responsible for reading `resp.Body` is the one that must close it. You have two clean patterns depending on your design:

#### Option A: Read and Close Inside the Worker (get) — Recommended
Keep the I/O work inside the worker goroutine, close the body with defer, and pass only parsed data (like []byte or StatusCode) over the channel—never the live *http.Response.

```go
type result struct {
	url        string
	statusCode int
	body       []byte
	err        error
	latency    time.Duration
}

func get(url string, ch chan<- result) {
	start := time.Now()
	resp, err := http.Get(url)
	if err != nil {
		ch <- result{url: url, err: err}
		return
	}
	defer resp.Body.Close() // Guaranteed to run after io.ReadAll finishes

	body, err := io.ReadAll(resp.Body)
	t := time.Since(start).Round(time.Millisecond)
	ch <- result{url: url, statusCode: resp.StatusCode, body: body, err: err, latency: t}
}
```
* Why this is best: get() owns resp.Body from creation to Close(). Whether ch is buffered or unbuffered, there is zero shared mutable state and zero race conditions.

#### Option B: Transfer Ownership to main() and Close There
If `main()` genuinely needs to stream `resp.Body`, `get()` must not call `resp.Body.Close()` at all—it hands over ownership completely, and `main()` closes it after reading:

```go
// Inside get():
ch <- result{url, resp, nil, t}
// Do NOT call resp.Body.Close() here

// Inside main():
r := <-result
if r.err != nil {
	// handle error
} else {
	defer r.resp.Body.Close() // Or call r.resp.Body.Close() at the end of each loop iteration
	// safe to read r.resp.Body here
}
```

Note: Inside a loop in main(), avoid defer r.resp.Body.Close() directly in the loop body since defer runs when main() exits, not at the end of each iteration—either call r.resp.Body.Close() explicitly at the end of the else block or wrap each iteration's processing in a helper function.

code:
```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"os"
	"time"
)

type result struct {
	url        string
	statusCode int
	body       []byte
	err        error
	latency    time.Duration
}

func get(url string, ch chan<- result) {
	start := time.Now()
	resp, err := http.Get(url)

	if err != nil {
		ch <- result{url: url, err: err}
		return
	}
	defer resp.Body.Close() // Guaranteed to run after io.ReadAll finishes
	t := time.Since(start).Round(time.Millisecond)

	body, err := io.ReadAll(resp.Body)
	ch <- result{url: url, statusCode: resp.StatusCode, body: body, err: err, latency: t}
}

func main() {
	urls := []string{
		"https://amazon.com",
		"https://google.com",
	}

	result := make(chan result)

	for _, url := range urls {
		go get(url, result)
	}

	for range urls {
		r := <-result
		if r.err != nil {
			fmt.Fprintf(os.Stderr, "error in fetching url:[%s]; err:%s\n", r.url, r.err.Error())
		} else {
			fmt.Fprintf(os.Stdout, "successful; url:[%s]; statusCode:%d, latency:%v, body:%+v\n", r.url, r.statusCode, r.latency, len(r.body))
		}
	}
}
```

Output:
```text
$ go run url-fetcher.go 
successful; url:[https://google.com]; statusCode:200, latency:754ms, body:89352
successful; url:[https://amazon.com]; statusCode:200, latency:998ms, body:3781
$
```