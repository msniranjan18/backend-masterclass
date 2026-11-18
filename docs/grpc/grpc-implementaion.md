# gRPC Implementaion

### Simple gRPC Implementation:

\[1. Write Contract] ---------> \[2. Run Compiler (protoc)] ---------> \[3. Write Business Logic]\
proto-files.proto                              Generates \*\_grpc.pb.go                       Your main.go files\
\
**Proto file**

{% code lineNumbers="true" expandable="true" %}
```protobuf
syntax = "proto3";

package profile;

// The Go package path where the generated code will live
option go_package = "./profile";

// 1. The Service (The functions we can call)
service ProfileService {
  rpc GetProfile (ProfileRequest) returns (ProfileResponse);
}

// 2. The Input Message
message ProfileRequest {
  string user_id = 1;
}

// 3. The Output Message
message ProfileResponse {
  string name = 1;
  int32 age = 2;
  string mobile = 3;
  string address = 4;
}
```
{% endcode %}

\
**Client code:**

{% code lineNumbers="true" expandable="true" %}
```go
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"

	// common repo for client and server both
	pb "github.com/your-org/shared-apis/profile" 
)

func main() {
	// Set up a connection to the server using the updated secure dialing options
	conn, err := grpc.Dial("localhost:50051", grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		log.Fatalf("Did not connect: %v", err)
	}
	defer conn.Close()

	// Initialize the client stub using the function from the external repo
	client := pb.NewProfileServiceClient(conn)

	// Create a context with a timeout for network safety
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()

	// Contact the server
	req := &pb.ProfileRequest{UserId: "compute_node_99"}
	res, err := client.GetProfile(ctx, req)
	if err != nil {
		log.Fatalf("Could not get profile: %v", err)
	}

	fmt.Printf("Data received from external API contract! Name: %s\n", res.GetName())
}
```
{% endcode %}

**Server side code:**

{% code lineNumbers="true" expandable="true" %}
```go
package main

import (
	"context"
	"log"
	"net"

	"google.golang.org/grpc"
	// common repo for client and server
	pb "github.com/your-org/shared-apis/profile" 
)

// server struct implements the interface defined in the external module
type server struct {
	pb.UnimplementedProfileServiceServer
}

// GetProfile handles the incoming data stream
func (s *server) GetProfile(ctx context.Context, req *pb.ProfileRequest) (*pb.ProfileResponse, error) {
	// Read data from the unmarshaled request structure
	log.Printf("Serving request for User ID: %s", req.GetUserId())

	// Simulate data retrieval
	return &pb.ProfileResponse{
		Name:    "Hypervisor-Cluster-Alpha",
		Age:     4,
		Mobile:  "+1-000-555-0100",
		Address: "Rack 4, Data Center East",
	}, nil
}

func main() {
	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatalf("Failed to listen: %v", err)
	}

	grpcServer := grpc.NewServer()
	
	// Register our server struct with the external service definition
	pb.RegisterProfileServiceServer(grpcServer, &server{})

	log.Println("Production gRPC Server listening on :50051...")
	if err := grpcServer.Serve(lis); err != nil {
		log.Fatalf("Failed to serve: %v", err)
	}
}
```
{% endcode %}





### **Same communication with REST**

**Client**

{% code lineNumbers="true" expandable="true" %}
```go
package main

import (
	"encoding/json"
	"fmt"
	"io"
	"log"
	"net/http"
)

// ProfileResponse must match the structure the server sends
type ProfileResponse struct {
	Name    string `json:"name"`
	Age     int    `json:"age"`
	Mobile  string `json:"mobile"`
	Address string `json:"address"`
}

func main() {
	// 1. Construct the URL with the plain text query parameter
	url := "http://localhost:8080/api/v1/profile?user_id=compute_node_99"

	// 2. Send the standard HTTP GET request
	resp, err := http.Get(url)
	if err != nil {
		log.Fatalf("Failed to send request: %v", err)
	}
	defer resp.Body.Close()

	// 3. Read the raw text bytes from the network response body
	bodyBytes, err := io.ReadAll(resp.Body)
	if err != nil {
		log.Fatalf("Failed to read response body: %v", err)
	}

	// 4. MANUAL UNMARSHALING: Convert the raw JSON text bytes back into a Go struct
	var res ProfileResponse
	err = json.Unmarshal(bodyBytes, &res)
	if err != nil {
		log.Fatalf("Failed to unmarshal JSON: %v", err)
	}

	fmt.Printf("Data received via REST JSON! Name: %s\n", res.Name)
}
```
{% endcode %}

Server:

{% code lineNumbers="true" expandable="true" %}
```go
package main

import (
	"encoding/json"
	"log"
	"net/http"
)

// ProfileResponse defines the JSON structure we will send back
type ProfileResponse struct {
	Name    string `json:"name"`
	Age     int    `json:"age"`
	Mobile  string `json:"mobile"`
	Address string `json:"address"`
}

func getProfileHandler(w http.ResponseWriter, r *http.Request) {
	// 1. Extract the data from the URL query parameters (e.g., ?user_id=compute_node_99)
	userID := r.URL.Query().Get("user_id")
	log.Printf("REST Server: Serving request for User ID: %s", userID)

	// 2. Simulate database lookup
	res := ProfileResponse{
		Name:    "Hypervisor-Cluster-Alpha",
		Age:     4,
		Mobile:  "+1-000-555-0100",
		Address: "Rack 4, Data Center East",
	}

	// 3. MANUAL MARSHALING: Convert the Go struct into plain text JSON
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusOK)
	json.NewEncoder(w).Encode(res) 
}

func main() {
	// Register the endpoint route
	http.HandleFunc("/api/v1/profile", getProfileHandler)

	log.Println("REST Server listening on :8080...")
	if err := http.ListenAndServe(":8080", nil); err != nil {
		log.Fatalf("Failed to start server: %v", err)
	}
}
```
{% endcode %}





## &#x20;4 communication modes in gRPC.

* **Unary** – Client sends single request, server sends single response. _(Standard HTTP request/response)_
* **Server Streaming** – Client sends one request, server streams back multiple responses.
* **Client Streaming** – Client streams multiple requests, server sends one response.
* **Bidirectional Streaming** – Both sides stream independently (full-duplex).

\
\
**1. The `profile.proto` File**

```proto
syntax = "proto3";

package profile;
option go_package = "./pb";

service ProfileService {
  // 1. Unary: Single request, single response
  rpc GetProfile (ProfileRequest) returns (ProfileResponse);

  // 2. Server Streaming: Single request, stream of responses
  rpc StreamProfileUpdates (ProfileRequest) returns (stream ProfileResponse);

  // 3. Client Streaming: Stream of requests, single response
  rpc UploadProfilePictures (stream UploadPictureRequest) returns (UploadStatusResponse);

  // 4. Bidirectional Streaming: Stream of requests, stream of responses
  rpc ChatSupport (stream ChatMessage) returns (stream ChatMessage);
}

message ProfileRequest { string user_id = 1; }
message ProfileResponse { string name = 1; string email = 2; }
message UploadPictureRequest { bytes chunk = 1; }
message UploadStatusResponse { bool success = 1; }
message ChatMessage { string user = 1; string text = 2; }
```

2. #### How the Go Client Uses Each Mode

Once `protoc` compiles this file, your Go client code interacts with each mode using distinct streaming interfaces. Here is how you write the client side for all four:

#### Mode 1: Unary (Standard Call)

```go
// Direct request and response
res, err := client.GetProfile(ctx, &pb.ProfileRequest{UserId: "123"})
fmt.Println("Profile:", res.GetName())
```

#### **Mode 2: Server Streaming (Receiving a Stream)**&#x20;

```go
// The server returns a stream object
stream, err := client.StreamProfileUpdates(ctx, &pb.ProfileRequest{UserId: "123"})

for {
    // We call Recv() in a loop to read incoming updates
    msg, err := stream.Recv()
    if err == io.EOF {
        break // 🏁 Server closed the stream safely
    }
    fmt.Println("New update received:", msg.GetName())
}
```

#### **Mode 3: Client Streaming (Sending a Stream) 📥**

```go
// We open the stream first
stream, err := client.UploadProfilePictures(ctx)

for _, chunk := range imageChunks {
    // We call Send() to push data bit by bit
    stream.Send(&pb.UploadPictureRequest{Chunk: chunk})
}

// Block and wait for the server's single final response
reply, err := stream.CloseAndRecv()
fmt.Println("Upload success:", reply.GetSuccess())
```

#### Mode 4: Bidirectional Streaming (Two-Way Street)

```go
stream, err := client.ChatSupport(ctx)

// 1. Spawn a goroutine to listen for incoming server messages concurrently
go func() {
    for {
        in, err := stream.Recv()
        if err == io.EOF { return }
        fmt.Printf("Support: %s\n", in.GetText())
    }
}()

// 2. Send messages from the main loop whenever you want
for {
    stream.Send(&pb.ChatMessage{User: "User1", Text: "Hello!"})
    time.Sleep(time.Second)
}
```
