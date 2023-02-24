在web开发过程中，我们往往还会使用另外一种认证方式进行身份验证，那就是：Token认证。基于Token的身份验证是无状态，不需要将用户信息服务存在服务器或者session中。



##### token认证过程

1.客户端在发送请求前，首先向服务器发起请求，服务器返回一个生成的token给客户端。

2.客户端将token保存下来，用于后续每次请求时，携带着token参数。
3.服务端在进行处理请求之前，会首先对token进行验证，只有token验证成功了，才会处理并返回相关的数据。验证一般写在拦截器中。



##### gRPC的自定义Token认证

在 gRPC 中，允许开发者自定义自己的认证规则，通过credentials.PerRPCCredentials设置自定义认证规则：

```
type PerRPCCredentials interface {
	GetRequestMetadata(ctx context.Context, uri ...string) (map[string]string, error)
	RequireTransportSecurity() bool
}
```

说明：

GetRequestMetadata：以 map 的形式返回本次调用的授权信息，ctx 是用来控制超时的，并不是从这个 ctx 中获取。

RequireTransportSecurity ：指该 Credentials 的传输是否需要需要 TLS 加密，如果返回 true 则说明该 Credentials 需要在一个有 TLS 认证的安全连接上传输

因此，开发者实现上面接口，就可以定义自己的token信息。



自定义Token认证

```go
package auth

import (
   "context"
   "google.golang.org/grpc/codes"
   "google.golang.org/grpc/metadata"
   "google.golang.org/grpc/status"
)

// MyAuth 自定义 Auth 需要实现 credentials.PerRPCCredentials 接口
type MyAuth struct {
   Username string
   Password string
}

//username和password是自定义字段，可以自己定义，必须小写
// GetRequestMetadata 定义授权信息的具体存放形式，最终会按这个格式存放到 metadata map 中。
func (a *MyAuth) GetRequestMetadata(context.Context, ...string) (map[string]string, error) {
   return map[string]string{"username": a.Username, "password": a.Password}, nil
}

// RequireTransportSecurity 是否需要基于 TLS 加密连接进行安全传输
func (a *MyAuth) RequireTransportSecurity() bool {
   return false
}

const (
   Admin = "admin"
   Root  = "root"
)

func NewMyAuth() *MyAuth {
   return &MyAuth{
      Username: "error",
      //Username: Admin,
      Password: Root,
   }
}

// IsValidAuth 具体的验证逻辑
func IsValidAuth(ctx context.Context) error {
   var (
      user     string
      password string
   )
   // 从 ctx 中获取 metadata
   md, ok := metadata.FromIncomingContext(ctx)
   if !ok {
      return status.Errorf(codes.InvalidArgument, "missing metadata")
   }
   // 从metadata中获取授权信息
   // 这里之所以通过md["username"]和md["password"] 可以取到对应的授权信息
   // 是因为我们自定义的 GetRequestMetadata 方法是按照这个格式返回的.
   if val, ok := md["username"]; ok {
      user = val[0]
   }
   if val, ok := md["password"]; ok {
      password = val[0]
   }
   // 简单校验一下 用户名密码是否正确.
   if user != Admin || password != Root {
      return status.Errorf(codes.Unauthenticated, "Unauthorized")
   }

   return nil
}
```



##### 客户端代码

客户端连接是，将自定义token作为参数传入

添加 Credentials 有两种方式：

1.建立连接时指定

授权信息保存在 conn 对象上，通过该连接发起的每个调用都会附带上该授权信息。

2.发起调用是指定

为每个调用指定不同的授权信息。

```go
package main

import (
   "context"
   "fmt"
   "golang.org/x/oauth2"
   "google.golang.org/grpc"
   "google.golang.org/grpc/credentials"
   "guokai.com/grpc-go-example/streamhello/auth"
   "guokai.com/grpc-go-example/streamhello/pb"
   "log"
   "time"
)

func callUnaryEcho(client pb.EchoClient, message string) {
   ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
   defer cancel()

   // 构建一个 PerRPCCredentials。 方式2:发起调用时指定
   //cred := oauth.NewOauthAccess(fetchToken())
   //resp, err := client.UnaryEcho(ctx, &pb.EchoRequest{Message: message},grpc.PerRPCCredentials(cred))

   resp, err := client.UnaryEcho(ctx, &pb.EchoRequest{Message: message})
   if err != nil {

      log.Fatalf("客户端一元rpc调用失败, %v: ", err)
   }
   fmt.Println("UnaryEcho: ", resp.Message)
}

func main() {

   // 构建一个 PerRPCCredentials。
   //perRPC := oauth.NewOauthAccess(fetchToken())
   myAuth := auth.NewMyAuth()

   creds, err := credentials.NewClientTLSFromFile("../server.crt", "")
   if err != nil {
      log.Fatalf("failed to load credentials: %v", err)
   }
   //方式1：建立连接时指定
   //conn, err := grpc.Dial("127.0.0.1:8088", grpc.WithTransportCredentials(creds), grpc.WithPerRPCCredentials(perRPC))
   conn, err := grpc.Dial("127.0.0.1:8088", grpc.WithTransportCredentials(creds), grpc.WithPerRPCCredentials(myAuth))
   if err != nil {
      log.Fatalf("did not connect: %v", err)
   }
   defer conn.Close()
   client := pb.NewEchoClient(conn)

   callUnaryEcho(client, "hello world")
}

// fetchToken 获取授权信息
func fetchToken() *oauth2.Token {
   return &oauth2.Token{
      AccessToken: "some-secret-token",
   }
}
```



服务端代码

服务端获取授权信息并校验有效性。

gPRC 传输时把授权信息存放在 metada ，所以需要先获取 metadata。通过metadata.FromIncomingContext从 ctx 中取出本次调用的 metadata，然后再从 MD 中取出授权信息并校验即可。

metadata结构

```go
type MD map[string][]string
```

授权信息在map中具体怎么存的由 PerRPCCredentials接口的**GetRequestMetadata**函数实现。

```go
package main

import (
   "context"
   "google.golang.org/grpc"
   "google.golang.org/grpc/codes"
   "google.golang.org/grpc/credentials"
   "google.golang.org/grpc/metadata"
   "google.golang.org/grpc/status"
   "guokai.com/grpc-go-example/streamhello/auth"
   "guokai.com/grpc-go-example/streamhello/pb"
   "log"
   "net"
   "strings"
)

type ecServer struct {
   pb.UnimplementedEchoServer
}

//一元rpc
func (s *ecServer) UnaryEcho(ctx context.Context, req *pb.EchoRequest) (*pb.EchoResponse, error) {
   return &pb.EchoResponse{Message: req.Message}, nil
}

// myEnsureValidToken 自定义函数 验证 token 有效性
func myEnsureValidToken(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
   // 如果返回err不为nil则说明token验证未通过
   err := auth.IsValidAuth(ctx)
   if err != nil {
      return nil, err
   }
   return handler(ctx, req)
}

// valid 校验认证信息有效性。
func valid(authorization []string) bool {
   if len(authorization) < 1 {
      return false
   }
   token := strings.TrimPrefix(authorization[0], "Bearer ")
   return token == "some-secret-token"
}

// ensureValidToken 定义一元拦截器 校验 token有效性。
func ensureValidToken(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
   // 如果 token不存在或者无效，直接返回错误，否则就调用真正的RPC方法。
   md, ok := metadata.FromIncomingContext(ctx)
   if !ok {
      return nil, errMissingMetadata
   }
   if !valid(md["authorization"]) {
      return nil, errInvalidToken
   }
   return handler(ctx, req)
}

//定义流验证器 验证 token有效性
func streamInterceptor(srv interface{}, ss grpc.ServerStream, info *grpc.StreamServerInfo, handler grpc.StreamHandler) error {
   // authentication (token verification)
   md, ok := metadata.FromIncomingContext(ss.Context())
   if !ok {
      return errMissingMetadata
   }
   if !valid(md["authorization"]) {
      return errInvalidToken
   }

   err := handler(srv, ss)
   return err
}

//定义错误
var (
   errMissingMetadata = status.Errorf(codes.InvalidArgument, "missing metadata")
   errInvalidToken    = status.Errorf(codes.Unauthenticated, "invalid token")
)

func main() {

   creds, err := credentials.NewServerTLSFromFile("../server.crt", "../server.key")
   if err != nil {
      log.Fatalf("failed to load key pair: %s", err)
   }
   //增加授权信息和tls安全验证
   //这2种方式都可以
   //s := grpc.NewServer(grpc.UnaryInterceptor(ensureValidToken), grpc.Creds(creds))
   s := grpc.NewServer(grpc.UnaryInterceptor(myEnsureValidToken), grpc.Creds(creds))
   pb.RegisterEchoServer(s, &ecServer{})
   lis, err := net.Listen("tcp", "127.0.0.1:8088")
   if err != nil {
      log.Fatalf("failed to listen: %v", err)
   }
   log.Println("Serving gRPC on 0.0.0.0：8088")
   if err := s.Serve(lis); err != nil {
      log.Fatalf("failed to serve: %v", err)
   }
}
```

结果：将代码中写成username=error，提示授权失败，修改成admin后就正确了

客户端

```
macbookpro@bogon ~/go/src/grpc-go-example/streamhello/auth/client $ go run client.go
2022/09/19 14:58:57 客户端一元rpc调用失败, rpc error: code = Unauthenticated desc = Unauthorized: 
exit status 1
```

正确结果

```
macbookpro@bogon ~/go/src/grpc-go-example/streamhello/auth/client $ go run client.go
UnaryEcho:  hello world
```



总结:

1.实现credentials.PerRPCCredentials接口就可以把数据当做 gRPC 中的 Credential 在添加到 metadata 中，跟着请求一起传递到服务端；

2.服务端从 ctx 中解析 metadata，然后从 metadata 中获取 授权信息并进行验证；

3.借助 Interceptor （拦截器）实现全局身份验证

4.客户端可以通过 DialOption （连接选项）为所有请求统一指定授权信息，或者通过 CallOption（调用选项） 为每一个请求分别指定授权信息。



参考

https://www.lixueduan.com/posts/grpc/06-auth/