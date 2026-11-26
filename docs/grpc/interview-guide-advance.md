# Interview Guide

> A book-style guide to the concepts, scenarios, failure modes and trade-offs of gRPC that come up in staff/principal-level interviews. All code is Go (`google.golang.org/grpc`), because that is where the sharpest production lessons live, but the concepts apply to every gRPC implementation.

---

## Table of Contents

1. [Mental Model: What gRPC Really Is](#part-1--mental-model-what-grpc-really-is)
2. [HTTP/2 Under the Hood](#part-2--http2-under-the-hood)
3. [Protocol Buffers Deep Dive](#part-3--protocol-buffers-deep-dive)
4. [The Four RPC Types in Go](#part-4--the-four-rpc-types-in-go)
5. [Status Codes and the Error Model](#part-5--status-codes-and-the-error-model)
6. [Deadlines, Cancellation and Propagation](#part-6--deadlines-cancellation-and-propagation)
7. [Interceptors and Metadata](#part-7--interceptors-and-metadata)
8. [Security: TLS, mTLS, Auth](#part-8--security-tls-mtls-auth)
9. [Retries, Hedging and Resilience](#part-9--retries-hedging-and-resilience)
10. [Load Balancing](#part-10--load-balancing)
11. [Streaming Deep Dive: Flow Control and Backpressure](#part-11--streaming-deep-dive-flow-control-and-backpressure)
12. [Connection Management and Keepalive](#part-12--connection-management-and-keepalive)
13. [Performance Tuning](#part-13--performance-tuning)
14. [Observability](#part-14--observability)
15. [Testing gRPC Services](#part-15--testing-grpc-services)
16. [API Evolution and Governance](#part-16--api-evolution-and-governance)
17. [Gateways, gRPC-Web, Connect and Browsers](#part-17--gateways-grpc-web-connect-and-browsers)
18. [Graceful Shutdown, Health and Operations](#part-18--graceful-shutdown-health-and-operations)
19. [System Design and Scenario Questions](#part-19--system-design-and-scenario-questions)
20. [Rapid-Fire Round and Cheat Sheet](#part-20--rapid-fire-round-and-cheat-sheet)

---

# Part 1 — Mental Model: What gRPC Really Is

gRPC is **a contract-first RPC framework** made of four layered ideas:

| Layer | What it gives you |
|---|---|
| **IDL (`.proto`)** | Language-neutral contract; code generation for clients and servers |
| **Serialization (Protocol Buffers)** | Compact, schema-driven binary encoding with explicit evolution rules |
| **Transport (HTTP/2)** | Multiplexed streams over one TCP connection, header compression, flow control, bidirectional streaming |
| **Runtime semantics** | Deadlines, cancellation, metadata, status codes, interceptors, name resolution, load balancing, retries |

```
 Client app                                          Server app
 ┌──────────────┐                                  ┌──────────────┐
 │ generated    │                                  │ generated    │
 │ stub         │                                  │ service iface│
 ├──────────────┤                                  ├──────────────┤
 │ interceptors │                                  │ interceptors │
 ├──────────────┤   HTTP/2 streams over 1 TCP/TLS  ├──────────────┤
 │ name resolver│ ───────────────────────────────► │ HTTP/2 server│
 │ LB policy    │                                  └──────────────┘
 │ subchannels  │
 └──────────────┘
```

## Q1.1 — Why would you pick gRPC over REST/JSON? When would you NOT?

**Answer.**

Pick gRPC when:

- **Internal service-to-service traffic** where both ends are controlled by you. Strong contracts and generated clients remove an entire class of integration bugs.
- **Latency/throughput matter**: binary encoding is typically 3–10x smaller than JSON and cheaper to parse; HTTP/2 multiplexing avoids head-of-line blocking at the HTTP layer and removes per-request connection setup.
- **Streaming** is first-class (server push, client upload, bidi) rather than bolted on with WebSockets/SSE.
- **Polyglot** environments: one proto, many languages.
- **Built-in deadlines/cancellation** and standard status codes give consistent cross-service behavior.

Avoid or adapt when:

- **Public/browser-facing APIs**: browsers cannot speak native gRPC (no control over HTTP/2 framing and trailers). Use gRPC-Web, Connect, or a REST gateway.
- **Human debuggability** is paramount (curl, browser dev tools). Binary payloads need `grpcurl`, reflection, or tooling.
- **Caching via HTTP intermediaries (CDN)** is crucial: gRPC uses POST for everything; standard HTTP caching does not apply.
- **Third-party consumers** unwilling to adopt protobuf tooling.
- **Infrastructure that doesn't understand HTTP/2 end-to-end** (old L7 proxies, some API gateways).

**Staff-level nuance.** The honest trade-off is *operational*, not performance. gRPC pushes complexity into: load balancing (long-lived connections break L4 balancing), observability tooling, schema governance, and browser reach. A staff engineer weighs the *platform cost* against the benefits and often picks "gRPC inside, REST/GraphQL at the edge".

## Q1.2 — Walk through what happens on the wire for a single unary call.

**Answer.**

1. Client resolves the target name (DNS/xDS), picks a subchannel through its LB policy, and ensures an HTTP/2 connection (TCP + TLS + HTTP/2 preface + SETTINGS exchange).
2. Client opens a new **HTTP/2 stream** and sends a **HEADERS** frame:

```
:method       POST
:scheme       https
:path         /orders.v1.OrderService/GetOrder
:authority    orders.internal:443
content-type  application/grpc+proto
te            trailers
grpc-timeout  2S                (deadline, if set)
grpc-encoding gzip              (if compressing)
authorization Bearer ...        (custom metadata)
user-agent    grpc-go/1.xx
```

3. Client sends one or more **DATA** frames holding **length-prefixed messages**:

```
 1 byte            4 bytes (big endian)       N bytes
┌───────────────┬─────────────────────────┬──────────────────┐
│ compressed? 0/1│ message length          │ protobuf payload │
└───────────────┴─────────────────────────┴──────────────────┘
```

4. Client half-closes (sets END_STREAM on its last DATA frame).
5. Server replies with **HEADERS** (initial metadata), **DATA** (response message(s)), and finally a trailing **HEADERS** frame (the *trailers*) carrying `grpc-status`, `grpc-message` and optional `grpc-status-details-bin`, with END_STREAM.

> **Trailers-Only response**: if the server fails before sending any message, it can send a single HEADERS frame containing both the headers and `grpc-status`, with END_STREAM.

**Why this matters in interviews:** it explains why proxies must support HTTP/2 **trailers**, why `grpc-status` rather than the HTTP status carries the real outcome (HTTP status is almost always 200), and why the 5-byte frame prefix means each message must be fully buffered before decoding (hence message size limits).

---

# Part 2 — HTTP/2 Under the Hood

## Q2.1 — Explain multiplexing, and what "head-of-line blocking" still exists with gRPC.

**Answer.**

HTTP/2 splits a connection into independent **streams**, each carrying one RPC. Frames from different streams are interleaved on a single TCP connection, so many concurrent RPCs share it without opening new sockets.

What it solves: **HTTP-level HOL blocking** (HTTP/1.1 needed a response before the next on that connection).

What it does **not** solve: **TCP-level HOL blocking**. One lost packet stalls *all* streams on that connection until retransmission completes. This is the key motivation for HTTP/3/QUIC, where streams are independent at the transport layer. In lossy networks (mobile, cross-region links), a single large HTTP/2 connection can underperform several connections.

**Practical consequences:**

- Max concurrent streams per connection is negotiated (`SETTINGS_MAX_CONCURRENT_STREAMS`). Once reached, new RPCs queue on the client. For very high QPS to one backend you may need a **pool of connections (channels)**.
- A single slow, huge stream can monopolize bandwidth unless flow control is tuned.

## Q2.2 — How does HTTP/2 flow control work, and how does it relate to gRPC backpressure?

**Answer.**

HTTP/2 has **two levels** of credit-based flow control: per-stream and per-connection. A sender may only send DATA up to the receiver's advertised window. The receiver replenishes credit with `WINDOW_UPDATE` frames **as the application consumes data**.

In gRPC this becomes natural **backpressure**:

- If a receiver's application stops calling `Recv()`, its window drains and stops advertising credit.
- The sender's `Send()` call **blocks** once the window and internal buffers are full.
- Therefore a slow consumer slows a fast producer instead of exhausting memory, *as long as you actually honor it* (don't spawn unbounded goroutines or queues between `Recv` and processing).

The default initial window is 64 KB, which hurts high-bandwidth-delay-product links. grpc-go has a **BDP estimator** that dynamically grows the window (disabled if you set a static window explicitly with `grpc.InitialWindowSize` or `grpc.InitialConnWindowSize`), so be careful: manually setting windows can *reduce* throughput versus dynamic estimation.

## Q2.3 — What is HPACK and why does it matter for gRPC?

**Answer.**

HPACK compresses HTTP/2 headers using a dynamic table shared per connection. gRPC repeats the same headers on every call (`:path`, `content-type`, `user-agent`, `te`, auth metadata). After the first RPC, they are replaced by tiny table indexes.

Implications:

- Large or **high-cardinality metadata** (e.g., a different JWT per call) defeats the dynamic table and bloats frames.
- Header size is bounded (`grpc.MaxHeaderListSize`, default 16 MB in grpc-go on receive but many proxies enforce 8–64 KB). Oversized headers produce confusing errors from proxies (`RESOURCE_EXHAUSTED`/`431`/`UNAVAILABLE`). Keep metadata small.

## Q2.4 — Why does gRPC require HTTP/2 end to end, and what breaks behind a proxy?

**Answer.**

Native gRPC needs: HTTP/2, trailers, full-duplex streaming, and unbuffered streaming responses. Typical breakages:

| Symptom | Likely cause |
|---|---|
| `UNAVAILABLE` / `RST_STREAM` with `PROTOCOL_ERROR` | Proxy downgraded to HTTP/1.1 |
| Missing `grpc-status` ("server closed the stream without sending trailers") | Proxy strips trailers or buffers/terminates the response |
| Streaming works only after completion | Proxy buffering responses (disable proxy buffering) |
| Streams die after N seconds of idleness | Proxy idle timeout shorter than stream lifetime |
| Random `GOAWAY` | Proxy `max_connection_duration` or LB draining |

---

# Part 3 — Protocol Buffers Deep Dive

## Q3.1 — Explain the protobuf wire format. Why are field numbers 1–15 special?

**Answer.**

A message is a sequence of `(tag, value)` pairs. The **tag** is a varint: `(field_number << 3) | wire_type`.

| Wire type | Value | Used for |
|---|---|---|
| 0 VARINT | variable-length int | `int32/64, uint32/64, bool, enum, sint*` |
| 1 I64 | 8 bytes | `fixed64, sfixed64, double` |
| 2 LEN | length-prefixed | `string, bytes, embedded messages, packed repeated` |
| 5 I32 | 4 bytes | `fixed32, sfixed32, float` |

Example: `int32 id = 1; string name = 2;` with `id=150, name="ab"` encodes as:

```
08 96 01      tag(1,VARINT)=0x08, varint(150)=0x96 0x01
12 02 61 62   tag(2,LEN)=0x12, len=2, "ab"
```

**Field numbers 1–15 take a single tag byte; 16–2047 take two.** Reserve 1–15 for hot, frequently-populated fields.

**Other consequences interviewers like:**

- **Negative `int32/int64` values take 10 bytes** as varints (sign-extended). Use `sint32/sint64` (ZigZag encoding) for values that are often negative.
- **Default values are not serialized in proto3** (non-`optional` scalars). That's why you cannot distinguish "unset" from "zero" without `optional`, wrappers, or a presence-tracking type.
- **Unknown fields are preserved** (since proto3 3.5) and re-serialized, which is what makes forward compatibility work through middleware.
- **Encoding is not canonical/deterministic by default.** Never use serialized bytes as a hash/cache key unless you use deterministic marshal options and accept they are not stable across languages/versions.

## Q3.2 — What are the rules for backward- and forward-compatible schema evolution?

**Answer.**

**Safe changes:**

- Add new fields with **new, never-reused numbers**.
- Remove a field **and `reserve` its number and name**.
- Rename a field (wire format uses numbers; but JSON mapping uses names, so check JSON consumers/gateways).
- Add new enum values (consumers must handle unknown values).
- Add new RPCs/services.
- Change `int32 ↔ int64 ↔ uint32 ↔ uint64 ↔ bool` in compatible ways (with truncation risks); `sint32 ↔ sint64`; `fixed32 ↔ sfixed32`.

**Breaking changes:**

- Reusing/changing a field number or changing its wire-type-incompatible type (`string` → `int32`).
- Changing `repeated` ↔ singular carelessly (packed vs non-packed edge cases).
- Moving a field into/out of a `oneof` (semantics change) — moving into an existing oneof is breaking.
- Renaming package/service/method (changes `:path`).
- Making semantics stricter (a field that was optional now required in validation).

```proto
message Order {
  reserved 4, 7;
  reserved "legacy_status", "old_total";
  string id = 1;
  int64  total_cents = 2;
  // 4 was legacy_status — never reuse
}
```

Enforce in CI with **`buf breaking --against '.git#branch=main'`**. A staff engineer owns the policy: schema review, breaking-change gating, and deprecation windows.

## Q3.3 — `optional`, wrappers, `oneof`, `FieldMask`: how do you model "unset vs zero" and partial updates?

**Answer.**

```proto
import "google/protobuf/field_mask.proto";
import "google/protobuf/wrappers.proto";
import "google/protobuf/timestamp.proto";

message Product {
  string id = 1;
  optional int32 discount_percent = 2;      // explicit presence: nil vs 0
  google.protobuf.StringValue note = 3;      // legacy wrapper presence
  oneof pricing {
    int64 fixed_cents = 4;
    PercentPrice percent = 5;
  }
  google.protobuf.Timestamp updated_at = 6;
}

message UpdateProductRequest {
  Product product = 1;
  google.protobuf.FieldMask update_mask = 2; // which fields to apply
}
```

- **`optional`** (proto3 since 3.15 GA) gives hasField semantics: generated Go uses pointers (`*int32`).
- **Wrappers** (`Int32Value` etc.) are older; message-typed so presence is native.
- **`oneof`** guarantees at most one field set and saves memory; setting one clears the others. Great for polymorphic payloads, but adding fields to a `oneof` is safe while *moving existing fields into one* is not.
- **`FieldMask`** is the standard for PATCH-style updates. Without it, "send the full object" updates silently zero fields that older clients don't know about (a classic data-loss bug during rolling upgrades).

```go
func (s *Server) UpdateProduct(ctx context.Context, req *pb.UpdateProductRequest) (*pb.Product, error) {
    existing, err := s.repo.Get(ctx, req.GetProduct().GetId())
    if err != nil { return nil, status.Error(codes.NotFound, "product not found") }

    if req.UpdateMask == nil || len(req.UpdateMask.Paths) == 0 {
        return nil, status.Error(codes.InvalidArgument, "update_mask required")
    }
    // Validate and normalize paths against the message descriptor
    if !fieldmask_utils.IsValid(existing, req.UpdateMask) { // e.g. google.golang.org/protobuf/types/known/fieldmaskpb: req.UpdateMask.IsValid(existing)
        return nil, status.Error(codes.InvalidArgument, "invalid update_mask path")
    }
    applyMask(existing, req.Product, req.UpdateMask)
    return s.repo.Save(ctx, existing)
}
```

(`fieldmaskpb.FieldMask.IsValid(msg)` validates paths against the message; hand-rolling the merge or using a library is typical.)

## Q3.4 — How do you handle `Any`, `Struct`, `map`, and when should you avoid them?

**Answer.**

- **`google.protobuf.Any`** carries a serialized message plus a type URL. Useful for extensible envelopes (error details, event payloads) but loses compile-time safety, creates a dependency on type registries and complicates schema governance. Use in infrastructure envelopes, not business APIs.
- **`Struct`/`Value`** is JSON-in-proto. Fine for genuinely dynamic metadata; avoid for core domain data (no validation, bigger, slower).
- **`map<K,V>`**: unordered; iteration order is not guaranteed; keys limited to integral/string types; wire-encoded as repeated entries (so duplicates resolve last-wins).
- **Large `bytes` blobs** inside messages should instead be **streamed in chunks** or uploaded to object storage with pre-signed URLs.

## Q3.5 — Message size limits and large payloads — what are the defaults and what is the right design?

**Answer.**

- grpc-go defaults: **max receive message = 4 MB**, max send = effectively unlimited (`math.MaxInt32`). Exceeding the receive limit yields `RESOURCE_EXHAUSTED`.
- Raising the limit is usually a smell. The whole message (5-byte header plus payload) is buffered and decoded in memory; large messages cause GC pressure, tail latency, and head-of-line effects on the shared connection.

**Better designs:** client-streaming upload in 64 KB–1 MB chunks, server-streaming download, pagination for lists, or signed URLs for blobs with only metadata in gRPC.

```go
srv := grpc.NewServer(
    grpc.MaxRecvMsgSize(16<<20), // only if justified
    grpc.MaxSendMsgSize(16<<20),
)
```

---

# Part 4 — The Four RPC Types in Go

## The contract

```proto
syntax = "proto3";

package orders.v1;
option go_package = "github.com/acme/orders/gen/orders/v1;ordersv1";

import "google/protobuf/timestamp.proto";

service OrderService {
  // 1. Unary
  rpc GetOrder(GetOrderRequest) returns (Order);
  // 2. Server streaming
  rpc WatchOrders(WatchOrdersRequest) returns (stream OrderEvent);
  // 3. Client streaming
  rpc UploadItems(stream UploadItemsRequest) returns (UploadItemsResponse);
  // 4. Bidirectional streaming
  rpc Negotiate(stream NegotiateMessage) returns (stream NegotiateMessage);
}

message GetOrderRequest { string order_id = 1; }
message Order {
  string id = 1;
  string customer_id = 2;
  int64  total_cents = 3;
  google.protobuf.Timestamp created_at = 4;
}
message WatchOrdersRequest { string customer_id = 1; }
message OrderEvent { string order_id = 1; string type = 2; google.protobuf.Timestamp at = 3; }
message UploadItemsRequest { string order_id = 1; bytes chunk = 2; }
message UploadItemsResponse { int64 bytes_received = 1; }
message NegotiateMessage { string text = 1; }
```

Generate:

```bash
# buf.gen.yaml approach (recommended)
buf generate
# or protoc:
protoc --go_out=. --go_opt=paths=source_relative \
       --go-grpc_out=. --go-grpc_opt=paths=source_relative \
       orders/v1/orders.proto
```

## Q4.1 — Server skeleton: what does a production-grade gRPC server in Go look like?

**Answer.**

```go
package main

import (
    "context"
    "log/slog"
    "net"
    "os"
    "os/signal"
    "syscall"
    "time"

    "google.golang.org/grpc"
    "google.golang.org/grpc/health"
    healthpb "google.golang.org/grpc/health/grpc_health_v1"
    "google.golang.org/grpc/keepalive"
    "google.golang.org/grpc/reflection"

    ordersv1 "github.com/acme/orders/gen/orders/v1"
)

type orderServer struct {
    ordersv1.UnimplementedOrderServiceServer // forward compat: new RPCs won't break the build
    repo Repo
}

func main() {
    lis, err := net.Listen("tcp", ":50051")
    if err != nil { slog.Error("listen", "err", err); os.Exit(1) }

    srv := grpc.NewServer(
        grpc.ChainUnaryInterceptor(recoveryUnary, loggingUnary, authUnary),
        grpc.ChainStreamInterceptor(recoveryStream, loggingStream, authStream),
        grpc.KeepaliveParams(keepalive.ServerParameters{
            MaxConnectionIdle:     5 * time.Minute,
            MaxConnectionAge:      30 * time.Minute, // forces periodic reconnect => rebalancing
            MaxConnectionAgeGrace: 30 * time.Second,
            Time:                  2 * time.Minute,  // server pings idle clients
            Timeout:               20 * time.Second,
        }),
        grpc.KeepaliveEnforcementPolicy(keepalive.EnforcementPolicy{
            MinTime:             30 * time.Second, // reject clients pinging faster
            PermitWithoutStream: true,
        }),
        grpc.MaxConcurrentStreams(1000),
    )

    ordersv1.RegisterOrderServiceServer(srv, &orderServer{repo: newRepo()})

    hs := health.NewServer()
    healthpb.RegisterHealthServer(srv, hs)
    hs.SetServingStatus("", healthpb.HealthCheckResponse_SERVING)
    reflection.Register(srv) // disable/guard in prod if exposing externally

    go func() {
        if err := srv.Serve(lis); err != nil { slog.Error("serve", "err", err) }
    }()

    // Graceful shutdown
    sig := make(chan os.Signal, 1)
    signal.Notify(sig, syscall.SIGINT, syscall.SIGTERM)
    <-sig
    hs.Shutdown() // flips all services to NOT_SERVING so LBs drain us
    done := make(chan struct{})
    go func() { srv.GracefulStop(); close(done) }()
    select {
    case <-done:
    case <-time.After(30 * time.Second):
        srv.Stop() // force after deadline: long-lived streams can block GracefulStop forever
    }
}
```

**Interview talking points embedded here:**

- Embedding `Unimplemented...Server` is *required by default* in newer `protoc-gen-go-grpc` and gives forward compatibility (new methods return `UNIMPLEMENTED`).
- `MaxConnectionAge` is the lever for rebalancing with L4 load balancers/DNS-based discovery (see Part 10).
- `GracefulStop` waits for **all** RPCs, including infinite streams; always have a fallback `Stop()`.

## Q4.2 — Unary RPC: implement it correctly with context, errors and validation.

**Answer.**

```go
func (s *orderServer) GetOrder(ctx context.Context, req *ordersv1.GetOrderRequest) (*ordersv1.Order, error) {
    if req.GetOrderId() == "" {
        return nil, status.Error(codes.InvalidArgument, "order_id is required")
    }
    // Respect the caller's deadline: pass ctx into every I/O call.
    o, err := s.repo.Find(ctx, req.OrderId)
    switch {
    case errors.Is(err, ErrNotFound):
        return nil, status.Errorf(codes.NotFound, "order %q not found", req.OrderId)
    case errors.Is(err, context.DeadlineExceeded):
        return nil, status.FromContextError(err).Err() // DEADLINE_EXCEEDED
    case errors.Is(err, context.Canceled):
        return nil, status.FromContextError(err).Err() // CANCELLED
    case err != nil:
        slog.ErrorContext(ctx, "repo failure", "err", err) // log details server-side
        return nil, status.Error(codes.Internal, "internal error") // don't leak internals
    }
    return toProto(o), nil
}
```

Client:

```go
conn, err := grpc.NewClient("dns:///orders.internal:443",
    grpc.WithTransportCredentials(credentials.NewTLS(tlsCfg)))
if err != nil { return err }
defer conn.Close()
client := ordersv1.NewOrderServiceClient(conn)

ctx, cancel := context.WithTimeout(ctx, 800*time.Millisecond) // ALWAYS set a deadline
defer cancel()
o, err := client.GetOrder(ctx, &ordersv1.GetOrderRequest{OrderId: "o-123"})
if err != nil {
    st, _ := status.FromError(err)
    switch st.Code() {
    case codes.NotFound: // handle
    case codes.DeadlineExceeded, codes.Unavailable: // maybe retry/backoff
    }
}
```

## Q4.3 — Server streaming: implement a watch/subscribe correctly, including cancellation and slow consumers.

**Answer.**

```go
func (s *orderServer) WatchOrders(req *ordersv1.WatchOrdersRequest, stream ordersv1.OrderService_WatchOrdersServer) error {
    ctx := stream.Context()
    events, unsubscribe := s.bus.Subscribe(req.GetCustomerId()) // returns <-chan *OrderEvent (buffered)
    defer unsubscribe()

    for {
        select {
        case <-ctx.Done():
            return status.FromContextError(ctx.Err()).Err() // client left / deadline hit / server stopping
        case ev, ok := <-events:
            if !ok {
                return status.Error(codes.Aborted, "subscription closed (slow consumer or shutdown)")
            }
            // Send blocks when HTTP/2 flow control window is full => natural backpressure.
            if err := stream.Send(ev); err != nil {
                return err // transport broken; stop producing
            }
        }
    }
}
```

**Design points:**

- The **handler returning ends the stream**, and the returned error becomes the trailers' status. Returning `nil` yields `OK`.
- **Always select on `ctx.Done()`**; otherwise goroutines and subscriptions leak when clients disappear.
- **Bounded per-subscriber buffers** with an explicit policy (drop oldest / disconnect slow consumer). Unbounded buffering is a memory bomb when one subscriber is slow.
- Clients should **resubscribe with a resume token/offset** because streams die routinely (deploys, `MaxConnectionAge`, LB). Streams are *not* durable; design for at-least-once with idempotent consumers.

Client side:

```go
stream, err := client.WatchOrders(ctx, &ordersv1.WatchOrdersRequest{CustomerId: "c1"})
if err != nil { return err }
for {
    ev, err := stream.Recv()
    if err == io.EOF { return nil }          // server finished with OK
    if err != nil  { return err }            // status error or transport failure
    handle(ev)
}
```

## Q4.4 — Client streaming: chunked upload with checksum and proper error flow.

**Answer.**

Server:

```go
func (s *orderServer) UploadItems(stream ordersv1.OrderService_UploadItemsServer) error {
    var total int64
    h := sha256.New()
    for {
        msg, err := stream.Recv()
        if err == io.EOF { // client called CloseSend
            slog.Info("upload done", "sha256", hex.EncodeToString(h.Sum(nil)))
            return stream.SendAndClose(&ordersv1.UploadItemsResponse{BytesReceived: total})
        }
        if err != nil { return err }
        total += int64(len(msg.Chunk))
        h.Write(msg.Chunk)
        if total > 1<<30 {
            return status.Error(codes.ResourceExhausted, "upload exceeds 1 GiB")
        }
    }
}
```

Client:

```go
stream, err := client.UploadItems(ctx)
if err != nil { return err }
buf := make([]byte, 256<<10)
for {
    n, rerr := file.Read(buf)
    if n > 0 {
        if err := stream.Send(&ordersv1.UploadItemsRequest{OrderId: id, Chunk: buf[:n]}); err != nil {
            // IMPORTANT: Send returns io.EOF when the stream was terminated by the server;
            // the real status is only available from CloseAndRecv / RecvMsg.
            if err == io.EOF { _, err = stream.CloseAndRecv() }
            return err
        }
    }
    if rerr == io.EOF { break }
    if rerr != nil { return rerr }
}
resp, err := stream.CloseAndRecv()
```

> **Gotcha:** `Send` returning `io.EOF` is *not* the real error. You must call `RecvMsg`/`CloseAndRecv` to obtain the status. Interviewers love this one.

## Q4.5 — Bidirectional streaming: how do you structure concurrency safely?

**Answer.**

Rules for gRPC streams in Go:

- It is safe to call `Send` from one goroutine and `Recv` from another concurrently.
- It is **not safe** to call `Send` (or `SendMsg`) from multiple goroutines concurrently on the same stream. Serialize with a mutex or a single writer goroutine.
- Same for `Recv`.

```go
func (s *orderServer) Negotiate(stream ordersv1.OrderService_NegotiateServer) error {
    g, ctx := errgroup.WithContext(stream.Context())
    out := make(chan *ordersv1.NegotiateMessage, 16) // bounded => backpressure to producers

    // Reader
    g.Go(func() error {
        for {
            in, err := stream.Recv()
            if err == io.EOF { close(out); return nil }
            if err != nil { return err }
            select {
            case out <- process(in):
            case <-ctx.Done():
                return ctx.Err()
            }
        }
    })
    // Single writer
    g.Go(func() error {
        for {
            select {
            case m, ok := <-out:
                if !ok { return nil }
                if err := stream.Send(m); err != nil { return err }
            case <-ctx.Done():
                return ctx.Err()
            }
        }
    })
    return g.Wait()
}
```

**Deadlock trap:** if the client only sends and never reads (or vice versa), flow-control windows fill and both sides block. Both sides must read and write concurrently.

## Q4.6 — How do you choose between unary, streaming, and polling/long-polling?

**Answer.**

| Need | Choice | Reason |
|---|---|---|
| Request/response, independent calls | Unary | Simplest; retries, LB per call, easy observability |
| Large result sets, ordered | Server streaming | Avoids huge messages; incremental processing |
| Large upload / aggregation | Client streaming | Chunking, backpressure, early abort |
| Live updates, subscriptions | Server streaming | Push without polling; reconnect with resume token |
| Interactive, full duplex (chat, collaborative) | Bidi | Both directions concurrently |
| Many tiny independent requests | Prefer **unary**, not a long stream | Streams pin to one backend, defeat per-RPC balancing, complicate retries |

**Staff-level insight:** *streams are sticky.* A long-lived stream is routed once and stays on that backend for hours. This creates hot spots, complicates rolling deploys, and prevents per-request retry/hedging. Use streams when the semantics truly demand them; otherwise unary calls on a balanced channel are operationally cheaper.

---

# Part 5 — Status Codes and the Error Model

## Q5.1 — List gRPC status codes and when to use each. Which are commonly misused?

**Answer.**

| Code | # | Meaning / when to use | Retry? |
|---|---|---|---|
| OK | 0 | Success | – |
| CANCELLED | 1 | Caller cancelled | No |
| UNKNOWN | 2 | Unmapped error | Rarely |
| INVALID_ARGUMENT | 3 | Bad input regardless of system state | No |
| DEADLINE_EXCEEDED | 4 | Deadline passed (outcome unknown) | Only if idempotent |
| NOT_FOUND | 5 | Entity doesn't exist | No |
| ALREADY_EXISTS | 6 | Create conflict | No |
| PERMISSION_DENIED | 7 | Authenticated but not allowed | No |
| RESOURCE_EXHAUSTED | 8 | Quota/rate limit/message too large | Maybe with backoff |
| FAILED_PRECONDITION | 9 | System not in required state (e.g., non-empty dir) | No, fix state first |
| ABORTED | 10 | Concurrency conflict (txn aborted, CAS fail) | Yes, at higher level |
| OUT_OF_RANGE | 11 | Past valid range (e.g., read beyond EOF) | No |
| UNIMPLEMENTED | 12 | Method not supported | No |
| INTERNAL | 13 | Invariant broken / server bug | Rarely |
| UNAVAILABLE | 14 | Transient; service unavailable | **Yes with backoff** |
| DATA_LOSS | 15 | Unrecoverable corruption | No |
| UNAUTHENTICATED | 16 | Missing/invalid credentials | No (refresh creds) |

**Commonly confused pairs:**

- `INVALID_ARGUMENT` vs `FAILED_PRECONDITION` vs `OUT_OF_RANGE`: argument bad *always* / state wrong *now* / argument bad *depending on state*.
- `UNAUTHENTICATED` (who are you?) vs `PERMISSION_DENIED` (you can't do this).
- `UNAVAILABLE` (safe to retry) vs `INTERNAL`/`UNKNOWN` (don't blindly retry).
- `ABORTED` (retry the higher-level transaction) vs `UNAVAILABLE` (retry the same call).
- Never return `INTERNAL` for validation errors, and never return `UNKNOWN` for errors you could classify.

**Propagation trap:** when service A calls B and B returns `NOT_FOUND`, A shouldn't blindly return the same code to its caller — a missing *downstream* resource is usually an `INTERNAL`/`FAILED_PRECONDITION` from A's caller's perspective. Translate codes at service boundaries.

## Q5.2 — How do you return rich, structured errors (field-level validation, retry info)?

**Answer.**

Use the **richer error model** (`google.rpc.Status` with `details` in `grpc-status-details-bin`):

```go
import (
    "google.golang.org/genproto/googleapis/rpc/errdetails"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
    "google.golang.org/protobuf/types/known/durationpb"
)

func validationErr(field, desc string) error {
    st := status.New(codes.InvalidArgument, "invalid request")
    st, err := st.WithDetails(&errdetails.BadRequest{
        FieldViolations: []*errdetails.BadRequest_FieldViolation{
            {Field: field, Description: desc},
        },
    })
    if err != nil { return status.Error(codes.Internal, "failed to attach details") }
    return st.Err()
}

func rateLimited(retryAfter time.Duration) error {
    st := status.New(codes.ResourceExhausted, "rate limit exceeded")
    st, _ = st.WithDetails(&errdetails.RetryInfo{RetryDelay: durationpb.New(retryAfter)})
    return st.Err()
}
```

Client:

```go
st, _ := status.FromError(err)
for _, d := range st.Details() {
    switch t := d.(type) {
    case *errdetails.BadRequest:
        for _, v := range t.FieldViolations { fmt.Println(v.Field, v.Description) }
    case *errdetails.RetryInfo:
        time.Sleep(t.RetryDelay.AsDuration())
    }
}
```

Standard detail types: `BadRequest`, `ErrorInfo` (stable machine-readable `reason` + `domain` + metadata — **prefer this for programmatic handling**), `RetryInfo`, `QuotaFailure`, `PreconditionFailure`, `ResourceInfo`, `DebugInfo` (never expose in prod externally), `LocalizedMessage`.

**Why not put an error enum in the response message?** Mixing success payload and errors forces clients to check two channels, defeats interceptors/metrics keyed on status code, and breaks retry policies which key on status codes. Use **status for failure, message for success.**

## Q5.3 — How do you map gRPC errors to HTTP for gateways?

**Answer.**

Canonical mapping (used by grpc-gateway and Google APIs): `INVALID_ARGUMENT/FAILED_PRECONDITION/OUT_OF_RANGE → 400`, `UNAUTHENTICATED → 401`, `PERMISSION_DENIED → 403`, `NOT_FOUND → 404`, `ALREADY_EXISTS/ABORTED → 409`, `RESOURCE_EXHAUSTED → 429`, `CANCELLED → 499`, `UNIMPLEMENTED → 501`, `INTERNAL/UNKNOWN/DATA_LOSS → 500`, `UNAVAILABLE → 503`, `DEADLINE_EXCEEDED → 504`.

---

# Part 6 — Deadlines, Cancellation and Propagation

## Q6.1 — Why are deadlines mandatory in production gRPC, and how do they differ from timeouts?

**Answer.**

gRPC has **no default deadline**: a call without one can wait forever, pinning goroutines, memory and streams. Under partial failure (slow dependency), unbounded waits cause **cascading failure**: threads/goroutines pile up, latency grows, upstream retries multiply load, and the system collapses.

A **deadline is an absolute point in time**; a timeout is a relative duration. The client converts it to a `grpc-timeout` header (remaining time at send). Each hop **re-derives** its own deadline from the received value, so the budget shrinks naturally along the call chain. In Go, the server handler's `ctx` already carries the deadline from `grpc-timeout`.

```
Client (budget 1000ms) ─► A (sees ~990ms) ─► B (sees ~900ms) ─► DB (sees ~850ms)
```

Rules:

1. **Always set deadlines on the client**, at the edge, based on SLO.
2. **Always pass `ctx` through** to downstream calls and DB queries — never `context.Background()` inside handlers.
3. **Reserve budget**: when making sequential calls, divide remaining time; don't give each downstream the full remaining deadline.
4. Servers should **check `ctx.Err()` before expensive work** and fail fast with `DEADLINE_EXCEEDED` instead of computing results nobody will read.

```go
func (s *server) Checkout(ctx context.Context, r *pb.CheckoutRequest) (*pb.CheckoutResponse, error) {
    dl, ok := ctx.Deadline()
    if ok && time.Until(dl) < 50*time.Millisecond {
        return nil, status.Error(codes.DeadlineExceeded, "insufficient time budget")
    }
    // sub-budget for the downstream call: leave 20% headroom for our own work
    sub, cancel := context.WithTimeout(ctx, time.Until(dl)*8/10)
    defer cancel()
    inv, err := s.inventory.Reserve(sub, &pb.ReserveRequest{...})
    ...
}
```

## Q6.2 — What does `DEADLINE_EXCEEDED` actually tell you? Why is it dangerous with retries?

**Answer.**

It tells you the **client gave up**, not that the server didn't do the work. The server may have completed the operation (a payment may be charged!) after the client stopped waiting. So:

- Retrying `DEADLINE_EXCEEDED` is safe **only for idempotent operations** or when you carry an **idempotency key**.
- If the server ignores cancellation, it keeps burning resources for a dead caller — wasted work during overload makes things worse.

```go
// Idempotency key from client (e.g., UUID per logical operation, reused across retries)
ctx = metadata.AppendToOutgoingContext(ctx, "idempotency-key", opID)
// Server: dedupe by key with a TTL store (Redis/DB unique constraint) and return the stored result on replay.
```

## Q6.3 — Cancellation: what happens when the client cancels?

**Answer.**

The client sends `RST_STREAM(CANCEL)`. On the server, the handler's `ctx` is cancelled (`ctx.Err() == context.Canceled`). The server must observe this to stop work. gRPC does **not** forcibly kill your goroutine; cooperative cancellation is on you. In streaming handlers, pending `Send`/`Recv` calls return errors once the stream is done.

Server-side cancellation of in-flight DB queries works only if you use `QueryContext`/`ExecContext` and your driver supports it.

---

# Part 7 — Interceptors and Metadata

## Q7.1 — What are interceptors? Show the four kinds and how to chain them.

**Answer.**

Interceptors are gRPC's middleware: cross-cutting concerns (auth, logging, metrics, tracing, recovery, rate limiting, validation) without touching handlers.

| Side | Unary | Stream |
|---|---|---|
| Server | `grpc.UnaryServerInterceptor` | `grpc.StreamServerInterceptor` |
| Client | `grpc.UnaryClientInterceptor` | `grpc.StreamClientInterceptor` |

Server unary:

```go
func loggingUnary(ctx context.Context, req any, info *grpc.UnaryServerInfo, h grpc.UnaryHandler) (any, error) {
    start := time.Now()
    resp, err := h(ctx, req)
    slog.InfoContext(ctx, "rpc",
        "method", info.FullMethod,
        "code", status.Code(err).String(),
        "dur_ms", time.Since(start).Milliseconds())
    return resp, err
}

func recoveryUnary(ctx context.Context, req any, info *grpc.UnaryServerInfo, h grpc.UnaryHandler) (resp any, err error) {
    defer func() {
        if r := recover(); r != nil {
            slog.ErrorContext(ctx, "panic", "method", info.FullMethod, "panic", r, "stack", string(debug.Stack()))
            err = status.Error(codes.Internal, "internal error")
        }
    }()
    return h(ctx, req)
}
```

Stream interceptors need a **wrapper around `ServerStream`** to intercept `SendMsg/RecvMsg` or to replace the context:

```go
type wrappedStream struct {
    grpc.ServerStream
    ctx context.Context
}
func (w *wrappedStream) Context() context.Context { return w.ctx }

func authStream(srv any, ss grpc.ServerStream, info *grpc.StreamServerInfo, h grpc.StreamHandler) error {
    ctx, err := authenticate(ss.Context())
    if err != nil { return err }
    return h(srv, &wrappedStream{ServerStream: ss, ctx: ctx})
}
```

**Ordering matters.** `grpc.ChainUnaryInterceptor(a, b, c)` runs `a → b → c → handler → c → b → a`. Typical order: **recovery → tracing/metrics → logging → auth → rate-limit → validation → handler.** Putting recovery first ensures panics in later interceptors are caught; putting auth before rate limit avoids letting unauthenticated traffic consume per-tenant quotas (or the opposite — IP-based limiting *before* auth to protect the auth backend). Be able to justify your order.

> **Gotcha:** interceptors on streams wrap the *stream setup*, not each message. For per-message behavior (validation, size accounting, auditing), wrap `RecvMsg/SendMsg`.

## Q7.2 — Client interceptor example: propagate metadata, add retries/timeouts defaults.

**Answer.**

```go
func defaultDeadlineUnary(d time.Duration) grpc.UnaryClientInterceptor {
    return func(ctx context.Context, method string, req, reply any, cc *grpc.ClientConn,
        invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
        if _, ok := ctx.Deadline(); !ok { // enforce: no call without a deadline
            var cancel context.CancelFunc
            ctx, cancel = context.WithTimeout(ctx, d)
            defer cancel()
        }
        ctx = metadata.AppendToOutgoingContext(ctx, "x-request-id", requestIDFrom(ctx))
        return invoker(ctx, method, req, reply, cc, opts...)
    }
}

conn, _ := grpc.NewClient(target,
    grpc.WithChainUnaryInterceptor(defaultDeadlineUnary(2*time.Second), otelUnary()),
)
```

## Q7.3 — Metadata: headers vs trailers, binary values, what to put there.

**Answer.**

Metadata = key/value pairs sent as HTTP/2 headers/trailers. Keys are **lowercase**; keys ending in `-bin` carry binary values (base64 on the wire). Keys with the `grpc-` prefix are reserved. Never put big payloads there.

```go
// Client: send headers, receive header + trailer
ctx = metadata.AppendToOutgoingContext(ctx, "tenant-id", "acme")
var header, trailer metadata.MD
resp, err := client.GetOrder(ctx, req, grpc.Header(&header), grpc.Trailer(&trailer))

// Server: read incoming, set outgoing
md, _ := metadata.FromIncomingContext(ctx)
tenant := md.Get("tenant-id")
grpc.SetHeader(ctx, metadata.Pairs("server-region", "ap-south-1")) // sent before first message
grpc.SetTrailer(ctx, metadata.Pairs("cost-units", "12"))           // sent with status; good for post-hoc info
```

Headers arrive **before** the response body; trailers arrive **after** and can include data computed during processing (timing, cost, pagination hints). Use metadata for **cross-cutting concerns** (auth, tracing, tenant, request-id), the message for **business data**.

**Security:** don't trust inbound metadata for identity unless an authenticated interceptor derived it; strip/override internal headers at the edge to prevent spoofing (`x-user-id`, etc.).

## Q7.4 — Write a server-side load-shedding / concurrency-limiting interceptor.

**Answer.**

```go
func limitConcurrency(max int) grpc.UnaryServerInterceptor {
    sem := make(chan struct{}, max)
    return func(ctx context.Context, req any, info *grpc.UnaryServerInfo, h grpc.UnaryHandler) (any, error) {
        select {
        case sem <- struct{}{}:
            defer func() { <-sem }()
            return h(ctx, req)
        default:
            // Fail fast: better a quick UNAVAILABLE/RESOURCE_EXHAUSTED than a slow timeout.
            return nil, status.Error(codes.ResourceExhausted, "server overloaded")
        }
    }
}
```

Staff-level comment: **fail fast and shed load** rather than queueing. Queues convert overload into latency, which causes timeouts and retries, which amplify overload (*metastable failure*). Pair with adaptive concurrency limits (Little's law / gradient-based limiters), per-tenant quotas, and make sure `RESOURCE_EXHAUSTED` vs `UNAVAILABLE` semantics match your client's retry policy (retrying shed requests blindly doubles pressure — use retry budgets/throttling).

---

# Part 8 — Security: TLS, mTLS, Auth

## Q8.1 — How do you secure gRPC? Explain TLS vs mTLS and per-RPC credentials.

**Answer.**

Two layers:

1. **Transport security** (`credentials.TransportCredentials`): TLS for encryption + server authentication; **mTLS** adds client certificates for service identity.
2. **Per-RPC credentials** (`credentials.PerRPCCredentials`): attach tokens (JWT/OAuth2) to each call as metadata. These *require* a secure channel by default (`RequireTransportSecurity() == true`).

Server mTLS:

```go
cert, _ := tls.LoadX509KeyPair("server.crt", "server.key")
caPool := x509.NewCertPool()
caPool.AppendCertsFromPEM(mustRead("ca.crt"))

creds := credentials.NewTLS(&tls.Config{
    Certificates: []tls.Certificate{cert},
    ClientAuth:   tls.RequireAndVerifyClientCert,
    ClientCAs:    caPool,
    MinVersion:   tls.VersionTLS13,
})
srv := grpc.NewServer(grpc.Creds(creds))
```

Client with token per RPC:

```go
type tokenCreds struct{ src oauth2.TokenSource }
func (t tokenCreds) GetRequestMetadata(ctx context.Context, uri ...string) (map[string]string, error) {
    tok, err := t.src.Token() // cached/refreshed
    if err != nil { return nil, err }
    return map[string]string{"authorization": "Bearer " + tok.AccessToken}, nil
}
func (tokenCreds) RequireTransportSecurity() bool { return true }

conn, _ := grpc.NewClient(target,
    grpc.WithTransportCredentials(credentials.NewTLS(clientTLS)),
    grpc.WithPerRPCCredentials(tokenCreds{src}),
)
```

Extract the peer identity in an interceptor:

```go
func authUnary(ctx context.Context, req any, info *grpc.UnaryServerInfo, h grpc.UnaryHandler) (any, error) {
    // 1. Service identity via mTLS
    if p, ok := peer.FromContext(ctx); ok {
        if ti, ok := p.AuthInfo.(credentials.TLSInfo); ok && len(ti.State.VerifiedChains) > 0 {
            spiffeOrCN := ti.State.PeerCertificates[0].Subject.CommonName
            ctx = context.WithValue(ctx, svcIdentityKey{}, spiffeOrCN)
        }
    }
    // 2. User identity via JWT in metadata
    md, _ := metadata.FromIncomingContext(ctx)
    vals := md.Get("authorization")
    if len(vals) == 0 { return nil, status.Error(codes.Unauthenticated, "missing token") }
    claims, err := verifyJWT(strings.TrimPrefix(vals[0], "Bearer "))
    if err != nil { return nil, status.Error(codes.Unauthenticated, "invalid token") }
    return h(context.WithValue(ctx, userKey{}, claims), req)
}
```

**Staff-level topics:**

- **Service mesh** (Istio/Linkerd) can terminate mTLS in sidecars; app code sees plaintext. Decide *where identity lives* (SPIFFE IDs) and be explicit about trust boundaries.
- **Certificate rotation**: use `tls.Config.GetCertificate`/`GetClientCertificate` or file watchers (e.g., `credentials/tls/certprovider`) to reload without restarts; short-lived certs (hours) reduce revocation needs.
- **Authorization** (who can call which method) is separate from authentication: keep a method→permission map keyed on `info.FullMethod`, or use grpc-go's `authz` package / OPA.
- **Reflection** exposes your entire API surface; disable or authenticate it on production edges.
- **Token passthrough vs token exchange**: forwarding a user's JWT through the call chain risks confused-deputy attacks; prefer token exchange / downscoped service tokens with an `on-behalf-of` claim.

## Q8.2 — `ServerName`, `:authority` and certificate verification pitfalls?

**Answer.**

TLS verifies the cert against the **target host** (or `grpc.WithAuthority`/`tls.Config.ServerName`). When connecting by IP, via a proxy, or through a Kubernetes service name that differs from the SAN, verification fails with "x509: certificate is valid for X, not Y". Fix by issuing certs with correct SANs (including `*.svc.cluster.local` forms), or overriding `ServerName` — **never** use `InsecureSkipVerify` in production.

---

# Part 9 — Retries, Hedging and Resilience

## Q9.1 — How do gRPC retries work? Explain transparent retries, retry policy and throttling.

**Answer.**

Three distinct mechanisms:

1. **Transparent retry**: if the RPC never left the client (e.g., connection was already dead) or the server never began processing (stream refused/GOAWAY with lower last-stream-id), the library retries automatically and invisibly, as it's guaranteed safe. No config.
2. **Retry policy** (service config): configurable retries on specified status codes with exponential backoff and jitter. Enabled by default in modern grpc-go when a service config defines it.
3. **Hedging**: send the same request to multiple backends with a delay, take the first response, cancel the rest. Reduces tail latency; only for idempotent/safe calls. Retry and hedging are mutually exclusive in a method config.

```go
const serviceConfig = `{
  "loadBalancingConfig": [{"round_robin":{}}],
  "methodConfig": [{
    "name": [{"service": "orders.v1.OrderService", "method": "GetOrder"}],
    "timeout": "1s",
    "retryPolicy": {
      "maxAttempts": 4,
      "initialBackoff": "0.05s",
      "maxBackoff": "0.5s",
      "backoffMultiplier": 2,
      "retryableStatusCodes": ["UNAVAILABLE"]
    }
  }],
  "retryThrottling": { "maxTokens": 10, "tokenRatio": 0.1 }
}`

conn, _ := grpc.NewClient("dns:///orders.internal:443",
    grpc.WithDefaultServiceConfig(serviceConfig),
    grpc.WithTransportCredentials(creds),
)
```

Details to know:

- `maxAttempts` is capped (5 in the spec) and counts the original attempt.
- Backoff is randomized: actual delay = `random(0, min(initialBackoff * multiplier^(n-1), maxBackoff))`.
- Server can control via **pushback** trailer `grpc-retry-pushback-ms` (negative/invalid value = don't retry; a value = delay).
- Retries are **per-RPC-call** budgets with **buffering**: the client buffers the request until it's committed (default buffer limit 256 KB per RPC by `WithPerRPCBufferLimit`/`MaxRetryRPCBufferSize`); beyond that, retries stop because the request can't be replayed. Streaming RPCs retry only before the first response is received (the RPC "commits").
- **Retry throttling** is a token bucket: each failure drains a token, each success refills `tokenRatio`; when tokens ≤ half of `maxTokens`, retries are suppressed. This prevents retry storms.

## Q9.2 — What causes a retry storm, and how do you prevent it?

**Answer.**

Retries are **load multipliers**: if each of N layers retries 3 times, a failing leaf receives up to 3^N attempts. When a dependency slows down, retries add load, which slows it more.

Prevention:

1. **Retry at one layer only** (usually closest to the failing dependency or the edge, not every layer).
2. **Retry budgets/throttling** (token bucket above, or ≤10% extra load).
3. **Exponential backoff with jitter**.
4. **Honor deadlines** — don't retry if the remaining budget is smaller than one expected attempt.
5. **Only retry safe codes** (`UNAVAILABLE`; `ABORTED` at transaction level) and idempotent methods.
6. **Server-side load shedding** and pushback signals.
7. **Circuit breakers** for sustained failure (outlier detection in Envoy/xDS).

## Q9.3 — How do you make a non-idempotent RPC safe to retry?

**Answer.**

Add an **idempotency key** supplied by the client (unique per logical operation, stable across retries) and have the server store `(key → response)` atomically with the effect, within a TTL.

```go
func (s *Pay) Charge(ctx context.Context, r *pb.ChargeRequest) (*pb.ChargeResponse, error) {
    key := r.GetIdempotencyKey()
    if key == "" { return nil, status.Error(codes.InvalidArgument, "idempotency_key required") }

    tx, _ := s.db.BeginTx(ctx, nil)
    defer tx.Rollback()
    // INSERT ... ON CONFLICT (key) DO NOTHING RETURNING ...
    if prior, found := s.lookup(ctx, tx, key); found {
        return prior, nil // replay-safe
    }
    resp, err := s.doCharge(ctx, tx, r)
    if err != nil { return nil, err }
    s.store(ctx, tx, key, resp)
    return resp, tx.Commit()
}
```

Also guard against the "in-flight duplicate" race with a unique constraint or lock on the key, and validate that a replayed key carries the same request fingerprint.

## Q9.4 — Hedging: when to use, how to configure?

**Answer.**

Use when p99 is dominated by random stragglers (GC pauses, noisy neighbors) and the call is **idempotent and cheap**.

```json
"hedgingPolicy": {
  "maxAttempts": 3,
  "hedgingDelay": "0.1s",
  "nonFatalStatusCodes": ["UNAVAILABLE", "INTERNAL"]
}
```

Set `hedgingDelay` near your p95 latency so only ~5% of calls send a second request. Always combine with throttling. Hedging a write without idempotency produces duplicate side effects.

---

# Part 10 — Load Balancing

## Q10.1 — Why does a Kubernetes `ClusterIP` Service not balance gRPC traffic properly?

**Answer.**

`kube-proxy` (iptables/IPVS) balances at **L4, per TCP connection**. gRPC uses **one long-lived HTTP/2 connection** and multiplexes all RPCs over it, so every request from a given client lands on the **same pod** for the connection's lifetime. After a scale-up, new pods remain idle; during deploys, load shifts unevenly; a few busy clients can overwhelm a pod.

**Solutions:**

| Approach | How | Trade-offs |
|---|---|---|
| **Client-side LB** | Headless Service (`clusterIP: None`) + `dns:///svc:port` + `round_robin` | Simple, no extra hop; DNS refresh latency; clients need endpoint awareness |
| **xDS** (proxyless gRPC) | Control plane (Istio/Traffic Director) pushes endpoints & policy | Powerful, complex; requires xDS infra |
| **L7 proxy** (Envoy, Linkerd, Istio sidecar, NGINX ≥1.13.10 w/ `grpc_pass`) | Proxy balances per-RPC | Extra hop/latency, but language-agnostic and centralized policy |
| **Lookaside (external LB)** | Client asks an LB service where to go (gRPCLB, legacy) | Largely superseded by xDS |
| **L4 + `MaxConnectionAge`** | Force periodic reconnects to rebalance | Crude, causes reconnect churn; works as a mitigation |

```yaml
apiVersion: v1
kind: Service
metadata: { name: orders }
spec:
  clusterIP: None       # headless: DNS returns all pod IPs
  selector: { app: orders }
  ports: [{ name: grpc, port: 50051 }]
```

```go
conn, err := grpc.NewClient("dns:///orders.default.svc.cluster.local:50051",
    grpc.WithDefaultServiceConfig(`{"loadBalancingConfig":[{"round_robin":{}}]}`),
    grpc.WithTransportCredentials(creds))
```

## Q10.2 — How does client-side load balancing work internally in grpc-go?

**Answer.**

Pipeline:

```
 target "dns:///orders:50051"
        │
   [Resolver]  ──► list of addresses (+ optional service config)
        │
   [Balancer / LB policy]  ──► creates one SubConn per address (or subset), tracks connectivity
        │
   [Picker]  ──► for each RPC, picks a ready SubConn (round_robin, pick_first, weighted, ring_hash, least_request...)
        │
   [Transport (HTTP/2)]
```

- **`pick_first`** (default): connect to the first working address, use only it. *Not load balancing.*
- **`round_robin`**: connects to all addresses, rotates per RPC.
- **`weighted_round_robin`**, **`ring_hash`** (consistent hashing for affinity/caches), **`least_request`** (via xDS) are available with xDS/newer releases.
- Connectivity states per SubConn: `IDLE → CONNECTING → READY ↔ TRANSIENT_FAILURE`, with exponential **connect backoff** (1s base, 1.6x multiplier, 120s max, 20% jitter by default).

**DNS resolver caveat:** grpc-go's DNS resolver **re-resolves only when a connection fails or is closed** (and via a min-interval throttle ~30s), *not* on a periodic TTL. So scale-ups aren't discovered until existing connections break — another reason `MaxConnectionAge` on the server is valuable. xDS or a watch-based custom resolver (Kubernetes EndpointSlices watch) avoids the problem.

## Q10.3 — How do you do session affinity / consistent hashing with gRPC?

**Answer.**

Use **`ring_hash`** or Maglev with a hash key from metadata/header (xDS route `hash_policy`), or application-level sharding (route by tenant/partition key to the owning shard). Remember that affinity conflicts with even distribution and complicates failover; prefer stateless services with external state, and use affinity only for cache locality.

A custom resolver for sharded topologies:

```go
// Register a custom scheme, e.g. "shard:///orders" resolving to shard-specific addresses,
// and use a custom picker that reads a key from the context.
type shardKey struct{}
ctx = context.WithValue(ctx, shardKey{}, "tenant-42")
// Picker.Pick(info balancer.PickInfo) can read info.Ctx.Value(shardKey{}) to choose a SubConn.
```

## Q10.4 — Single connection vs connection pool: when do you need more than one `ClientConn`?

**Answer.**

A `ClientConn` creates connections per resolved backend address (with `round_robin`, one per backend). Reasons to use **multiple `ClientConn`s (a pool)** to the same target:

- A single HTTP/2 connection hits **max concurrent streams** (e.g., 100 in Envoy/Java, configurable in Go) → queueing latency.
- **TCP HOL blocking** and per-connection congestion window limit throughput on lossy/long-fat networks.
- A single connection is bounded by one TCP/TLS stream's CPU (one goroutine reads frames) — very high QPS can bottleneck.

Do **not** create a new `ClientConn` per request: it is expensive (TCP+TLS handshake, DNS, goroutines) and leaks if not closed. `ClientConn` is safe for concurrent use and designed to be long-lived and shared. Pooling means N long-lived conns with round-robin selection across them.

## Q10.5 — What are GOAWAY and connection draining? How do deploys avoid errors?

**Answer.**

`GOAWAY` tells the peer "don't start new streams on this connection; existing ones may finish". `GracefulStop()` and `MaxConnectionAge` use it. A two-phase GOAWAY (first with max stream id, then actual after RTT) avoids racing new streams.

Zero-downtime rollout checklist:

1. Pod receives SIGTERM; **readiness flips to NOT_SERVING first** (health service) so LBs stop sending new traffic.
2. Wait a short time for LB/endpoint propagation (`preStop` sleep) — endpoints removal is eventually consistent.
3. `GracefulStop()` — sends GOAWAY, waits for in-flight RPCs.
4. Hard `Stop()` after a grace period; ensure `terminationGracePeriodSeconds` exceeds it.
5. Clients: retry `UNAVAILABLE` for idempotent calls; transparent retry covers refused streams; streaming clients reconnect with resume.

---

# Part 11 — Streaming Deep Dive: Flow Control and Backpressure

## Q11.1 — What delivery and ordering guarantees does a gRPC stream give?

**Answer.**

- **In-order, reliable delivery within a single stream** (it rides on TCP and HTTP/2 stream ordering).
- **No ordering across different streams/RPCs**, even on one connection.
- **No exactly-once** and no durability: if a stream breaks, messages in flight/buffered are lost from the app's perspective. The client can't know which messages the server processed (maybe processed but ack lost).
- Therefore, build **application-level acks, sequence numbers and resume tokens** for reliable streaming; make consumers idempotent.

```proto
message EventEnvelope { uint64 seq = 1; bytes payload = 2; }
message Subscribe { string subscriber_id = 1; uint64 resume_after_seq = 2; }
```

## Q11.2 — Design a resumable event-subscription stream.

**Answer.**

1. Server stores events with a monotonic sequence (log/Kafka offset).
2. Client opens `Subscribe(resume_after_seq=N)`.
3. Server replays from `N+1` then tails live events; sends in order with `seq`.
4. Client persists the last processed `seq` **after** processing (at-least-once) — or atomically with side effects for effectively-once.
5. On any error (`UNAVAILABLE`, EOF from rolling deploy), client reconnects with backoff + jitter using its last `seq`.
6. Server bounds per-subscriber buffers; if a subscriber falls too far behind, server closes the stream with `OUT_OF_RANGE`/`ABORTED` and the client re-syncs from a snapshot.
7. Heartbeats (empty events) or keepalive pings detect half-open connections.

```go
for attempt := 0; ; attempt++ {
    err := consume(ctx, client, &lastSeq)   // runs one stream until it fails
    if ctx.Err() != nil { return ctx.Err() }
    backoff := min(time.Duration(1<<min(attempt, 6))*100*time.Millisecond, 30*time.Second)
    time.Sleep(jitter(backoff))
}
```

## Q11.3 — A stream "hangs" with no errors. How do you diagnose half-open connections?

**Answer.**

A network partition, NAT/LB idle-timeout drop, or crashed peer without FIN/RST leaves TCP "up" but dead. Without traffic, neither side notices. Mitigations:

- **Keepalive pings** (client `keepalive.ClientParameters{Time, Timeout, PermitWithoutStream}`) detect dead peers; server must permit that frequency via `EnforcementPolicy.MinTime`, else it replies `GOAWAY too_many_pings`.
- Application-level **heartbeats** and **per-message deadlines** (a watchdog timer reset on each `Recv`).
- Choose keepalive less than the shortest idle timeout in the path (cloud LBs commonly 60s–350s).

```go
grpc.WithKeepaliveParams(keepalive.ClientParameters{
    Time: 30 * time.Second, Timeout: 10 * time.Second, PermitWithoutStream: true,
})
```

Avoid overly aggressive pings: with thousands of clients, pings cost CPU and may trigger `ENHANCE_YOUR_CALM`/`too_many_pings`.

---

# Part 12 — Connection Management and Keepalive

## Q12.1 — Explain the `ClientConn` lifecycle: `Dial` vs `NewClient`, lazy connect, idle mode, backoff.

**Answer.**

- **`grpc.NewClient`** (grpc-go ≥1.63) is the recommended constructor. It **does not connect immediately**; it starts in **IDLE** and connects on the first RPC. It uses the `dns` resolver by default (the old `grpc.Dial` defaulted to `passthrough`). Blocking dial options (`WithBlock`) are deprecated/discouraged because they hide failures and slow startup.
- After inactivity, channels enter **idle mode** (default 30 minutes) to release resources, then reconnect on demand.
- Failed connections retry with exponential **connect backoff** (distinct from RPC retry). While in `TRANSIENT_FAILURE`, RPCs fail fast with `UNAVAILABLE` unless `grpc.WaitForReady(true)` is set, which queues them until the channel is ready or the deadline hits.

```go
// Fail-fast (default) – good for request paths:
resp, err := client.GetOrder(ctx, req)
// Wait-for-ready – good for startup/batch where the server may not be up yet:
resp, err = client.GetOrder(ctx, req, grpc.WaitForReady(true))
```

Monitor state:

```go
for s := conn.GetState(); s != connectivity.Shutdown; s = conn.GetState() {
    if !conn.WaitForStateChange(ctx, s) { return }
    slog.Info("conn state", "state", conn.GetState())
}
```

## Q12.2 — `too_many_pings` / `ENHANCE_YOUR_CALM`: what is it and how do you fix it?

**Answer.**

The server's `KeepaliveEnforcementPolicy` (default `MinTime = 5 minutes`, `PermitWithoutStream = false`) tells it to close connections whose clients ping more often than allowed. Symptom: clients see `Unavailable: transport is closing` and server logs "Client sent too many pings"/GOAWAY `too_many_pings`. Fix: align both sides: client `Time ≥ server MinTime`, and set `PermitWithoutStream: true` on the server if clients ping while idle.

## Q12.3 — What are the key server/connection limits to set in production?

**Answer.**

| Setting | Why |
|---|---|
| `MaxRecvMsgSize` | Protect memory from oversized messages |
| `MaxConcurrentStreams` | Bound per-connection concurrency |
| `MaxConnectionAge(+Grace)` | Rebalance, pick up new backends, rotate certs/credentials |
| `MaxConnectionIdle` | Reclaim idle clients |
| `KeepaliveEnforcementPolicy` | Defend against ping floods |
| `ReadBufferSize/WriteBufferSize` | Memory vs throughput trade-off |
| `NumStreamWorkers` (experimental) | Reuse goroutines for handlers, lowering allocation under heavy load |
| Handler-level timeouts | Never trust clients to set deadlines |

---

# Part 13 — Performance Tuning

## Q13.1 — A gRPC service has high p99 latency under load. Walk through your investigation.

**Answer (structured approach).**

1. **Define the symptom**: which RPC, what percentile, since when, correlated with deploy/traffic/dependency?
2. **Separate client-observed vs server-observed latency.** If client p99 >> server handler p99, the gap is: queueing on the client (max concurrent streams, connection pool), network, LB/proxy hops, TLS, or server accept/queueing *before* the handler runs. Instrument both sides.
3. **Check saturation**: CPU (protobuf marshal, TLS, compression), GC pauses (large messages), goroutine counts, file descriptors, connection counts, `MaxConcurrentStreams` queueing.
4. **Check head-of-line effects**: big messages/streams sharing a connection with latency-sensitive calls → isolate onto separate channels.
5. **Examine retries/hedging**: are they amplifying load?
6. **Dependencies**: downstream latency and deadlines; connection pool settings for DB.
7. **Profile**: `pprof` (CPU/heap/goroutine/mutex/block), `go tool trace`; look at `proto.Marshal`, `compress/gzip`, `crypto/tls`, GC.
8. **Fix** with data: reduce allocations (pooling, avoid copying `[]byte`), use streaming for large payloads, tune windows, add connection pool, change compression, fix LB distribution, add load shedding.

## Q13.2 — Compression: when does it help or hurt?

**Answer.**

gRPC supports per-message compression negotiated via `grpc-encoding`/`grpc-accept-encoding` (gzip built-in; others like snappy/zstd via registered compressors).

```go
import _ "google.golang.org/grpc/encoding/gzip"
resp, err := client.GetOrder(ctx, req, grpc.UseCompressor(gzip.Name))
// server-side default:  grpc.RPCCompressor / or per-call grpc.SetSendCompressor(ctx, "gzip")
```

- Helps for large, repetitive payloads over slow/expensive links (cross-region, egress cost).
- Hurts for small messages (CPU cost > savings), already-compressed data (images), and CPU-bound servers. Compression also increases latency per message and complicates profiling.
- Security: compressing secrets mixed with attacker-controlled data over TLS can enable CRIME/BREACH-style attacks; relevant if attacker can influence part of the plaintext.

## Q13.3 — How do you reduce allocations and GC pressure in a hot gRPC path?

**Answer.**

- Reuse buffers: grpc-go uses an internal buffer pool (`mem.BufferPool`) for received messages in recent versions.
- Avoid `proto.Clone` and unnecessary conversions between domain and proto types; use `proto.Size` + `MarshalAppend` for custom paths.
- For many small fields, consider **arena/pooled** message reuse (`sync.Pool` with `Reset()`), but beware retaining references (data races/leaks).
- Use `bytes` fields judiciously: copying big `[]byte` at every layer multiplies cost.
- Prefer **streaming** for big payloads to cap per-message memory.
- Tune GC (`GOGC`, `GOMEMLIMIT`) with real traces.
- Consider vtproto/`protoc-gen-go-vtproto` for faster marshal/unmarshal with pooling (trade-off: custom codec and build complexity).

## Q13.4 — Explain how grpc-go processes concurrent requests (threading model).

**Answer.**

For each accepted connection, a goroutine reads HTTP/2 frames (`http2Server.HandleStreams`). For each new stream, grpc-go starts a **goroutine to run the handler** (or uses a worker pool if `NumStreamWorkers` is enabled). Writes are coalesced by a loopy writer goroutine per connection. Implications:

- Goroutine-per-RPC is cheap but **unbounded by default**: overload creates unbounded goroutines → memory exhaustion. Add concurrency limits/load shedding.
- Handlers share a connection's framer/writer; a handler blocked on `Send` due to flow control only blocks itself, not the connection.
- CPU-bound handlers should limit parallelism (worker pools) to avoid starving the reader/writer goroutines.

---

# Part 14 — Observability

## Q14.1 — What should you instrument for every gRPC service?

**Answer.**

**The RED + saturation set**, per method and per status code:

- **R**ate: requests/sec by `grpc_method`, `grpc_service`.
- **E**rrors: by `grpc_code` (don't treat `NOT_FOUND`/`INVALID_ARGUMENT` as server errors in SLOs; usually count `UNAVAILABLE`, `INTERNAL`, `DEADLINE_EXCEEDED`, `UNKNOWN`, `DATA_LOSS`).
- **D**uration: histograms (not averages), client-side and server-side.
- **Saturation**: in-flight RPCs, active streams, goroutines, connection counts, message sizes, queue depth, retry/hedge counts.
- **Stream-specific**: messages sent/received per stream, stream duration, active subscribers.

```go
import "go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"

srv := grpc.NewServer(grpc.StatsHandler(otelgrpc.NewServerHandler()))
conn, _ := grpc.NewClient(target, grpc.WithStatsHandler(otelgrpc.NewClientHandler()))
```

(`StatsHandler` is the modern hook, replacing interceptor-based otelgrpc, and sees stream-level and per-message events.)

## Q14.2 — Distributed tracing: how does context propagate across gRPC calls?

**Answer.**

Trace context travels as **metadata** (W3C `traceparent`/`tracestate`, or B3). The OpenTelemetry stats handler injects on the client and extracts on the server, so spans link across services. Requirements: pass `ctx` unchanged through your code; propagate through goroutines you spawn; set propagators globally:

```go
otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
    propagation.TraceContext{}, propagation.Baggage{}))
```

Staff-level: sampling strategy (head vs tail-based), span attributes (`rpc.system=grpc`, `rpc.service`, `rpc.method`, `rpc.grpc.status_code`), cardinality control, and exemplars linking metrics to traces.

## Q14.3 — Debugging tools you should know.

**Answer.**

```bash
# Reflection-enabled server:
grpcurl -plaintext localhost:50051 list
grpcurl -plaintext localhost:50051 describe orders.v1.OrderService
grpcurl -plaintext -d '{"order_id":"o-1"}' localhost:50051 orders.v1.OrderService/GetOrder

# Without reflection:
grpcurl -import-path ./proto -proto orders/v1/orders.proto -d '{"order_id":"o-1"}' host:443 orders.v1.OrderService/GetOrder

# Health check:
grpcurl -plaintext localhost:50051 grpc.health.v1.Health/Check

# Enable verbose logs:
GRPC_GO_LOG_VERBOSITY_LEVEL=99 GRPC_GO_LOG_SEVERITY_LEVEL=info ./server
# Go channelz (introspection of channels/subchannels/sockets):
#   import "google.golang.org/grpc/channelz/service"; service.RegisterChannelzServiceToServer(srv)
# Packet-level: Wireshark with HTTP/2 + protobuf dissectors (needs TLS keylog: SSLKEYLOGFILE)
```

**Channelz** is an underused staff-level tool: it exposes per-connection stats (streams started/succeeded/failed, last message sent time, flow-control windows) so you can see imbalance and stuck connections in production.

---

# Part 15 — Testing gRPC Services

## Q15.1 — How do you test gRPC services without opening real ports? (bufconn)

**Answer.**

```go
import (
    "google.golang.org/grpc/test/bufconn"
)

func newTestClient(t *testing.T, impl ordersv1.OrderServiceServer) ordersv1.OrderServiceClient {
    t.Helper()
    lis := bufconn.Listen(1 << 20)
    srv := grpc.NewServer(grpc.ChainUnaryInterceptor(loggingUnary))
    ordersv1.RegisterOrderServiceServer(srv, impl)
    go srv.Serve(lis)
    t.Cleanup(srv.Stop)

    conn, err := grpc.NewClient("passthrough:///bufnet",
        grpc.WithContextDialer(func(ctx context.Context, _ string) (net.Conn, error) { return lis.DialContext(ctx) }),
        grpc.WithTransportCredentials(insecure.NewCredentials()),
    )
    if err != nil { t.Fatal(err) }
    t.Cleanup(func() { conn.Close() })
    return ordersv1.NewOrderServiceClient(conn)
}

func TestGetOrder_NotFound(t *testing.T) {
    c := newTestClient(t, &orderServer{repo: fakeRepo{}})
    _, err := c.GetOrder(context.Background(), &ordersv1.GetOrderRequest{OrderId: "nope"})
    if got := status.Code(err); got != codes.NotFound {
        t.Fatalf("code = %v, want NotFound", got)
    }
}
```

This exercises the full stack — serialization, interceptors, status mapping — in-process and fast. Note `passthrough:///` resolver is needed with a custom dialer in `NewClient`.

## Q15.2 — What testing strategy covers a gRPC platform?

**Answer.**

1. **Unit tests** on domain logic without gRPC.
2. **bufconn integration tests** for handlers + interceptors (status codes, metadata, deadlines).
3. **Contract/compat tests**: `buf breaking`, golden serialized messages from older versions deserialized by newer code (and vice versa).
4. **Fault injection**: simulate slow handlers, `UNAVAILABLE`, connection drops (e.g., toxiproxy, custom interceptors returning errors) to validate retries, deadlines, reconnection.
5. **Streaming tests**: cancellation mid-stream, slow consumer, server shutdown, resume correctness.
6. **Load/soak tests** (`ghz`): throughput, p99, connection reuse, memory over hours.
7. **Rolling-upgrade tests** with mixed versions (N and N-1) to prove wire compatibility.

```bash
ghz --insecure --proto orders/v1/orders.proto --call orders.v1.OrderService/GetOrder \
    -d '{"order_id":"o-1"}' -c 50 -n 100000 --connections 5 localhost:50051
```

---

# Part 16 — API Evolution and Governance

## Q16.1 — How do you version gRPC APIs?

**Answer.**

- **Package-level versioning**: `package orders.v1;` → `orders.v2`. Method path includes the package, so `v1` and `v2` services can run side by side in the same server.
- Within a major version: **only backward-compatible changes**, enforced by tooling.
- Deprecate with `[deprecated = true]` options, documentation and telemetry on usage per client (you can only delete when metrics show zero callers).
- Run **v1 and v2 simultaneously**, with v1 implemented as an adapter over the v2 core when possible.

```proto
service OrderService {
  rpc GetOrder(GetOrderRequest) returns (Order) { option deprecated = true; }
}
```

## Q16.2 — What API design principles matter at scale (Google AIP-style)?

**Answer.**

- **Resource-oriented design**: `GetX`, `ListX`, `CreateX`, `UpdateX` (with `FieldMask`), `DeleteX`; custom methods `Verb` only when needed.
- **Pagination**: `page_size` + opaque `page_token` + `next_page_token`; never offset for large sets.
- **Request/response messages per RPC**, never reuse messages across RPCs and never use primitive/`google.protobuf.Empty` as request or response (can't extend later).
- **Long-running operations** via `google.longrunning.Operation` (create returns an operation to poll/cancel).
- **Idempotency** via request IDs on creates.
- **Validation** (protovalidate/`protoc-gen-validate`) declared in the schema.
- **Consistent error model** and stable `ErrorInfo.reason` codes.
- **Avoid leaking storage models** into the API; the proto is a contract, not an ORM.
- **Never expose internal enums/IDs** that you can't commit to supporting forever.

```proto
message ListOrdersRequest {
  int32  page_size  = 1;  // server caps to max
  string page_token = 2;  // opaque, e.g., base64 of cursor + signature
  string filter     = 3;
  string order_by   = 4;
}
message ListOrdersResponse {
  repeated Order orders = 1;
  string next_page_token = 2; // empty => end
}
```

## Q16.3 — How would you set up schema governance in a multi-team organization?

**Answer.**

- **Single source of truth**: a monorepo or Buf Schema Registry (BSR) for protos; generated code is published as versioned packages, or generated in-build.
- **Lint + breaking-change CI** (`buf lint`, `buf breaking`) as required checks.
- **Ownership & review**: CODEOWNERS per package; API review for new public APIs; style guide (naming, pagination, errors).
- **Compat testing** across language runtimes.
- **Deprecation workflow** with usage telemetry and announced sunset dates.
- **Documentation generation** from proto comments (protoc-gen-doc / BSR docs).
- **Dependency management**: shared "common" protos with strict stability guarantees (changing them breaks everyone).

---

# Part 17 — Gateways, gRPC-Web, Connect and Browsers

## Q17.1 — How do you expose gRPC services to browsers and REST clients?

**Answer.**

| Option | How it works | Notes |
|---|---|---|
| **gRPC-Web** | Browser client speaks a gRPC-like protocol over HTTP/1.1 or 2 to an Envoy/proxy that translates to gRPC | No client/bidi streaming from browsers (only unary and server streaming) |
| **Connect protocol** (connectrpc.com) | Protobuf-based RPC supporting gRPC, gRPC-Web and its own HTTP/1.1-friendly protocol from one handler | Simple `net/http` integration, curl-able JSON, browser-friendly |
| **grpc-gateway** | Generates a REST/JSON reverse proxy from `google.api.http` annotations | Great for public REST APIs with OpenAPI generation |
| **Envoy `grpc_json_transcoder`** | Proxy-level JSON↔gRPC transcoding | No generated code; config-driven |

```proto
import "google/api/annotations.proto";
service OrderService {
  rpc GetOrder(GetOrderRequest) returns (Order) {
    option (google.api.http) = { get: "/v1/orders/{order_id}" };
  }
}
```

```go
// grpc-gateway in the same process
mux := runtime.NewServeMux()
_ = ordersv1.RegisterOrderServiceHandlerFromEndpoint(ctx, mux, "localhost:50051",
    []grpc.DialOption{grpc.WithTransportCredentials(insecure.NewCredentials())})
http.ListenAndServe(":8080", mux)
```

**Trade-offs to mention:** a gateway adds a hop and a second API surface to maintain; JSON mapping rules (lowerCamelCase names, int64 as strings, enums as names, `Any` encoding) cause subtle client bugs; HTTP annotation changes are API changes.

## Q17.2 — gRPC vs REST vs GraphQL vs message queues: how do you decide?

**Answer.**

| Concern | gRPC | REST/JSON | GraphQL | Queue/Event bus |
|---|---|---|---|---|
| Coupling | Synchronous, contract-coupled | Synchronous, loosely typed | Client-driven shape | Asynchronous, decoupled in time |
| Best for | Internal low-latency RPC, streaming | Public APIs, cacheable resources | Many client shapes, BFF | Durable workflows, fan-out |
| Weak at | Browser, caching, human readability | Streaming, strict contracts | Complexity, N+1, caching | Request/response latency |
| Failure mode | Cascading latency | Same | Expensive queries | Backlog, ordering, duplicates |

A staff-level answer: *"Use the synchronous call only when the caller needs the answer to proceed; otherwise emit an event."* Synchronous gRPC chains multiply availability risk (availability = product of each dependency's availability).

---

# Part 18 — Graceful Shutdown, Health and Operations

## Q18.1 — Explain gRPC health checking and how Kubernetes should use it.

**Answer.**

Standard `grpc.health.v1.Health` service: `Check` (unary) and `Watch` (server stream). Status per service name (empty string = whole server): `SERVING`, `NOT_SERVING`, `SERVICE_UNKNOWN`.

Kubernetes (1.24+) supports **native gRPC probes**:

```yaml
readinessProbe:
  grpc: { port: 50051, service: "orders.v1.OrderService" }
  periodSeconds: 5
livenessProbe:
  grpc: { port: 50051 }
  periodSeconds: 10
  failureThreshold: 3
lifecycle:
  preStop: { exec: { command: ["sleep", "10"] } }
terminationGracePeriodSeconds: 45
```

Key design rules:

- **Liveness** should be shallow (process is not deadlocked); **readiness** reflects ability to serve (dependencies, warm-up, draining). Making liveness depend on downstreams causes restart storms during dependency outages.
- Flip readiness to `NOT_SERVING` on SIGTERM *before* `GracefulStop`.
- Client-side health checking (`healthCheckConfig` in service config) lets clients avoid unhealthy backends without waiting for the platform.

## Q18.2 — Describe a zero-downtime rolling deployment for a service with long-lived streams.

**Answer.**

1. Readiness → NOT_SERVING; wait for endpoint removal (`preStop` sleep).
2. Server sends GOAWAY (`GracefulStop`); new RPCs go elsewhere.
3. For streams: proactively **tell clients to reconnect** (send a "going away" control message/close with `UNAVAILABLE` after a jittered delay) so reconnects are spread out rather than a thundering herd.
4. Bound the drain with a deadline; force-close stragglers; clients resume using sequence tokens.
5. Use `maxSurge`/`maxUnavailable` and PodDisruptionBudgets so capacity doesn't dip below what reconnection storms need.
6. **Connection-draining thundering herd**: when a pod dies, thousands of clients reconnect simultaneously: use jittered backoff and server-side admission control.

## Q18.3 — What are common production incidents with gRPC and their causes?

**Answer.**

| Incident | Root cause | Fix |
|---|---|---|
| New pods get no traffic after scale-out | L4 LB + persistent connections / DNS not re-resolved | Client-side LB w/ headless svc, xDS, `MaxConnectionAge` |
| Sudden spike of `UNAVAILABLE` during deploy | No readiness flip/drain, clients not retrying | Health + GracefulStop + retry policy |
| Memory growth / OOM | Unbounded goroutines, large messages, slow-consumer buffers, leaked streams | Limits, bounded queues, ctx handling |
| `RESOURCE_EXHAUSTED: received message larger than max` | 4 MB default | Redesign (stream/paginate); raise limit only if justified |
| Retry storm during partial outage | Retries at multiple layers | Single-layer retry, throttling, budgets |
| Latency cliffs under load | Max concurrent streams/connection saturation | Pool channels, tune limits |
| Idle streams randomly die | LB/NAT idle timeout | Keepalive/heartbeat < timeout |
| `too_many_pings` | Mismatched keepalive policy | Align client/server settings |
| Goroutine leak | Handler ignoring `ctx.Done()` | Always select on ctx; tests with `goleak` |
| Cascading failure | Missing deadlines | Mandatory deadlines + propagation + load shedding |

---

# Part 19 — System Design and Scenario Questions

## Scenario 1 — Design a real-time order-tracking system for 1M concurrent clients using gRPC streaming.

**Strong answer outline:**

**Requirements**: 1M concurrent subscribers, <2s event latency, no loss of order updates (at-least-once OK), mobile clients with flaky networks.

**Architecture:**

```
Mobile/Web ──► Edge L7 proxy (Envoy) ──► Stream Gateway pods ──► Pub/Sub (Kafka/NATS) ◄── Order services
                                              │
                                              └─► Subscription registry (Redis) / sharding by user_id
```

- **Stream Gateway tier** holds client streams; stateless regarding business data; each pod serves e.g. 20–50k streams (memory per stream matters: buffers, goroutines ~ a few KB each plus buffers; validate by load test).
- Gateways subscribe to **per-shard topics** (hash(user_id) → shard) rather than per-user subscriptions to the bus. Fan-out in memory from shard consumers to local subscriber channels.
- **Resume tokens** = (partition, offset) per user or global sequence; on reconnect the gateway replays from the event log (bounded retention) or sends a snapshot if too far behind.
- **Backpressure**: bounded per-stream channel; slow consumers disconnected with a documented code; clients reconnect and resync.
- **Connection distribution**: L7 LB w/ least-connections for *new* streams; `MaxConnectionAge` with jitter (e.g., 30–60 min) to rebalance; graceful drain on deploy with spread-out reconnects.
- **Keepalive** 30s, `MaxConcurrentStreams` modest per connection; HTTP/2 connection per client app.
- **Auth**: JWT validated on stream open, with re-validation on a timer (token expiry mid-stream!) — either require periodic re-auth messages or close at expiry.
- **Observability**: active streams gauge, per-stream lag, reconnect rate, event delivery latency histograms.
- **Mobile reality**: radio switches kill TCP; expect constant reconnects. Consider gRPC-Web/WebSocket fallbacks for browsers; keep payloads tiny (push notifications for background).
- **Failure modes**: reconnect storms (jitter+admission control), hot shards (consistent hashing + sub-sharding), log retention exceeded (snapshot fallback).

## Scenario 2 — Your service calls 5 downstream gRPC services; p99 spikes to 10s and then everything times out. What happened and how do you harden it?

**Strong answer:**

Likely cascade: one downstream slowed → handlers blocked waiting → goroutines/connections piled up → memory/CPU pressure → more timeouts → retries from upstream added load.

Hardening, in order of impact:

1. **Deadlines everywhere + propagation** with budget splitting.
2. **Bulkheads**: separate concurrency limits/channels per dependency so one slow dependency can't consume all capacity.
3. **Timeouts shorter than the caller's**; **fail fast** when remaining budget is too small.
4. **Retry budgets/throttling**, retries only at one layer.
5. **Circuit breaker / outlier ejection** on dependency.
6. **Degrade gracefully**: serve cached/partial data when a non-critical dependency is down; make calls parallel with `errgroup` and tolerate optional failures.
7. **Load shedding** with priority classes (critical vs best-effort).
8. **Observability**: per-dependency latency and in-flight metrics + alerts on saturation, not just errors.

```go
g, gctx := errgroup.WithContext(ctx)
var prof *pb.Profile; var recs *pb.Recs
g.Go(func() (err error) { c, cancel := context.WithTimeout(gctx, 150*time.Millisecond); defer cancel(); prof, err = users.Get(c, ...); return })
g.Go(func() error {
    c, cancel := context.WithTimeout(gctx, 100*time.Millisecond); defer cancel()
    r, err := recsSvc.Get(c, ...)
    if err != nil { slog.Warn("recs degraded", "err", err); return nil } // optional dependency: tolerate
    recs = r; return nil
})
if err := g.Wait(); err != nil { return nil, err }
```

## Scenario 3 — Migrate a REST/JSON monolith's internal calls to gRPC with zero downtime.

**Strong answer:**

1. **Define protos** from existing contracts; don't redesign domain at the same time.
2. **Dual-serve**: add a gRPC server beside the REST handlers sharing the same service layer; use grpc-gateway or Connect to eventually generate REST from proto to avoid two implementations.
3. **Shadow traffic/compare**: mirror real requests and diff responses (including error semantics).
4. **Migrate callers incrementally** behind feature flags, per-client percentage rollout, with metrics comparing latency/error rates.
5. **Infrastructure readiness**: HTTP/2-capable ingress/mesh, client-side or L7 LB, observability for gRPC codes, retries/deadline policy defaults.
6. **Retire REST** only after traffic is zero for a deprecation window.
7. Risks: error-code mapping differences, JSON ↔ proto semantics (default values, unknown fields, int64), timeouts defaults (REST often had none; gRPC deadlines now enforced), and auth propagation.

## Scenario 4 — Design a file upload/download service with gRPC for multi-GB files.

**Strong answer:**

- **Don't** send multi-GB through your gRPC service if object storage is available: issue **pre-signed URLs** over gRPC (unary) and let clients upload/download directly with HTTP (resumable/multipart). gRPC carries control-plane only.
- If you must stream through gRPC: **client-streaming with chunks of 64 KB–1 MB**, first message = metadata (name, size, checksum, resume offset), then chunk messages with offsets; server verifies per-chunk or final checksum; supports **resumable uploads** by returning the committed offset on reconnect.
- Server-streaming for downloads with `offset`/`length` request for resume/range semantics.
- Tune flow control windows for high-BDP links, apply backpressure through bounded buffers, set `MaxRecvMsgSize` just above the chunk size, apply per-user rate limiting and quotas, and verify integrity.
- Avoid compression for already compressed media.

```proto
message UploadRequest {
  oneof payload {
    FileHeader header = 1;   // must be first
    Chunk chunk = 2;
    Trailer trailer = 3;     // checksum
  }
}
```

## Scenario 5 — Two services with a circular gRPC dependency cause deadlocks under load. Explain and fix.

**Strong answer:**

A calls B and B calls A (directly or via a cycle) while each handler holds limited resources (concurrency limiter, connection stream slots, worker pool, DB connections). Under load, all of A's slots wait on B and all of B's slots wait on A: **distributed deadlock** (resolved only by timeouts).

Fixes: break cycles in the dependency graph (extract shared logic to a third service/library); use **separate pools/limits for inbound vs outbound** work; make one direction asynchronous (events); enforce deadlines to bound the damage; add dependency-graph linting using tracing data.

## Scenario 6 — Multi-tenant platform: one noisy tenant degrades others. How do you isolate them over gRPC?

**Strong answer:**

- Identify tenant in **metadata** (derived from authenticated identity, never client-supplied claims alone).
- **Per-tenant rate limits and concurrency limits** in an interceptor (token bucket per tenant; shared via Redis for global limits or locally with approximate quotas).
- **Weighted fair queuing/priority** at the server: reserve capacity for premium tenants; shed best-effort first.
- **Per-tenant metrics** (with bounded cardinality) and anomaly alerts.
- **Cell-based architecture**: route large tenants to dedicated cells (via `ring_hash`/routing), limiting blast radius.
- Return `RESOURCE_EXHAUSTED` with `RetryInfo` so well-behaved clients back off.

```go
func tenantLimiter(l *Limiters) grpc.UnaryServerInterceptor {
    return func(ctx context.Context, req any, info *grpc.UnaryServerInfo, h grpc.UnaryHandler) (any, error) {
        t := tenantFrom(ctx) // set by auth interceptor from verified token
        if !l.For(t).Allow() {
            st, _ := status.New(codes.ResourceExhausted, "tenant quota exceeded").
                WithDetails(&errdetails.RetryInfo{RetryDelay: durationpb.New(500 * time.Millisecond)})
            return nil, st.Err()
        }
        return h(ctx, req)
    }
}
```

---

# Part 20 — Rapid-Fire Round and Cheat Sheet

## Rapid-fire Q&A

**1. What HTTP method and status does gRPC use?** Always `POST`; HTTP status is nearly always `200`, the real result is `grpc-status` in trailers.

**2. Can gRPC work over HTTP/1.1?** Not natively. Only through gRPC-Web/Connect-style protocols with a translating layer.

**3. What's the default max message size in Go?** 4 MB receive; send effectively unlimited.

**4. What does `UNAVAILABLE` mean for retry?** Transient; retrying (with backoff) is the canonical response.

**5. Is `ClientConn` thread-safe?** Yes; share one (or a small pool) per target.

**6. Can I call `stream.Send` from many goroutines?** No. One sender goroutine and one receiver goroutine max per stream.

**7. How do I cancel an in-flight RPC?** Cancel the context passed to the call; the client sends `RST_STREAM`.

**8. What's the difference between `Stop` and `GracefulStop`?** `GracefulStop` stops accepting new RPCs and waits for existing ones; `Stop` closes immediately, cancelling active RPCs.

**9. What ordering guarantees exist?** In-order within a stream; none across streams.

**10. How do you version a proto without breaking clients?** Add new fields with new numbers; reserve removed ones; never change types incompatibly; use `buf breaking`.

**11. `Unimplemented...Server` embedding: why?** Forward compatibility: adding RPCs doesn't break existing implementations.

**12. What is `WaitForReady`?** Queue RPCs while the channel is connecting/in transient failure instead of failing fast.

**13. Why do gRPC clients need `te: trailers`?** Required by the spec so intermediaries/servers know the client can handle trailers where status is delivered.

**14. How is `grpc-timeout` encoded?** Integer + unit (`H`,`M`,`S`,`m`,`u`,`n`), up to 8 digits, e.g., `250m` = 250 ms.

**15. What does a "trailers-only" response mean?** Error/empty responses sent as a single HEADERS frame with `grpc-status` and END_STREAM.

**16. Is gRPC secure by default?** No. You must configure TLS/credentials; `insecure.NewCredentials()` is plaintext.

**17. What's the point of `reserved`?** Prevents reuse of removed field numbers/names that old data/clients might still use.

**18. `google.protobuf.Empty` as request/response — good idea?** Avoid; you can't add fields later without breaking the API. Define dedicated empty messages.

**19. How do you pass large lists?** Paginate or stream; do not return unbounded `repeated` fields.

**20. How does a client know the server isn't serving a method?** `UNIMPLEMENTED` status code.

## Pre-production checklist

- [ ] Deadlines set on every client call; deadline propagated via `ctx`
- [ ] TLS/mTLS configured; reflection restricted; auth + authz interceptors
- [ ] Recovery, logging, metrics, tracing via stats handler/interceptors
- [ ] Concurrency/load-shedding limits and `MaxRecvMsgSize` set
- [ ] Keepalive client/server policies aligned and < network idle timeouts
- [ ] Load-balancing strategy decided (headless+round_robin / xDS / L7) and verified under scale-out
- [ ] `MaxConnectionAge` with jitter for rebalancing
- [ ] Retry policy only for safe codes + throttling; idempotency keys for writes
- [ ] Health service + readiness flipping + graceful shutdown with hard stop fallback
- [ ] Proto lint/breaking checks in CI; reserved removed fields; deprecation policy
- [ ] Bounded buffers for streams; resume tokens for reconnects
- [ ] Load test including rolling deploys and failure injection
- [ ] Runbooks: status-code dashboards, channelz access, pprof endpoints

## Quick reference: essential snippets

```go
// Client with sensible production defaults
conn, err := grpc.NewClient("dns:///svc.ns.svc.cluster.local:50051",
    grpc.WithTransportCredentials(credentials.NewTLS(tlsCfg)),
    grpc.WithDefaultServiceConfig(serviceConfigJSON),   // LB + retry + per-method timeout
    grpc.WithKeepaliveParams(keepalive.ClientParameters{Time: 30*time.Second, Timeout: 10*time.Second, PermitWithoutStream: true}),
    grpc.WithStatsHandler(otelgrpc.NewClientHandler()),
    grpc.WithChainUnaryInterceptor(defaultDeadlineUnary(2*time.Second)),
)

// Status helpers
st, ok := status.FromError(err)          // ok=false if err is not a gRPC status error
code := status.Code(err)                 // codes.OK for nil
status.FromContextError(ctx.Err())       // map context errors to gRPC codes
```

## Further reading

- gRPC core concepts, performance best practices and the **gRPC over HTTP/2 protocol spec** (`PROTOCOL-HTTP2.md` in the grpc repo)
- gRFCs: **A6 (retries)**, **A8 (keepalive)**, **A27/A28 (xDS)**, **A32 (service config)**; the gRFC repo is where behavior is formally specified
- Google **API Improvement Proposals (AIP)** for resource-oriented API design
- Protobuf **Language Guide** and **Encoding** docs
- Buf documentation for linting, breaking-change detection and schema registry
- Envoy docs on gRPC proxying, health checking and outlier detection

> **How to use this guide:** for each question, practice answering in three layers — *what it is*, *why it exists / what breaks without it*, and *how you'd operate it in production*. Staff-level answers always include trade-offs, failure modes and the metrics you'd watch.
