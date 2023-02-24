#### Grpc介绍

gRPC是由Google公司开源的一款高性能的远程过程调用(RPC)框架，可以在任何环境下运行。该框架提供了负载均衡，跟踪，智能监控，身份验证等功能，可以实现系统间的高效连接。

#### gRPC官方网站

gRPC官方网站：[https://grpc.io/](https://grpc.io/)

#### gRPC源码

gRPC的官方源码库存放于github网站，可以公开访问。gRPC源码库主页链接如下：[https://github.com/grpc/grpc](https://github.com/grpc/grpc)

gRPC开源库支持诸如：C++，C#，Dart，Go，Java，Node，Objective-C，PHP，Python，Ruby，WebJS等多种语言，开发者可以自行在gRPC的github主页库选择查看对应语言的实现。

#### gRPC调用执行过程

因为gRPC支持多种语言的实现，因此gRPC支持客户端与服务器在多种语言环境中部署运行和互相调用。多语言环境交互示例如下图所示：

![多语言环境交互示例](/Users/macbookpro/Downloads/Golang-100-Days-master/Day81-82(gRPC远程调用机制)/img/WX20190802-154324@2x.png)

gRPC中默认采用的数据格式化方式是protocol buffers。

在服务器端，服务器实现服务声明的方法，并运行一个 gRPC 服务器来处理客户端发来的调用请求。gRPC 底层会对传入的请求进行解码，执行被调用的服务方法，并对服务响应进行编码。

在客户端，客户端有一个称为存根（stub）的本地对象，它实现了与服务相同的方法。然后，客户端可以在本地对象上调用这些方法，将调用的参数包装在适当的 protocol buffers 消息类型中—— gRPC 在向服务器发送请求并返回服务器的 protocol buffers 响应之后进行处理。



#### 安装grpc

```bash
go get -u google.golang.org/grpc
```



#### 安装protobuf相关库

1.安装proto编译器，从下面连接选择自己的系统，安装

https://github.com/protocolbuffers/protobuf/releases

在终端执行这个命令，查看protoc 是否安装完成

```bash
protoc --version
```

2.安装proto的go插件

这个插件会根据.proto生成一个.pb.go文件,包含所有`.proto`文件中定义的类型及其序列化方法。

```bash
go install google.golang.org/protobuf/cmd/protoc-gen-go@v1.28
```

终端执行，查看protoc-gen-go 安装完成

```bash
protoc-gen-go --version
```

注意:提示下面错误，需要给安装命令前加上sudo权限，下面的grpc插件安装出错也同样这样处理。

go install google.golang.org/protobuf/cmd/protoc-gen-go: copying /var/folders/wt/52g8qsv96_vbx744kz6spnpc0000gn/T/go-build2226110402/b001/exe/a.out: open /Users/macbookpro/go/bin/protoc-gen-go: permission denied

3.安装grpc插件

这个插件会生成一个后缀为`_grpc.pb.go`的文件，其中包含：

- 一种接口类型(或存根) ，供客户端调用的服务方法。
- 服务器要实现的接口类型。

```
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@v1.2
```

终端执行，查看protoc-gen-go-grpc 安装完成

```bash
protoc-gen-go-grpc --version
```



#### Grpc框架使用

##### 编写.proto文件定义服务

```protobuf
service HelloService {
  rpc SayHello (HelloRequest) returns (HelloResponse);
}

message HelloRequest {
  string greeting = 1;
}

message HelloResponse {
  string reply = 1;
}
```

说明:定义了一个HelloService服务

在gRPC中你可以定义四种类型的服务方法。

- 普通 rpc，客户端向服务器发送一个请求，然后得到一个响应，就像普通的函数调用一样。

  ```protobuf
  rpc SayHello(HelloRequest) returns (HelloResponse);
  ```

- 服务器流式 rpc，其中客户端向服务器发送请求，并获得一个流来读取一系列消息。客户端从返回的流中读取，直到没有更多的消息。gRPC 保证在单个 RPC 调用中的消息是有序的。

  ```protobuf
  rpc LotsOfReplies(HelloRequest) returns (stream HelloResponse);
  ```

- 客户端流式 rpc，其中客户端写入一系列消息并将其发送到服务器，同样使用提供的流。一旦客户端完成了消息的写入，它就等待服务器读取消息并返回响应。同样，gRPC 保证在单个 RPC 调用中对消息进行排序。

  ```protobuf
  rpc LotsOfGreetings(stream HelloRequest) returns (HelloResponse);
  ```

- 双向流式 rpc，其中双方使用读写流发送一系列消息。这两个流独立运行，因此客户端和服务器可以按照自己喜欢的顺序读写: 例如，服务器可以等待接收所有客户端消息后再写响应，或者可以交替读取消息然后写入消息，或者其他读写组合。每个流中的消息是有序的。

  ```
  rpc LotsOfGreetings(stream HelloRequest) returns (stream HelloResponse);
  ```

  

#### 实战

**普通rpc/一元rpc：客户端向服务器发送单个请求并获得单个响应，就像正常的函数调用一样。**

```
rpc SayHello(HelloRequest)  returns  (HelloResponse);
```

1.编写proto文件hello.proto

```protobuf

```

2.终端执行，生成grpc文件

```go
protoc --go_out=. --go_opt=paths=source_relative --go-grpc_out=. --go-grpc_opt=paths=source_relative pb/hello.proto
```

此时，目录结构：

```bash
grpc-go-example
	└── sayhello
    ├── pb
    		├── hello.pb.go
    	  └── hello.proto
    	  └── hello_grpc.go
```

3.server端代码

```go
package main

import (
   "context"
   "google.golang.org/grpc"
   "guokai.com/grpc-go-example/sayhello/pb"
   "log"
   "net"
)

type greeterServer struct {
   pb.UnimplementedGreeterServer
}

func (g *greeterServer) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
   log.Printf("Received: %v", in.GetName())
   return &pb.HelloReply{Message: "hello " + in.GetName()}, nil
}

func main() {
   listen, err := net.Listen("tcp", ":8088")
   if err != nil {
      log.Fatalf("failed to listen:%v", err)
   }
   s := grpc.NewServer() //创建grpc服务器
   //将服务描述(server)及其具体实现(greeterServer)注册到 gRPC 中去
   pb.RegisterGreeterServer(s, &greeterServer{})
   //启动服务
   if err := s.Serve(listen); err != nil {
      log.Fatalf("failed to serve: %v", err)
   }
}
```

说明：

​	1.定义一个结构体，必须包含`pb.UnimplementedGreeterServer` 对象；

​	2.实现 .proto文件中定义的API；

​	3.将服务描述及其具体实现注册到 gRPC 中；

​	4.启动服务。

4.client代码

```go
package main

import (
   "context"
   "google.golang.org/grpc"
   "google.golang.org/grpc/credentials/insecure"
   "guokai.com/grpc-go-example/sayhello/pb"
   "log"
   "os"
   "time"
)

var (
   defaultName = "world"
)

func main() {
   //创建客户端连接，此处禁用安全传输
   conn, err := grpc.Dial("127.0.0.1:8088", grpc.WithTransportCredentials(insecure.NewCredentials()))
   if err != nil {
      log.Fatalf("did not connect: %v", err)
   }
   defer conn.Close()
   c := pb.NewGreeterClient(conn)
   //通过命令行参数指定name
   name := defaultName
   if len(os.Args) > 1 {
      name = os.Args[1]
   }
   ctx, cancel := context.WithTimeout(context.Background(), time.Second)
   defer cancel()
   res, err := c.SayHello(ctx, &pb.HelloRequest{Name: name})
   if err != nil {
      log.Fatalf("could not greet: %v", err)
   }
   log.Printf("Greeting: %s", res.GetMessage())
}
```

说明：

​	1.首先使用 `grpc.Dial()` 与 gRPC 服务器建立连接；

​	2.使用`pb.NewGreeterClient(conn)`获取客户端；

​	3.通过客户端调用ServiceAPI方法`client.SayHello`。

5.测试

启动serve服务和client服务

go run server/main.go

go run client/main.go guok

结果:

```
2022/09/17 22:07:48 Greeting: hello guok
```

最终目录结构

```
grpc-go-example
	└── sayhello
		├── client
				├── main.go
    ├── pb
    		├── hello.pb.go
    	  └── hello.proto
    	  └── hello_grpc.go
    ├── server
				├── main.go
```



流式rpc

1.编写hello.proto

```protobuf
syntax = "proto3";

option go_package = "guokai.com/grpc-go-example/streamhello/pb";

package pb;

// Echo 服务，包含了4种类型API
service Echo {
  // 普通rpc
  rpc UnaryEcho(EchoRequest) returns (EchoResponse) {}
  // 服务流rpc
  rpc ServerStreamingEcho(EchoRequest) returns (stream EchoResponse) {}
  // 客户端流rpc
  rpc ClientStreamingEcho(stream EchoRequest) returns (EchoResponse) {}
  // 双向流rpc
  rpc BidirectionalStreamingEcho(stream EchoRequest) returns (stream EchoResponse) {}
}

message EchoRequest {
  string message = 1;
}

message EchoResponse {
  string message = 1;
}
```

在streamhello目录终端执行

```sh
protoc --go_out=. --go_opt=paths=source_relative --go-grpc_out=. --go-grpc_opt=paths=source_relative pb/hello.proto
```

此时目录结构：

```
grpc-go-example
	└── streamhello
    ├── pb
    		├── hello.pb.go
    	  └── hello.proto
    	  └── hello_grpc.go
```



服务器流式Rpc

服务端流：服务端可以发送多个数据给客户端。

使用场景: 例如图片处理的时候，客户端提供一张原图，服务端依次返回多个处理后的图片。

说明：可以多次调用 stream.Send() 来返回多个数据

1.服务端代码

```go
package main

import (
   "google.golang.org/grpc"
   "guokai.com/grpc-go-example/streamhello/pb"
   "log"
   "net"
)

type Echo struct {
   pb.UnimplementedEchoServer
}

// 服务流rpc
// ServerStreamingEcho 客户端发送一个请求 服务端以流的形式循环发送多个响应
/*
1. 获取客户端请求参数
2. 处理完成后返回多个响应
3. 最后返回nil表示完成响应
*/
func (e *Echo) ServerStreamingEcho(req *pb.EchoRequest, stream pb.Echo_ServerStreamingEchoServer) error {
   log.Printf("Recved %v", req.GetMessage())
   // 具体返回多少个response根据业务逻辑调整
   for i := 0; i < 2; i++ {
      //通过send方法不断推送数据
      err := stream.Send(&pb.EchoResponse{Message: req.GetMessage()})
      if err != nil {
         log.Fatalf("Send error:%v", err)
         return err
      }
   }
   // 返回nil表示完成响应
   return nil
}

func main() {
   listen, err := net.Listen("tcp", ":8088")
   if err != nil {
      log.Fatalf("failed to listen:%v", err)
   }
   s := grpc.NewServer() //创建grpc服务器
   //将服务描述(server)及其具体实现(echoServer)注册到 gRPC 中去
   pb.RegisterEchoServer(s, &Echo{})
   //启动服务
   if err := s.Serve(listen); err != nil {
      log.Fatalf("failed to serve: %v", err)
   }
}
```

2.客户端代码

说明：调用方法获取到的不是单纯的响应，而是一个 stream。通过 stream.Recv() 接收服务端的多个返回值。

```go
package main

import (
   "context"
   "google.golang.org/grpc"
   "google.golang.org/grpc/credentials/insecure"
   "guokai.com/grpc-go-example/streamhello/pb"
   "io"
   "log"
)

func main() {
   //建立连接，获取client，此处禁用安全传输
   conn, err := grpc.Dial("127.0.0.1:8088", grpc.WithTransportCredentials(insecure.NewCredentials()))
   if err != nil {
      log.Fatalf("did not connect: %v", err)
   }
   defer conn.Close()
   client := pb.NewEchoClient(conn)
   ServerStreamingEcho(client)
}

/*
1. 建立连接 获取client
2. 通过 client 获取stream
3. for循环中通过stream.Recv()依次获取服务端推送的消息
4. err==io.EOF则表示服务端关闭stream了
*/
func ServerStreamingEcho(client pb.EchoClient) {
   //2.调用获取stream
   stream, err := client.ServerStreamingEcho(context.Background(), &pb.EchoRequest{Message: "hello world"})
   if err != nil {
      log.Fatalf("could not echo: %v", err)
   }
   // 3. for循环获取服务端推送的消息
   for {
      // 通过 Recv() 不断获取服务端send()推送的消息
      resp, err := stream.Recv()
      // 4. err==io.EOF则表示服务端关闭stream了 退出
      if err == io.EOF {
         log.Println("server closed")
         break
      }
      if err != nil {
         log.Printf("Recv error:%v", err)
         continue
      }
      log.Printf("Recv data:%v", resp.GetMessage())
   }
}
```

结果：

服务端

```bash
macbookpro@bogon ~/go/src/grpc-go-example/streamhello/server-stream $ go run server.go 
2022/09/18 10:39:27 Recved hello world
```

客户端

```bash
macbookpro@bogon ~/go/src/grpc-go-example/streamhello/server-stream $ go run  client.go 
2022/09/18 10:39:27 Recv data:hello world
2022/09/18 10:39:27 Recv data:hello world
```



客户端流RPC

和 ServerStream 相反，客户端可以发送多个数据。

1.服务端代码

说明：循环调用 stream.Recv() 获取数据，err == io.EOF 则表示数据已经全部获取了，最后通过 stream.SendAndClose() 返回响应。

```go
package main

import (
   "google.golang.org/grpc"
   "guokai.com/grpc-go-example/streamhello/pb"
   "io"
   "log"
   "net"
)

type Echo struct {
   pb.UnimplementedEchoServer
}

// ClientStreamingEcho 客户端流
/*
1. for循环中通过stream.Recv()不断接收client传来的数据
2. err == io.EOF表示客户端已经发送完毕关闭连接了,此时等待服务端处理完并返回消息
3. stream.SendAndClose() 发送消息并关闭连接(虽然在客户端流里服务器这边并不需要关闭 但是方法还是叫的这个名字，内部也只会调用Send())
*/
func (e *Echo) ClientStreamingEcho(stream pb.Echo_ClientStreamingEchoServer) error {
   // 1.for循环接收客户端发送的消息
   for {
      // 2. 通过 Recv() 不断获取客户端 send()推送的消息
      req, err := stream.Recv() // Recv内部也是调用RecvMsg
      // 3. err == io.EOF表示已经获取全部数据
      if err == io.EOF { //客户端发送结束
         log.Println("client closed")
         // 4.SendAndClose 返回并关闭连接
         // 在客户端发送完毕后服务端即可返回响应
         return stream.SendAndClose(&pb.EchoResponse{Message: "ok"})
      }
      if err != nil {
         return err
      }
      log.Printf("Recved %v", req.GetMessage())
   }
}

func main() {
   listen, err := net.Listen("tcp", ":8088")
   if err != nil {
      log.Fatalf("failed to listen:%v", err)
   }
   s := grpc.NewServer() //创建grpc服务器
   //将服务描述(server)及其具体实现(echoServer)注册到 gRPC 中去
   pb.RegisterEchoServer(s, &Echo{})
   //启动服务
   if err := s.Serve(listen); err != nil {
      log.Fatalf("failed to serve: %v", err)
   }
}
```

2.客户端代码

说明：通过多次调用 stream.Send() 向服务端推送多个数据，最后调用 stream.CloseAndRecv() 关闭stream并接收服务端响应。

```go
package main

import (
   "context"
   "google.golang.org/grpc"
   "google.golang.org/grpc/credentials/insecure"
   "guokai.com/grpc-go-example/streamhello/pb"
   "log"
)

func main() {
   //建立连接，获取client，此处禁用安全传输
   conn, err := grpc.Dial("127.0.0.1:8088", grpc.WithTransportCredentials(insecure.NewCredentials()))
   if err != nil {
      log.Fatalf("did not connect: %v", err)
   }
   defer conn.Close()
   client := pb.NewEchoClient(conn)
   clientStream(client)
}

// clientStream 客户端流
/*
1. 建立连接并获取client
2. 获取 stream 并通过 Send 方法不断推送数据到服务端
3. 发送完成后通过stream.CloseAndRecv() 关闭steam并接收服务端返回结果
*/
func clientStream(client pb.EchoClient) {
   // 2.获取 stream 并通过 Send 方法不断推送数据到服务端
   stream, err := client.ClientStreamingEcho(context.Background())
   if err != nil {
      log.Fatalf("Sum() error: %v", err)
   }
   for i := 0; i < 2; i++ {
      err := stream.Send(&pb.EchoRequest{Message: "hello world"})
      if err != nil {
         log.Printf("send error: %v", err)
         continue
      }
   }
   // 3. 发送完成后通过stream.CloseAndRecv() 关闭steam并接收服务端返回结果
   // (服务端则根据err==io.EOF来判断client是否关闭stream)
   resp, err := stream.CloseAndRecv()
   if err != nil {
      log.Fatalf("CloseAndRecv() error: %v", err)
   }
   log.Printf("sum: %v", resp.GetMessage())
}
```

结果：

服务端

```bash
macbookpro@bogon ~/go/src/grpc-go-example/streamhello/client-stream $ go run server.go 
2022/09/18 10:53:24 Recved hello world
2022/09/18 10:53:24 Recved hello world
2022/09/18 10:53:24 client closed
```

客户端

```bash
macbookpro@bogon ~/go/src/grpc-go-example/streamhello/client-stream $ go run client.go
2022/09/18 10:53:24 sum: ok
```



双向流

双向推送流则可以看做是结合了 ServerStream 和 ClientStream 二者。客户端和服务端都可以向对方推送多个数据。

说明：服务端和客户端的这两个Stream 是独立的。

1.服务端代码

说明：一般使用两个 Goroutine，一个接收数据，一个推送数据。最后通过 return nil 表示已经完成响应。

```go
package main

import (
   "fmt"
   "google.golang.org/grpc"
   "guokai.com/grpc-go-example/streamhello/pb"
   "io"
   "log"
   "net"
   "sync"
)

type Echo struct {
   pb.UnimplementedEchoServer
}

// BidirectionalStreamingEcho 双向流服务端
/*
// 1. 建立连接 获取client
// 2. 通过client调用方法获取stream
// 3. 开两个goroutine（使用 chan 传递数据） 分别用于Recv()和Send()
// 3.1 一直Recv()到err==io.EOF(即客户端关闭stream)
// 3.2 Send()则自己控制什么时候Close 服务端stream没有close 只要跳出循环就算close了。 具体见https://github.com/grpc/grpc-go/issues/444
*/
func (e *Echo) BidirectionalStreamingEcho(stream pb.Echo_BidirectionalStreamingEchoServer) error {
   var (
      waitGroup sync.WaitGroup
      msgCh     = make(chan string)
   )
   waitGroup.Add(1)
   go func() {
      defer waitGroup.Done()
      for v := range msgCh {
         err := stream.Send(&pb.EchoResponse{Message: v})
         if err != nil {
            fmt.Println("Send error:", err)
            continue
         }
      }
   }()

   waitGroup.Add(1)
   go func() {
      defer waitGroup.Done()
      for {
         req, err := stream.Recv()
         if err == io.EOF {
            break
         }
         if err != nil {
            log.Fatalf("recv error:%v", err)
         }
         fmt.Printf("Recved :%v \n", req.GetMessage())
         msgCh <- req.GetMessage()
      }
   }()
   waitGroup.Wait()

   // 返回nil表示已经完成响应
   return nil
}

func main() {
   listen, err := net.Listen("tcp", ":8088")
   if err != nil {
      log.Fatalf("failed to listen:%v", err)
   }
   s := grpc.NewServer() //创建grpc服务器
   //将服务描述(server)及其具体实现(echoServer)注册到 gRPC 中去
   pb.RegisterEchoServer(s, &Echo{})
   //启动服务
   if err := s.Serve(listen); err != nil {
      log.Fatalf("failed to serve: %v", err)
   }
}
```

2.客户端代码

说明：和服务端类似，不过客户端推送结束后需要主动调用 stream.CloseSend() 函数来关闭Stream

```go
package main

import (
   "context"
   "fmt"
   "google.golang.org/grpc"
   "google.golang.org/grpc/credentials/insecure"
   "guokai.com/grpc-go-example/streamhello/pb"
   "io"
   "log"
   "sync"
   "time"
)

func main() {
   //建立连接，获取client，此处禁用安全传输
   conn, err := grpc.Dial("127.0.0.1:8088", grpc.WithTransportCredentials(insecure.NewCredentials()))
   if err != nil {
      log.Fatalf("did not connect: %v", err)
   }
   defer conn.Close()
   client := pb.NewEchoClient(conn)
   bidirectionalStream(client)
}

// bidirectionalStream 双向流
/*
1. 建立连接 获取client
2. 通过client获取stream
3. 开两个goroutine 分别用于Recv()和Send()
   3.1 一直Recv()到err==io.EOF(即服务端关闭stream)
   3.2 Send()则由自己控制
4. 发送完毕调用 stream.CloseSend()关闭stream 必须调用关闭 否则Server会一直尝试接收数据 一直报错...
*/
func bidirectionalStream(client pb.EchoClient) {
   var wg sync.WaitGroup
   // 2. 调用方法获取stream
   stream, err := client.BidirectionalStreamingEcho(context.Background())
   if err != nil {
      panic(err)
   }
   // 3.开两个goroutine 分别用于Recv()和Send()
   wg.Add(1)
   go func() {
      defer wg.Done()
      for {
         req, err := stream.Recv()
         if err == io.EOF {
            fmt.Println("Server Closed")
            break
         }
         if err != nil {
            continue
         }
         fmt.Printf("Recv Data:%v \n", req.GetMessage())
      }
   }()

   wg.Add(1)
   go func() {
      defer wg.Done()
      for i := 0; i < 2; i++ {
         err := stream.Send(&pb.EchoRequest{Message: "hello world"})
         if err != nil {
            log.Printf("send error:%v\n", err)
         }
         time.Sleep(time.Second)
      }
      // 4. 发送完毕关闭stream
      err := stream.CloseSend()
      if err != nil {
         log.Printf("Send error:%v\n", err)
         return
      }
   }()
   wg.Wait()
}
```

结果：

服务端

```
macbookpro@bogon ~/go/src/grpc-go-example/streamhello/both-stream $ go run server.go 
Recved :hello world 
Recved :hello world 
```

客户端

```
macbookpro@bogon ~/go/src/grpc-go-example/streamhello/both-stream $ go run client.go
Recv Data:hello world 
Recv Data:hello world 
```

总结：

客户端或者服务端都有对应的推送或者接收对象，只要不断循环 Recv(),或者 Send() 就能接收或者推送了！

gRPC Stream 和 goroutine 配合简直完美。通过 Stream 我们可以更加灵活的实现自己的业务。如订阅，大数据传输等。

**Client发送完成后需要手动调用Close()或者CloseSend()方法关闭stream，Server端则`return nil`就会自动 Close**。

**1）ServerStream**

- 服务端处理完成后`return nil`代表响应完成
- 客户端通过 `err == io.EOF`判断服务端是否响应完成

**2）ClientStream**

- 客户端发送完毕通过`CloseAndRecv关闭stream 并接收服务端响应
- 服务端通过 `err == io.EOF`判断客户端是否发送完毕，完毕后使用`SendAndClose`关闭 stream并返回响应。

**3）BidirectionalStream**

- 客户端服务端都通过stream向对方推送数据
- 客户端推送完成后通过`CloseSend关闭流，通过`err == io.EOF`判断服务端是否响应完成
- 服务端通过`err == io.EOF`判断客户端是否响应完成,通过`return nil`表示已经完成响应

通过`err == io.EOF`来判定是否把对方推送的数据全部获取到了。

客户端通过`CloseAndRecv`或者`CloseSend`关闭 Stream，服务端则通过`SendAndClose`或者直接 `return nil`来返回响应。



参考：

https://www.lixueduan.com/posts/grpc/03-stream/#3-unaryapi
