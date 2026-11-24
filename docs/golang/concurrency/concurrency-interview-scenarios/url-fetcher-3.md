# url fetcher - 3
To prevent slow or hanging servers from blocking your goroutines indefinitely (in url-fetcher-2.md), you have two main approaches in Go's net/http. Both cover the entire request lifecycle—from DNS lookup and TLS handshake all the way through io.ReadAll(resp.Body).

## 1. Custom http.Client (Simplest for a Uniform Per-Request Timeout)
Instead of calling `http.Get()` (which uses `http.DefaultClient` with no `timeout`), instantiate a single reusable `*http.Client` with `Timeout` set and pass it to your workers.


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

func get(sharedHttpClient *http.Client, url string, ch chan<- result, wg *sync.WaitGroup) {
	defer wg.Done()
	start := time.Now()
	resp, err := sharedHttpClient.Get(url)

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
	var wg sync.WaitGroup

	// Covers connection + headers + full body read for each individual request
	sharedHttpClient := &http.Client{
		Timeout: 1 * time.Second, // timeout for each request
	}

	for _, url := range urls {
		wg.Add(1)
		go get(sharedHttpClient, url, result, &wg)
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
successful; url:[https://google.com]; statusCode:200, latency:636ms, body:89513
error in fetching url:[https://amazon.com]; err:Get "https://www.amazon.com/": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
$
```

Notice that amazon server did not responded within 1 second resulting timeout thus the error "context deadline exceeded"


## 2. context.WithTimeout + http.NewRequestWithContext (Best for Batch Deadlines or Cancellation)
Use `context.WithTimeout` when you want one overall deadline across the entire batch (or if you need to propagate parent cancellation signals). Notice how `defer cancel()` must happen after `for r := range result` finishes—if you `defer cancel()` inside `get()` before `io.ReadAll()` completes, or cancel before reading the body, `io.ReadAll()` will fail with `context canceled`.

```go
package main

import (
	"context"
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

func get(parentCtx context.Context, sharedHttpClient *http.Client, url string, ch chan<- result, wg *sync.WaitGroup) {
	defer wg.Done()

	childCtx, childCancel := context.WithTimeout(parentCtx, 1*time.Second)
	defer childCancel()

	start := time.Now()

	req, err := http.NewRequestWithContext(childCtx, http.MethodGet, url, nil)
	if err != nil {
		ch <- result{url: url, err: err}
		return
	}

	resp, err := sharedHttpClient.Do(req)
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
	var wg sync.WaitGroup

	// Covers connection + headers + full body read for each individual request
	sharedHttpClient := &http.Client{
		Timeout: 3 * time.Second, // timeout for each request
	}

	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()
	for _, url := range urls {
		wg.Add(1)
		go get(ctx, sharedHttpClient, url, result, &wg)
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
```
$ go run url-fetcher.go 
successful; url:[https://google.com]; statusCode:200, latency:668ms, body:89512
error in fetching url:[https://amazon.com]; err:Get "https://www.amazon.com/": context deadline exceeded
$
```

This version moves from a script-level concurrency example to production-grade Go HTTP engineering. By decoupling the request construction (NewRequestWithContext), transport management (sharedHttpClient), and execution (Do), you gain full control over cancellation trees, TCP connection pooling, and resource limits.
## 1. Why This Architecture Beats Your Previous Versions
| Aspect | Old (`http.Get` / `client.Get`) | Current (`NewRequestWithContext` + `sharedHttpClient.Do`) |
|---|---|---|
| **Cancellation** | None (`http.Get` uses `context.Background()` internally). A slow server holds the goroutine and socket open forever. | Hierarchical cancellation: canceling `parentCtx` (at 10s or on shutdown) immediately aborts every in-flight TCP dial, TLS handshake, and body read across all workers. |
| **Request Customization** | Locked to GET with default headers. | Allows injecting custom headers (`Authorization`, `User-Agent`, trace IDs) before `Do(req)` executes. |
| **Connection Pooling** | If created per-goroutine, every worker opens a brand-new TCP/TLS handshake and leaks idle connection pools. | Reuses the underlying `http.Transport` connection pool across all goroutines, dramatically reducing TLS handshake overhead and file descriptor usage. |
| **Timeout Granularity** | Either no timeout at all (`http.Get`) or a single flat timeout (`client.Get`). | Two-tier deadlines: 10s overall batch SLA (`parentCtx`) + 1s individual request cap (`childCtx`). |
| **Body Ownership** | Often closed by the wrong goroutine, causing race conditions (`http: read on closed response body`) if `main()` tries to read it. | Worker owns the body end-to-end via `defer resp.Body.Close()`, passing only parsed data (`[]byte`, `int`) over the channel. |
| **Error Propagation** | Errors frequently swallowed or printed inside the worker, mixing I/O noise with business logic. | Errors travel as structured `result.err` and are handled centrally in `main()`, keeping workers clean and testable. |
| **Testability** | Hard to stub, since `http.Get` hits the real network directly. | Inject a fake `*http.Client` (or custom `Transport`) for unit tests without hitting the network. |
|---|---|---|