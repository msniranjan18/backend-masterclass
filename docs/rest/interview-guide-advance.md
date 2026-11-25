# Interview Guide - Advance

**Production-Level Interview Questions & Scenario Deep Dives**

* Q1: Debugging Ephemeral Port Exhaustion (`TIME_WAIT`)
* Q2: Designing a High-Throughput, Idempotent Payment Webhook API
* Q3: Handling Long-Running Operations (Async REST vs. HTTP/2 Streaming)
* Q4: Memory Bloat Under High Concurrency (JSON & Buffer Optimization)
* Q5: Cascading Failures & Context Cancellation Propagation to DB/Downstream

---

### Question 1: Ephemeral Port Exhaustion (`TIME_WAIT`) Under Load

> **Scenario:** Your Go REST gateway acts as a reverse proxy/aggregator calling internal microservices over HTTP/1.1. During a flash sale, outbound requests start failing with `dial tcp 10.0.1.42:8080: connect: cannot assign requested address`. `netstat` shows 28,000 sockets in `TIME_WAIT`. Why is this happening in Go, and how do you fix it?

#### Production-Level Answer:
1. **Root Cause:** In TCP, whichever side initiates the active close enters the `TIME_WAIT` state for `2 * MSL` (typically 60 seconds on Linux) to ensure delayed packets aren't misrouted to a new connection reusing the same 4-tuple `(src_ip, src_port, dst_ip, dst_port)`. Because the destination IP and port are fixed (`10.0.1.42:8080`), your Go service is limited by the OS ephemeral port range (`net.ipv4.ip_local_port_range`, roughly 28,232 ports).
2. **Why Go Triggered It:** Two classic Go mistakes cause connections not to be reused:
   * **Unconsumed Response Bodies:** In Go's `net/http`, a persistent HTTP/1.1 connection is only returned to the idle pool if the response body is **read completely to `io.EOF` AND closed**. If a handler checks `resp.StatusCode != 200` and calls `resp.Body.Close()` without draining it, Go tears down the underlying TCP socket!
   * **Default `MaxIdleConnsPerHost`:** `http.DefaultTransport` caps idle connections per host at **2**. Under 500 concurrent requests, 498 connections are closed immediately after each request finishes.
3. **The Fix:**
   * Always drain response bodies before closing (using `io.LimitReader` to prevent malicious huge bodies from hanging the drain).
   * Configure a custom `http.Transport` with `MaxIdleConnsPerHost` matched to your peak concurrency per downstream host, or migrate internal traffic to **HTTP/2** (multiplexes thousands of concurrent streams over a single TCP connection).

```go
// Proper response body handling to guarantee TCP connection reuse
func safeCloseBody(resp *http.Response) {
	if resp == nil || resp.Body == nil {
		return
	}
	// Drain up to 64KB to allow keep-alive reuse without risking unbounded reads
	_, _ = io.Copy(io.Discard, io.LimitReader(resp.Body, 64*1024))
	_ = resp.Body.Close()
}
```

---

### Question 2: Preventing Cascading Failures via Context Cancellation

> **Scenario:** A client makes a `GET /v1/dashboard` request to your Go API, which executes 3 parallel PostgreSQL queries and 2 downstream HTTP calls. The client disconnects after 200ms, or the API Gateway times out at 1s. However, your database CPU remains pegged at 100% during traffic spikes. Why, and how do you architect the Go service to prevent this?

#### Production-Level Answer:
1. **Root Cause:** When an HTTP client disconnects or a gateway closes the connection, Go's `net/http` server detects the closed socket (via a background read on the connection) and cancels `r.Context()`. However, if your repository layer uses `context.Background()` instead of threading `r.Context()` down to `db.QueryContext(ctx, ...)` and `http.NewRequestWithContext(ctx, ...)`, the database continues executing expensive queries whose results will be thrown away.
2. **Nuance with Writes:** While **read** operations should always inherit `r.Context()` so they abort immediately on client disconnect, **mutating operations** (like recording a financial ledger entry after charging a card) should sometimes decouple cancellation (`context.WithoutCancel(r.Context())` introduced in Go 1.21) so a client disconnect mid-flight doesn't leave distributed state partially committed!
3. **Parallel Fan-Out with `errgroup`:** Use `golang.org/x/sync/errgroup` with `errgroup.WithContext(r.Context())`. If *any* of the 5 parallel downstream calls fails, `errgroup` immediately cancels the derived context, aborting the other 4 in-flight DB/HTTP calls rather than waiting for all of them to finish.

```go
func (s *DashboardService) GetDashboard(ctx context.Context, userID string) (*Dashboard, error) {
	g, gCtx := errgroup.WithContext(ctx)

	var (
		profile *UserProfile
		orders  []Order
		recs    []Recommendation
	)

	// If GetProfile fails in 10ms, gCtx is canceled immediately, aborting GetOrders and GetRecs
	g.Go(func() error {
		var err error
		profile, err = s.userRepo.GetProfile(gCtx, userID)
		return err
	})

	g.Go(func() error {
		var err error
		orders, err = s.orderRepo.GetRecentOrders(gCtx, userID)
		return err
	})

	g.Go(func() error {
		var err error
		recs, err = s.recClient.FetchRecommendations(gCtx, userID)
		return err
	})

	if err := g.Wait(); err != nil {
		return nil, err
	}

	return &Dashboard{Profile: profile, Orders: orders, Recommendations: recs}, nil
}
```

---

### Question 3: Designing Long-Running Operations in Strict REST

> **Scenario:** Clients call `POST /v1/reports` to generate a massive compliance export that takes 45 seconds. Holding an HTTP connection open for 45 seconds violates our API Gateway's 30-second hard timeout and ties up server resources. How do you model this cleanly in REST and implement it in Go?

#### Production-Level Answer:
1. **HTTP Semantics (`202 Accepted` + Asynchronous Resource Pattern):**
   * `POST /v1/reports` validates the request, persists a job record with `status: PENDING`, publishes a message to a durable queue (or transactional outbox), and immediately returns **`202 Accepted`** with a `Location: /v1/report-jobs/{jobID}` header and a `Retry-After: 5` header.
   * Clients poll `GET /v1/report-jobs/{jobID}`. While running, it returns `200 OK` with `{"status": "PROCESSING", "progress": 60}`.
   * Once complete, `GET /v1/report-jobs/{jobID}` either returns `200 OK` with a signed S3 URL or responds with **`303 See Other`** pointing `Location` to the final resource `GET /v1/reports/{reportID}`.
2. **Cancellation Support:** Expose `DELETE /v1/report-jobs/{jobID}` to allow clients to cancel in-flight jobs cleanly.

```go
func (h *ReportHandler) CreateReport(w http.ResponseWriter, r *http.Request) {
	req, err := DecodeJSONStrict[CreateReportReq](w, r, 1<<20)
	if err != nil {
		WriteProblem(w, r, ProblemDetail{Type: "invalid-request", Title: "Invalid Payload", Status: http.StatusBadRequest, Detail: err.Error()})
		return
	}

	// Persist job & enqueue via Transactional Outbox
	job, err := h.service.EnqueueReportJob(r.Context(), req)
	if err != nil {
		WriteProblem(w, r, ProblemDetail{Type: "internal-error", Title: "Failed to queue job", Status: http.StatusInternalServerError})
		return
	}

	w.Header().Set("Location", fmt.Sprintf("/api/v1/report-jobs/%s", job.ID))
	w.Header().Set("Retry-After", "5")
	_ = WriteJSON(w, http.StatusAccepted, job)
}
```

---

### Question 4: Diagnosing & Fixing GC Thrashing on High-RPS JSON Endpoints

> **Scenario:** A high-throughput Go REST endpoint serving 25,000 RPS (`GET /v1/feature-flags`) has a p99 latency of 180ms despite p50 being 4ms. CPU profiling (`pprof`) shows 45% of CPU time is spent in `runtime.gcBgMarkWorker` and `encoding/json`. How do you diagnose and optimize this?

#### Production-Level Answer:
1. **Diagnosis:**
   * Run `go tool pprof -alloc_space http://localhost:6060/debug/pprof/heap` to inspect total bytes allocated over time (not just `inuse_space`).
   * In Go, reflection-based `encoding/json` inspects struct tags at runtime and allocates intermediate slices/strings for every field on every request. At 25k RPS, allocating 200KB per request creates **5 GB/sec of heap allocations**, forcing the Garbage Collector to run continuously and steal CPU cycles (causing p99 tail-latency spikes).
2. **Architectural & Code Fixes (in order of impact):**
   * **Caching Serialized Bytes (Zero-Serialization Hot Path):** If feature flags change infrequently (e.g., every few seconds) per environment/segment, cache the **already-marshaled `[]byte` JSON** in an `atomic.Value` or `singleflight`-protected cache. Serving raw `[]byte` reduces per-request allocations to **zero**.
   * **HTTP Caching Headers (`ETag` / `Cache-Control`):** Return an `ETag` (SHA-256 of the serialized payload). If the client sends `If-None-Match` matching the `ETag`, return `304 Not Modified` with an empty body—saving bandwidth and serialization.
   * **Code-Generated JSON (`easyjson` / `ffjson`) or `encoding/json/v2`:** Eliminate reflection overhead by generating static marshaling code.

```go
type CachedPayload struct {
	RawJSON []byte
	ETag    string
}

type FeatureFlagCache struct {
	current atomic.Pointer[CachedPayload]
}

func (c *FeatureFlagCache) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	payload := c.current.Load()
	if payload == nil {
		http.Error(w, "cache warming", http.StatusServiceUnavailable)
		return
	}

	// Conditional GET support: zero body transfer if unchanged
	if r.Header.Get("If-None-Match") == payload.ETag {
		w.WriteHeader(http.StatusNotModified)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	w.Header().Set("ETag", payload.ETag)
	w.Header().Set("Cache-Control", "public, max-age=5, stale-while-revalidate=30")
	w.WriteHeader(http.StatusOK)
	_, _ = w.Write(payload.RawJSON) // Zero reflection, zero heap allocation!
}
```

---

### Question 5: Dynamic Write Deadlines for Mixed Endpoints (REST + SSE)

> **Scenario:** Your Go server hosts both standard REST endpoints (which should time out if a response takes >10s) and a Server-Sent Events (SSE) endpoint (`GET /v1/events/stream`) that streams real-time updates for up to 30 minutes. Setting `WriteTimeout: 10s` on `http.Server` kills the SSE stream after 10 seconds. Removing `WriteTimeout` leaves standard REST endpoints vulnerable to slow-reading clients. How do you solve both in the same `http.Server`?

#### Production-Level Answer:
Leave `WriteTimeout` unset (or high) on `http.Server`, or use **`http.ResponseController`** (introduced in Go 1.20) to set per-handler write deadlines dynamically!

* In a global middleware for standard REST routes, call `rc.SetWriteDeadline(time.Now().Add(10 * time.Second))`.
* In the SSE streaming handler, override the deadline on every event write (`rc.SetWriteDeadline(time.Now().Add(30 * time.Second))`) and call `rc.Flush()` immediately so events aren't buffered in user-space.

```go
func DynamicWriteTimeoutMiddleware(timeout time.Duration) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			rc := http.NewResponseController(w)
			// Enforce strict write deadline for standard REST requests
			_ = rc.SetWriteDeadline(time.Now().Add(timeout))
			next.ServeHTTP(w, r)
		})
	}
}

func SSEStreamHandler(w http.ResponseWriter, r *http.Request) {
	rc := http.NewResponseController(w)

	w.Header().Set("Content-Type", "text/event-stream")
	w.Header().Set("Cache-Control", "no-cache")
	w.Header().Set("Connection", "keep-alive")

	ticker := time.NewTicker(5 * time.Second)
	defer ticker.Stop()

	for {
		select {
		case <-r.Context().Done():
			return // Client disconnected
		case t := <-ticker.C:
			// Extend the write deadline per chunk so slow clients are still dropped,
			// while healthy long-lived streams stay alive indefinitely.
			if err := rc.SetWriteDeadline(time.Now().Add(10 * time.Second)); err != nil {
				return
			}
			if _, err := fmt.Fprintf(w, "data: {\"time\": \"%s\"}\n\n", t.Format(time.RFC3339)); err != nil {
				return
			}
			if err := rc.Flush(); err != nil {
				return
			}
		}
	}
}
```