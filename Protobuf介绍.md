#### 1.概述

**Protocol buffers** 是一种语言无关、平台无关的可扩展机制或者说是**数据交换格式**，用于**序列化结构化数据**。与 XML、JSON 相比，Protocol buffers 序列化后的码流更小、速度更快、操作更简单。



#### 2.protoc编译器

protoc 用于编译 protocolbuf (.proto文件) 。protoc 是 C++ 写的，比较简单的安装方式是直接下载编译好的二进制文件。

1.下载

release 版本下载地址如下:

```
https://github.com/protocolbuffers/protobuf/releases
```

下载操作系统对应版本然后解压。

<img src="/Users/macbookpro/Library/Application Support/typora-user-images/image-20220917102846330.png" alt="image-20220917102846330" style="zoom:67%;" />



2.配置环境变量

protoc编译器正常运行需要环境变量配置，将protocke执行文件所在目录添加到当前系统的环境变量中。windows系统下可以直接在Path目录中进行添加；macOS系统下可以将protoc可执行文件拷贝至**/usr/local/include**目录下。

```
如果不想安装到/user/local目录下，可以解压到其他目录，然后把解压目录下的bin加入环境变量。
以我的mac环境变量为例，我的终端使用zsh替换了，vi .zshrc,在文件添加下面内容
#protoc
export PATH="/Users/macbookpro/go_unit/protoc-3.19.1-osx-x86_64/bin:$PATH"

修改完成，终端执行  source ~/.zshrc 保存

终端执行 proto --version 查看是否安装成功
输出类似这种的 libprotoc 3.11.2说明安装成功
```



#### 3.go插件

除了安装 protoc 之外还需要安装各个语言对应的**编译插件**，我用的 Go 语言，所以还需要安装一个 Go 语言的编译插件 protoc-gen-go。这个工具用来将 `.proto` 文件转换为 Golang 代码。

```
go install google.golang.org/protobuf/cmd/protoc-gen-go@v1.28
```



#### 4.实战

##### 1.创建proto文件

在项目目录建立grpc-go-example/helloworld目录

创建pb目录，在目录新建proto文件，文件名hello_world.proto

```
//声明proto的版本 只有proto3才支持gRPC
syntax = "proto3";

// 将编译后文件输出在 guokai.com/grpc-go-example/helloworld/pb 目录
option go_package = "guokai.com/grpc-go-example/helloworld/pb";

// 指定当前proto文件属于pb包
package pb;

//定义具体的信息
//repeated 表示字段可重复，即用来表示 Go 语言中的数组类型。
message Student {
  string name = 1;
  bool male = 2;
  repeated int32 scores = 3;
}
```

说明：

1.protobuf 有2个版本，默认版本是 proto2，如果需要 proto3，则需要在非空非注释第一行使用 `syntax = "proto3"` 标明版本。

2.`package`，指定包名，是可选的，用来防止不同的消息类型有命名冲突。

3.消息类型 

使用 `message` 关键字定义，Student 是类型名，name, male, scores 是该类型的 3 个字段，类型分别为 string, bool 和 []int32。字段可以是标量类型，也可以是合成类型。

`repeated` 表示字段可重复，即用来表示 Go 语言中的数组类型。

每个字符 `=`后面的数字称为标识符，每个字段都需要提供一个唯一的标识符。标识符用来在消息的二进制格式中识别各个字段，一旦使用就不能够再改变，标识符的取值范围为 [1, 2^29 - 1] 。

4.注释

proto 文件可以写注释，单行注释 `//`，多行注释 `/* ... */`

5.一个 .proto 文件中可以写多个消息类型，即对应多个结构体(struct)



##### 2.终端执行，编译proto

```sh
protoc --proto_path=./ --go_out=./ --go_opt=paths=source_relative pb/hello_world.proto
```

可以看到，在pb目录生成了hello_world.pb.go文件

```bash
grpc-go-example
└── helloworld
    └── pb
        ├── helloworld.pb.go
        └── helloworld.proto
```

如果想将生成的文件保存到其他目录，如保存到proto目录，需要先创建proto文件夹

```bash
protoc --proto_path=pb --go_out=proto --go_opt=paths=source_relative hello_world.proto
```

```bash
grpc-go-example
├── helloworld
    └── pb
    │    ├── helloworld.pb.go
    │    └── helloworld.proto
		└── proto
        ├── helloworld.pb.go
```



说明：

1. –proto_path或者`-I`：指定 import 路径，可以指定多个参数，编译时按顺序查找，不指定时默认查找当前目录。如果没有引入其他的 .proto 文件，该参数可以省略。

   解释： .proto 文件中也可以引入（import）其他的 .proto 文件，这里主要用于**指定引入文件位置**。

2. –go_out：golang编译支持，指定输出文件路径。其他语言则替换即可，比如 `--java_out` 等。支持的语言有 cpp/java/python/ruby/objc/csharp/php等语言。

3. –go_opt：指定参数，比如`--go_opt=paths=source_relative`就是表明生成文件输出使用相对路径。

4. pb/hello_world.proto  要编译的proto文件放到最后面

终端例子说明：

```makefile
下面3条命令表示的含义是一样的
protoc的编译命令比较习惯写在Makefile中，一是容易查找，二是执行起来简单。

# Makefile
GOPATH:=$(shell go env GOPATH)

# proto1 相关指令
.PHONY: proto1
path1:
	protoc --go_out=. proto1/greeter/greeter.proto
path2:
	protoc -I. --go_out=. proto1/greeter/greeter.proto # 点号表示当前路径，注意-I参数没有等于号
path3:
	protoc --proto_path=. --go_out=. proto1/greeter/greeter.proto
```



Import同目录下的protobuf文件

price.proto

```protobuf
//声明proto的版本 只有proto3才支持gRPC
syntax = "proto3";

// 将编译后文件输出在 guokai.com/grpc-go-example/book/pb 目录
option go_package = "guokai.com/grpc-go-example/book/pb";

// 指定当前proto文件属于pb包
package pb;

//定义具体的信息
message Price {
    int64 market_price = 1;  // 建议使用下划线的命名方式
    int64 sale_price = 2;
}
```



book.proto

```protobuf
syntax = "proto3";

// 声明protobuf中的包名
package pb;

// 声明生成的Go代码的导入路径
option go_package = "guokai.com/grpc-go-example/book/pb";

// 引入同目录下的protobuf文件（注意起始位置为proto_path的下层）
import "pb/price.proto";

//price.proto与book.proto属于同一包，无需写成pb.price
message Book {
    string title = 1;
    Price price = 2;
}
```

在proto目录终端下执行

```bash
protoc --proto_path=./ --go_out=./ --go_opt=paths=source_relative book/book.proto book/price.proto
```

生成的最终效果

```bash
grpc-go-example
└── proto
    └── book
        ├── book.pb.go
        ├── book.proto
        ├── price.pb.go
        └── price.proto
```



import其他目录文件

在proto目录新建author文件夹，定义author.proto文件

```protobuf
// grpc-go-example/proto/author/author.proto

syntax = "proto3";

// 声明protobuf中的包名
package author;

// 声明生成的Go代码的导入路径
option go_package = "guokai.com/grpc-go-example/proto/author";

message Info {
    string name = 1;
}
```



更新book.proto

```protobuf
syntax = "proto3";

// 声明protobuf中的包名
package book;

// 声明生成的Go代码的导入路径
option go_package = "guokai.com/grpc-go-example/proto/book";

// 引入同目录下的protobuf文件（注意起始位置为proto_path的下层）
import "book/price.proto";

// 引入其他目录下的protobuf文件
import "author/author.proto";

message Book {
    string title = 1;
    Price price = 2;
    author.Info authorInfo = 3;  // 需要带package前缀
}
```

在proto目录终端执行

```bash
protoc --proto_path=./ --go_out=./ --go_opt=paths=source_relative book/book.proto book/price.proto author/author.proto
```

结果：

```bash
grpc-go-example
└── proto
    ├── author
    │   ├── author.pb.go
    │   └── author.proto
    └── book
        ├── book.pb.go
        ├── book.proto
        ├── price.pb.go
        └── price.proto
```



import google proto文件

修改book.proto

```protobuf
syntax = "proto3";

// 声明protobuf中的包名
package book;

// 声明生成的Go代码的导入路径
option go_package = "guokai.com/grpc-go-example/proto/book";

// 引入同目录下的protobuf文件（注意起始位置为proto_path的下层）
import "book/price.proto";

// 引入其他目录下的protobuf文件
import "author/author.proto";

// 引入google/protobuf/timestamp.proto文件
import "google/protobuf/timestamp.proto";

message Book {
    string title = 1;
    Price price = 2;
    author.Info authorInfo = 3;  // 需要带package前缀
    // Timestamp是大写T!大写T!大写T!
    google.protobuf.Timestamp date = 4;  // 注意包名前缀
}
```

文件里的google/protobuf/timestamp.proto是在我们下载的proto编译器的include目录导入的，

你也可以简单粗暴的把下载好的proto文件拷贝到你的proto目录执行。

```bash
protoc --proto_path=./ --go_out=./ --go_opt=paths=source_relative book/book.proto book/price.proto author/author.proto
```

执行后发现加载到了时间参数。



#### 生成Grpc代码

通常都是配合 gRPC 来使用 protobuf ，基于`.proto`文件生成Go代码的同时也要生成 gRPC 代码。

要想生成 gRPC 代码就需要先安装 `protoc-gen-go-grpc` 插件。

```bash
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@v1.2
```

上述命令会默认将插件安装到`$GOPATH/bin`，为了`protoc`编译器能找到这些插件，请确保你的`$GOPATH/bin`在环境变量中。

在终端执行，查看是否安装成功

```
protoc-gen-go-grpc --version
```



#### 实战

新建grpc-go-example/hello/proto目录，在目录下新建文件hello.proto

```
//声明proto的版本 只有proto3才支持 gRPC
syntax = "proto3";
// 将编译后文件输出在 guokai.com/grpc-go-example/hello/proto 目录
option go_package = "guokai.com/grpc-go-example/hello/proto";
// 指定当前proto文件属于pb包
package proto;

// 定义一个名叫 greeting 的服务
service Greeter {
  // 该服务包含一个 SayHello 方法 HelloRequest、HelloReply分别为该方法的输入与输出
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// 具体的参数定义
message HelloRequest {
  string name = 1;
}

// 具体的响应信息
message HelloReply {
  string message = 1;
}
```

在hello目录终端下执行

```bash
protoc --proto_path=./ --go_out=./ --go_opt=paths=source_relative --go-grpc_out=./ --go-grpc_opt=paths=source_relative proto/hello.proto 
```

结果：

```bash
grpc-go-example
└── hello
    ├── proto
        ├── hello.pb.go
        └── hello.proto
        └── hello_grpc.pb.go
```



#### gRPC-Gateway

[gRPC-Gateway](https://github.com/grpc-ecosystem/grpc-gateway) 也是日常开发中比较常用的一个工具，它同样也是根据 protobuf 生成相应的代码。

安装

```bash
go get github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-grpc-gateway
```

定义proto文件

```protobuf
// demo/proto/book/book.proto

syntax = "proto3";

// 声明protobuf中的包名
package book;

// 声明生成的Go代码的导入路径
option go_package = "github.com/Q1mi/demo/proto/book";

// 引入同目录下的protobuf文件  引入路径从终端命令开始算
import "book/price.proto";
// 引入其他目录下的protobuf文件
import "author/author.proto";
// 引入google/protobuf/timestamp.proto文件
import "google/protobuf/timestamp.proto";
// 引入google/api/annotations.proto文件
import "google/api/annotations.proto";

message Book {
    string title = 1;
    Price price = 2;
    author.Info authorInfo = 3;  // 需要带package前缀
    // Timestamp是大写T!大写T!大写T!
    google.protobuf.Timestamp date = 4;  // 注意包名前缀
}

service BookService{
    rpc Create(Book)returns(Book){
        option (google.api.http) = {
            post: "/v1/book"
            body: "*"
        };
    };
}
```

引入了`google/api/annotations.proto` ，这个文件是由googleapi定义在https://github.com/googleapis/googleapis。

直接下载文件，放到proto编译器的include目录或者项目目录。

编译加上gRPC-Gateway相关的 `--grpc-gateway_out=proto --grpc-gateway_opt=paths=source_relative` 参数。

```bash
protoc --proto_path=proto --go_out=proto --go_opt=paths=source_relative --go-grpc_out=proto --go-grpc_opt=paths=source_relative --grpc-gateway_out=proto --grpc-gateway_opt=paths=source_relative book/book.proto book/price.proto author/author.proto
```

上面命令会编译得到一个`book.pb.gw.go`文件。



**为了方便编译可以在项目定义makefile**

```makefile
.PHONY: gen help

PROTO_DIR=proto

gen:
	protoc \
	--proto_path=$(PROTO_DIR) \
	--go_out=$(PROTO_DIR) \
	--go_opt=paths=source_relative \
	--go-grpc_out=$(PROTO_DIR) \
	--go-grpc_opt=paths=source_relative \
	--grpc-gateway_out=$(PROTO_DIR) \
	--grpc-gateway_opt=paths=source_relative \
	$(shell find $(PROTO_DIR) -iname "*.proto")

help:
	@echo "make gen - 生成pb及grpc代码"
```

后续想要编译只需在项目目录下执行`make gen`即可。



proto文件格式如下，想编译所有proto

```
grpc-go-example
└── proto
    ├── common.proto
    ├── greeter
    │   └── greeter.proto
    └── user
        └── user.proto
```

在终端执行下面命令编译

```
protoc --proto_path=. --go_out=. proto/*.proto proto/user/*proto proto/greeter/*proto
```





**github老版本的终端执行方式，目前已经迁移到google.golang.org**，不推荐使用老方式

`--go_out=paths=import:.`、`--go_out=paths=source_relative:.`，或者`--go_out=plugins=grpc:.`

`--go_out`参数是用来指定 protoc-gen-go 插件的工作方式和Go代码的生成位置，而上面的写法正是表明该插件的工作方式。

`--go_out`主要的两个参数为`plugins` 和 `paths`，分别表示生成Go代码所使用的插件，以及生成的Go代码的位置。

`--go_out`的写法是，参数之间用逗号隔开，最后加上冒号来指定代码的生成位置

比如`--go_out=plugins=grpc,paths=import:.`。

`paths`参数有两个选项，分别是 `import` 和 `source_relative`，默认为 import，表示按照生成的Go代码包的全路径去创建目录层级，source_relative 表示按照 **proto源文件的目录层级**去创建Go代码的目录层级，如果目录已存在则不用创建

```protobuf
option go_package="proto1/pb_go";

# 指令1：paths为import，pb文件最终在 pb_go 目录下
$ protoc --proto_path=. --go_out=. proto1/greeter/greeter_v2.proto
$ protoc --proto_path=. --go_out=paths=import:. proto1/greeter/greeter_v2.proto

# 指令2：paths为source_relative，pb文件最终在 proto1/greeter 目录下
$ protoc --proto_path=. --go_out=paths=source_relative:. proto1/greeter/greeter_v2.proto
```





其他语法

#### Reserved

Reserved用来指明此message不使用某些字段，也就是忽略这些字段。

可以通过字段编号范围或者字段名称指定保留的字段：

如更新时，可能某些字段会删除，被加载老版本数据时，会造成数据冲突，所以可以将这些字段保留reserved

```
syntax = "proto3";
package abc;
message AllNormalypes {
  reserved 2, 4 to 6;
  reserved "field14", "field11";
  double field1 = 1;
  // float field2 = 2;
  int32 field3 = 3;
  // int64 field4 = 4;
  // uint32 field5 = 5;
  // uint64 field6 = 6;
  sint32 field7 = 7;
  sint64 field8 = 8;
  fixed32 field9 = 9;
  fixed64 field10 = 10;
  // sfixed32 field11 = 11;
  sfixed64 field12 = 12;
  bool field13 = 13;
  // string field14 = 14;
  bytes field15 = 15;
}
```

声明保留的字段你就不要再定义了，否则编译的时候会出错。



#### 枚举类型

```protobuf
message Student {
  string name = 1;
  enum Gender {
    FEMALE = 0;
    MALE = 1;
  }
  Gender gender = 2;
  enum Status {
  	option allow_alias = true;
  	UNKNOWN = 0;
  	STARTED = 1;
  	RUNNING = 1;
	}
	Status status =3;
  repeated int32 scores = 4;
}
```

1.避免在同一个package定义重名的枚举字段

2.设置`allow_alias`，允许字段编号重复，`RUNNING`是`STARTED`的别名。

3.枚举类型的第一个选项的标识符必须是0，这也是枚举类型的默认值



其他类型

`Result`是另一个消息类型，在 SearchReponse 作为一个消息字段类型使用

```protobuf
message SearchResponse {
  repeated Result results = 1;
}
message Result {
  string url = 1;
  string title = 2;
  repeated string snippets = 3;
}
```

嵌套写也是支持的：

```protobuf
message SearchResponse {
  message Result {
    string url = 1;
    string title = 2;
    repeated string snippets = 3;
  }
  repeated Result results = 1;
}
```

如果定义在其他文件中，可以导入其他消息类型来使用：

```protobuf
import "myproject/other_protos.proto";
```



#### 任意类型(Any)

Any 可以表示不在 .proto 中定义任意的内置类型。



```protobuf
import "google/protobuf/any.proto";

message ErrorStatus {
  string message = 1;
  repeated google.protobuf.Any details = 2;
}
```

为了使用`Any`类型，你需要引入`google/protobuf/any.proto`。



### oneof

```protobuf
message SampleMessage {
  oneof test_oneof {
    string name = 4;
    SubMessage sub_message = 9;
  }
}
```

Oneof有判断字段是否设置的功能



#### map

```
message MapRequest {
  map<string, int32> points = 1;
}
```

参考：

https://juejin.cn/post/6949927882126966820



推荐编写规范

- 文件(Files)
  - 文件名使用小写下划线的命名风格，例如 lower_snake_case.proto
  - 每行不超过 80 字符
  - 使用 2 个空格缩进

- 包(Packages)
  - 包名应该和目录结构对应，例如文件在`my/package/`目录下，包名应为 `my.package`
- 消息和字段(Messages & Fields)
  - 消息名使用首字母大写驼峰风格(CamelCase)，例如`message StudentRequest { ... }`
  - 字段名使用小写下划线的风格，例如 `string status_code = 1`
  - 枚举类型，枚举名使用首字母大写驼峰风格，例如 `enum FooBar`，枚举值使用全大写下划线隔开的风格(CAPITALS_WITH_UNDERSCORES )，例如 FOO_DEFAULT=1
- 服务(Services)
  - RPC 服务名和方法名，均使用首字母大写驼峰风格，例如`service FooService{ rpc GetSomething() }`



执行protoc --go_out=. simple.proto时报错

![image-20220908151133781](/Users/macbookpro/Library/Application Support/typora-user-images/image-20220908151133781.png)

解决: protoc-gen-go1.3.2以后需要在 xxx.proto 文件中加入下面的一行

option go_package ="文件路径;包名";



参考

https://www.liwenzhou.com/posts/Go/protobuf/   李文周

https://colobu.com/2019/10/03/protobuf-ultimate-tutorial-in-go/#%E7%BC%96%E7%A0%81