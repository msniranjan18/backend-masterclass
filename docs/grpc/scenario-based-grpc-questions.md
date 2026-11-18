# Scenario-Based gRPC Questions

### 1. High Latency Between Microservices

#### Scenario

Your Go-based microservices communicate over gRPC. Suddenly, p99 latency increases from 20ms to 500ms under high traffic.

#### What would you investigate?

#### Expected Senior-Level Answer

A senior engineer should investigate:

* Connection reuse vs reconnects
* HTTP/2 connection exhaustion
* Head-of-line blocking
* Large protobuf payloads
* CPU spikes due to serialization/deserialization
* Network latency between zones/regions
* Improper goroutine handling
* Missing deadlines/timeouts
* DNS resolution delays
* Load balancer behavior

Should also mention:

* Distributed tracing
* Prometheus metrics
* grpc interceptors for timing
* Profiling using `pprof`

***

### 2. Handling Backpressure in Streaming RPC

#### Scenario

You implemented bidirectional streaming for real-time telemetry ingestion. Clients are sending messages faster than the server can process.

#### How would you handle backpressure?

#### Expected Answer

Must discuss:

* Flow control in HTTP/2
* Buffered channels carefully
* Worker pools
* Rate limiting
* Stream throttling
* Applying context cancellation
* Using bounded queues instead of unbounded buffering
* Dropping or batching messages if acceptable

Senior-level insight:

> Never allow unlimited goroutine spawning or memory growth in streaming systems.

***

### 3. gRPC Service-to-Service Authentication

#### Scenario

Your organization wants zero-trust communication between internal microservices.

#### How would you secure gRPC communication?

#### Expected Answer

Should include:

* mTLS implementation
* Certificate rotation
* SPIFFE/SPIRE concepts (bonus)
* JWT auth via metadata
* Unary/stream interceptors for auth validation
* RBAC enforcement
* TLS cipher suite considerations

Strong answer includes:

* Separation of authentication vs authorization
* Secret management integration

***

### 4. Rolling Deployment Compatibility Issue

#### Scenario

After deploying a new version of a Go gRPC service, old clients start failing.

#### What likely happened and how do you prevent it?

#### Expected Answer

Must discuss protobuf backward compatibility:

Bad changes:

* Removing fields
* Reusing field numbers
* Changing field types

Good practices:

* Reserve deleted field numbers
* Additive schema evolution
* Versioned APIs
* Contract testing

Should mention:

```
reserved 4;reserved "old_field";
```

***

### 5. Debugging Memory Leak in Streaming Server

#### Scenario

A long-running gRPC streaming service continuously increases memory usage.

#### How would you debug it?

#### Expected Answer

Must discuss:

* Goroutine leaks
* Unclosed streams
* Context cancellation not respected
* Blocked channels
* Large buffered messages
* Improper retry loops

Tools:

* `pprof`
* Goroutine dump
* Heap analysis
* Runtime metrics

Senior insight:

> Streaming RPCs are long-lived connections, so resource cleanup discipline is critical.

***

### 6. Retry Storm During Partial Outage

#### Scenario

One downstream gRPC dependency becomes slow. Suddenly all services experience cascading failures.

#### What happened?

#### Expected Answer

Must identify:

* Retry storm
* Thundering herd problem
* Connection pool exhaustion

Mitigation strategies:

* Exponential backoff
* Circuit breakers
* Retry budgets
* Hedging carefully
* Timeouts/deadlines
* Load shedding

Should mention:

> Retries amplify traffic during failures if not controlled.

***

### 7. Designing a Real-Time Notification System

#### Scenario

You need to design a scalable notification system using Go and gRPC for millions of connected clients.

#### What architecture would you choose?

#### Expected Answer

Should discuss:

* Bidirectional streaming
* Connection fanout
* Gateway layer
* Sticky sessions/load balancing
* Horizontal scaling
* Kafka/NATS integration
* Connection state management
* Heartbeats/keepalive

Must mention:

* Stream reconnection handling
* Idempotency
* Message ordering guarantees

***

### 8. Observability in gRPC Systems

#### Scenario

Production incidents occur frequently, but debugging is difficult because requests traverse 15+ microservices.

#### How would you improve observability?

#### Expected Answer

Must include:

* OpenTelemetry
* Distributed tracing
* Correlation/request IDs in metadata
* Structured logging
* Metrics collection
* Unary and stream interceptors

Key metrics:

* Latency
* Error rate
* Active streams
* Retry count
* Payload size

Strong answer:

> Observability must be built into middleware/interceptors rather than business logic.

***

### 9. Large Payload Performance Problem

#### Scenario

A gRPC endpoint transfers 50MB protobuf responses and services start crashing under load.

#### What would you change?

#### Expected Answer

Must discuss:

* Server streaming instead of unary
* Compression (gzip/snappy)
* Pagination/chunking
* Memory pressure from unmarshalling
* Max message size configs

Important insight:

> gRPC is optimized for many small efficient messages, not huge monolithic payloads.

***

### 10. Load Balancing Problem in Kubernetes

#### Scenario

Your Go gRPC clients in Kubernetes always hit the same backend pods unevenly.

#### Why can this happen?

#### Expected Answer

Should explain:

* HTTP/2 persistent connections
* Connection-level load balancing issue
* kube-proxy limitations
* L4 vs L7 balancing

Solutions:

* Client-side load balancing
* Service mesh
* DNS round robin
* xDS
* Envoy

Senior insight:

> Traditional Kubernetes service balancing behaves differently with long-lived gRPC connections compared to REST.







