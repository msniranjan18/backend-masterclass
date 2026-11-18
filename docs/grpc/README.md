# gRPC

**1. What is gRPC?**\
gRPC is a high-performance RPC framework using HTTP/2 and Protobuf, supporting four communication patterns with automatic code generation – ideal for microservices and real-time streaming.

* **High-performance, open-source RPC framework** by Google.
* Uses **HTTP/2** for transport, **Protocol Buffers (Protobuf)** as IDL.
* Enables client-server communication with support for **streaming** (unary, server, client, bidirectional).<br>

**2. Core Concepts**

| Concept              | Short Explanation                                                          |
| -------------------- | -------------------------------------------------------------------------- |
| **Protocol Buffers** | Language-neutral, binary serialization format (smaller, faster than JSON). |
| **.proto file**      | Defines messages (data structure) & service (RPC methods).                 |
| **gRPC Server**      | Implements service interface, listens for client calls.                    |
| **gRPC Stub**        | Client-side proxy (generated from .proto) that invokes RPC methods.        |
| **HTTP/2**           | Multiplexing, server push, header compression → low latency.               |



**3. Communication Types (4 Modes)**

1. **Unary** – Client sends single request, server sends single response. _(Standard HTTP request/response)_
2. **Server Streaming** – Client sends one request, server streams back multiple responses.
3. **Client Streaming** – Client streams multiple requests, server sends one response.
4. **Bidirectional Streaming** – Both sides stream independently (full-duplex).



**4. Key Features**

* **Code Generation** – Generate client/server stubs for multiple languages from `.proto`.
* **Deadlines / Timeouts** – Avoid hanging requests; client sets max duration.
* **Cancellation** – Both sides can cancel in-flight calls.
* **Metadata** – Key-value pairs (e.g., auth tokens) sent with RPC calls.
* **Interceptors** – Middleware for logging, auth, metrics (similar to REST middleware).
* **Load Balancing** – Client-side or proxy-based (e.g., gRPC LB, Envoy).
* **Error Handling** – Standard status codes (e.g., `NOT_FOUND`, `UNAVAILABLE`, `INTERNAL`).



**5. gRPC vs REST (Interview Favorite)**

| Feature               | gRPC                 | REST                                          |
| --------------------- | -------------------- | --------------------------------------------- |
| **Protocol**          | HTTP/2               | HTTP/1.1 or HTTP/2                            |
| **Payload**           | Protobuf (binary)    | JSON (text)                                   |
| **Performance**       | Very high            | Moderate                                      |
| **Code Generation**   | Yes (built-in)       | Manual / 3rd party (e.g., Swagger)            |
| **Streaming**         | Native (all 4 types) | Request/response only (limited via WebSocket) |
| **Browsers**          | Requires gRPC-web    | Native                                        |
| **Human Readability** | Low                  | High                                          |



**6. Common Use Cases**

* **Microservices** (low latency, polyglot environments)
* **Real-time streaming** (chat, live feed, game state)
* **Internal APIs** (where performance > human-readability)
* **Mobile clients** (smaller payload, battery efficient)



**7. Weaknesses / Limitations**

* **Limited browser support** (needs gRPC-web + proxy)
* **No built-in load balancing** (requires additional setup)
* **Not human-readable** (harder to debug without tools like grpcurl)
* **Steep learning curve** (Protobuf, HTTP/2 concepts)



**8. Typical Workflow (Short Steps)**

1. Define `.proto` (message + service)
2. Compile with `protoc` → generate client/server code
3. Implement server logic (override generated interface)
4. Run gRPC server
5. Client creates stub → calls method → gets response



**9. Important Status Codes (Memory aid)**

* `0` – OK
* `1` – CANCELLED
* `3` – INVALID\_ARGUMENT
* `5` – NOT\_FOUND
* `7` – PERMISSION\_DENIED
* `14` – UNAVAILABLE
* `16` – UNAUTHENTICATED



**10. Quick Comparison with GraphQL**

|           | gRPC                | GraphQL               |
| --------- | ------------------- | --------------------- |
| Verb      | RPC-style (actions) | Query (data fetching) |
| Schema    | Protobuf            | GraphQL schema        |
| Streaming | Native              | Limited               |
| Caching   | Medium              | Easy via client       |

####
