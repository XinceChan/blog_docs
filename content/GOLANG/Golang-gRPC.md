```json
{
  "date":"2023.01.18 23:40",
  "author":"XinceChan",
  "tags":["Golang", "gRPC"],
  "musicId":"1977749761"
}
```

RPC(Remote Procedure Call, 远程过程调用)是一种不需要了解底层网络技术就可以通过网络从远程计算机程序上请求服务的协议。RPC协议假定存在某些传输协议（如TCP或UDP），并通过这些协议在通信程序之间传输数据信息。

### Go RPC

gRPC是谷歌开源的一款跨平台、高性能发的RPC框架，它可以在任何环境下运行。在实际开发过程中，主要使用它来进行后端微服务的开发。

在gRPC框架中，客户端应用程序可以像本地对象那样直接调用另一台计算机上的服务器应用程序中的方法，从而更容易地创建分布式应用程序和服务。与许多RPC系统一样，gRPC系统一样，gRPC框架基于定义服务的思想，通过设计参数和返回类型来远程调用方法。在服务器端，实现这个接口并运行gRPC服务器以处理客户端调用。客户端提供方法（客户端与服务器端的方法相同）。

#### 安装protobuf

1. Install the protocol compiler plugins for Go using the following commands:

```go
$ go install google.golang.org/protobuf/cmd/protoc-gen-go@v1.28
$ go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@v1.2
```

2. Update your `PATH` so that the `protoc` compiler can find the plugins:

```go
$ export PATH="$PATH:$(go env GOPATH)/bin"
```

#### 定义protobuf文件

```protobuf
syntax="proto3";

option go_package = "./proto";

package greet_service;

service GreetService {
    rpc SayHello (NoParam) returns (HelloResponse);

    rpc SayHelloServerStreaming(NameList) returns (stream HelloResponse);

    rpc SayHelloClientStreaming(stream HelloRequest) returns (MessagesList);

    rpc SayHelloBidirectionalStreaming(stream HelloRequest) returns (stream HelloResponse);
}

message NoParam{};

message HelloRequest{
    string name = 1;
}

message HelloResponse{
    // [修饰符] 类型 字段名 = 标识符;
    string message = 1;
};

message NameList{
    repeated string names = 1;
}

message MessagesList{
    repeated string messages = 1;
}
```

运行如下命令：

```go
protoc --go_out=. --go-grpc_out=. proto/greet.proto
```

生成两个文件

```go
├── greet.pb.go
├── greet_grpc.pb.go
```

#### 服务器端代码编写

1. 实现相应的接口

```go
type GreetServiceServer interface {
	SayHello(context.Context, *NoParam) (*HelloResponse, error)
	SayHelloServerStreaming(*NameList, GreetService_SayHelloServerStreamingServer) error
	SayHelloClientStreaming(GreetService_SayHelloClientStreamingServer) error
	SayHelloBidirectionalStreaming(GreetService_SayHelloBidirectionalStreamingServer) error
	mustEmbedUnimplementedGreetServiceServer()
}
```

2. 使用gRPC建立服务，监听端口
3. 将实现的服务注册到gRPC中。

grpc-demo/server/main.go

```go
package main

import (
	"log"
	"net"

	pb "github.com/XinceChan/grpc-demo/proto"
	"google.golang.org/grpc"
)

const (
	port = ":8080"
)

type helloServer struct {
	pb.GreetServiceServer
}

func main() {
	lis, err := net.Listen("tcp", port)
	if err != nil {
		log.Fatalf("Failed to start the server %v\n", err)
	}
	grpcServer := grpc.NewServer()
	// 注册
	pb.RegisterGreetServiceServer(grpcServer, &helloServer{})
	log.Printf("server started at %v", lis.Addr())
	if err := grpcServer.Serve(lis); err != nil {
		log.Fatalf("Failed to start: %v", err)
	}
}
```

grpc-demo/server/unary.go

```go
func (s *helloServer) SayHello(ctx context.Context, req *pb.NoParam) (*pb.HelloResponse, error) {
	return &pb.HelloResponse{
		Message: "Hello",
	}, nil
}
```

grpc-demo/server/server_stream.go

```go
func (s *helloServer) SayHelloServerStreaming(req *pb.NameList, stream pb.GreetService_SayHelloServerStreamingServer) error {
	log.Printf("got request with names : %v", req.Names)
	for _, name := range req.Names {
		res := &pb.HelloResponse{
			Message: "Hello, " + name,
		}
		if err := stream.Send(res); err != nil {
			return err
		}
		time.Sleep(1 * time.Second)
	}
	return nil
}
```

grpc-demo/server/client_stream.go

```go
func (s *helloServer) SayHelloClientStreaming(stream pb.GreetService_SayHelloClientStreamingServer) error {
	var messages []string
	for {
		req, err := stream.Recv()
		if err == io.EOF {
			return stream.SendAndClose(&pb.MessagesList{Messages: messages})
		}
		if err != nil {
			return err
		}
		log.Printf("Got request with name: %v", req.Name)
		messages = append(messages, "Hello", req.Name)
	}
}
```

grpc-demo/server/bi_stream.go

```go
func (s *helloServer) SayHelloBidirectionalStreaming(stream pb.GreetService_SayHelloBidirectionalStreamingServer) error {
	for {
		req, err := stream.Recv()
		if err == io.EOF {
			return nil
		}
		if err != nil {
			return err
		}
		log.Printf("Got request with name: %v", req.Name)
		res := &pb.HelloResponse{
			Message: "Hello " + req.Name,
		}
		if err := stream.Send(res); err != nil {
			return err
		}
	}
}
```

客户端代码编写

grpc-demo/client/main.go

```go
package main

import (
	"log"

	pb "github.com/XinceChan/grpc-demo/proto"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
)

const (
	port = ":8080"
)

func main() {
	conn, err := grpc.Dial("localhost"+port, grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		log.Fatalf("did not connet: %v", err)
	}
	defer conn.Close()

	// 实例化
	client := pb.NewGreetServiceClient(conn)

	names := &pb.NameList{
		Names: []string{"Xince", "MingChen", "Alice", "Bob"},
	}

	// callSayHello(client)
	// callSayHelloServerStream(client, names)
	// callSayHelloClientStream(client, names)
	callHelloBidirectionalStream(client, names)
}
```

grpc-demo/client/unary.go

```go
func callSayHello(client pb.GreetServiceClient) {
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()

	res, err := client.SayHello(ctx, &pb.NoParam{})
	if err != nil {
		log.Fatalf("could not greet: %v", err)
	}
	log.Printf("%s", res.Message)
}
```

grpc-demo/client/server_stream.go

```go
func callSayHelloServerStream(client pb.GreetServiceClient, names *pb.NameList) {
	log.Printf("Streaming started")
	stream, err := client.SayHelloServerStreaming(context.Background(), names)
	if err != nil {
		log.Fatalf("could not send names: %v", err)
	}

	for {
		message, err := stream.Recv()
		if err == io.EOF {
			break
		}
		if err != nil {
			log.Fatalf("error while streaming %v", err)
		}
		log.Println(message)
	}
	log.Printf("Streaming finished")
}
```

grpc-demo/client/client_stream.go

```go
func callSayHelloClientStream(client pb.GreetServiceClient, names *pb.NameList) {
	log.Printf("Client streaming started")
	stream, err := client.SayHelloClientStreaming(context.Background())
	if err != nil {
		log.Fatalf("could not send names: %v", err)
	}

	for _, name := range names.Names {
		req := &pb.HelloRequest{
			Name: name,
		}
		if err := stream.Send(req); err != nil {
			log.Fatalf("Error while sending %v", err)
		}
		log.Printf("Sent the request with name: %s", name)
		time.Sleep(time.Second)
	}

	res, err := stream.CloseAndRecv()
	log.Printf("Client streaming finished")
	if err != nil {
		log.Fatalf("Error while receiving %v", err)
	}
	log.Printf("%v", res.Messages)
}
```

grpc-demo/client/bi_stream.go

```go
func callHelloBidirectionalStream(client pb.GreetServiceClient, names *pb.NameList) {
	log.Printf("Bidirectional Streaming Started")
	stream, err := client.SayHelloBidirectionalStreaming(context.Background())
	if err != nil {
		log.Fatalf("could not send names: %v", err)
	}
	waitc := make(chan struct{})

	go func() {
		for {
			message, err := stream.Recv()
			if err == io.EOF {
				break
			}
			if err != nil {
				log.Fatalf("Error while streaming %v", err)
			}
			log.Println(message)
		}
		close(waitc)
	}()

	for _, name := range names.Names {
		req := &pb.HelloRequest{
			Name: name,
		}
		if err := stream.Send(req); err != nil {
			log.Fatalf("Error while sending %v", err)
		}
		log.Printf("Sent the request with name: %s", name)
		time.Sleep(time.Second)
	}
	stream.CloseSend()
	<-waitc
	log.Printf("Bidirectional streaming finished")
}
```

### 测试输出

#### unary

```go
admin@192 server % go run *.go
2023/01/18 23:35:12 server started at [::]:8080

admin@192 client % go run *.go
2023/01/18 23:35:34 Hello
```

#### service_stream

```go
admin@192 client % go run *.go
2023/01/18 23:36:25 Streaming started
2023/01/18 23:36:25 message:"Hello, Xince"
2023/01/18 23:36:26 message:"Hello, MingChen"
2023/01/18 23:36:27 message:"Hello, Alice"
2023/01/18 23:36:28 message:"Hello, Bob"
2023/01/18 23:36:29 Streaming finished
```

#### client_stream

```go
admin@192 client % go run *.go
2023/01/18 23:36:59 Client streaming started
2023/01/18 23:36:59 Sent the request with name: Xince
2023/01/18 23:37:00 Sent the request with name: MingChen
2023/01/18 23:37:01 Sent the request with name: Alice
2023/01/18 23:37:02 Sent the request with name: Bob
2023/01/18 23:37:03 Client streaming finished
2023/01/18 23:37:03 [Hello Xince Hello MingChen Hello Alice Hello Bob]
```

#### bi_stream

```go
admin@192 client % go run *.go
2023/01/18 23:37:30 Bidirectional Streaming Started
2023/01/18 23:37:30 Sent the request with name: Xince
2023/01/18 23:37:30 message:"Hello Xince"
2023/01/18 23:37:31 Sent the request with name: MingChen
2023/01/18 23:37:31 message:"Hello MingChen"
2023/01/18 23:37:32 Sent the request with name: Alice
2023/01/18 23:37:32 message:"Hello Alice"
2023/01/18 23:37:33 Sent the request with name: Bob
2023/01/18 23:37:33 message:"Hello Bob"
2023/01/18 23:37:34 Bidirectional streaming finished
```

### 总结

Go语言已经提供了良好的RPC支持。通过gRPC，可以很方便地开发分布式的Web应用程序。

#### 相关参考：

gRPC: https://grpc.io/docs/languages/go/quickstart/

Protocol Buffer Compiler Installation: https://grpc.io/docs/protoc-installation/