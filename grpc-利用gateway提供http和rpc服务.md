gRPC-Gateway 是Google protocol buffers compiler(protoc)的一个插件。读取 protobuf 定义然后生成反向代理服务器，将 RESTful HTTP API 转换为 gRPC。

换句话说就是将 gRPC 转为 RESTful HTTP API。



1. HTTP 请求到达 gRPC-Gateway ，它将 JSON 数据解析为 Protobuf 消息。
2. 使用解析的 Protobuf 消息发出正常的 Go gRPC 客户端请求。
3. Go gRPC 客户端将 Protobuf 结构编码为 Protobuf 二进制格式，然后将其发送到 gRPC 服务器。
4. gRPC 服务器处理请求并以 Protobuf 二进制格式返回响应。
5. Go gRPC 客户端将其解析为 Protobuf 消息，并将其返回给 gRPC-Gateway
6. gRPC-Gateway将 Protobuf 消息编码为 JSON 并返回给原始客户端。



#### gRPC-Gateway 安装

```
go get github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-grpc-gateway
```



#### 实战

定义proto文件gateway.proto

```protobuf
//声明proto的版本
syntax = "proto3";
//定义编译文件输出目录
option go_package = "guokai.com/grpc-go-example/gateway/pb";
//定义包名
package pb;

//引入其他proto
import "google/api/annotations.proto";

//定义Greeter服务
service Greeter {
  rpc SayHello (HelloRequest) returns (HelloReply) {
    option (google.api.http) = {
      post: "/v1/greeter/sayhello",
      body: "*"
    };
//    option (google.api.http) = {
//      get: "/v1/greeter/sayhello",
//    };
  }
}

message HelloRequest {
  string name = 1;
}

message HelloReply {
  string message = 1;
}
```



注意：proto文件里面引入了google/api/annotations.proto，和google/api/http.proto

需要从连接下载上面的文件，保存到proto编译器的include目录或者项目目录

https://github.com/googleapis/googleapis/tree/master/google/api

<img src="/Users/macbookpro/Library/Application Support/typora-user-images/image-20220919160601941.png" alt="image-20220919160601941" style="zoom:67%;" />

终端执行命令

```bash
protoc --proto_path=./ --go_out=./ --go_opt=paths=source_relative --go-grpc_out=./ --go-grpc_opt=paths=source_relative --grpc-gateway_out=./ --grpc-gateway_opt=paths=source_relative pb/gateway.proto
```

最终目录结构

```
grpc-go-example
└── gateway
    ├── pb
        ├── gateway.pb.go
        ├── gateway.pb.gw.go
        ├── gateway.protoo
        └── gateway_grpc.pb.go
```

说明：

和普通rpc的proto相比

1.引入annotations.proto

2.增加 http 相关注解

```
rpc SayHello (HelloRequest) returns (HelloReply) {
    option (google.api.http) = {
      post: "/v1/greeter/sayhello"
      body: "*"
    };
  }
```

每个方法都必须添加 google.api.http 注解后 gRPC-Gateway 才能生成对应 http 方法。

post为请求方式   /v1/greeter/sayhello 为请求路径

更多语法查看 https://github.com/googleapis/googleapis/blob/master/google/api/http.proto

body 字段用于接收http request body 中的参数，不指定则无法接收。



客户端代码

和普通rpc一样

```go
package main

import (
   "context"
   "google.golang.org/grpc"
   "guokai.com/grpc-go-example/gateway/pb"
   "log"
)

func main() {
   conn, err := grpc.Dial("127.0.0.1:8088", grpc.WithInsecure())
   if err != nil {
      panic(err)
   }
   defer conn.Close()
   c := pb.NewGreeterClient(conn)
   r, err := c.SayHello(context.Background(), &pb.HelloRequest{Name: "world"})
   if err != nil {
      log.Fatalf("could not greet: %v", err)
   }
   log.Printf("Greeting: %s", r.Message)
}
```



服务端代码

原有 gRPC 基础上在启动一个 HTTP 服务

```go
package main

import (
   "context"
   "github.com/grpc-ecosystem/grpc-gateway/v2/runtime"
   "google.golang.org/grpc"
   "guokai.com/grpc-go-example/gateway/pb"
   "log"
   "net"
   "net/http"
)

type server struct {
   pb.UnimplementedGreeterServer
}

func (s *server) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
   return &pb.HelloReply{Message: "hello " + in.Name}, nil
}

func main() {

   // 创建grpc服务器
   s := grpc.NewServer()
   // 将Greeter 服务绑定到服务器
   pb.RegisterGreeterServer(s, &server{})
   // 监听端口
   lis, err := net.Listen("tcp", "127.0.0.1:8088")
   if err != nil {
      log.Fatalf("failed to listen: %v", err)
   }
   log.Println("Serving gRPC on 0.0.0.0:8088")
   go func() {
      //启动服务
      if err := s.Serve(lis); err != nil {
         log.Fatalf("failed to serve: %v", err)
      }
   }()

   // 2. 启动 HTTP 服务
   // Create a client connection to the gRPC server we just started
   // This is where the gRPC-Gateway proxies the requests
   conn, err := grpc.Dial(
      "127.0.0.1:8088",
      grpc.WithInsecure(),
   )
   if err != nil {
      log.Fatalln("Failed to dial server:", err)
   }

   gwmux := runtime.NewServeMux()
   // Register Greeter
   err = pb.RegisterGreeterHandler(context.Background(), gwmux, conn)
   if err != nil {
      log.Fatalln("Failed to register gateway:", err)
   }

   gwServer := &http.Server{
      Addr:    "127.0.0.1:8087", //定义http端口
      Handler: gwmux,
   }

   log.Println("Serving gRPC-Gateway on http://127.0.0.1:8087")
   log.Fatalln(gwServer.ListenAndServe())
}
```

结果测试

grpc请求

客户端

```
macbookpro@bogon ~/go/src/grpc-go-example/gateway/client $ go run main.go           
2022/09/19 16:53:41 Greeting: hello world
```

http请求

```
curl -X POST http://127.0.0.1:8087/v1/greeter/sayhello -d '{"name": "world"}'
```

结果

```
{"message":"hello world"}
```



参考

https://www.lixueduan.com/posts/grpc/07-grpc-gateway/