gRPC 内置支持 SSL/TLS，可以通过 SSL/TLS 证书建立安全连接，对传输的数据进行加密处理。

#### 生成证书

##### 生成私钥

执行下面的命令生成私钥文件——`server.key`

```
openssl ecparam -genkey -name secp384r1 -out server.key
```

这里生成的是ECC私钥，当然你也可以使用RSA。

##### 生成自签名的证书

为了在证书中添加SANs信息，我们将下面自定义配置保存到`server.cnf`文件中。

```conf
[ req ]
default_bits       = 4096
default_md		= sha256
distinguished_name = req_distinguished_name
req_extensions     = req_ext

[ req_distinguished_name ]
countryName                 = Country Name (2 letter code)
countryName_default         = CN
stateOrProvinceName         = State or Province Name (full name)
stateOrProvinceName_default = BEIJING
localityName                = Locality Name (eg, city)
localityName_default        = BEIJING
organizationName            = Organization Name (eg, company)
organizationName_default    = DEV
commonName                  = Common Name (e.g. server FQDN or YOUR name)
commonName_max              = 64
commonName_default          = liwenzhou.com

[ req_ext ]
subjectAltName = @alt_names

[alt_names]
DNS.1   = localhost
DNS.2   = liwenzhou.com
IP      = 127.0.0.1
```

执行下面的命令生成自签名证书——`server.crt`

```bash
openssl req -nodes -new -x509 -sha256 -days 3650 -config server.cnf -extensions 'req_ext' -key server.key -out server.crt
```

##### 建立安全连接

服务端代码

```go
package main

import (
   "context"
   "google.golang.org/grpc"
   "google.golang.org/grpc/credentials"
   "guokai.com/grpc-go-example/streamhello/pb"
   "log"
   "net"
)

type Echo struct {
   pb.UnimplementedEchoServer
}

func (e *Echo) UnaryEcho(ctx context.Context, req *pb.EchoRequest) (*pb.EchoResponse, error) {
   log.Printf("Recved: %v", req.GetMessage())
   resp := &pb.EchoResponse{Message: req.GetMessage()}
   return resp, nil
}

func main() {
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
   s := grpc.NewServer(grpc.Creds(creds)) //创建grpc服务器
   //将服务描述(server)及其具体实现(echoServer)注册到 gRPC 中去
   pb.RegisterEchoServer(s, &Echo{})
   //启动服务
   if err := s.Serve(listen); err != nil {
      log.Fatalf("failed to serve: %v", err)
   }
}
```



客户端代码

```go
package main

import (
   "context"
   "fmt"
   "google.golang.org/grpc"
   "google.golang.org/grpc/credentials"
   "guokai.com/grpc-go-example/streamhello/pb"
   "log"
   "time"
)

func main() {

   // 客户端通过ca证书来验证服务的提供的证书
   certFile := "./server.crt"
   creds, err := credentials.NewClientTLSFromFile(certFile, "")
   if err != nil {
      log.Fatalf("failed to load credentials: %v", err)
   }
   // 建立连接时指定使用 TLS
   conn, err := grpc.Dial("127.0.0.1:8088", grpc.WithTransportCredentials(creds))
   if err != nil {
      log.Fatalf("did not connect: %v", err)
   }
   defer conn.Close()
   client := pb.NewEchoClient(conn)
   unary(client)
}

//普通rpc
func unary(client pb.EchoClient) {
   ctx, cancel := context.WithTimeout(context.Background(), time.Second)
   defer cancel()
   resp, err := client.UnaryEcho(ctx, &pb.EchoRequest{Message: "hello world"})
   if err != nil {
      log.Printf("send error:%v\n", err)
   }
   fmt.Printf("Recved:%v \n", resp.GetMessage())
}
```

除了这种自签名证书的方式外，生产环境对外通信时通常需要使用受信任的CA证书。



参考：

https://www.lixueduan.com/posts/grpc/04-encryption-tls/