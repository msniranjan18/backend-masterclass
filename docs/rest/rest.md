# Production-Grade RESTful Systems in Go

---

## Table of Contents

1. **Chapter 1: Go's HTTP Internals & Network Mechanics**
   * The `net/http` Architecture (Goroutine-per-Connection Model)
   * Connection Lifecycles, Keep-Alives, and Socket Exhaustion
   * Server & Transport Timeouts (Preventing Slowloris and Resource Leaks)
2. **Chapter 2: Routing, Middleware Architecture & Context Propagation**
   * Go 1.22+ `http.ServeMux` vs. Third-Party Routers
   * Idiomatic Middleware Chaining & `http.ResponseController`
   * Context-Driven Cancellation, Deadlines, and Request-Scoped Values
3. **Chapter 3: Serialization, Validation, and Memory Efficiency**
   * Streaming JSON (`json.NewDecoder` vs. `json.Unmarshal`)
   * Strict Parsing & Defense Against Payload Bombs
   * Zero-Allocation Patterns & `sync.Pool` in Hot Paths
4. **Chapter 4: Resilience, Concurrency Control & Distributed Patterns**
   * Graceful Shutdown & Connection Draining
   * Idempotency Keys & Distributed Locking
   * Rate Limiting (Token Bucket) & Circuit Breaking
   * Outbox Pattern & Transactional Consistency
5. **Chapter 5: Observability, Error Handling & API Evolution**
   * RFC 9457 (Problem Details for HTTP APIs)
   * Structured Logging (`log/slog`) & OpenTelemetry Tracing
   * API Versioning & Backward Compatibility Strategies

---

## Chapter 1: Go's HTTP Internals & Network Mechanics

At the Production level, writing a REST API in Go isn't about knowing how to register a route; it's about understanding what happens between the Linux kernel's TCP stack, the Go runtime scheduler (GMP model), and your handler function.

### 1.1 The `net/http` Architecture

Unlike Node.js (single-threaded event loop) or traditional Java/Tomcat (thread-per-request with OS threads), Go uses a **goroutine-per-connection** model backed by non-blocking I/O (`epoll` on Linux, `kqueue` on macOS/BSD) integrated directly into the Go runtime's network poller (`netpoller`).

1. **Accept Loop:** `http.Server.Serve(l net.Listener)` runs an infinite loop calling `l.Accept()`.
2. **Goroutine Spawn:** For every accepted TCP connection, Go spawns a new goroutine (`go c.serve(connCtx)`). Because goroutines start with a tiny stack (2KB), a single Go process can easily handle 100,000+ concurrent idle connections (e.g., HTTP Keep-Alive or SSE).
3. **Network Poller Integration:** When a goroutine reads from `net.Conn` and no data is ready on the socket, the Go runtime parks the goroutine (putting it in `_Gwaiting` state) and registers the file descriptor with `epoll`. The underlying OS thread (`M`) is immediately freed to execute other runnable goroutines (`G`).

### 1.2 Server & Transport Timeouts (The #1 Production Footgun)

**Never use `http.ListenAndServe(":8080", handler)` in production.** The default `http.Server` has *zero* timeouts configured. A malicious client or a degraded mobile network can hold a connection open indefinitely, exhausting memory and file descriptors (Slowloris attack).

#### Anatomy of HTTP Server Timeouts:
* **`ReadHeaderTimeout`**: Maximum time allowed to read request headers. Protects against Slowloris attacks before routing even begins.
* **`ReadTimeout`**: Covers the entire duration from connection accept to the end of reading the request body.
* **`WriteTimeout`**: Covers the time from the end of request header read to the end of writing the response. *(Note: For streaming endpoints like SSE, this must be managed dynamically via `http.ResponseController`).*
* **`IdleTimeout`**: Maximum time to wait for the next request when keep-alives are enabled.

```go
package main

import (
	"log/slog"
	"net/http"
	"os"
	"time"
)

func NewProductionServer(addr string, handler http.Handler) *http.Server {
	return &http.Server{
		Addr:              addr,
		Handler:           handler,
		ReadHeaderTimeout: 5 * time.Second,
		ReadTimeout:       15 * time.Second,
		WriteTimeout:      15 * time.Second,
		IdleTimeout:       60 * time.Second,
		MaxHeaderBytes:    1 << 20, // 1 MB max header size
		ErrorLog:          slog.NewLogLogger(slog.NewJSONHandler(os.Stderr, nil), slog.LevelError),
	}
}
```

### 1.3 Outbound `http.Client` & `http.Transport` Tuning

When your REST service calls downstream REST APIs, using `http.DefaultClient` is another critical anti-pattern:
1. It has no overall `Timeout`.
2. `http.DefaultTransport` sets `MaxIdleConnsPerHost = 2`. Under high concurrency to a single downstream microservice, Go will constantly open and close TCP connections, leading to TLS handshake latency spikes and `TIME_WAIT` socket exhaustion.

```go
func NewProductionHTTPClient() *http.Client {
	transport := &http.Transport{
		Proxy: http.ProxyFromEnvironment,
		// Connection pooling tuning
		MaxIdleConns:        1000,
		MaxIdleConnsPerHost: 100, // Crucial for microservice-to-microservice communication
		MaxConnsPerHost:     200, // Hard cap to prevent overwhelming downstream
		IdleConnTimeout:     90 * time.Second,

		// Network & TLS handshake timeouts
		TLSHandshakeTimeout:   5 * time.Second,
		ResponseHeaderTimeout: 10 * time.Second,
		ExpectContinueTimeout: 1 * time.Second,
		ForceAttemptHTTP2:     true,
	}

	return &http.Client{
		Transport: transport,
		Timeout:   30 * time.Second, // Hard ceiling for the entire request lifecycle
	}
}
```

---

## Chapter 2: Routing, Middleware Architecture & Context Propagation

### 2.1 Go 1.22+ Enhanced `http.ServeMux`

Prior to Go 1.22, the standard library router lacked method-based routing and path wildcards, forcing teams to adopt `chi`, `gorilla/mux`, or `gin`. Today, `net/http.ServeMux` supports method matching, path parameters (`{id}`), trailing catch-alls (`{path...}`), and strict exact-match anchoring (`{$}`).

```go
func SetupRoutes() http.Handler {
	mux := http.NewServeMux()

	// Method + Wildcard routing
	mux.HandleFunc("GET /api/v1/orders/{orderID}", handleGetOrder)
	mux.HandleFunc("POST /api/v1/orders", handleCreateOrder)
	
	// Exact root match using {$} so it doesn't act as a catch-all prefix
	mux.HandleFunc("GET /{$}", handleHealthCheck)

	return mux
}

func handleGetOrder(w http.ResponseWriter, r *http.Request) {
	// Native path parameter extraction
	orderID := r.PathValue("orderID")
	if orderID == "" {
		http.Error(w, "missing orderID", http.StatusBadRequest)
		return
	}
	// ...
}
```

### 2.2 Middleware Chaining & `http.ResponseController`

A common challenge in middleware (like logging or metrics) is capturing the HTTP status code written by downstream handlers. Historically, developers wrapped `http.ResponseWriter` in a custom struct—which often accidentally stripped optional interfaces like `http.Flusher` or `http.Hijacker`.

Since Go 1.20, **`http.ResponseController`** solves interface-stripping by unwrapping custom `ResponseWriter`s via an `Unwrap() http.ResponseWriter` method.

```go
type statusRecorder struct {
	http.ResponseWriter
	statusCode int
	written    bool
}

func (r *statusRecorder) WriteHeader(code int) {
	if !r.written {
		r.statusCode = code
		r.written = true
		r.ResponseWriter.WriteHeader(code)
	}
}

func (r *statusRecorder) Write(b []byte) (int, error) {
	if !r.written {
		r.WriteHeader(http.StatusOK)
	}
	return r.ResponseWriter.Write(b)
}

// Unwrap allows http.ResponseController to access the underlying ResponseWriter
// for Flushing, Hijacking, or setting dynamic WriteDeadlines!
func (r *statusRecorder) Unwrap() http.ResponseWriter {
	return r.ResponseWriter
}

func TelemetryMiddleware(logger *slog.Logger) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			start := time.Now()
			rec := &statusRecorder{ResponseWriter: w, statusCode: http.StatusOK}

			next.ServeHTTP(rec, r)

			logger.Info("http_request",
				slog.String("method", r.Method),
				slog.String("path", r.URL.Path),
				slog.Int("status", rec.statusCode),
				slog.Duration("duration", time.Since(start)),
			)
		})
	}
}
```

### 2.3 Context Propagation & Typed Keys

Never use raw `string` types as keys in `context.WithValue`—this leads to package collisions. Always use unexported struct types and provide type-safe getters/setters. Furthermore, **never store mutable state or dependencies (like DB connections) inside `context.Context`**. Context is strictly for request-scoped metadata (trace IDs, auth claims) and cancellation signals.

```go
type ctxKey int

const (
	tenantIDKey ctxKey = iota
	userClaimsKey
)

func WithTenantID(ctx context.Context, tenantID string) context.Context {
	return context.WithValue(ctx, tenantIDKey, tenantID)
}

func TenantIDFromContext(ctx context.Context) (string, bool) {
	id, ok := ctx.Value(tenantIDKey).(string)
	return id, ok
}
```

---

## Chapter 3: Serialization, Validation, and Memory Efficiency

### 3.1 Hardened JSON Decoding

Using `json.Unmarshal` after `io.ReadAll(r.Body)` exposes your service to **Out-Of-Memory (OOM) DoS attacks** if a client sends a 5GB JSON payload.

Production-grade JSON decoding enforces four rules:
1. Bound the body size using `http.MaxBytesReader`.
2. Stream directly from `r.Body` using `json.NewDecoder`.
3. Reject unknown fields (`dec.DisallowUnknownFields()`) if strict contract validation is required.
4. Ensure there is no trailing garbage after the first JSON object.

```go
func DecodeJSONStrict[T any](w http.ResponseWriter, r *http.Request, maxBytes int64) (T, error) {
	var dst T

	// 1. Enforce max body size at the reader level (automatically closes/errors if exceeded)
	r.Body = http.MaxBytesReader(w, r.Body, maxBytes)

	dec := json.NewDecoder(r.Body)
	dec.DisallowUnknownFields()

	// 2. Decode primary JSON object
	if err := dec.Decode(&dst); err != nil {
		return dst, fmt.Errorf("json decode error: %w", err)
	}

	// 3. Ensure only a single JSON value was sent (no trailing data)
	if err := dec.Decode(&struct{}{}); err != io.EOF {
		return dst, errors.New("request body must only contain a single JSON object")
	}

	return dst, nil
}
```

### 3.2 Zero-Allocation & `sync.Pool` Encoding

When serializing responses, writing directly to `http.ResponseWriter` via `json.NewEncoder(w).Encode(v)` has a subtle flaw: **if encoding fails midway through (e.g., an unsupported type or broken marshaler), the `200 OK` header has already been flushed to the wire**, making it impossible to return a `500 Internal Server Error`.

To solve this without creating massive Garbage Collector (GC) pressure from allocating temporary `bytes.Buffer`s on every request, use a `sync.Pool` of `bytes.Buffer`s.

```go
var bufPool = sync.Pool{
	New: func() any {
		return bytes.NewBuffer(make([]byte, 0, 4096)) // Pre-allocate 4KB
	},
}

func WriteJSON(w http.ResponseWriter, status int, payload any) error {
	buf := bufPool.Get().(*bytes.Buffer)
	buf.Reset()
	defer func() {
		// Prevent memory leaks: don't return abnormally huge buffers to the pool
		if buf.Cap() <= 64*1024 {
			bufPool.Put(buf)
		}
	}()

	if err := json.NewEncoder(buf).Encode(payload); err != nil {
		return err
	}

	w.Header().Set("Content-Type", "application/json; charset=utf-8")
	w.WriteHeader(status)
	_, err := w.Write(buf.Bytes())
	return err
}
```

---

## Chapter 4: Resilience, Concurrency Control & Distributed Patterns

### 4.1 Graceful Shutdown & In-Flight Draining

In Kubernetes or containerized environments, pods receive `SIGTERM` during deployments or autoscaling scale-downs. If your Go process exits immediately, active requests are severed (`502 Bad Gateway` at the load balancer).

**Crucial Nuance:** Load balancers (like AWS ALB or Envoy) remove endpoints asynchronously after `SIGTERM` is sent to the container. Adding a brief readiness-probe failure delay before calling `srv.Shutdown()` prevents race conditions where new traffic arrives right as the listener closes.

```go
func RunServerWithGracefulShutdown(ctx context.Context, srv *http.Server, logger *slog.Logger) error {
	errCh := make(chan error, 1)

	go func() {
		logger.Info("starting HTTP server", slog.String("addr", srv.Addr))
		if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
			errCh <- err
		}
	}()

	// Wait for OS signal (SIGINT/SIGTERM) or server startup failure
	select {
	case err := <-errCh:
		return err
	case <-ctx.Done():
		logger.Info("shutdown signal received, draining connections...")
	}

	// Optional: Flip readiness check to unhealthy and wait 2-5s for LB endpoints to update
	// time.Sleep(3 * time.Second)

	shutdownCtx, cancel := context.WithTimeout(context.Background(), 25*time.Second)
	defer cancel()

	// Shutdown stops accepting new connections and waits for active handlers to return
	if err := srv.Shutdown(shutdownCtx); err != nil {
		logger.Error("graceful shutdown timed out, forcing close", slog.Any("err", err))
		return srv.Close()
	}

	logger.Info("server stopped cleanly")
	return nil
}
```

### 4.2 Idempotency Keys in Mutating REST Endpoints

Network retries are inevitable. If a client sends `POST /v1/transfers` and times out waiting for the response, retrying without idempotency causes double charges.

A Production-level implementation handles **concurrent duplicate requests** (e.g., two retries arriving within 5ms of each other) using atomic state transitions (`IN_PROGRESS` vs. `COMPLETED`) backed by Redis or a relational database unique constraint.

```go
type IdempotencyRecord struct {
	Key          string
	RequestHash  string
	Status       string // "IN_PROGRESS", "COMPLETED"
	ResponseCode int
	ResponseBody []byte
}

func IdempotencyMiddleware(store IdempotencyStore) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			// Only apply to mutating methods
			if r.Method != http.MethodPost && r.Method != http.MethodPatch {
				next.ServeHTTP(w, r)
				return
			}

			key := r.Header.Get("Idempotency-Key")
			if key == "" {
				http.Error(w, "Idempotency-Key header required", http.StatusBadRequest)
				return
			}

			// Hash request body to prevent key reuse with different payloads
			bodyBytes, _ := io.ReadAll(r.Body)
			r.Body = io.NopCloser(bytes.NewBuffer(bodyBytes))
			reqHash := fmt.Sprintf("%x", sha256.Sum256(bodyBytes))

			// Atomically attempt to acquire lock / insert IN_PROGRESS record
			record, acquired, err := store.AcquireOrGet(r.Context(), key, reqHash)
			if err != nil {
				http.Error(w, "internal error", http.StatusInternalServerError)
				return
			}

			if !acquired {
				// 1. Payload mismatch check (RFC draft standard: 422 Unprocessable Entity)
				if record.RequestHash != reqHash {
					http.Error(w, "Idempotency-Key reused with different payload", http.StatusUnprocessableEntity)
					return
				}
				// 2. Concurrent request still running
				if record.Status == "IN_PROGRESS" {
					w.Header().Set("Retry-After", "1")
					http.Error(w, "Request currently in progress", http.StatusConflict)
					return
				}
				// 3. Replay cached response
				w.Header().Set("Content-Type", "application/json")
				w.Header().Set("X-Idempotent-Replay", "true")
				w.WriteHeader(record.ResponseCode)
				w.Write(record.ResponseBody)
				return
			}

			// Capture response for storage
			rec := &responseCapture{ResponseWriter: w, body: &bytes.Buffer{}, code: http.StatusOK}
			next.ServeHTTP(rec, r)

			// Persist completed response (or delete lock on 5xx so client can retry)
			if rec.code >= 500 {
				_ = store.Delete(r.Context(), key)
			} else {
				_ = store.Complete(r.Context(), key, rec.code, rec.body.Bytes())
			}
		})
	}
}
```

### 4.3 Distributed Rate Limiting & Load Shedding

* **Load Shedding (Concurrency Limiting):** Protects the local instance from CPU/memory starvation when downstream databases slow down. Implemented using a buffered channel semaphore or `golang.org/x/sync/semaphore`.
* **Rate Limiting (Per-Tenant/Client):** Protects fairness across tenants. Use `golang.org/x/time/rate` for in-memory Token Bucket limiting, or Redis Lua scripts (GCRA - Generic Cell Rate Algorithm) for distributed cluster-wide limits.

```go
// LoadShedder rejects excess concurrent requests immediately with 503 Service Unavailable
func LoadShedder(maxConcurrent int) func(http.Handler) http.Handler {
	sem := make(chan struct{}, maxConcurrent)

	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			select {
			case sem <- struct{}{}:
				defer func() { <-sem }()
				next.ServeHTTP(w, r)
			default:
				// Fail fast rather than queuing and causing tail-latency spikes
				w.Header().Set("Retry-After", "2")
				http.Error(w, "server overloaded", http.StatusServiceUnavailable)
			}
		})
	}
}
```

---

## Chapter 5: Observability, Error Handling & API Evolution

### 5.1 Standardized Errors: RFC 9457 (Problem Details)

Ad-hoc error responses (`{"error": "bad input"}` in one endpoint, `{"msg": "invalid", "code": 12}` in another) create brittle clients. Adopt **RFC 9457** (`application/problem+json`).

```go
type ProblemDetail struct {
	Type     string         `json:"type"`               // URI identifying the error type
	Title    string         `json:"title"`              // Short human-readable summary
	Status   int            `json:"status"`             // HTTP status code
	Detail   string         `json:"detail,omitempty"`   // Specific explanation for this occurrence
	Instance string         `json:"instance,omitempty"` // Request URI or trace ID
	Errors   map[string]any `json:"errors,omitempty"`   // Extension: field-level validation errors
}

func WriteProblem(w http.ResponseWriter, r *http.Request, p ProblemDetail) {
	if p.Instance == "" {
		p.Instance = r.URL.Path
	}
	w.Header().Set("Content-Type", "application/problem+json")
	w.WriteHeader(p.Status)
	_ = json.NewEncoder(w).Encode(p)
}
```

### 5.2 Pagination at Scale: Keyset (Cursor) vs. Offset

* **Offset Pagination (`?limit=50&offset=100000`):** In SQL (`LIMIT 50 OFFSET 100000`), the database must scan and discard 100,050 rows. Latency degrades linearly ($O(N)$), and concurrent inserts cause duplicate/skipped items across pages.
* **Keyset / Cursor Pagination (`?limit=50&cursor=eyJpZCI6MTIzfQ==`):** Uses an indexed `WHERE (created_at, id) < ($1, $2) ORDER BY created_at DESC, id DESC LIMIT 50`. Complexity is $O(\log N)$ regardless of page depth, and results remain stable under concurrent writes.
