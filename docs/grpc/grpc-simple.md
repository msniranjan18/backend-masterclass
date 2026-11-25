# Production-Grade gRPC Systems in Go - Simpler

## Table of Contents

1. **Chapter 1: How gRPC Works Under the Hood (HTTP/2 & `grpc-go`)**

   * HTTP/2 Multiplexing, Streams, and Flow Control (`WINDOW_UPDATE`)

   * Wire Format: Length-Prefixed Framing & Protobuf Binary Encoding

   * `grpc-go` Internal Architecture (`loopyWriter` and Goroutine Workers)

2. **Chapter 2: Protobuf Schema Design & Safe Updates Over Time**

   * How to Update `.proto` Files Without Breaking Old Clients

   * Well-Known Types, `FieldMask` for Partial Updates, and `oneof` Choices

   * Implementing Safe Partial Updates in Go

3. **Chapter 3: Production Server Configuration, Keepalives & Connection Management**

   * Keepalive Settings (Detecting Dead Connections & Blocking Ping Spam)

   * `MaxConnectionAge` (Solving Load Balancer "Stickiness" in Kubernetes)

   * Message Size Limits, Buffer Pools, and Window Size Tuning

4. **Chapter 4: Client-Side Load Balancing, Name Resolution & Retries**

   * Why Standard Load Balancers Struggle with gRPC (L4 vs. L7)

   * Kubernetes Headless DNS (`dns:///`) + `round_robin` Balancing

   * Automatic Retries & Exponential Backoff via Service Config

5. **Chapter 5: Interceptors, Context Deadlines & Rich Error Handling**

   * Deadline Propagation (Stopping Slow Queries from Crashing Everything)

   * Panic Recovery & Wrapping Streaming Contexts

   * Rich Error Models (Sending Detailed Validation Errors Back to Clients)

6. **Chapter 6: Streaming Architectures, Backpressure & Flow Control**

   * Thread-Safety Rules for gRPC Streams (What You Can and Cannot Do Concurrently)

   * Building a Safe Bidirectional Stream with Bounded Channels

7. **Chapter 7: Security, mTLS, Health Checks & Graceful Shutdown**

   * Zero-Downtime Mutual TLS (mTLS) with Live Certificate Rotation

   * Kubernetes Health Checks (`grpc.health.v1`) & Graceful Shutdown

## Chapter 1: How gRPC Works Under the Hood (HTTP/2 & `grpc-go`)

To build reliable, fast gRPC services in Go, it helps to understand what happens *between* your Go function and the network cable.

### 1.1 HTTP/2 Multiplexing and Flow Control

Think of **HTTP/1.1** like a single-lane bridge: if a large truck (a slow request) is crossing, every car behind it has to wait (this is called *Head-of-Line Blocking*). To work around this, HTTP/1.1 clients open many separate TCP connections at once.

**gRPC uses HTTP/2**, which works like a multi-lane highway over a **single TCP connection**:

* **One Connection, Many Streams:** A client opens just *one* TCP connection to a server and sends hundreds of requests at the exact same time. Each request gets its own virtual lane called a **Stream** (numbered 1, 3, 5, etc.). Responses can come back in any order without blocking each other.

* **How Messages Are Packaged on the Wire:** Because multiple streams share one TCP pipe, gRPC wraps every message in a tiny 5-byte envelope so the receiver knows where one message ends and the next begins:

  * **Byte 1 (Compression Flag):** `0` means uncompressed; `1` means compressed (e.g., gzip).

  * **Bytes 2–5 (Message Size):** A number telling the receiver the exact byte length of the message.

  * **Remaining Bytes (Payload):** Your actual data, packed tightly in binary format by Protocol Buffers.

* **Built-in Speed Control (`WINDOW_UPDATE`):** Imagine a fast sender pouring water into a funnel faster than the receiver can drink it. Without speed control, the receiver runs out of memory and crashes. HTTP/2 solves this using **Flow Control**:

  * The receiver gives the sender a "credit allowance" (by default, 64 KB).

  * Every time the sender transmits data, it spends its credit.

  * As your Go server reads messages using `stream.Recv()`, `grpc-go` sends a `WINDOW_UPDATE` message back to the sender saying, *"I cleared space in my buffer, here is more credit."*

  * If your Go server gets busy and stops calling `stream.Recv()`, it stops sending `WINDOW_UPDATE` credits. Once the sender runs out of credit, its `stream.Send()` call automatically pauses until your server catches up. This prevents memory crashes automatically.

### 1.2 How `grpc-go` Handles Network I/O Internally

Inside `grpc-go`, every active connection uses a smart design to avoid CPU bottlenecks:

1. **One Reader Goroutine:** A single background goroutine reads incoming bytes from the network socket, checks the `StreamID` on each packet, and drops the data into that specific stream's inbox buffer.

2. **One Writer Goroutine (`loopyWriter`):** If 1,000 goroutines tried to write directly to the network socket at the same time, they would fight over locks and slow down. Instead, `grpc-go` funnels all outgoing messages into a lock-free queue handled by a single `loopyWriter` goroutine. `loopyWriter` bundles multiple small messages together and writes them to the OS network socket in one big batch, drastically reducing CPU overhead.

3. **Worker Pool (`NumStreamWorkers`):** By default, Go creates a brand-new goroutine for every incoming request. Under extreme traffic (50,000+ requests/sec), creating and destroying goroutines constantly creates extra work for Go's Garbage Collector. Setting `grpc.NumStreamWorkers` tells `grpc-go` to reuse a fixed pool of goroutines instead.

```go
package main

import (
	"runtime"
	"google.golang.org/grpc"
)

// ConfigureWorkerPoolServer reuses a fixed pool of goroutines to handle incoming requests.
// This reduces memory churn and Garbage Collector work when handling tens of thousands
// of requests per second.
func ConfigureWorkerPoolServer(opts ...grpc.ServerOption) *grpc.Server {
	baseOpts := []grpc.ServerOption{
		grpc.NumStreamWorkers(uint32(runtime.NumCPU())),
	}
	baseOpts = append(baseOpts, opts...)
	return grpc.NewServer(baseOpts...)
}

```

## Chapter 2: Protobuf Schema Design & Safe Updates Over Time

In production, you will never deploy your server and all your clients at the exact same second. Old mobile apps or older microservices will still talk to your updated server. Following a few strict `.proto` rules guarantees old and new code never break each other.

### 2.1 How to Update `.proto` Files Safely

Unlike JSON—which sends field names like `"user_id": 123` over the network—Protobuf only sends the **field number** (like `1: 123`). This makes messages tiny and fast, but it creates important rules:

1. **Never Reuse Deleted Field Numbers:** Suppose field `6` used to be `string discount_code`. Later, you delete it, and a year later someone adds `int32 discount_code = 6`. An old client still sending a `string` in slot `6` will cause decoding errors or corrupted data. **Always mark deleted numbers and names as `reserved`** so the compiler forbids reusing them.

2. **Safe Type Changes:** You can safely change an `int32` to an `int64` because Protobuf uses the same variable-length encoding for both. However, changing `int32` to `sint32` will corrupt numbers because `sint32` uses a different encoding for negative numbers.

3. **Always Wrap Inputs and Outputs in Messages:** Even if an RPC needs no parameters right now, create `message ListUsersRequest {}` instead of using `google.protobuf.Empty`. Six months from now, when you need to add pagination (`page_size`), you can simply add a field to `ListUsersRequest` without breaking any existing callers.

4. **Always Make Enum `0` an "Unspecified" Value:** In Protobuf, if a sender forgets to set an enum field, its value defaults to `0`. If `0` means `ORDER_STATUS_PAID`, you won't know whether the order was actually paid or if an older client simply didn't send that field. Always set `0` to `UNSPECIFIED`.

### 2.2 Production Protobuf Design Example

Here is a clean, production-ready `.proto` file using `reserved` fields, `oneof` (for mutually exclusive choices), and `FieldMask` (for partial updates):

```
syntax = "proto3";

package commerce.orders.v1;

option go_package = "github.com/acme/platform/gen/go/commerce/orders/v1;ordersv1";

import "google/protobuf/field_mask.proto";
import "google/protobuf/timestamp.proto";

service OrderService {
  rpc CreateOrder(CreateOrderRequest) returns (CreateOrderResponse);
  rpc GetOrder(GetOrderRequest) returns (GetOrderResponse);
  // Partial update pattern using FieldMask (similar to HTTP PATCH)
  rpc UpdateOrder(UpdateOrderRequest) returns (UpdateOrderResponse);
  // Server-side streaming for real-time order updates
  rpc WatchOrderStatus(WatchOrderStatusRequest) returns (stream WatchOrderStatusResponse);
}

enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;
  ORDER_STATUS_CREATED = 1;
  ORDER_STATUS_PAID = 2;
  ORDER_STATUS_SHIPPED = 3;
  ORDER_STATUS_CANCELLED = 4;
}

message Order {
  // Block anyone from ever reusing these retired field numbers or names
  reserved 6, 7;
  reserved "legacy_tax_code", "discount_flag";

  string id = 1;
  string customer_id = 2;
  OrderStatus status = 3;
  int64 total_amount_cents = 4;
  string currency = 5;

  // "oneof" guarantees an Order can have EITHER a CardPayment OR a BankTransferPayment, never both
  oneof payment_instrument {
    CardPayment card = 8;
    BankTransferPayment bank_transfer = 9;
  }

  google.protobuf.Timestamp created_at = 10;
  google.protobuf.Timestamp updated_at = 11;
}

message CardPayment {
  string last_four = 1;
  string brand = 2;
}

message BankTransferPayment {
  string iban_masked = 1;
  string routing_code = 2;
}

message UpdateOrderRequest {
  Order order = 1;
  // A list of field names the client actually wants to change (e.g., ["status", "currency"]).
  // Without this, we wouldn't know if total_amount_cents = 0 means "set price to $0" or "don't touch price".
  google.protobuf.FieldMask update_mask = 2;
}

```

### 2.3 Implementing Safe Partial Updates (`FieldMask`) in Go

When a client sends an `UpdateOrderRequest`, `fieldmaskpb` validates that the requested field names actually exist and applies only those specific changes:

```go
package orders

import (
	ordersv1 "github.com/acme/platform/gen/go/commerce/orders/v1"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
)

func ApplyOrderUpdate(existing *ordersv1.Order, req *ordersv1.UpdateOrderRequest) error {
	if req.GetOrder() == nil || req.GetUpdateMask() == nil {
		return status.Error(codes.InvalidArgument, "order and update_mask are required")
	}

	// 1. Check that every field name in update_mask actually exists on the Order message
	if !req.GetUpdateMask().IsValid(existing) {
		return status.Error(codes.InvalidArgument, "invalid field path in update_mask")
	}

	// 2. Clean up duplicate or overlapping paths
	req.GetUpdateMask().Normalize()

	// 3. Only update the exact fields listed in the mask
	for _, path := range req.GetUpdateMask().GetPaths() {
		switch path {
		case "status":
			if req.GetOrder().GetStatus() == ordersv1.OrderStatus_ORDER_STATUS_UNSPECIFIED {
				return status.Error(codes.InvalidArgument, "status cannot be UNSPECIFIED")
			}
			existing.Status = req.GetOrder().GetStatus()
		case "total_amount_cents":
			existing.TotalAmountCents = req.GetOrder().GetTotalAmountCents()
		case "currency":
			existing.Currency = req.GetOrder().GetCurrency()
		default:
			return status.Errorf(codes.InvalidArgument, "field %q cannot be updated", path)
		}
	}
	return nil
}

```

## Chapter 3: Production Server Configuration, Keepalives & Connection Management

Calling `grpc.NewServer()` with zero options leaves your server vulnerable to memory spikes, dead connections, and uneven traffic distribution in Kubernetes.

### 3.1 Solving Load Balancer "Stickiness" with `MaxConnectionAge`

Here is one of the most common surprises teams hit when moving gRPC to Kubernetes:

1. Suppose you have **3 server pods** handling traffic, and a spike hits, so Kubernetes scales up to **10 server pods**.

2. Surprisingly, the **7 new pods sit at 0% CPU** while the original 3 pods remain overloaded!

**Why does this happen?** Standard load balancers (like AWS NLB or Kubernetes `ClusterIP`) balance **TCP connections**, not individual requests. Because gRPC keeps a single TCP connection open forever and sends all requests over it, existing clients stay glued ("sticky") to the original 3 pods.

**How to fix it:** Set `MaxConnectionAge` on your gRPC server.

* Every 15 minutes (with a random $\pm 10\%$ jitter so all connections don't drop at once), the server politely sends an HTTP/2 `GOAWAY` signal to the client.

* The client finishes any active requests within `MaxConnectionAgeGrace`, closes the old connection, and opens a brand-new connection—which the load balancer now routes to the newly created pods!

### 3.2 Complete Production `grpc.Server` Configuration

```go
package server

import (
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/keepalive"
)

func NewProductionGRPCServer(
	unaryInterceptors []grpc.UnaryServerInterceptor,
	streamInterceptors []grpc.StreamServerInterceptor,
) *grpc.Server {
	// 1. ServerParameters controls how the server maintains and recycles connections
	kaParams := keepalive.ServerParameters{
		// Close completely idle connections after 5 minutes to free up server memory/sockets
		MaxConnectionIdle: 5 * time.Minute,
		// Politely ask clients to reconnect every 15 minutes so traffic rebalances across new pods
		MaxConnectionAge: 15 * time.Minute,
		// Give active requests 30 seconds to finish after asking the client to reconnect
		MaxConnectionAgeGrace: 30 * time.Second,
		// Send a heartbeat PING if no data moved for 1 minute (prevents cloud NAT firewalls from silently dropping the connection)
		Time: 1 * time.Minute,
		// If the client doesn't reply to the PING within 20 seconds, assume the client crashed and close the socket
		Timeout: 20 * time.Second,
	}

	// 2. EnforcementPolicy protects your server from buggy clients spamming PING packets
	kaPolicy := keepalive.EnforcementPolicy{
		MinTime:             30 * time.Second, // Disconnect clients that ping more than once every 30s
		PermitWithoutStream: true,             // Allow pings even when no active RPC is running
	}

	opts := []grpc.ServerOption{
		grpc.KeepaliveParams(kaParams),
		grpc.KeepaliveEnforcementPolicy(kaPolicy),

		// Cap message sizes so a single massive payload cannot cause an Out-Of-Memory crash
		grpc.MaxRecvMsgSize(4 * 1024 * 1024),  // Max 4 MB incoming request
		grpc.MaxSendMsgSize(16 * 1024 * 1024), // Max 16 MB outgoing response

		// Limit a single client connection to 250 simultaneous requests so one noisy client can't starve others
		grpc.MaxConcurrentStreams(250),

		// Increase HTTP/2 flow control buffers from 64KB to 1MB for faster large transfers
		grpc.InitialWindowSize(1 << 20),
		grpc.InitialConnWindowSize(1 << 20),

		// Attach middleware interceptors
		grpc.ChainUnaryInterceptor(unaryInterceptors...),
		grpc.ChainStreamInterceptor(streamInterceptors...),
	}

	return grpc.NewServer(opts...)
}

```

## Chapter 4: Client-Side Load Balancing, Name Resolution & Retries

### 4.1 Client-Side Load Balancing in Kubernetes

If your gRPC client connects to a standard Kubernetes Service (`orders-svc:9090`), Kubernetes gives the client **one single virtual IP address**. The client opens one TCP connection to that IP, sending 100% of its traffic to a single pod.

To spread requests evenly across *all* pods without installing a complex service mesh (like Istio):

1. Create a **Headless Service** in Kubernetes (`clusterIP: None`). When the client looks up the DNS name, Kubernetes returns the real IP addresses of *every* healthy pod.

2. Tell `grpc-go` to use its built-in DNS watcher by prefixing the address with `dns:///` (note the three slashes) and enabling `"round_robin"` load balancing.

3. `grpc-go` automatically opens one connection to each pod and rotates every outgoing RPC across all pods evenly.

### 4.2 Automatic Retries & Backoff via Service Config

Networks glitch. Pods restart. Instead of writing messy `for` loops with `time.Sleep` around every gRPC client call, `grpc-go` has a built-in retry engine configured via JSON (`DefaultServiceConfig`).

* If a request fails with `UNAVAILABLE` (meaning the connection dropped before the server could process it), `grpc-go` automatically waits a tiny randomized backoff time (`0.1s`, then `0.2s`, then `0.4s`) and retries on a different pod.

* Once the server starts sending a response back, `grpc-go` guarantees it will **never** retry that call, preventing duplicate side effects.

```go
package client

import (
	"fmt"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials"
	"google.golang.org/grpc/keepalive"
)

// serviceConfigJSON tells the gRPC client how to load-balance across pods
// and automatically retry transient network failures.
const serviceConfigJSON = `{
	"loadBalancingConfig": [
		{ "round_robin": {} }
	],
	"methodConfig": [
		{
			"name": [
				{ "service": "commerce.orders.v1.OrderService" }
			],
			"waitForReady": true,
			"timeout": "5s",
			"retryPolicy": {
				"MaxAttempts": 4,
				"InitialBackoff": "0.1s",
				"MaxBackoff": "2s",
				"BackoffMultiplier": 2.0,
				"RetryableStatusCodes": [
					"UNAVAILABLE",
					"ABORTED"
				]
			}
		},
		{
			"name": [
				{ "service": "commerce.orders.v1.OrderService", "method": "CreateOrder" }
			],
			"retryPolicy": {
				"MaxAttempts": 2,
				"InitialBackoff": "0.2s",
				"MaxBackoff": "1s",
				"BackoffMultiplier": 2.0,
				"RetryableStatusCodes": [
					"UNAVAILABLE"
				]
			}
		}
	]
}`

func NewProductionGRPCClient(target string, creds credentials.TransportCredentials) (*grpc.ClientConn, error) {
	// Example target for a Kubernetes Headless Service:
	// "dns:///orders-headless.prod.svc.cluster.local:9090"
	opts := []grpc.DialOption{
		grpc.WithTransportCredentials(creds),
		grpc.WithDefaultServiceConfig(serviceConfigJSON),

		// Keepalive Time (45s) MUST be higher than the server's EnforcementPolicy.MinTime (30s),
		// otherwise the server will disconnect the client for pinging too frequently!
		grpc.WithKeepaliveParams(keepalive.ClientParameters{
			Time:                45 * time.Second,
			Timeout:             15 * time.Second,
			PermitWithoutStream: true,
		}),

		grpc.WithInitialWindowSize(1 << 20),
		grpc.WithInitialConnWindowSize(1 << 20),
	}

	// grpc.NewClient initializes the client immediately without blocking startup;
	// connections are established automatically in the background.
	conn, err := grpc.NewClient(target, opts...)
	if err != nil {
		return nil, fmt.Errorf("failed to create grpc client for %s: %w", target, err)
	}
	return conn, nil
}

```

## Chapter 5: Interceptors, Context Deadlines & Rich Error Handling

Interceptors are gRPC's equivalent of HTTP middleware—they let you run logging, authentication, metrics, and timeout checks across all endpoints in one place.

### 5.1 Deadline Propagation (Stopping Domino-Effect Failures)

Imagine **Service A** calls **Service B**, which calls **Service C**, which queries a slow database.

If Service A gives up after 2 seconds (`DEADLINE_EXCEEDED`), you don't want Service B and Service C to keep wasting CPU and database connections working on a request nobody is waiting for anymore.

**gRPC handles this automatically:** When Service A sets `context.WithTimeout(ctx, 2*time.Second)`, `grpc-go` sends the remaining time budget (`grpc-timeout`) in the request headers. Service B receives that exact remaining budget in its `ctx`, and passes whatever time is left to Service C. When the 2 seconds expire, **all three services cancel their `ctx` at the exact same moment**.

What if a client forgets to set a timeout? Use an interceptor to enforce a safe default deadline:

```go
package interceptors

import (
	"context"
	"log/slog"
	"runtime/debug"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
)

// DeadlineEnforcementInterceptor guarantees every incoming RPC has a timeout.
// If the caller already set a shorter timeout, the caller's timeout is respected.
func DeadlineEnforcementInterceptor(defaultTimeout time.Duration) grpc.UnaryServerInterceptor {
	return func(ctx context.Context, req any, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (any, error) {
		if _, hasDeadline := ctx.Deadline(); !hasDeadline {
			var cancel context.CancelFunc
			ctx, cancel = context.WithTimeout(ctx, defaultTimeout)
			defer cancel()
		}
		return handler(ctx, req)
	}
}

// RecoveryInterceptor catches unexpected panics in your code so a single bug
// returns a clean codes.Internal error instead of crashing the entire server process.
func RecoveryInterceptor(logger *slog.Logger) grpc.UnaryServerInterceptor {
	return func(ctx context.Context, req any, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp any, err error) {
		defer func() {
			if r := recover(); r != nil {
				logger.Error("grpc_handler_panic",
					slog.String("method", info.FullMethod),
					slog.Any("panic", r),
					slog.String("stack", string(debug.Stack())),
				)
				err = status.Errorf(codes.Internal, "internal server error")
			}
		}()
		return handler(ctx, req)
	}
}

```

### 5.2 Attaching Values to Streaming Contexts

For streaming RPCs, `context.Context` lives inside the `grpc.ServerStream` object rather than being a separate argument. To pass request-scoped values (like a trace ID or authenticated user ID) into your stream handler, wrap `grpc.ServerStream` and override `Context()`:

```go
type wrappedServerStream struct {
	grpc.ServerStream
	ctx context.Context
}

func (w *wrappedServerStream) Context() context.Context {
	return w.ctx
}

func StreamContextEnricherInterceptor() grpc.StreamServerInterceptor {
	return func(srv any, ss grpc.ServerStream, info *grpc.StreamServerInfo, handler grpc.StreamHandler) error {
		enrichedCtx := context.WithValue(ss.Context(), requestStartKey{}, time.Now())
		wrapped := &wrappedServerStream{
			ServerStream: ss,
			ctx:          enrichedCtx,
		}
		return handler(srv, wrapped)
	}
}

type requestStartKey struct{}

```

### 5.3 Sending Detailed Validation Errors (`errdetails`)

Returning `codes.InvalidArgument` with a plain string like `"bad request"` forces frontend or client developers to parse error strings with regexes. Instead, attach Google's standard `errdetails.BadRequest` message to return structured, field-by-field errors:

```go
package errors

import (
	"google.golang.org/genproto/googleapis/rpc/errdetails"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
)

// NewValidationError attaches structured field-level error details to a gRPC status.
func NewValidationError(fieldViolations map[string]string) error {
	st := status.New(codes.InvalidArgument, "request validation failed")

	badReq := &errdetails.BadRequest{}
	for field, desc := range fieldViolations {
		badReq.FieldViolations = append(badReq.FieldViolations, &errdetails.BadRequest_FieldViolation{
			Field:       field,
			Description: desc,
		})
	}

	detailedStatus, err := st.WithDetails(badReq)
	if err != nil {
		return st.Err()
	}
	return detailedStatus.Err()
}

// ParseValidationError shows how a Go client extracts those structured field errors.
func ParseValidationError(err error) map[string]string {
	st := status.Convert(err)
	if st.Code() != codes.InvalidArgument {
		return nil
	}

	violations := make(map[string]string)
	for _, detail := range st.Details() {
		if badReq, ok := detail.(*errdetails.BadRequest); ok {
			for _, v := range badReq.GetFieldViolations() {
				violations[v.GetField()] = v.GetDescription()
			}
		}
	}
	return violations
}

```

## Chapter 6: Streaming Architectures, Backpressure & Flow Control

### 6.1 The Golden Thread-Safety Rules for gRPC Streams

When writing streaming handlers in Go, violating these two rules causes rare, hard-to-debug crashes in production:

1. **Reading and Writing Simultaneously is Safe:** You can have **one goroutine calling `stream.Recv()`** and **one goroutine calling `stream.Send()`** at the exact same time.

2. **Two Goroutines Calling `stream.Send()` is NOT Safe:** `grpc-go`'s stream writer is not thread-safe. If two worker goroutines call `stream.Send()` at the same time on the same stream, your program will panic or corrupt packets. **Always funnel outgoing messages through a single writer goroutine (using a channel).**

### 6.2 Production Bidirectional Streaming Pattern

Here is a complete, safe pattern for a high-throughput bidirectional stream:

* **1 Reader goroutine** pulls events from the network into a bounded `inboundCh`.

* **4 Worker goroutines** process events concurrently and push results into a bounded `outboundCh`.

* **1 Writer goroutine** safely calls `stream.Send()` one message at a time.

* If the client stops reading responses, `stream.Send()` blocks $\rightarrow$ `outboundCh` fills up $\rightarrow$ workers pause $\rightarrow$ `inboundCh` fills up $\rightarrow$ `stream.Recv()` pauses $\rightarrow$ HTTP/2 automatically tells the client to stop sending. Memory usage stays flat and predictable.

```go
package streaming

import (
	"errors"
	"io"
	"sync"

	ordersv1 "github.com/acme/platform/gen/go/commerce/orders/v1"
	"golang.org/x/sync/errgroup"
)

func HandleBidirectionalStream(stream ordersv1.StreamService_SyncOrdersServer) error {
	// errgroup cancels ctx automatically if ANY goroutine returns an error
	g, ctx := errgroup.WithContext(stream.Context())

	// Bounded buffers (capacity 64) prevent unbounded memory growth
	inboundCh := make(chan *ordersv1.OrderEvent, 64)
	outboundCh := make(chan *ordersv1.OrderAck, 64)

	// 1. Single Reader Goroutine: reads messages from the client
	g.Go(func() error {
		defer close(inboundCh)
		for {
			event, err := stream.Recv()
			if errors.Is(err, io.EOF) {
				return nil // Client finished sending all messages cleanly
			}
			if err != nil {
				return err // Client disconnected or timed out
			}

			select {
			case inboundCh <- event:
			case <-ctx.Done():
				return ctx.Err()
			}
		}
	})

	// 2. Worker Pool: processes incoming events in parallel
	const numWorkers = 4
	var workersWg sync.WaitGroup
	workersWg.Add(numWorkers)

	for i := 0; i < numWorkers; i++ {
		g.Go(func() error {
			defer workersWg.Done()
			for {
				select {
				case <-ctx.Done():
					return ctx.Err()
				case event, ok := <-inboundCh:
					if !ok {
						return nil // No more incoming events
					}
					ack, err := processOrderEvent(ctx, event)
					if err != nil {
						return err
					}
					select {
					case outboundCh <- ack:
					case <-ctx.Done():
						return ctx.Err()
					}
				}
			}
		})
	}

	// Close outboundCh once all workers have finished processing remaining items
	go func() {
		workersWg.Wait()
		close(outboundCh)
	}()

	// 3. Single Writer Goroutine: safely sends responses one by one
	g.Go(func() error {
		for {
			select {
			case <-ctx.Done():
				return ctx.Err()
			case ack, ok := <-outboundCh:
				if !ok {
					return nil // All acknowledgments sent cleanly
				}
				if err := stream.Send(ack); err != nil {
					return err
				}
			}
		}
	})

	return g.Wait()
}

```

## Chapter 7: Security, mTLS, Health Checks & Graceful Shutdown

### 7.1 Zero-Downtime Mutual TLS (mTLS) with Live Certificate Rotation

In production environments, services prove their identity to each other using **Mutual TLS (mTLS)**—meaning both the server *and* the client verify each other's TLS certificates.

Modern security tools (like cert-manager or SPIFFE/SPIRE) automatically replace certificate files on disk every 24 hours. If you load your certificate once at startup using `credentials.NewServerTLSFromFile()`, your service will start failing all connections the moment that certificate expires.

**The Solution:** Store the active certificate inside an `atomic.Pointer` and use `tls.Config.GetCertificate`. Every time a new client connects, Go automatically fetches the newest certificate from memory with zero downtime and zero restarts.

```go
package security

import (
	"crypto/tls"
	"crypto/x509"
	"fmt"
	"os"
	"sync/atomic"

	"google.golang.org/grpc/credentials"
)

// DynamicCertReloader holds the current TLS certificate in an atomic pointer
// so background file watchers can swap in renewed certificates safely without restarts.
type DynamicCertReloader struct {
	cert atomic.Pointer[tls.Certificate]
}

func NewDynamicCertReloader(certFile, keyFile string) (*DynamicCertReloader, error) {
	r := &DynamicCertReloader{}
	if err := r.Reload(certFile, keyFile); err != nil {
		return nil, err
	}
	return r, nil
}

// Reload can be called whenever a file watcher detects updated cert/key files on disk.
func (r *DynamicCertReloader) Reload(certFile, keyFile string) error {
	cert, err := tls.LoadX509KeyPair(certFile, keyFile)
	if err != nil {
		return fmt.Errorf("load x509 keypair: %w", err)
	}
	r.cert.Store(&cert)
	return nil
}

// NewMTLSServerCredentials enforces client certificate verification (mTLS)
// while dynamically serving the latest certificate on every handshake.
func NewMTLSServerCredentials(caCertPath string, reloader *DynamicCertReloader) (credentials.TransportCredentials, error) {
	caBytes, err := os.ReadFile(caCertPath)
	if err != nil {
		return nil, fmt.Errorf("read CA cert: %w", err)
	}

	caPool := x509.NewCertPool()
	if !caPool.AppendCertsFromPEM(caBytes) {
		return nil, fmt.Errorf("failed to append CA certificate")
	}

	tlsCfg := &tls.Config{
		MinVersion: tls.VersionTLS13,
		ClientAuth: tls.RequireAndVerifyClientCert,
		ClientCAs:  caPool,
		GetCertificate: func(*tls.ClientHelloInfo) (*tls.Certificate, error) {
			return reloader.cert.Load(), nil
		},
	}

	return credentials.NewTLS(tlsCfg), nil
}

```

### 7.2 Standard Health Checks & Graceful Shutdown

When Kubernetes deploys a new version of your app, it sends a `SIGTERM` signal to shut down old pods. If your Go server exits immediately, active requests fail with errors.

By combining gRPC's standard health check service (`grpc.health.v1`) with `GracefulStop()`:

1. As soon as `SIGTERM` arrives, you flip the health check status to `NOT_SERVING` so load balancers stop sending *new* requests to this pod.

2. `srv.GracefulStop()` stops accepting new connections and waits patiently for all currently running RPCs to finish.

3. A 25-second safety timer ensures that if a buggy client hangs forever, `srv.Stop()` forces a clean exit before Kubernetes kills the container (`SIGKILL`).

```go
package lifecycle

import (
	"context"
	"log/slog"
	"net"
	"os/signal"
	"syscall"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/health"
	healthpb "google.golang.org/grpc/health/grpc_health_v1"
	"google.golang.org/grpc/reflection"
)

func RunGRPCServer(ctx context.Context, addr string, srv *grpc.Server, logger *slog.Logger) error {
	lis, err := net.Listen("tcp", addr)
	if err != nil {
		return err
	}

	// 1. Register standard gRPC Health Check protocol (natively supported by Kubernetes probes)
	healthSrv := health.NewServer()
	healthpb.RegisterHealthServer(srv, healthSrv)
	healthSrv.SetServingStatus("", healthpb.HealthCheckResponse_SERVING)
	healthSrv.SetServingStatus("commerce.orders.v1.OrderService", healthpb.HealthCheckResponse_SERVING)

	// 2. Register Server Reflection so tools like grpcurl can inspect your API
	reflection.Register(srv)

	ctx, stop := signal.NotifyContext(ctx, syscall.SIGINT, syscall.SIGTERM)
	defer stop()

	errCh := make(chan error, 1)
	go func() {
		logger.Info("gRPC server listening", slog.String("addr", addr))
		if err := srv.Serve(lis); err != nil {
			errCh <- err
		}
	}()

	select {
	case err := <-errCh:
		return err
	case <-ctx.Done():
		logger.Info("shutdown signal received, marking health status as NOT_SERVING")
	}

	// Tell load balancers and Kubernetes probes to stop routing new traffic here
	healthSrv.SetServingStatus("", healthpb.HealthCheckResponse_NOT_SERVING)
	healthSrv.SetServingStatus("commerce.orders.v1.OrderService", healthpb.HealthCheckResponse_NOT_SERVING)

	// Drain in-flight requests cleanly, with a 25-second safety timeout
	stopped := make(chan struct{})
	go func() {
		srv.GracefulStop()
		close(stopped)
	}()

	select {
	case <-stopped:
		logger.Info("gRPC server drained all connections cleanly")
	case <-time.After(25 * time.Second):
		logger.Warn("graceful drain timed out, forcing immediate shutdown")
		srv.Stop()
	}

	return nil
}

```