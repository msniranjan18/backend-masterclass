# url fetcher - 2
```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"os"
	"sync"
	"time"
)

type result struct {
	url        string
	statusCode int
	body       []byte
	err        error
	latency    time.Duration
}

func get(url string, ch chan<- result, wg *sync.WaitGroup) {
	defer wg.Done()
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

	// result := make(chan result)
	result := make(chan result, len(urls)) // notice: buffered chan with len(urls) means eq to no of goroutine
	// if the url list is big then we should not use the len(url); instead a fixed size buffer is recommended

	var wg sync.WaitGroup

	for _, url := range urls {
		wg.Add(1)
		go get(url, result, &wg)
	}

	wg.Wait()
	close(result)

	// go func() {
	// 	wg.Wait()
	// 	close(result)
	// }()

	// for range urls {
	// 	r := <-result
	for r := range result {
		if r.err != nil {
			fmt.Fprintf(os.Stderr, "error in fetching url:[%s]; err:%s\n", r.url, r.err.Error())
		} else {
			fmt.Fprintf(os.Stdout, "successful; url:[%s]; statusCode:%d, latency:%v, body:%+v\n", r.url, r.statusCode, r.latency, len(r.body))
		}
	}
}
```

```text
$go run url-fetcher.go 
successful; url:[https://google.com]; statusCode:200, latency:1.019s, body:89326
successful; url:[https://amazon.com]; statusCode:200, latency:1.262s, body:3781
$
```


This version switches to the classic fan-out / fan-in with WaitGroup + buffered channel pattern. Here is a breakdown of how this implementation behaves, why the buffer size is critical here, and the subtle trade-offs in the commented-out sections.

## 1. Why make(chan result, len(urls)) Prevents a Deadlock Here
In this version, `wg.Wait()` runs synchronously in main before the for `r := range result` loop starts reading from the channel:

*   Because the buffer size (`len(urls)`) matches the exact number of goroutines spawned (and each goroutine sends at most 1 item), every `get()` goroutine can push its result into the buffer without blocking, call `defer wg.Done()`, and exit.   

*   Once all goroutines exit, `wg.Wait()` unblocks, `close(result)` seals the channel, and `for r := range result` drains all buffered items until hitting the closed marker.

*   What if result were unbuffered (`make(chan result)`)? This exact code would deadlock immediately: every `get()` goroutine would block on `ch <- result{...}` waiting for a receiver, `defer wg.Done()` would never run, and main would stay stuck forever on `wg.Wait()` without ever reaching `for r := range result`.

## 2. Synchronous wg.Wait() vs. The Commented-Out go func()
*   Current (`wg.Wait()` inline): main blocks until all URLs finish (even the slowest one) before printing any output. All results sit in the channel buffer until the batch completes, then print in a burst.

*   Commented-out (`go func() { wg.Wait(); close(result) }()`): Moving `wg.Wait()` and `close(result)` into a separate monitor goroutine lets main reach `for r := range result` immediately—streaming each result to `stdout` the instant it arrives while still safely closing the channel once the last worker finishes (and works regardless of whether the channel is buffered or unbuffered).


## Clean code:
```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"os"
	"sync"
	"time"
)

type result struct {
	url        string
	statusCode int
	body       []byte
	err        error
	latency    time.Duration
}

func get(url string, ch chan<- result, wg *sync.WaitGroup) {
	defer wg.Done()
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
	//result := make(chan result, len(urls)) // notice: buffered chan with len(urls) means eq to no of goroutine
	// if the url list is big then we should not use the len(url); instead a fixed size buffer is recommended

	var wg sync.WaitGroup

	for _, url := range urls {
		wg.Add(1)
		go get(url, result, &wg)
	}

	// wg.Wait()
	// close(result)

	go func() {
		wg.Wait()
		close(result)
	}()

	// for range urls {
	// 	r := <-result
	for r := range result {
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
successful; url:[https://google.com]; statusCode:200, latency:614ms, body:89455
successful; url:[https://amazon.com]; statusCode:200, latency:990ms, body:3781
$
```

This version hits the idiomatic Go sweet spot for fan-out/fan-in concurrency—clean ownership, streaming output, and zero deadlocks.

### Key Points
*   Safe Resource Ownership: `get()` reads the full body via `io.ReadAll(resp.Body)` and closes it with `defer resp.Body.Close()` before exiting, passing only plain values (`statusCode`, `[]byte`, `latency`) over the channel. There is no shared `*http.Response` across goroutines and zero risk of reading a closed body.

*   Non-Blocking Closer Goroutine (`go func() { wg.Wait(); close(result) }()`): Running `wg.Wait()` in a separate goroutine allows `main()` to immediately enter `for r := range result` and print results as soon as each URL finishes—avoiding the deadlock that would happen if `wg.Wait()` ran inline with an unbuffered channel.

*   Clean range Termination: Because `close(result)` is guaranteed to run only after every `wg.Done()` completes, for `r := range result` drains all results and exits cleanly without needing to manually count `len(urls)`.

*   Buffer Sizing Flexibility: Works safely with an unbuffered channel (`make(chan result)`), and using a small fixed-size buffer for large URL slices (instead of len(urls)) is spot on—it prevents holding all response []byte slices in the channel buffer at once.

*   One Remaining Production Gap: `http.Get(url)` uses `http.DefaultClient` with no `timeout`, so a hanging server can keep a goroutine and `wg.Wait()` blocked indefinitely.


