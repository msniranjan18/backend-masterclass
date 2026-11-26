# Complete Guide to REST Servers & Clients in Go: From Basic to Production

## Table of Contents

1. **Part I: Building REST Servers (Basic $\rightarrow$ Production)**
   * **Chapter 1: Level 1 — The Basic REST Server (What It Is & Why It Breaks in Production)**
   * **Chapter 2: Level 2 — Clean Architecture, Go 1.22+ Routing, Validation & RFC 9457 Errors**
   * **Chapter 3: Level 3 — Production Server Hardening (Timeouts, Middleware, Graceful Shutdown & Idempotency)**
2. **Part II: Building REST Clients (Basic $\rightarrow$ Production)**
   * **Chapter 4: Level 1 — The Basic REST Client (And the Hidden Traps That Crash Services)**
   * **Chapter 5: Level 2 — Custom `http.Transport`, Connection Pooling & Safe Body Draining**
   * **Chapter 6: Level 3 — Production Client Resilience (Retries with Jitter, Circuit Breakers & Observability)**
3. **Part III: End-to-End Integration**
   * **Chapter 7: Putting It Together — Production Server + Client Blueprint**

---

# Part I: Building REST Servers (Basic $\rightarrow$ Production)

## Chapter 1: Level 1 — The Basic REST Server

When learning Go, almost everyone starts with a minimal HTTP server using `http.HandleFunc` and `http.ListenAndServe`.

### 1.1 Basic Server Code

```go
package main

import (
	"encoding/json"
	"net/http"
)

type Task struct {
	ID    string `json:"id"`
	Title string `json:"title"`
	Done  bool   `json:"done"`
}

func main() {
	http.HandleFunc("/tasks", func(w http.ResponseWriter, r *http.Request) {
		if r.Method != http.MethodGet {
			http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
			return
		}
		tasks := []Task{{ID: "1", Title: "Ship API", Done: false}}
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(tasks)
	})

	// WARNING: Never use this in production!
	http.ListenAndServe(":8080", nil)
}
```

### 1.2 Why This Fails in Production

1. **Zero Timeouts:** `http.ListenAndServe` uses a default `http.Server` with *no* `ReadHeaderTimeout`, `ReadTimeout`, or `WriteTimeout`. A slow or malicious client can hold sockets open forever (Slowloris attack), exhausting file descriptors and RAM.
2. **Global Mutable Router (`http.DefaultServeMux`):** Any imported third-party package can secretly register routes onto `http.DefaultServeMux` (such as `/debug/pprof`). Always instantiate your own `http.NewServeMux()`.
3. **No Body Size Limit:** Reading `r.Body` without `http.MaxBytesReader` lets a client send a 10 GB JSON payload and crash your pod with an Out-Of-Memory (OOM) kill.
4. **Partial Write Failures:** `json.NewEncoder(w).Encode(tasks)` writes HTTP `200 OK` headers immediately. If encoding fails halfway through, the client receives broken JSON with a `200 OK` status.
5. **Abrupt Shutdown:** When Kubernetes sends `SIGTERM` during a deploy, `ListenAndServe` terminates immediately, dropping all active in-flight requests (`502 Bad Gateway`).

---

## Chapter 2: Level 2 — Clean Architecture, Modern Routing & Standardized Errors

At the intermediate/structured level, we separate **HTTP transport concerns** (decoding JSON, status codes, headers) from **business logic**, use Go 1.22+ method-and-path routing, enforce strict JSON decoding, and return machine-readable errors following **RFC 9457 (`application/problem+json`)**.

### 2.1 Strict JSON Helpers (`sync.Pool` + `MaxBytesReader`)

* **Inbound:** We wrap `r.Body` with `http.MaxBytesReader`, disallow unknown fields if strict contract enforcement is desired, and verify there is no trailing garbage after the JSON object.
* **Outbound:** We encode into a pooled `bytes.Buffer` first. If marshaling fails, we can cleanly return a `500 Internal Server Error` before writing any bytes to the wire.

```go
package httpx

import (
	"bytes"
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"net/http"
	"sync"
)

var bufPool = sync.Pool{
	New: func() any {
		return bytes.NewBuffer(make([]byte, 0, 4096))
	},
}

// DecodeJSON safely decodes a single JSON object with a strict byte size cap.
func DecodeJSON[T any](w http.ResponseWriter, r *http.Request, maxBytes int64) (T, error) {
	var v T
	r.Body = http.MaxBytesReader(w, r.Body, maxBytes)

	dec := json.NewDecoder(r.Body)
	dec.DisallowUnknownFields()

	if err := dec.Decode(&v); err != nil {
		return v, fmt.Errorf("invalid JSON payload: %w", err)
	}
	// Ensure there isn't a second JSON object concatenated in the body
	if err := dec.Decode(&struct{}{}); !errors.Is(err, io.EOF) {
		return v, errors.New("body must contain only a single JSON object")
	}
	return v, nil
}

// WriteJSON serializes payload into a pooled buffer before writing headers,
// guaranteeing we never send a 200 OK with half-written broken JSON.
func WriteJSON(w http.ResponseWriter, status int, payload any) error {
	buf := bufPool.Get().(*bytes.Buffer)
	buf.Reset()
	defer func() {
		if buf.Cap() <= 64*1024 { // Avoid retaining huge buffers in memory
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

### 2.2 RFC 9457 Problem Details (Standardized Error Responses)

Instead of ad-hoc `{"error": "something failed"}` strings, return structured RFC 9457 Problem Details so API consumers can handle errors programmatically.

```go
package httpx

import (
	"encoding/json"
	"net/http"
)

type ProblemDetail struct {
	Type     string            `json:"type"`
	Title    string            `json:"title"`
	Status   int               `json:"status"`
	Detail   string            `json:"detail,omitempty"`
	Instance string            `json:"instance,omitempty"`
	Fields   map[string]string `json:"fields,omitempty"` // Optional validation errors
}

func WriteProblem(w http.ResponseWriter, r *http.Request, p ProblemDetail) {
	if p.Instance == "" {
		p.Instance = r.URL.Path
	}
	w.Header().Set("Content-Type", "application/problem+json; charset=utf-8")
	w.WriteHeader(p.Status)
	_ = json.NewEncoder(w).Encode(p)
}
```

### 2.3 Go 1.22+ `http.ServeMux` Handler Implementation

Go 1.22+ natively supports HTTP method matching (`GET`, `POST`, `PATCH`, `DELETE`) and path wildcards (`{id}`) in `net/http` without needing external routers.

```go
package orders

import (
	"context"
	"errors"
	"net/http"

	"yourapp/httpx"
)

var (
	ErrNotFound = errors.New("order not found")
	ErrConflict = errors.New("order already exists")
)

type Order struct {
	ID          string `json:"id"`
	CustomerID  string `json:"customer_id"`
	AmountCents int64  `json:"amount_cents"`
	Status      string `json:"status"`
}

type CreateOrderReq struct {
	CustomerID  string `json:"customer_id"`
	AmountCents int64  `json:"amount_cents"`
}

type Service interface {
	Create(ctx context.Context, req CreateOrderReq) (Order, error)
	GetByID(ctx context.Context, id string) (Order, error)
}

type Handler struct {
	svc Service
}

func NewHandler(svc Service) *Handler {
	return &Handler{svc: svc}
}

func (h *Handler) RegisterRoutes(mux *http.ServeMux) {
	mux.HandleFunc("POST /api/v1/orders", h.handleCreate)
	mux.HandleFunc("GET /api/v1/orders/{id}", h.handleGet)
}

func (h *Handler) handleCreate(w http.ResponseWriter, r *http.Request) {
	req, err := httpx.DecodeJSON[CreateOrderReq](w, r, 1<<20) // 1 MB max body
	if err != nil {
		httpx.WriteProblem(w, r, httpx.ProblemDetail{
			Type:   "https://api.example.com/errors/invalid-json",
			Title:  "Invalid Request Body",
			Status: http.StatusBadRequest,
			Detail: err.Error(),
		})
		return
	}

	// Domain validation
	if req.CustomerID == "" || req.AmountCents <= 0 {
		httpx.WriteProblem(w, r, httpx.ProblemDetail{
			Type:   "https://api.example.com/errors/validation-failed",
			Title:  "Validation Failed",
			Status: http.StatusUnprocessableEntity,
			Fields: map[string]string{
				"customer_id":  "must not be empty",
				"amount_cents": "must be greater than 0",
			},
		})
		return
	}

	order, err := h.svc.Create(r.Context(), req)
	if err != nil {
		httpx.WriteProblem(w, r, httpx.ProblemDetail{
			Type:   "https://api.example.com/errors/internal",
			Title:  "Internal Server Error",
			Status: http.StatusInternalServerError,
		})
		return
	}

	w.Header().Set("Location", "/api/v1/orders/"+order.ID)
	_ = httpx.WriteJSON(w, http.StatusCreated, order)
}

func (h *Handler) handleGet(w http.ResponseWriter, r *http.Request) {
	id := r.PathValue("id")
	order, err := h.svc.GetByID(r.Context(), id)
	if err != nil {
		if errors.Is(err, ErrNotFound) {
			httpx.WriteProblem(w, r, httpx.ProblemDetail{
				Type:   "https://api.example.com/errors/not-found",
				Title:  "Order Not Found",
				Status: http.StatusNotFound,
				Detail: "No order exists with ID " + id,
			})
			return
		}
		httpx.WriteProblem(w, r, httpx.ProblemDetail{
			Type:   "https://api.example.com/errors/internal",
			Title:  "Internal Server Error",
			Status: http.StatusInternalServerError,
		})
		return
	}

	_ = httpx.WriteJSON(w, http.StatusOK, order)
}
```

---

## Chapter 3: Level 3 — Production Server Hardening

A production REST server must survive network glitches, traffic spikes, panic bugs, duplicate client retries, and rolling Kubernetes deployments without dropping a single request.

### 3.1 Production Middleware Chain (Recovery, Load Shedding, Telemetry, Idempotency)

#### Key Design Details:
* **`statusRecorder` with `Unwrap()`:** Wrapping `http.ResponseWriter` to capture the status code can break `http.Flusher` or `http.ResponseController`. Implementing `Unwrap() http.ResponseWriter` allows Go 1.20+ `http.ResponseController` to access the underlying network connection cleanly.
* **Load Shedding:** Using a bounded semaphore channel (`make(chan struct{}, maxConcurrent)`) protects your service from CPU/memory collapse during sudden traffic spikes by failing fast with `503 Service Unavailable` + `Retry-After`.
* **Panic Recovery:** Catches unexpected runtime panics, logs the stack trace, and returns a clean `500` error instead of crashing the whole server.

```go
package middleware

import (
	"log/slog"
	"net/http"
	"runtime/debug"
	"time"
)

type Middleware func(http.Handler) http.Handler

func Chain(h http.Handler, mws ...Middleware) http.Handler {
	for i := len(mws) - 1; i >= 0; i-- {
		h = mws[i](h)
	}
	return h
}

type statusRecorder struct {
	http.ResponseWriter
	status  int
	written bool
}

func (r *statusRecorder) WriteHeader(code int) {
	if !r.written {
		r.status = code
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

// Unwrap preserves http.ResponseController capabilities (Flusher, SetWriteDeadline).
func (r *statusRecorder) Unwrap() http.ResponseWriter {
	return r.ResponseWriter
}

func Recovery(logger *slog.Logger) Middleware {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			defer func() {
				if rec := recover(); rec != nil {
					logger.Error("http_handler_panic",
						slog.Any("panic", rec),
						slog.String("path", r.URL.Path),
						slog.String("stack", string(debug.Stack())),
					)
					http.Error(w, "internal server error", http.StatusInternalServerError)
				}
			}()
			next.ServeHTTP(w, r)
		})
	}
}

func AccessLog(logger *slog.Logger) Middleware {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			start := time.Now()
			rec := &statusRecorder{ResponseWriter: w, status: http.StatusOK}
			next.ServeHTTP(rec, r)

			logger.Info("http_request",
				slog.String("method", r.Method),
				slog.String("path", r.URL.Path),
				slog.Int("status", rec.status),
				slog.Duration("latency", time.Since(start)),
			)
		})
	}
}

func LoadShedder(maxInFlight int) Middleware {
	sem := make(chan struct{}, maxInFlight)
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			select {
			case sem <- struct{}{}:
				defer func() { <-sem }()
				next.ServeHTTP(w, r)
			default:
				w.Header().Set("Retry-After", "2")
				http.Error(w, "server busy, try again shortly", http.StatusServiceUnavailable)
			}
		})
	}
}
```

### 3.2 Hardened `http.Server` & Graceful Shutdown with Health Probes

When Kubernetes terminates a pod, it sends `SIGTERM` while asynchronously removing the pod's IP from load balancers (ALB/Envoy/kube-proxy).

**Production Shutdown Sequence:**
1. Catch `SIGINT` / `SIGTERM`.
2. Flip `/readyz` health check to `503 Service Unavailable`.
3. Wait briefly (`2–5 seconds`) so load balancers stop routing *new* traffic to this pod.
4. Call `srv.Shutdown(shutdownCtx)` to finish all active requests cleanly before exiting.

```go
package server

import (
	"context"
	"errors"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"sync/atomic"
	"syscall"
	"time"
)

type Server struct {
	httpServer *http.Server
	isReady    atomic.Bool
	logger     *slog.Logger
}

func New(addr string, handler http.Handler, logger *slog.Logger) *Server {
	s := &Server{logger: logger}
	s.isReady.Store(true)

	rootMux := http.NewServeMux()
	// Liveness probe: is the process alive?
	rootMux.HandleFunc("GET /healthz", func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
		_, _ = w.Write([]byte("ok"))
	})
	// Readiness probe: should the load balancer send traffic here?
	rootMux.HandleFunc("GET /readyz", func(w http.ResponseWriter, r *http.Request) {
		if !s.isReady.Load() {
			http.Error(w, "shutting down", http.StatusServiceUnavailable)
			return
		}
		w.WriteHeader(http.StatusOK)
		_, _ = w.Write([]byte("ready"))
	})
	// Mount application routes
	rootMux.Handle("/", handler)

	s.httpServer = &http.Server{
		Addr:              addr,
		Handler:           rootMux,
		ReadHeaderTimeout: 5 * time.Second,  // Defends against Slowloris header attacks
		ReadTimeout:       15 * time.Second, // Max time to read entire request body
		WriteTimeout:      15 * time.Second, // Max time to write response
		IdleTimeout:       60 * time.Second, // Max idle keep-alive duration
		MaxHeaderBytes:    1 << 20,          // 1 MB max headers
		ErrorLog:          slog.NewLogLogger(logger.Handler(), slog.LevelError),
	}
	return s
}

func (s *Server) Run() error {
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()

	errCh := make(chan error, 1)
	go func() {
		s.logger.Info("HTTP server listening", slog.String("addr", s.httpServer.Addr))
		if err := s.httpServer.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
			errCh <- err
		}
	}()

	select {
	case err := <-errCh:
		return err
	case <-ctx.Done():
		s.logger.Info("shutdown signal received, starting graceful drain")
	}

	// 1. Mark pod unready so load balancers stop sending new requests
	s.isReady.Store(false)
	// Allow propagation time for Kubernetes Endpoints / Load Balancer health checks
	time.Sleep(3 * time.Second)

	// 2. Drain active connections with a 20-second hard ceiling
	shutdownCtx, cancel := context.WithTimeout(context.Background(), 20*time.Second)
	defer cancel()

	if err := s.httpServer.Shutdown(shutdownCtx); err != nil {
		s.logger.Error("graceful shutdown timed out, forcing close", slog.Any("err", err))
		return s.httpServer.Close()
	}

	s.logger.Info("HTTP server stopped cleanly")
	return nil
}
```

---

# Part II: Building REST Clients (Basic $\rightarrow$ Production)

## Chapter 4: Level 1 — The Basic REST Client (And Hidden Traps)

Most Go developers start by calling `http.Get(url)` or creating a bare `&http.Client{}`.

### 4.1 Basic Client Code

```go
package main

import (
	"encoding/json"
	"fmt"
	"net/http"
)

func fetchOrderBasic(id string) error {
	// WARNING: Uses http.DefaultClient (ZERO timeout!)
	resp, err := http.Get("https://api.example.com/v1/orders/" + id)
	if err != nil {
		return err
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		return fmt.Errorf("unexpected status: %d", resp.StatusCode)
	}

	var data map[string]any
	return json.NewDecoder(resp.Body).Decode(&data)
}
```

### 4.2 Why This Fails in Production

1. **Infinite Hang (`http.DefaultClient` has no timeout):** If the remote server accepts the TCP connection and hangs forever, your goroutine blocks forever. Under load, thousands of stuck goroutines exhaust your service's RAM.
2. **Broken TCP Connection Reuse (`TIME_WAIT` Exhaustion):**
   * In Go's `net/http`, a persistent HTTP/1.1 connection is **only** returned to the keep-alive pool if you **both** read the response body completely to `io.EOF` **and** close it.
   * When `resp.StatusCode != 200` above, the function returns early without reading `resp.Body`. Go assumes unread bytes are still sitting on the wire and **destroys the TCP socket**! Under heavy traffic, this causes ephemeral port exhaustion (`connect: cannot assign requested address`).
3. **Tiny Per-Host Connection Pool:** `http.DefaultTransport` sets `MaxIdleConnsPerHost = 2`. If your service makes 500 concurrent calls to `payment-service`, 498 TCP connections are opened and closed on every single request, adding massive TLS handshake latency.
4. **Creating a New `http.Client` Per Request:** Never instantiate `&http.Client{}` inside a helper function! Each `http.Client` owns its own connection pool; creating one per request leaks idle sockets and disables connection reuse completely. **Always share a single `http.Client` instance across your application.**

---

## Chapter 5: Level 2 — Custom `http.Transport`, Connection Pooling & Safe Body Draining

At Level 2, we configure a dedicated `http.Transport`, propagate `context.Context` on every request, and guarantee safe body draining so TCP connections are always reused.

### 5.1 Safe Response Body Draining Helper

To reuse TCP connections without risking an Out-Of-Memory attack from a rogue server sending a 10 GB error body, drain up to a bounded limit (e.g., `64 KB`) before calling `Close()`.

```go
package restclient

import (
	"io"
	"net/http"
)

// DrainAndClose reads up to 64KB of any remaining response body so Go's HTTP/1.1
// transport can safely reuse the underlying TCP connection for future requests.
func DrainAndClose(resp *http.Response) {
	if resp == nil || resp.Body == nil {
		return
	}
	_, _ = io.Copy(io.Discard, io.LimitReader(resp.Body, 64*1024))
	_ = resp.Body.Close()
}
```

### 5.2 Tuned `http.Client` and `http.Transport`

```go
package restclient

import (
	"net"
	"net/http"
	"time"
)

func NewTunedHTTPClient() *http.Client {
	dialer := &net.Dialer{
		Timeout:   5 * time.Second,  // Max time to establish TCP connection
		KeepAlive: 30 * time.Second, // OS-level TCP keep-alive probes
	}

	transport := &http.Transport{
		Proxy:                 http.ProxyFromEnvironment,
		DialContext:           dialer.DialContext,
		ForceAttemptHTTP2:     true,
		MaxIdleConns:          1000,             // Total idle connections across all hosts
		MaxIdleConnsPerHost:   100,              // Crucial: raised from default 2 to 100!
		MaxConnsPerHost:       250,              // Hard cap to avoid overwhelming downstream
		IdleConnTimeout:       90 * time.Second, // Close idle connections after 90s
		TLSHandshakeTimeout:   5 * time.Second,
		ResponseHeaderTimeout: 10 * time.Second, // Max wait for server to send first header byte
		ExpectContinueTimeout: 1 * time.Second,
	}

	return &http.Client{
		Transport: transport,
		Timeout:   30 * time.Second, // Hard ceiling covering dial + TLS + body read
	}
}
```

---

## Chapter 6: Level 3 — Production Client Resilience (Retries with Jitter, Circuit Breakers & Observability)

In distributed systems, networks blip, pods restart, and downstream services occasionally throw `429 Too Many Requests` or `503 Service Unavailable`.

### 6.1 Composable `http.RoundTripper` Middleware Architecture

In Go, `http.Client` uses the `http.RoundTripper` interface (`RoundTrip(*http.Request) (*http.Response, error)`). By wrapping `http.Transport` in custom `RoundTripper`s, you get clean client-side middleware for **logging, distributed tracing, authentication headers, and retries**—without cluttering your business logic!

#### Golden Rules for HTTP Client Retries:
1. **Only Retry Safe/Idempotent Operations:** Retry `GET`, `HEAD`, `OPTIONS`, `PUT`, `DELETE`, or `POST` requests carrying an explicit `Idempotency-Key` header.
2. **Only Retry Transient Errors:** Retry network timeouts/connection resets, `408 Request Timeout`, `429 Too Many Requests`, `502 Bad Gateway`, `503 Service Unavailable`, and `504 Gateway Timeout`. **Never** retry `400 Bad Request`, `401 Unauthorized`, or `404 Not Found`.
3. **Replayable Request Bodies (`req.GetBody`):** Once an `http.Request` body is sent once, its `io.Reader` is exhausted. Before retrying, you must reset `req.Body` using `req.GetBody()`.
4. **Exponential Backoff with Full Jitter:** Adding randomness (*jitter*) prevents thousands of clients from retrying at the exact same millisecond and creating a "thundering herd" outage.
5. **Respect `context.Context` Cancellation:** Always select on `<-req.Context().Done()` during backoff sleeps so a canceled user request aborts retries immediately.

### 6.2 Production Retry & Telemetry `RoundTripper` Implementation

```go
package restclient

import (
	"bytes"
	"fmt"
	"io"
	"log/slog"
	"math"
	"math/rand/v2"
	"net/http"
	"strconv"
	"time"
)

// RetryTransport wraps an underlying http.RoundTripper with exponential backoff + jitter.
type RetryTransport struct {
	Base        http.RoundTripper
	MaxRetries  int
	BaseBackoff time.Duration
	MaxBackoff  time.Duration
	Logger      *slog.Logger
}

func (t *RetryTransport) RoundTrip(req *http.Request) (*http.Response, error) {
	// Ensure request body can be replayed across retries if present
	if req.Body != nil && req.GetBody == nil {
		bodyBytes, err := io.ReadAll(req.Body)
		if err != nil {
			return nil, fmt.Errorf("read request body for retry buffering: %w", err)
		}
		_ = req.Body.Close()
		req.GetBody = func() (io.ReadCloser, error) {
			return io.NopCloser(bytes.NewReader(bodyBytes)), nil
		}
		req.Body, _ = req.GetBody()
	}

	var resp *http.Response
	var err error

	for attempt := 0; attempt <= t.MaxRetries; attempt++ {
		if attempt > 0 {
			// Reset body stream before retry
			if req.GetBody != nil {
				req.Body, err = req.GetBody()
				if err != nil {
					return nil, err
				}
			}

			wait := t.calculateBackoff(attempt, resp)
			// Drain previous failed response body so its socket can return to the pool
			DrainAndClose(resp)

			if t.Logger != nil {
				t.Logger.Warn("retrying outbound HTTP request",
					slog.String("method", req.Method),
					slog.String("url", req.URL.String()),
					slog.Int("attempt", attempt),
					slog.Duration("backoff", wait),
				)
			}

			timer := time.NewTimer(wait)
			select {
			case <-req.Context().Done():
				timer.Stop()
				return nil, req.Context().Err()
			case <-timer.C:
			}
		}

		resp, err = t.Base.RoundTrip(req)
		if !t.shouldRetry(req, resp, err) {
			return resp, err
		}
	}

	return resp, err
}

func (t *RetryTransport) shouldRetry(req *http.Request, resp *http.Response, err error) bool {
	// Check if method is safe to retry (idempotent methods or explicit Idempotency-Key)
	isIdempotent := req.Method == http.MethodGet ||
		req.Method == http.MethodHead ||
		req.Method == http.MethodPut ||
		req.Method == http.MethodDelete ||
		req.Header.Get("Idempotency-Key") != ""

	if !isIdempotent {
		return false
	}

	// Do not retry if caller's context was canceled or timed out
	if req.Context().Err() != nil {
		return false
	}

	// Transient network-level error (DNS glitch, connection reset, TLS timeout)
	if err != nil {
		return true
	}

	// Retryable HTTP status codes
	switch resp.StatusCode {
	case http.StatusRequestTimeout,      // 408
		http.StatusTooManyRequests,      // 429
		http.StatusBadGateway,           // 502
		http.StatusServiceUnavailable,   // 503
		http.StatusGatewayTimeout:       // 504
		return true
	default:
		return false
	}
}

func (t *RetryTransport) calculateBackoff(attempt int, resp *http.Response) time.Duration {
	// Respect server's Retry-After header if provided (e.g., on 429 or 503)
	if resp != nil {
		if retryAfter := resp.Header.Get("Retry-After"); retryAfter != "" {
			if seconds, err := strconv.Atoi(retryAfter); err == nil && seconds > 0 {
				d := time.Duration(seconds) * time.Second
				if d <= t.MaxBackoff {
					return d
				}
				return t.MaxBackoff
			}
		}
	}

	// Exponential backoff with full jitter: random(0, min(MaxBackoff, BaseBackoff * 2^attempt))
	exp := float64(t.BaseBackoff) * math.Pow(2, float64(attempt-1))
	maxSleep := min(time.Duration(exp), t.MaxBackoff)
	return rand.N(maxSleep)
}
```

### 6.3 Typed, Ergonomic Production SDK Client Wrapper

Wrap your configured `*http.Client` in a clean domain client struct that handles base URLs, `Idempotency-Key` injection, structured RFC 9457 error parsing, and safe body draining automatically.

```go
package ordersclient

import (
	"bytes"
	"context"
	"encoding/json"
	"fmt"
	"io"
	"log/slog"
	"net/http"
	"time"

	"yourapp/httpx"
	"yourapp/restclient"
)

type APIError struct {
	StatusCode int
	Problem    httpx.ProblemDetail
}

func (e *APIError) Error() string {
	return fmt.Sprintf("orders API error (status %d): %s - %s",
		e.StatusCode, e.Problem.Title, e.Problem.Detail)
}

type Client struct {
	baseURL    string
	httpClient *http.Client
}

func New(baseURL string, logger *slog.Logger) *Client {
	baseClient := restclient.NewTunedHTTPClient()

	// Wrap the tuned transport with our production RetryTransport
	baseClient.Transport = &restclient.RetryTransport{
		Base:        baseClient.Transport,
		MaxRetries:  3,
		BaseBackoff: 100 * time.Millisecond,
		MaxBackoff:  3 * time.Second,
		Logger:      logger,
	}

	return &Client{
		baseURL:    baseURL,
		httpClient: baseClient,
	}
}

type Order struct {
	ID          string `json:"id"`
	CustomerID  string `json:"customer_id"`
	AmountCents int64  `json:"amount_cents"`
	Status      string `json:"status"`
}

type CreateOrderInput struct {
	IdempotencyKey string `json:"-"`
	CustomerID     string `json:"customer_id"`
	AmountCents    int64  `json:"amount_cents"`
}

func (c *Client) CreateOrder(ctx context.Context, in CreateOrderInput) (Order, error) {
	var out Order

	bodyBytes, err := json.Marshal(in)
	if err != nil {
		return out, fmt.Errorf("marshal request: %w", err)
	}

	req, err := http.NewRequestWithContext(ctx, http.MethodPost, c.baseURL+"/api/v1/orders", bytes.NewReader(bodyBytes))
	if err != nil {
		return out, fmt.Errorf("build request: %w", err)
	}
	req.Header.Set("Content-Type", "application/json")
	req.Header.Set("Accept", "application/json")
	if in.IdempotencyKey != "" {
		// Enables safe automatic retries in RetryTransport!
		req.Header.Set("Idempotency-Key", in.IdempotencyKey)
	}

	resp, err := c.httpClient.Do(req)
	if err != nil {
		return out, fmt.Errorf("execute create order: %w", err)
	}
	// Guarantees socket reuse regardless of success or error path
	defer restclient.DrainAndClose(resp)

	if resp.StatusCode != http.StatusCreated && resp.StatusCode != http.StatusOK {
		var problem httpx.ProblemDetail
		// Limit error body read to 64KB
		_ = json.NewDecoder(io.LimitReader(resp.Body, 64*1024)).Decode(&problem)
		return out, &APIError{StatusCode: resp.StatusCode, Problem: problem}
	}

	// Limit success body read to 4MB to protect client from oversized payloads
	if err := json.NewDecoder(io.LimitReader(resp.Body, 4<<20)).Decode(&out); err != nil {
		return out, fmt.Errorf("decode order response: %w", err)
	}

	return out, nil
}
```

---

# Part III: End-to-End Integration

## Chapter 7: Putting It Together — Production Server + Client Blueprint

Below is a complete `main.go` wiring together the production REST server (with middleware, health probes, and graceful shutdown) and demonstrating how a production REST client calls it with context deadlines and idempotent retries.

```go
package main

import (
	"context"
	"log/slog"
	"net/http"
	"os"
	"sync"
	"time"

	"yourapp/middleware"
	"yourapp/orders"
	"yourapp/ordersclient"
	"yourapp/server"
)

// In-memory thread-safe store for demonstration
type memoryOrderService struct {
	mu     sync.RWMutex
	orders map[string]orders.Order
}

func (m *memoryOrderService) Create(ctx context.Context, req orders.CreateOrderReq) (orders.Order, error) {
	m.mu.Lock()
	defer m.mu.Unlock()
	id := "ord_" + time.Now().Format("150405.000")
	o := orders.Order{
		ID:          id,
		CustomerID:  req.CustomerID,
		AmountCents: req.AmountCents,
		Status:      "CREATED",
	}
	m.orders[id] = o
	return o, nil
}

func (m *memoryOrderService) GetByID(ctx context.Context, id string) (orders.Order, error) {
	m.mu.RLock()
	defer m.mu.RUnlock()
	o, ok := m.orders[id]
	if !ok {
		return orders.Order{}, orders.ErrNotFound
	}
	return o, nil
}

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelInfo}))

	// 1. Setup Domain Service & HTTP Handlers
	svc := &memoryOrderService{orders: make(map[string]orders.Order)}
	orderHandler := orders.NewHandler(svc)

	apiMux := http.NewServeMux()
	orderHandler.RegisterRoutes(apiMux)

	// 2. Wrap routes in Production Middleware Stack
	protectedHandler := middleware.Chain(
		apiMux,
		middleware.Recovery(logger),
		middleware.AccessLog(logger),
		middleware.LoadShedder(500), // Cap at 500 concurrent in-flight requests
	)

	// 3. Start Hardened Server in background for demo
	srv := server.New(":8080", protectedHandler, logger)
	go func() {
		if err := srv.Run(); err != nil {
			logger.Error("server exited with error", slog.Any("err", err))
		}
	}()

	time.Sleep(200 * time.Millisecond) // Wait briefly for listener bind

	// 4. Use the Production REST Client with Context Timeout & Idempotency Key
	client := ordersclient.New("http://localhost:8080", logger)

	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	created, err := client.CreateOrder(ctx, ordersclient.CreateOrderInput{
		IdempotencyKey: "idem-key-98765",
		CustomerID:     "cust_42",
		AmountCents:    4999,
	})
	if err != nil {
		logger.Error("client call failed", slog.Any("err", err))
		os.Exit(1)
	}

	logger.Info("order successfully created via production client",
		slog.String("order_id", created.ID),
		slog.Int64("amount_cents", created.AmountCents),
	)
}
```

### Summary Checklist: Basic vs. Production REST in Go

| Concern | Basic (`net/http` defaults) | Production-Grade Implementation |
| :--- | :--- | :--- |
| **Server Timeouts** | None (vulnerable to Slowloris DoS) | `ReadHeaderTimeout`, `ReadTimeout`, `WriteTimeout`, `IdleTimeout` |
| **JSON Decoding** | `io.ReadAll` + `Unmarshal` (OOM risk) | `http.MaxBytesReader` + streaming `json.NewDecoder` |
| **JSON Encoding** | Direct `Encode(w)` (partial 200 OK bug) | Pooled `bytes.Buffer` (`sync.Pool`) before `WriteHeader` |
| **Error Format** | Unstructured strings | RFC 9457 (`application/problem+json`) |
| **Shutdown** | Immediate exit (`502` errors) | Flip `/readyz` $\rightarrow$ brief drain delay $\rightarrow$ `srv.Shutdown(ctx)` |
| **Client Pooling** | `MaxIdleConnsPerHost = 2` (port exhaustion) | Custom `http.Transport` with `MaxIdleConnsPerHost = 100+` |
| **Client Body Close** | `resp.Body.Close()` without reading | `DrainAndClose` (`io.LimitReader` + `io.Discard`) for socket reuse |
| **Client Retries** | Manual loops or none | Composable `http.RoundTripper` with exponential backoff + full jitter |