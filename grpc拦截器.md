#### 1.概述

gRPC 提供了 Interceptor （拦截器）功能，包括客户端拦截器和服务端拦截器。可以在接收到请求或者发起请求之前优先对请求中的数据做一些处理后再转交给指定的服务处理并响应，很适合在这里处理验证、日志等流程。

gRPC-go 在 v1.28.0版本增加了多 interceptor 支持，可以在不借助第三方库（go-grpc-middleware）的情况下添加多个 interceptor 了。

go-grpc-middleware 中也提供了多种常用 interceptor ，可以直接使用。

在 gRPC 中，根据拦截的方法类型不同可以分为拦截 Unary 方法的**一元拦截器**，和作用于 Stream 方法的**流拦截器**。

同时还分为**服务端拦截器**和**客户端拦截器**，所以一共有以下4种类型:

- grpc.UnaryServerInterceptor
- grpc.StreamServerInterceptor
- grpc.UnaryClientInterceptor
- grpc.StreamClientInterceptor

#### 2.定义

#### 客户端拦截器

使用客户端拦截器 只需要在 Dial的时候指定相应的 DialOption 即可。

##### Unary Interceptor 一元拦截器

客户端一元拦截器类型为 grpc.UnaryClientInterceptor，具体如下：

```
type UnaryClientInterceptor func(ctx context.Context, method string, req, reply interface{}, cc *ClientConn, invoker UnaryInvoker, opts ...CallOption) error
```

可以看到，所谓的**拦截器其实就是一个函数**，可以分为预处理(pre-processing)、调用RPC方法(invoking RPC method)、后处理(post-processing)三个阶段。

参数含义如下:

- ctx：Go语言中的上下文，一般和 Goroutine 配合使用，起到超时控制的效果
- method：当前调用的 RPC 方法名
- req：本次请求的参数，只有在处理前阶段修改才有效
- reply：本次请求响应，需要在处理后阶段才能获取到
- cc：gRPC 连接信息
- invoker：可以看做是当前 RPC 方法，一般在拦截器中调用 invoker 能达到调用 RPC 方法的效果，当然底层也是 gRPC 在处理。
- opts：本次调用指定的 options 信息

作为一个客户端拦截器，可以在处理前检查 req 看看本次请求带没带 token 之类的鉴权数据，没有的话就可以在拦截器中加上。

##### Stream Interceptor流拦截器

```
type StreamClientInterceptor func(ctx context.Context, desc *StreamDesc, cc *ClientConn, method string, streamer Streamer, opts ...CallOption) (ClientStream, error)
```

由于 StreamAPI 和 UnaryAPI有所不同，因此拦截器方面也有所区别，比如 req 参数变成了 streamer 。同时其拦截过程也有所不同，不在是处理 req resp，而是对 streamer 这个流对象进行包装，比如说实现自己的 SendMsg 和 RecvMsg 方法。

然后在这些方法中的预处理(pre-processing)、调用RPC方法(invoking RPC method)、后处理(post-processing)各个阶段加入自己的逻辑。

#### 服务端拦截器

服务端拦截器和客户端拦截器类似，就不做过多描述。使用客户端拦截器 只需要在 NewServer 的时候指定相应的 ServerOption 即可。

##### Unary Interceptor 一元拦截器

```
type UnaryServerInterceptor func(ctx context.Context, req interface{}, info *UnaryServerInfo, handler UnaryHandler) (resp interface{}, err error)
```

参数具体含义如下：

- ctx context.Context：请求上下文
- req interface{}：RPC 方法的请求参数
- info *UnaryServerInfo：RPC 方法的所有信息
- handler UnaryHandler：RPC 方法真正执行的逻辑

#### Stream Interceptor

```
type StreamServerInterceptor func(srv interface{}, ss ServerStream, info *StreamServerInfo, handler StreamHandler) error
```



##### 一元拦截器

一元拦截器可以分为3个阶段：

- 1）预处理(pre-processing)
- 2）调用RPC方法(invoking RPC method)
- 3）后处理(post-processing)



1.客户端代码

说明：invoker(ctx, method, req, reply, cc, opts...) 是真正调用 RPC 方法。因此我们可以在调用前后增加自己的逻辑：比如调用前检查一下参数之类的，调用后记录一下本次请求处理耗时等。

建立连接时通过 grpc.WithUnaryInterceptor 指定要加载的拦截器即可。

```go
package main

import (
   "context"
   "flag"
   "fmt"
   "google.golang.org/grpc"
   "google.golang.org/grpc/credentials"
   "guokai.com/grpc-go-example/streamhello/pb"
   "log"
   "time"
)

// unaryInterceptor 一个简单的 unary interceptor 示例。
func unaryInterceptor(ctx context.Context, method string, req, reply interface{}, cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
   // pre-processing
   start := time.Now()
   log.Println("客户端发送数据之前.....")
   err := invoker(ctx, method, req, reply, cc, opts...) // invoking RPC method
   // post-processing
   end := time.Now()
   log.Println("客户端发送数据之后.....")
   log.Printf("RPC: %s, req:%v start time: %s, end time: %s, err: %v", method, req, start.Format(time.RFC3339), end.Format(time.RFC3339), err)
   return err
}

func callUnaryEcho(client pb.EchoClient, message string) {
   ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
   defer cancel()
   resp, err := client.UnaryEcho(ctx, &pb.EchoRequest{Message: message})
   if err != nil {
      log.Fatalf("client.UnaryEcho(_) = _, %v: ", err)
   }
   fmt.Println("UnaryEcho: ", resp.Message)
}


func main() {
   flag.Parse()
   certFile := "./server.crt"
   creds, err := credentials.NewClientTLSFromFile(certFile,"")
   if err != nil {
      log.Fatalf("failed to load credentials: %v", err)
   }

   // 建立连接时指定要加载的拦截器
   conn, err := grpc.Dial("127.0.0.1:8088", grpc.WithTransportCredentials(creds), grpc.WithUnaryInterceptor(unaryInterceptor))
   if err != nil {
      log.Fatalf("did not connect: %v", err)
   }
   defer conn.Close()

   client := pb.NewEchoClient(conn)
   callUnaryEcho(client,"hello world")
}
```



2.服务端代码

说明：服务端的一元拦截器和客户端类似，服务端是在 NewServer 时指定拦截器：

```go
package main

import (
   "context"
   "flag"
   "google.golang.org/grpc"
   "google.golang.org/grpc/credentials"
   "guokai.com/grpc-go-example/streamhello/pb"
   "log"
   "net"
   "time"
)

type Echo struct {
   pb.UnimplementedEchoServer
}

func (e *Echo) UnaryEcho(ctx context.Context, req *pb.EchoRequest) (*pb.EchoResponse, error) {
   log.Printf("Recved: %v", req.GetMessage())
   resp := &pb.EchoResponse{Message: req.GetMessage()}
   return resp, nil
}

// unaryInterceptor 一元拦截器：记录请求日志
func unaryInterceptorServer(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
   start := time.Now()
   log.Println("服务端执行方法之前....")
   m, err := handler(ctx, req)
   end := time.Now()
   // 记录请求参数 耗时 错误信息等数据
   log.Println("服务端执行方法之后....")
   log.Printf("RPC: %s,req:%v start time: %s, end time: %s, err: %v", info.FullMethod, req, start.Format(time.RFC3339), end.Format(time.RFC3339), err)
   return m, err
}

func main() {
   flag.Parse()

   listen, err := net.Listen("tcp", ":8088")
   if err != nil {
      log.Fatalf("failed to listen:%v", err)
   }

   // 指定使用服务端证书创建一个 TLS credentials。
   certFile := "./server.crt"
   keyFile := "./server.key"
   creds, err := credentials.NewServerTLSFromFile(certFile, keyFile)
   if err != nil {
      log.Fatalf("failed to create credentials: %v", err)
   }
   // 指定使用 TLS credentials。
   s := grpc.NewServer(grpc.Creds(creds),grpc.UnaryInterceptor(unaryInterceptorServer)) //创建grpc服务器
   //将服务描述(server)及其具体实现(echoServer)注册到 gRPC 中去
   pb.RegisterEchoServer(s, &Echo{})
   log.Println("Server gRPC on 0.0.0.0:8088")
   //启动服务
   if err := s.Serve(listen); err != nil {
      log.Fatalf("failed to serve: %v", err)
   }
}
```

结果：

服务端

```
2022/09/18 23:43:00 Server gRPC on 0.0.0.0:8088
2022/09/18 23:43:10 服务端执行之前....
2022/09/18 23:43:10 Recved: hello world
2022/09/18 23:43:10 RPC: /pb.Echo/UnaryEcho,req:message:"hello world" start time: 2022-09-18T23:43:10+08:00, end time: 2022-09-18T23:43:10+08:00, err: <nil>
```

客户端

```
macbookpro@bogon ~/go/src/grpc-go-example/streamhello/interceptor $ go run client.go 
2022/09/18 23:43:10 客户端发送数据之前.....
2022/09/18 23:43:10 RPC: /pb.Echo/UnaryEcho, req:message:"hello world" start time: 2022-09-18T23:43:10+08:00, end time: 2022-09-18T23:43:10+08:00, err: <nil>
Recved:hello world 
```



##### 流拦截器

流拦截器过程和一元拦截器有所不同，同样可以分为3个阶段：

- 1）预处理(pre-processing)
- 2）调用RPC方法(invoking RPC method)
- 3）后处理(post-processing)

预处理阶段和一元拦截器类似，但是调用RPC方法和后处理这两个阶段则完全不同。

StreamAPI 的请求和响应都是通过 Stream 进行传递的，更进一步是通过 Streamer 调用 SendMsg 和 RecvMsg 这两个方法获取的。

然后 Streamer 又是调用RPC方法来获取的，所以在流拦截器中我们可以**对 Streamer 进行包装，然后实现 SendMsg 和 RecvMsg 这两个方法**。

1.客户端代码

说明：连接时通过 grpc.WithStreamInterceptor 指定要加载的拦截器

```go
package main

import (
   "context"
   "flag"
   "fmt"
   "google.golang.org/grpc"
   "google.golang.org/grpc/credentials"
   "guokai.com/grpc-go-example/streamhello/pb"
   "io"
   "log"
   "time"
)

// wrappedStream  用于包装 grpc.ClientStream 结构体并拦截其对应的方法。
type wrappedStream struct {
   grpc.ClientStream
}

func newWrappedStream(s grpc.ClientStream) grpc.ClientStream {
   return &wrappedStream{s}
}

func (w *wrappedStream) RecvMsg(m interface{}) error {
   log.Printf("Receive a message (Type: %T) at %v", m, time.Now().Format(time.RFC3339))
   return w.ClientStream.RecvMsg(m)
}

func (w *wrappedStream) SendMsg(m interface{}) error {
   log.Printf("Send a message (Type: %T) at %v", m, time.Now().Format(time.RFC3339))
   return w.ClientStream.SendMsg(m)
}

// streamInterceptor 一个简单的 stream interceptor 示例。
func streamInterceptor(ctx context.Context, desc *grpc.StreamDesc, cc *grpc.ClientConn, method string, streamer grpc.Streamer, opts ...grpc.CallOption) (grpc.ClientStream, error) {
   s, err := streamer(ctx, desc, cc, method, opts...)
   if err != nil {
      return nil, err
   }
   return newWrappedStream(s), nil
}

//客户端流
func callBidiStreamingEcho(client pb.EchoClient) {
   ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
   defer cancel()
   c, err := client.BidirectionalStreamingEcho(ctx)
   if err != nil {
      return
   }
   for i := 0; i < 2; i++ {
      if err := c.Send(&pb.EchoRequest{Message: fmt.Sprintf("Request %d", i+1)}); err != nil {
         log.Fatalf("failed to send request due to error: %v", err)
      }
   }
   err = c.CloseSend()
   if err != nil {
      log.Printf("close send err:%v", err)
   }
   for {
      resp, err := c.Recv()
      if err == io.EOF {
         break
      }
      if err != nil {

         log.Fatalf("failed to receive response due to error: %v", err)
      }
      fmt.Println("BidiStreaming Echo: ", resp.Message)
   }
}

func main() {
   flag.Parse()

   certFile := "../server.crt"
   creds, err := credentials.NewClientTLSFromFile(certFile, "")
   if err != nil {
      log.Fatalf("failed to load credentials: %v", err)
   }

   // 建立连接时指定要加载的拦截器
   conn, err := grpc.Dial("127.0.0.1:8088", grpc.WithTransportCredentials(creds), grpc.WithStreamInterceptor(streamInterceptor),grpc.WithBlock())
   if err != nil {
      log.Fatalf("did not connect: %v", err)
   }
   defer conn.Close()

   client := pb.NewEchoClient(conn)
   callBidiStreamingEcho(client)
}
```



2.服务端代码

```go
package main

import (
   "flag"
   "fmt"
   "google.golang.org/grpc"
   "google.golang.org/grpc/credentials"
   "guokai.com/grpc-go-example/streamhello/pb"
   "io"
   "log"
   "net"
   "time"
)

type server struct {
   pb.UnimplementedEchoServer
}

//客户端流
func (s *server) BidirectionalStreamingEcho(stream pb.Echo_BidirectionalStreamingEchoServer) error {
   for {
      in, err := stream.Recv()
      if err != nil {
         if err == io.EOF {
            return nil
         }
         fmt.Printf("server: error receiving from stream: %v\n", err)
         return err
      }
      fmt.Printf("bidi echoing message %q\n", in.Message)
      err = stream.Send(&pb.EchoResponse{Message: in.Message})
      if err != nil {
         fmt.Printf("server: error send to stream: %v\n", err)
      }
   }
}

type wrappedStream struct {
   grpc.ServerStream
}

func newWrappedStream(s grpc.ServerStream) grpc.ServerStream {
   return &wrappedStream{s}
}

func (w *wrappedStream) RecvMsg(m interface{}) error {
   log.Printf("Receive a message (Type: %T) at %s", m, time.Now().Format(time.RFC3339))
   return w.ServerStream.RecvMsg(m)
}

func (w *wrappedStream) SendMsg(m interface{}) error {
   log.Printf("Send a message (Type: %T) at %v", m, time.Now().Format(time.RFC3339))
   return w.ServerStream.SendMsg(m)
}


func streamInterceptor(srv interface{}, ss grpc.ServerStream, info *grpc.StreamServerInfo, handler grpc.StreamHandler) error {
   // 包装 grpc.ServerStream 以替换 RecvMsg SendMsg这两个方法。
   err := handler(srv, newWrappedStream(ss))
   if err != nil {
      log.Printf("RPC failed with error %v", err)
   }
   return err
}

func main() {
   flag.Parse()

   listen, err := net.Listen("tcp", ":8088")
   if err != nil {
      log.Fatalf("failed to listen:%v", err)
   }

   certFile := "../server.crt"
   keyFile := "../server.key"
   creds, err := credentials.NewServerTLSFromFile(certFile, keyFile)
   if err != nil {
      log.Fatalf("failed to create credentials: %v", err)
   }

   s := grpc.NewServer(grpc.Creds(creds), grpc.StreamInterceptor(streamInterceptor))
   //将服务描述(server)及其具体实现(echoServer)注册到 gRPC 中去
   pb.RegisterEchoServer(s, &server{})
   log.Println("Server gRPC on 0.0.0.0：8088")
   //启动服务
   if err := s.Serve(listen); err != nil {
      log.Fatalf("failed to serve: %v", err)
   }
}
```

结果：

服务端

```
macbookpro@bogon ~/go/src/grpc-go-example/streamhello/streamInterceptor/seerver $ go run server.go
2022/09/19 10:06:11 Server gRPC on 0.0.0.0：8088
2022/09/19 10:06:21 Receive a message (Type: *pb.EchoRequest) at 2022-09-19T10:06:21+08:00
bidi echoing message "Request 1"
2022/09/19 10:06:21 Send a message (Type: *pb.EchoResponse) at 2022-09-19T10:06:21+08:00
2022/09/19 10:06:21 Receive a message (Type: *pb.EchoRequest) at 2022-09-19T10:06:21+08:00
bidi echoing message "Request 2"
2022/09/19 10:06:21 Send a message (Type: *pb.EchoResponse) at 2022-09-19T10:06:21+08:00
2022/09/19 10:06:21 Receive a message (Type: *pb.EchoRequest) at 2022-09-19T10:06:21+08:00
```

客户端

```
macbookpro@bogon ~/go/src/grpc-go-example/streamhello/streamInterceptor/client $ go run client.go 
2022/09/19 10:06:21 Send a message (Type: *pb.EchoRequest) at 2022-09-19T10:06:21+08:00
2022/09/19 10:06:21 Send a message (Type: *pb.EchoRequest) at 2022-09-19T10:06:21+08:00
2022/09/19 10:06:21 Receive a message (Type: *pb.EchoResponse) at 2022-09-19T10:06:21+08:00
BidiStreaming Echo:  Request 1
2022/09/19 10:06:21 Receive a message (Type: *pb.EchoResponse) at 2022-09-19T10:06:21+08:00
BidiStreaming Echo:  Request 2
2022/09/19 10:06:21 Receive a message (Type: *pb.EchoResponse) at 2022-09-19T10:06:21+08:00
```



#### 总结

1.拦截器可以分为4种类型

- grpc.UnaryServerInterceptor  一元服务端拦截器
- grpc.StreamServerInterceptor  流服务端拦截器
- grpc.UnaryClientInterceptor 一元客户端拦截器
- grpc.StreamClientInterceptor 流客户端拦截器

拦截器本质上就是一个特定类型的函数，所以实现拦截器只需要实现对应类型方法（**方法签名相同**）即可。

2.拦截器处理过程

**一元拦截器**

- 1）预处理
- 2）调用RPC方法
- 3）后处理

**流拦截器**

- 1）预处理
- 2）调用RPC方法 获取 Streamer
- 3）后处理
  - 调用 SendMsg 、RecvMsg 之前
  - 调用 SendMsg 、RecvMsg
  - 调用 SendMsg 、RecvMsg 之后

**配置多个拦截器时，会按照参数传入顺序依次执行**

**如果想配置一个 Recovery 拦截器则必须放在第一个**，放在最后则无法捕获前面执行的拦截器中触发的 panic。

同时也可以将 一元和流拦截器一起配置，gRPC 会根据不同方法选择对应类型的拦截器执行。

推荐一下这个 [go-grpc-middleware](https://github.com/grpc-ecosystem/go-grpc-middleware)，该仓库提供了多种常用拦截器。



参考：

https://www.lixueduan.com/posts/grpc/05-interceptor/