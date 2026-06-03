# 4.2 · gRPC

## Overview

gRPC is a high-performance, contract-first RPC framework using HTTP/2 and Protocol Buffers. It supports four communication patterns and is widely used for inter-service communication in microservices.

---

## Core Concepts

### Communication Patterns
| Pattern | Description |
|---------|-------------|
| Unary | Client sends one request, server sends one response |
| Server-streaming | Client sends one request, server streams multiple responses |
| Client-streaming | Client streams multiple requests, server sends one response |
| Bidirectional-streaming | Both sides stream concurrently |

### HTTP/2 Advantages
- Multiplexed streams over a single TCP connection
- Header compression (HPACK)
- Binary framing (vs HTTP/1.1 text)
- Server push

### Service Definition (`.proto`)
- Defined in Protocol Buffers
- `protoc` + `protoc-gen-go-grpc` generates client/server stubs
- Single source of truth for API contract

### Interceptors (Middleware)
- **Unary interceptor** — wraps each RPC call
- **Stream interceptor** — wraps streaming RPCs
- Use for: auth, logging, tracing, rate limiting, retry, recovery

### Status Codes
gRPC has its own status codes: `OK`, `CANCELLED`, `INVALID_ARGUMENT`, `NOT_FOUND`, `ALREADY_EXISTS`, `PERMISSION_DENIED`, `INTERNAL`, `UNAVAILABLE`, `DEADLINE_EXCEEDED`, etc.

### Load Balancing
- **Client-side LB** — client picks endpoint from resolver (e.g., `grpc.WithDefaultServiceConfig`)
- **Proxy LB** — via Envoy, Nginx, or a service mesh
- gRPC uses connection-level LB by default (not request-level) — needs L7 proxy for request-level

### Deadlines & Cancellation
- Propagate `context` with deadline through the entire call chain
- Peer is notified on cancellation; both client and server should respect `ctx.Done()`

---

## Key Commands / Code Snippets

```protobuf
// orders.proto
syntax = "proto3";
package orders.v1;

service OrderService {
  rpc CreateOrder(CreateOrderRequest) returns (CreateOrderResponse);
  rpc StreamOrders(StreamOrdersRequest) returns (stream Order);
}

message CreateOrderRequest {
  string user_id = 1;
  repeated Item items = 2;
}
message CreateOrderResponse { string order_id = 1; }
message Order { string id = 1; string status = 2; }
message StreamOrdersRequest { string user_id = 1; }
message Item { string sku = 1; int32 qty = 2; }
```

```bash
# Generate Go code
protoc --go_out=. --go-grpc_out=. orders.proto

# Test with grpcurl
grpcurl -plaintext localhost:50051 list
grpcurl -plaintext -d '{"user_id":"u1"}' localhost:50051 orders.v1.OrderService/StreamOrders
```

```go
// Unary interceptor (logging + recovery)
func loggingInterceptor(
    ctx context.Context, req interface{},
    info *grpc.UnaryServerInfo, handler grpc.UnaryHandler,
) (interface{}, error) {
    start := time.Now()
    resp, err := handler(ctx, req)
    log.Printf("method=%s duration=%s err=%v", info.FullMethod, time.Since(start), err)
    return resp, err
}

// Server with options
s := grpc.NewServer(
    grpc.ChainUnaryInterceptor(authInterceptor, loggingInterceptor),
    grpc.KeepaliveParams(keepalive.ServerParameters{
        MaxConnectionIdle: 5 * time.Minute,
    }),
)

// Client with deadline
ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
defer cancel()
resp, err := client.CreateOrder(ctx, &pb.CreateOrderRequest{UserId: "u1"})
if status.Code(err) == codes.DeadlineExceeded { /* handle */ }
```

---

## Common Interview Questions

**Q: Why use gRPC over REST?**
> gRPC has a strongly-typed contract (`.proto`), uses binary serialization (smaller payloads, faster parse), supports streaming natively, and uses HTTP/2 (multiplexing, lower latency). REST is better for public APIs, browser clients, and human-readability.

**Q: How does gRPC handle errors differently from HTTP?**
> gRPC uses its own status code system (OK, NOT_FOUND, INTERNAL, etc.) carried in HTTP/2 trailers. Use `status.Error(codes.NotFound, "user not found")` and `status.Code(err)` on the client side.

**Q: What are gRPC interceptors?**
> Middleware that wraps RPC handlers. Unary interceptors get the request/response; stream interceptors wrap the stream object. Chain multiple with `grpc.ChainUnaryInterceptor`.

**Q: How does load balancing work with gRPC?**
> gRPC connections are long-lived (HTTP/2), so round-robin at the DNS/IP level doesn't distribute load per-request. Use a name resolver + client-side LB policy, or route through an L7 proxy (Envoy/Linkerd) that speaks gRPC natively.

---

## Gotchas & Best Practices

- Always propagate `context` — it carries deadlines and cancellation
- Don't forget to check `ctx.Done()` in long-running server streams
- Register health check service (`grpc_health_v1`) for load balancer integration
- Use `grpc.WaitForReady(true)` to avoid immediate failure on startup
- Prefer `status.Errorf` for structured errors — raw Go errors lose gRPC status codes over the wire
- Enable keepalive to detect dead connections early
- Document `.proto` files — they are the API contract

---

## Resources

- [ ] [gRPC Go Quick Start](https://grpc.io/docs/languages/go/quickstart/)
- [ ] [grpc-go GitHub](https://github.com/grpc/grpc-go)
- [ ] [grpcurl](https://github.com/fullstorydev/grpcurl)
- [ ] [Evans — gRPC REPL](https://github.com/ktr0731/evans)

---

## My Notes

<!-- ADD YOUR PERSONAL NOTES HERE -->

