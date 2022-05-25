# 环境

macOS Catalina 10.15.7

go version go1.15.2 darwin/amd64



# 准备

1. [安装编码器 protoc](../protobuf)；
2. 安装go插件 `go install google.golang.org/protobuf/cmd/protoc-gen-go`；
3. 安装grpc插件 `go install google.golang.org/grpc/cmd/protoc-gen-go-grpc`；

  go install默认会将插件安装到`$GOPATH/bin`



# server项目结构

```shell
grpc-server $ pwd
/Users/demon/Desktop/test/gotest/grpc-server
 
grpc-server $ tree
.
├── go.mod
├── main.go
└── pb
    └── hello.proto
```



**go.mod**

```go
module gserver

go 1.15

```



**hello.proto**

```protobuf
syntax = "proto3"; // 版本声明，使用Protocol Buffers v3版本

option go_package = "./pb";  // 指定go package名称；xx根据需要替换

package pb; // 包名

// 定义一个打招呼服务
service Greeter {
    // SayHello 方法
    rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// 包含人名的一个请求消息
message HelloRequest {
    string name = 1;
}

// 包含问候语的响应消息
message HelloReply {
    string message = 1;
}
```



**编译生成**

```shell
grpc-server $ protoc -I=. --go_out=. --go-grpc_out=.  ./pb/hello.proto


grpc-server $ tree
.
├── go.mod
├── main.go
└── pb
    ├── hello.pb.go
    ├── hello.proto
    └── hello_grpc.pb.go
```



**main内容**

```go
package main

import (
	"context"
	"fmt"
	"log"
	"net"

	"google.golang.org/grpc"
	"gserver/pb"
)


// server is used to implement helloworld.GreeterServer.
type server struct {
	pb.UnimplementedGreeterServer
}

// SayHello implements helloworld.GreeterServer
func (s *server) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
	log.Printf("Received: %v", in.GetName())
	return &pb.HelloReply{Message: "Hello " + in.GetName()}, nil
}

func main() {
	port := 9999
	lis, err := net.Listen("tcp", fmt.Sprintf(":%d", port))
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}
	s := grpc.NewServer()
	pb.RegisterGreeterServer(s, &server{})
	log.Printf("server listening at %v", lis.Addr())
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}
```



**跑server**

```go
grpc-server $ go run main.go 
go: finding module for package google.golang.org/grpc
go: finding module for package google.golang.org/grpc/status
go: finding module for package google.golang.org/grpc/codes
go: finding module for package google.golang.org/protobuf/runtime/protoimpl
go: finding module for package google.golang.org/protobuf/reflect/protoreflect
go: downloading google.golang.org/protobuf v1.28.0
go: found google.golang.org/grpc in google.golang.org/grpc v1.46.2
go: found google.golang.org/grpc/codes in google.golang.org/grpc v1.46.2
go: found google.golang.org/grpc/status in google.golang.org/grpc v1.46.2
go: found google.golang.org/protobuf/reflect/protoreflect in google.golang.org/protobuf v1.28.0
go: found google.golang.org/protobuf/runtime/protoimpl in google.golang.org/protobuf v1.28.0
go: downloading golang.org/x/net v0.0.0-20201021035429-f5854403a974
go: downloading google.golang.org/genproto v0.0.0-20200526211855-cb27e3aa2013
go: downloading golang.org/x/text v0.3.3
2022/05/18 15:20:04 server listening at [::]:9999

```



**server端源码分析**





# client项目结构

```shell
grpc-client $ pwd
/Users/demon/Desktop/test/gotest/grpc-client

grpc-client $ tree
.
├── go.mod
├── go.sum
├── main.go
└── pb
    ├── hello.pb.go
    ├── hello.proto
    └── hello_grpc.pb.go
```

pb下的文件夹从server里拷贝过来。

**main文件内容如下：**

```go
package main

import (
	"context"
	"flag"
	"log"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
	"gclient/pb"
)

const (
	defaultName = "world"
)

var (
	addr = flag.String("addr", "localhost:9999", "the address to connect to")
	name = flag.String("name", defaultName, "Name to greet")
)

func main() {
	flag.Parse()
	// Set up a connection to the server.
	conn, err := grpc.Dial(*addr, grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}
	defer conn.Close()
	c := pb.NewGreeterClient(conn)

	// Contact the server and print out its response.
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()
	r, err := c.SayHello(ctx, &pb.HelloRequest{Name: *name})
	if err != nil {
		log.Fatalf("could not greet: %v", err)
	}
	log.Printf("Greeting: %s", r.GetMessage())
}
```



**跑client**

```go
grpc-client $ go run main.go 
go: finding module for package google.golang.org/grpc/credentials/insecure
go: finding module for package google.golang.org/protobuf/runtime/protoimpl
go: finding module for package google.golang.org/grpc
go: finding module for package google.golang.org/protobuf/reflect/protoreflect
go: finding module for package google.golang.org/grpc/codes
go: finding module for package google.golang.org/grpc/status
go: found google.golang.org/grpc in google.golang.org/grpc v1.46.2
go: found google.golang.org/grpc/credentials/insecure in google.golang.org/grpc v1.46.2
go: found google.golang.org/grpc/codes in google.golang.org/grpc v1.46.2
go: found google.golang.org/grpc/status in google.golang.org/grpc v1.46.2
go: found google.golang.org/protobuf/reflect/protoreflect in google.golang.org/protobuf v1.28.0
go: found google.golang.org/protobuf/runtime/protoimpl in google.golang.org/protobuf v1.28.0
2022/05/19 11:05:12 Greeting: Hello world

```





# 用到的包

## [net/url](https://pkg.go.dev/net/url#Parse)

```go
package main

import (
	"fmt"
	"net/url"
)

func main() {
	target := "localhost:9999"
	u, err := url.Parse(target)
	if err != nil {}
	fmt.Println("     Schema:", u.Scheme) // "localhost"
	fmt.Println("     Opaque:", u.Opaque)	// "9999"
	fmt.Println("       User:", u.User)
	fmt.Println("       Host:", u.Host)
	fmt.Println("       Path:", u.Path)
	fmt.Println("    RawPath:", u.RawPath)
	fmt.Println(" ForceQuery:", u.ForceQuery) // false
	fmt.Println("   Fragment:", u.Fragment)
	fmt.Println("RawFragment:", u.RawFragment)
  
	target = "passthrough:///localhost:9999"
	u, err = url.Parse(target)
	if err != nil {}
	fmt.Println("     Schema:", u.Scheme) // passthrough
	fmt.Println("     Opaque:", u.Opaque)
	fmt.Println("       User:", u.User)
	fmt.Println("       Host:", u.Host)
	fmt.Println("       Path:", u.Path)	// /localhost:9999
	fmt.Println("    RawPath:", u.RawPath)
	fmt.Println(" ForceQuery:", u.ForceQuery)	// false
	fmt.Println("   Fragment:", u.Fragment)
	fmt.Println("RawFragment:", u.RawFragment)
  
}

```





# 资料

官网: https://grpc.io/docs/languages/go/quickstart/