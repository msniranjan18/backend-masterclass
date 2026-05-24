# 1.1 · Golang

## Overview

Go is a statically typed, compiled language built for concurrency and performance. It is the primary language for cloud-native backend systems, Kubernetes operators, and high-throughput microservices.

---

## Core Concepts

### Language Fundamentals
- Types, interfaces, structs, methods, embedded types
- Pointers and value semantics (when to use `*T` vs `T`)
- `defer`, `panic`, `recover`
- Error handling: `errors.New`, `fmt.Errorf("%w")`, `errors.Is`, `errors.As`
- Blank identifier `_`, named return values
- `iota` in const blocks

### Concurrency
- Goroutines — lightweight (~2KB stack, grows dynamically)
- Channels: buffered vs unbuffered, directional (`chan<-`, `<-chan`)
- `select` for multiplexing
- `sync.Mutex`, `sync.RWMutex`, `sync.WaitGroup`, `sync.Once`, `sync.Pool`
- `sync/atomic` for lock-free counters
- `context.Context` — cancellation, deadlines, values
- Common patterns: worker pool, fan-out/fan-in, pipeline, semaphore

### Memory & Runtime
- GC: concurrent tri-color mark-and-sweep
- Stack vs heap; escape analysis (`go build -gcflags="-m"`)
- GOGC (default 100 = GC when heap doubles), GOMEMLIMIT
- `runtime.NumGoroutine()`, `runtime.GC()`

### Interfaces & Composition
- Implicit satisfaction — no `implements` keyword
- Interface segregation: prefer small interfaces (`io.Reader`, `io.Writer`)
- `any` (empty interface) — avoid overuse
- nil interface != nil concrete pointer (common gotcha)
- Embedding structs and interfaces

### Standard Library Highlights
- `net/http`, `encoding/json`, `encoding/xml`
- `io`, `bufio`, `bytes`, `strings`
- `os`, `filepath`, `time`, `log/slog`
- `testing`, `net/http/httptest`
- `math/rand/v2`, `crypto/rand`

---

## Key Commands / Code Snippets

```go
// Worker pool pattern
func workerPool(jobs <-chan Job, results chan<- Result, n int) {
    var wg sync.WaitGroup
    for i := 0; i < n; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for job := range jobs {
                results <- process(job)
            }
        }()
    }
    wg.Wait()
    close(results)
}

// Context with timeout
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
if err := doWork(ctx); err != nil {
    if errors.Is(err, context.DeadlineExceeded) { /* handle */ }
}

// Error wrapping
return fmt.Errorf("fetchUser(%d): %w", id, err)

// Unwrapping
var notFound *NotFoundError
if errors.As(err, &notFound) { /* handle */ }

// sync.Once for singleton
var (
    instance *DB
    once     sync.Once
)
func GetDB() *DB {
    once.Do(func() { instance = initDB() })
    return instance
}

// Table-driven test
func TestAdd(t *testing.T) {
    cases := []struct{ a, b, want int }{
        {1, 2, 3}, {0, 0, 0}, {-1, 1, 0},
    }
    for _, tc := range cases {
        t.Run(fmt.Sprintf("%d+%d", tc.a, tc.b), func(t *testing.T) {
            if got := Add(tc.a, tc.b); got != tc.want {
                t.Errorf("got %d, want %d", got, tc.want)
            }
        })
    }
}
```

```bash
# Build & run
go build ./...
go run ./cmd/server

# Test with race detector
go test -race -count=1 ./...

# Benchmark
go test -bench=BenchmarkFoo -benchmem -benchtime=5s ./...

# Profile CPU
go test -bench=. -cpuprofile=cpu.prof ./...
go tool pprof cpu.prof

# Module management
go mod tidy && go mod vendor

# Lint (golangci-lint)
golangci-lint run --enable=govet,errcheck,staticcheck

# Generate code
go generate ./...
```

---

## Common Interview Questions

**Q: What is the difference between a goroutine and an OS thread?**
> Goroutines are multiplexed onto OS threads by the Go scheduler (M:N model). They start with a ~2KB stack (vs ~1–8MB for threads) that grows on demand. Goroutines are cheap to create — you can run millions concurrently.

**Q: Explain Go's memory model. How do you safely share data between goroutines?**
> The Go memory model specifies when one goroutine's writes are visible to another. Use channels or `sync` primitives (`Mutex`, `atomic`) to establish "happens-before" guarantees. Never access shared mutable state without synchronization.

**Q: When would you use `sync.Mutex` vs a channel?**
> Use `Mutex` to protect shared state (counters, maps, caches). Use channels for communication and ownership transfer. Channels are not a replacement for mutexes when the only goal is guarding a variable.

**Q: What is `context.Context` used for?**
> Propagating deadlines, cancellation signals, and request-scoped values (e.g., trace IDs) across goroutines and API boundaries. It enables graceful shutdown and request-level timeouts.

**Q: What is a goroutine leak and how do you detect it?**
> A goroutine leak occurs when goroutines are started but never exit (e.g., blocked on a channel nobody closes). Detect with `runtime.NumGoroutine()` or pprof goroutine dumps.

**Q: Explain interface satisfaction. Can a nil pointer satisfy an interface?**
> A type satisfies an interface implicitly. A nil *T can satisfy an interface — the interface value has a non-nil type but a nil pointer value. This can cause unexpected non-nil interface comparisons.

**Q: How does Go's garbage collector work?**
> Concurrent tri-color mark-and-sweep. The GC runs concurrently with the program, minimizing stop-the-world pauses. Tune with `GOGC` (GC trigger percentage) and `GOMEMLIMIT`.

---

## Gotchas & Best Practices

- Always `defer cancel()` after `context.With*`
- Loop variable capture: `for i, v := range slice { go func() { use(i, v) }() }` — capture `v` by value or use `v := v` inside the loop (fixed in Go 1.22+)
- Slice aliasing: `append` may or may not create a new backing array
- `map` is not goroutine-safe — protect with `sync.RWMutex` or use `sync.Map`
- `nil` slice and empty slice behave differently with JSON (`null` vs `[]`)
- Avoid `init()` for complex setup; prefer explicit initialization
- Prefer returning errors over panics in library code
- Use `errors.Is`/`errors.As` — never compare errors with `==` (except `nil`)
- Keep interfaces in the consumer package, not the producer package

---

## Resources

- [ ] [Effective Go](https://go.dev/doc/effective_go)
- [ ] [Go Memory Model](https://go.dev/ref/mem)
- [ ] [Go Concurrency Patterns (Rob Pike)](https://go.dev/blog/pipelines)
- [ ] [100 Go Mistakes](https://100go.co/)
- [ ] [Uber Go Style Guide](https://github.com/uber-go/guide)
- [ ] [Go Proverbs](https://go-proverbs.github.io/)

---

## My Notes

<!-- ADD YOUR PERSONAL NOTES HERE -->

