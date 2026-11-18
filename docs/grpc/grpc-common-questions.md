# gRPC Common Questions

### 1. What is gRPC?

gRPC is a high-performance RPC framework developed by Google that uses HTTP/2 and Protocol Buffers for efficient communication between distributed services.

***

### 2. Why is gRPC faster than REST?

* Uses binary serialization (protobuf) instead of JSON
* Uses HTTP/2 multiplexing
* Smaller payload size
* Persistent connections

***

### 3. What transport protocol does gRPC use?

gRPC uses HTTP/2.

***

### 4. What serialization format does gRPC use by default?

Protocol Buffers (protobuf).

***

### 5. What is a `.proto` file?

A `.proto` file defines:

* Message schema
* RPC methods
* Service contracts

***

### 6. What command generates Go code from proto files?

```
protoc --go_out=. --go-grpc_out=. file.proto
```

***

### 7. What are the types of gRPC communication?

1. Unary RPC
2. Server Streaming
3. Client Streaming
4. Bidirectional Streaming

***

### 8. What is Unary RPC?

Single request → single response.

Example:

```
Client ---> Server        <---
```

***

### 9. What is Server Streaming RPC?

Client sends one request, server returns multiple responses.

***

### 10. What is Client Streaming RPC?

Client sends multiple requests, server returns one response.

***

### 11. What is Bidirectional Streaming?

Both client and server send streams simultaneously.

***

### 12. What package is commonly used for gRPC in Go?

```
google.golang.org/grpc
```

***

### 13. What is the default port commonly used in gRPC examples?

Usually:

```
50051
```

***

### 14. What is a stub in gRPC?

Generated client-side code used to invoke remote RPC methods like local functions.

***

### 15. What is the role of protobuf in gRPC?

* Defines API contracts
* Serializes/deserializes messages efficiently
* Enables language interoperability

***

### 16. How does gRPC support multiple languages?

Because protobuf is language-neutral and code generators exist for many languages.

***

### 17. What is context used for in gRPC Go?

`context.Context` is used for:

* Deadlines
* Cancellation
* Metadata propagation
* Tracing

***

### 18. How do you set a timeout in gRPC Go?

```
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)defer cancel()
```

***

### 19. What happens if a gRPC deadline exceeds?

The request is cancelled and returns:

```
DeadlineExceeded
```

***

### 20. What are gRPC status codes?

Standard error codes returned by gRPC.

Examples:

* OK
* NotFound
* Internal
* Unavailable
* DeadlineExceeded

***

### 21. How do you return a gRPC error in Go?

```
return nil, status.Error(codes.NotFound, "user not found")
```

***

### 22. What is metadata in gRPC?

Key-value pairs sent with requests/responses, similar to HTTP headers.

Used for:

* Auth tokens
* Tracing IDs
* Request IDs

***

### 23. How do you send metadata in Go gRPC client?

```
md := metadata.Pairs("authorization", "token")ctx := metadata.NewOutgoingContext(context.Background(), md)
```

***

### 24. What are interceptors in gRPC?

Middleware for handling:

* Logging
* Authentication
* Metrics
* Retry
* Tracing

Similar to HTTP middleware.

***

### 25. Difference between Unary and Stream interceptors?

* Unary interceptor → Unary RPC
* Stream interceptor → Streaming RPC

***

### 26. What is mTLS in gRPC?

Mutual TLS where both client and server authenticate each other using certificates.

***

### 27. Why is gRPC good for microservices?

* Fast communication
* Strong contracts
* Streaming support
* Efficient internal service communication
* Multi-language support

***

### 28. What are some limitations of gRPC?

* Browser support is limited
* Harder to debug than REST
* Binary payloads are not human-readable
* Requires protobuf setup

***

### 29. What is gRPC reflection?

Allows tools like `grpcurl` to discover available services dynamically without proto files.

***

### 30. What is grpcurl?

`grpcurl` is a CLI tool used to test and debug gRPC APIs, similar to `curl` for REST.

Example:

```
grpcurl localhost:50051 list
```
