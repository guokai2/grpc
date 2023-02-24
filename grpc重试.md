gRPC 中已经内置了 retry 功能，可以直接使用，不需要我们手动来实现，非常方便。



为了测试 retry 功能，服务端做了一点调整。

记录客户端的请求次数，只有满足条件的那一次（这里就是请求次数模4等于0的那一次）才返回成功，其他时候都返回失败。



服务端代码

```go
package main

import (
   "context"
   "fmt"
   "google.golang.org/grpc"
   "google.golang.org/grpc/codes"
   "google.golang.org/grpc/status"
   "guokai.com/grpc-go-example/streamhello/pb"
   "log"
   "net"
   "sync"
)

//定义重试结构体
type failingServer struct {
   pb.UnimplementedEchoServer
   mu sync.Mutex

   reqCounter uint
   reqModulo  uint
}

// maybeFailRequest 手动模拟请求失败 一共请求n次，前n-1次都返回失败，最后一次返回成功。
func (s *failingServer) maybeFailRequest() error {
   s.mu.Lock()
   defer s.mu.Unlock()
   s.reqCounter++
   if (s.reqModulo > 0) && (s.reqCounter%s.reqModulo == 0) {
      return nil
   }

   return status.Errorf(codes.Unavailable, "maybeFailRequest: failing it")
}

//定义重试结构体实现的服务方法
func (s *failingServer) UnaryEcho(ctx context.Context, req *pb.EchoRequest) (*pb.EchoResponse, error) {
   if err := s.maybeFailRequest(); err != nil {
      log.Println("request failed count:", s.reqCounter)
      return nil, err
   }

   log.Println("request succeeded count:", s.reqCounter)
   return &pb.EchoResponse{Message: req.Message}, nil
}

func main() {
   lis, err := net.Listen("tcp", "127.0.0.1:8088")
   if err != nil {
      log.Fatalf("failed to listen: %v", err)
   }
   fmt.Println("listen on address", "127.0.0.1:8088")
   s := grpc.NewServer() //创建grpc服务器

   // 指定第4次请求才返回成功，用于测试 gRPC 的 retry 功能。
   failingservice := &failingServer{
      reqModulo: 4,
   }
   //将服务注册到grpc服务器
   pb.RegisterEchoServer(s, failingservice)
   if err := s.Serve(lis); err != nil {
      log.Fatalf("failed to serve: %v", err)
   }
}
```



客户端代码

```go
package main

import (
   "context"
   "google.golang.org/grpc"
   "guokai.com/grpc-go-example/streamhello/pb"
   "log"
   "time"
)

var (
   // 更多配置信息查看官方文档： https://github.com/grpc/grpc/blob/master/doc/service_config.md
   // service语法为<package>.<service> package就是proto文件中指定的package，service也是proto文件中指定的 Service Name。
   // method 可以不指定 即当前service下的所以方法都使用该配置。
   retryPolicy = `{
      "methodConfig": [{
        "name": [{"service": "pb.Echo","method":"UnaryEcho"}],
        "retryPolicy": {
           "MaxAttempts": 4,
           "InitialBackoff": ".01s",
           "MaxBackoff": ".01s",
           "BackoffMultiplier": 1.0,
           "RetryableStatusCodes": [ "UNAVAILABLE" ]
        }
      }]}`
)

func main() {
   //建立连接，通过grpc.WithDefaultServiceConfig()配置重试次数
   conn, err := grpc.Dial("127.0.0.1:8088", grpc.WithInsecure(), grpc.WithDefaultServiceConfig(retryPolicy))
   if err != nil {
      log.Fatalf("did not connect: %v", err)
   }
   defer func() {
      if e := conn.Close(); e != nil {
         log.Printf("failed to close connection: %s", e)
      }
   }()
   client := pb.NewEchoClient(conn)
   unary(client)
}

//一元rpc
func unary(client pb.EchoClient) {
   ctx, cancel := context.WithTimeout(context.Background(), time.Second)
   defer cancel()
   resp, err := client.UnaryEcho(ctx, &pb.EchoRequest{Message: "Try and Success"})
   if err != nil {
      log.Printf("UnaryEcho error:%v\n", err)
   }
   //打印服务器响应消息
   log.Printf("UnaryEcho reply: %v", resp.GetMessage())
}
```

结果：

服务端

```
macbookpro@guokai ~/go/src/grpc-go-example/streamhello/retry/server $ go run main.go 
listen on address 127.0.0.1:8088
2022/09/21 11:42:59 request failed count: 1
2022/09/21 11:42:59 request failed count: 2
2022/09/21 11:42:59 request failed count: 3
2022/09/21 11:42:59 request succeeded count: 4
```

客户端

```
macbookpro@guokai ~/go/src/grpc-go-example/streamhello/retry/client $ go run main.go 
2022/09/21 11:42:59 UnaryEcho reply: Try and Success
```



配置说明

Service Config 是以 JSON 格式配置的，具体文档见 [service_config.md](https://github.com/grpc/grpc/blob/master/doc/service_config.md)。



```json
{
		"methodConfig": [{
		  "name": [{"service": "pb.Echo","method":"UnaryEcho"}],
          "wait_for_ready": false,
          "timeout": 1000ms,
          "max_request_message_bytes": 1024,
          "max_response_message_bytes": 1024,
      //下面2个是重点，重试策略和对冲策略，一个rpc只能配置一种重试策略
      //重试必须要服务端响应后才会发起请求
		  "retryPolicy": {
			  "maxAttempts": 4,
			  "initialBackoff": ".01s",
			  "maxBackoff": ".01s",
			  "backoffMultiplier": 1.0,
			  "retryableStatusCodes": [ "UNAVAILABLE" ]
		  },
			//对冲指在不影响响应情况主动发送单次调用的多个请求
			//过程:首先会像正常的 RPC 调用一样发送第一次请求，如果 hedgingDelay 时间内没有响应，那么直接发送第二次请求，以此类推，直到发送了 maxAttempts 次
		  "hedgingPolicy":{
              "maxAttempts":4,
              "hedgingDelay":"0.1s",
              "nonFatalStatusCodes": [ "" ]
          }}]
}

```



重试配置

```json
{
		"methodConfig": [{
		  "name": [{"service": "echo.Echo","method":"UnaryEcho"}],
		  "retryPolicy": {
			  "MaxAttempts": 4,
			  "InitialBackoff": ".01s",
			  "MaxBackoff": ".01s",
			  "BackoffMultiplier": 1.0,
			  "RetryableStatusCodes": [ "UNAVAILABLE" ]
		  }}]
}
```

说明：

1.name 配置信息作用的 RPC 服务或方法

- service：通过服务名匹配，语法为`<package>.<service>`package就是proto文件中指定的package，service也是proto文件中指定的 Service Name。
- method：匹配具体某个方法，proto文件中定义的方法名。

 2.retryPolicy重试策略

- MaxAttempts：最大尝试次数
- InitialBackoff：默认退避时间
- MaxBackoff：最大退避时间
- BackoffMultiplier：退避时间增加倍率
- RetryableStatusCodes：服务端返回什么错误码才重试

重试机制一般会搭配退避算法一起使用。



过程：第一次请求失败后，等1秒（随便取的一个数）再次请求，又失败后就等2秒在请求，一直重试直到超过指定重试次数或者等待时间就不在重试。



如果不使用退避算法，失败后就一直重试只会增加服务器的压力。

因为服务器压力大，导致的请求失败，那么根据退避算法等待一定时间后再次请求可能就能成功。反之直接请求可能会因为压力过大导致服务崩溃。

- 第一次重试间隔是 `random(0, initialBackoff)`
- 第 n 次的重试间隔为 `random(0, min( initialBackoff*backoffMultiplier**(n-1) , maxBackoff))`