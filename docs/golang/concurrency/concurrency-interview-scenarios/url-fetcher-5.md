# url fetcher - 5
Worker - Multiple
Producer - Multiple
Consumer - Multiple

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"io"
	"net"
	"net/http"
	"os"
	"os/signal"
	"sync"
	"syscall"
	"time"
)

const (
	producerPoolSize = 2
	workerPoolSize   = 5
	consumerPoolSize = 3
	jobBufferSize    = 6
	resultBufferSize = 4
	maxResponseSize  = 5 << 20 // 5 MB
)

type Job struct {
	producerId int
	url        string
}

type Result struct {
	producerId int
	workerId   int
	url        string
	statusCode int
	body       []byte
	err        error
	latency    time.Duration
}

func get(parentCtx context.Context,
	sharedHttpClient *http.Client,
	id int,
	url string) Result {

	childCtx, childCancel := context.WithTimeout(parentCtx, 2*time.Second)
	defer childCancel()

	start := time.Now()

	req, err := http.NewRequestWithContext(childCtx, http.MethodGet, url, nil)
	if err != nil {
		return Result{workerId: id, url: url, err: err}
	}

	resp, err := sharedHttpClient.Do(req)
	if err != nil {
		return Result{workerId: id, url: url, err: err}
	}

	defer resp.Body.Close() // Guaranteed to run after io.ReadAll finishes

	//body, err := io.ReadAll(resp.Body)
	body, err := io.ReadAll(io.LimitReader(resp.Body, maxResponseSize+1))
	if err != nil {
		return Result{workerId: id, url: url, err: err}
	}
	if len(body) > maxResponseSize {
		return Result{workerId: id, url: url, err: fmt.Errorf("response body exceeds maximum size")}
	}

	t := time.Since(start).Round(time.Millisecond)

	return Result{
		workerId:   id,
		url:        url,
		statusCode: resp.StatusCode,
		body:       body,
		err:        err,
		latency:    t,
	}
}

func worker(parentCtx context.Context,
	id int,
	sharedHttpClient *http.Client,
	job <-chan Job, // ingress chan
	result chan<- Result, // egress chan
	wg *sync.WaitGroup) {
	defer wg.Done()

	for {
		// Avoid picking up additional jobs after cancellation.
		if parentCtx.Err() != nil { // This reduces unnecessary job processing after cancellation.
			return
		}

		select {
		case j, ok := <-job:
			if !ok { // means job is closed
				return
			}

			// Cancellation might have happened while receiving.
			if parentCtx.Err() != nil { // This reduces unnecessary job processing after cancellation.
				return
			}

			r := get(parentCtx, sharedHttpClient, id, j.url)

			select {
			case result <- r:
			case <-parentCtx.Done():
				return
			}

		case <-parentCtx.Done():
			return
		}
	}
}

func consumer(id int, ch <-chan Result, wg *sync.WaitGroup) {
	defer wg.Done()

	for r := range ch {
		if r.err != nil {
			fmt.Fprintf(os.Stderr, "[P-%d -> W-%d -> C-%d] ERROR; url=[%s]: %v\n",
				r.producerId, r.workerId, id, r.url, r.err)
		} else {
			fmt.Fprintf(os.Stdout, "[P-%d -> W-%d -> C-%d] OK; url=[%s]: status=%d, latency=%v, bytes=%d\n",
				r.producerId, r.workerId, id, r.url, r.statusCode, r.latency, len(r.body))
		}
	}
}

func producer(ctx context.Context, id int, urls []string, jobs chan<- Job, wg *sync.WaitGroup) {
	defer wg.Done()
	for _, url := range urls {
		select {
		case jobs <- Job{producerId: id, url: url}:
		case <-ctx.Done():
			return
		}
	}
}

func main() {
	urls := []string{
		"https://google.com",
		"https://example.com",
		"https://httpbin.org/get",

		//Different HTTP status codes
		"https://httpbin.org/status/200",
		"https://httpbin.org/status/400",
		"https://httpbin.org/status/404",
		"https://httpbin.org/status/503",

		//Slow responses
		"https://httpbin.org/delay/5",

		//Redirect handling
		"https://httpbin.org/redirect/3",
		"https://httpbin.org/absolute-redirect/2",
		"https://httpbin.org/relative-redirect/2",

		//Large response bodies
		"https://httpbin.org/bytes/1048576",
		"https://httpbin.org/stream/100",
		"https://httpbin.org/stream/500",
	}

	// Root signal context -> child batch timeout context
	sigCtx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()

	ctx, cancel := context.WithTimeout(sigCtx, 10*time.Second)
	defer cancel()

	transport := &http.Transport{
		DialContext: (&net.Dialer{
			Timeout:   2 * time.Second,
			KeepAlive: 30 * time.Second,
		}).DialContext,
		TLSHandshakeTimeout:   2 * time.Second,
		ResponseHeaderTimeout: 2 * time.Second,
		MaxIdleConns:          100,
		MaxIdleConnsPerHost:   10,
		MaxConnsPerHost:       10,
		IdleConnTimeout:       90 * time.Second,
	}

	sharedHttpClient := &http.Client{
		Transport: transport, // Contexts control per-request and batch timeouts
	}
	defer sharedHttpClient.CloseIdleConnections()

	jobs := make(chan Job, jobBufferSize)
	results := make(chan Result, resultBufferSize)

	// 1. Start Consumers
	var wgConsumer sync.WaitGroup
	for i := 1; i <= consumerPoolSize; i++ {
		wgConsumer.Add(1)
		go consumer(i, results, &wgConsumer)
	}

	// 2. Start Workers + closer for results
	var wgWorker sync.WaitGroup
	// spawm the workers
	for i := 1; i <= workerPoolSize; i++ {
		wgWorker.Add(1)
		go worker(ctx, i, sharedHttpClient, jobs, results, &wgWorker)
	}
	go func() {
		wgWorker.Wait()
		close(results)
	}()

	// Producer: respects ctx.Done() so it never blocks if batch times out early
	// 3. Start Producers (partitioning input slice across producerPoolSize) + closer for jobs
	var wgProducer sync.WaitGroup
	chunkSize := (len(urls) + producerPoolSize - 1) / producerPoolSize
	for i := 0; i < producerPoolSize; i++ {
		start := i * chunkSize
		if start >= len(urls) {
			break
		}
		end := min(start+chunkSize, len(urls))

		wgProducer.Add(1)
		go producer(ctx, i+1, urls[start:end], jobs, &wgProducer)
	}
	go func() {
		wgProducer.Wait()
		close(jobs)
	}()

	// Wait for final stage to finish draining
	wgConsumer.Wait()

	if errors.Is(sigCtx.Err(), context.Canceled) {
		fmt.Fprintln(os.Stderr, "Pipeline stopped via OS signal.")
	} else if errors.Is(ctx.Err(), context.DeadlineExceeded) {
		fmt.Fprintln(os.Stderr, "Pipeline stopped due to 10s batch deadline.")
	}
}

```

### Output
#### CTRL+C
```text
$go run url-fetcher.go 
^C[P-0 -> W-4 -> C-1] ERROR; url=[https://httpbin.org/status/200]: Get "https://httpbin.org/status/200": context canceled
[P-0 -> W-1 -> C-2] ERROR; url=[https://google.com]: Get "https://google.com": context canceled
^C <------------------------------- ctrl+c
Pipeline stopped via OS signal.
$
```
### parentCtx.Done()
```text
$ go run url-fetcher.go 
[P-0 -> W-5 -> C-1] OK; url=[https://example.com]: status=200, latency=345ms, bytes=577
[P-0 -> W-4 -> C-2] OK; url=[https://google.com]: status=200, latency=615ms, bytes=89347
[P-0 -> W-3 -> C-3] OK; url=[https://httpbin.org/get]: status=200, latency=844ms, bytes=217
[P-0 -> W-1 -> C-1] OK; url=[https://httpbin.org/status/200]: status=200, latency=844ms, bytes=0
[P-0 -> W-2 -> C-2] OK; url=[https://httpbin.org/status/400]: status=400, latency=844ms, bytes=0
[P-0 -> W-1 -> C-1] ERROR; url=[https://httpbin.org/redirect/3]: Get "https://httpbin.org/redirect/3": context deadline exceeded
[P-0 -> W-3 -> C-3] ERROR; url=[https://httpbin.org/delay/5]: Get "https://httpbin.org/delay/5": context deadline exceeded
Pipeline stopped due to context batch deadline.
$
```
### Normal 
```text
$ go run url-fetcher.go 
[P-0 -> W-4 -> C-1] OK; url=[https://example.com]: status=200, latency=412ms, bytes=577
[P-0 -> W-3 -> C-2] OK; url=[https://google.com]: status=200, latency=753ms, bytes=89455
[P-0 -> W-2 -> C-3] OK; url=[https://httpbin.org/status/200]: status=200, latency=838ms, bytes=0
[P-0 -> W-5 -> C-1] OK; url=[https://httpbin.org/get]: status=200, latency=836ms, bytes=217
[P-0 -> W-1 -> C-2] OK; url=[https://httpbin.org/status/400]: status=400, latency=836ms, bytes=0
[P-0 -> W-3 -> C-3] OK; url=[https://httpbin.org/status/503]: status=503, latency=339ms, bytes=0
[P-0 -> W-4 -> C-1] OK; url=[https://httpbin.org/status/404]: status=404, latency=680ms, bytes=0
[P-0 -> W-3 -> C-2] OK; url=[https://httpbin.org/relative-redirect/2]: status=200, latency=750ms, bytes=275
[P-0 -> W-1 -> C-3] OK; url=[https://httpbin.org/absolute-redirect/2]: status=200, latency=1.006s, bytes=275
[P-0 -> W-5 -> C-1] OK; url=[https://httpbin.org/redirect/3]: status=200, latency=1.176s, bytes=275
[P-0 -> W-1 -> C-2] OK; url=[https://httpbin.org/stream/500]: status=400, latency=255ms, bytes=107
[P-0 -> W-3 -> C-3] OK; url=[https://httpbin.org/stream/100]: status=200, latency=582ms, bytes=17490
[P-0 -> W-2 -> C-1] ERROR; url=[https://httpbin.org/delay/5]: Get "https://httpbin.org/delay/5": net/http: timeout awaiting response headers
[P-0 -> W-4 -> C-2] ERROR; url=[https://httpbin.org/bytes/1048576]: context deadline exceeded
(base)
```



