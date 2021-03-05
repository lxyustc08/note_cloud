- [GRPC Generate Go Code Reference](#grpc-generate-go-code-reference)
  - [Methods on generated server interfaces](#methods-on-generated-server-interfaces)
    - [Unary methods](#unary-methods)
    - [Server-streaming methods](#server-streaming-methods)
    - [Client-streaming methods](#client-streaming-methods)
    - [Bidi-streaming methods](#bidi-streaming-methods)
  - [Methods on generated client interfaces](#methods-on-generated-client-interfaces)
    - [Unary Methods](#unary-methods-1)
    - [Server-Streaming methods](#server-streaming-methods-1)
    - [Client-Streaming methods](#client-streaming-methods-1)
    - [Bidi-Streaming methods](#bidi-streaming-methods-1)
  - [Packages and Namespaces](#packages-and-namespaces)

# GRPC Generate Go Code Reference

+ 线程安全性
  + 对于客户端侧的RPC调用以及服务端侧的RPC handle均可运行在多个并发的goroutines中，他们是线程安全的。
  + 对于每个streams而言，其是双向且顺序的，因此每个stream并不支持并发读或者并发写，但支持在读写之间并行 `reads are safely concurrent with writes`

## Methods on generated server interfaces

在服务器侧，`.proto`文件中定义的每个service将生成对应的注册函数，如若在`.proto`文件中定义了一个`RouteGuide service`，则对应如下函数的生成：

```golang
func RegisterRouteGuideServer(s grpc.ServiceRegistrar, srv RouteGuideServer)
```

利用该函数，用户在实现RouteGuide service相关接口后，实现`RouteGuide service`的注册

### Unary methods

对于一元方法而言，gRPC将生成如下的接口

```golang
Foo(context.Context, *MsgA)(*MsgB,error)
```
其中`MsgA`是客户端侧发送的protobuf消息，`MsgB`是服务器侧发送的protobuf消息

### Server-streaming methods

对于服务器端流方法而言，gRPC将生成如下的接口

```golang
Foo(*MsgA,<ServiceName>_FooServer) error
```

其中，`MsgA`是客户端侧发送的单个请求，`<ServiceName>_FooServer`代表服务器发送至客户端的流程`MsgB`消息

对于`<ServiceName>_FooServer`而言，其结构如下：

```golang
type <ServiceName>_FooServer interface {
	Send(*MsgB) error
	grpc.ServerStream
}
```

内嵌grpc.ServerStream域。

服务器侧的handler通过内嵌的grpc.ServerStream域发送流式的protobuf messages至客户端。流式消息通过handler方法的return结束

### Client-streaming methods

对于客户端侧流方法而言，gRPC将生成如下的接口

```golang
Foo(<ServerName>_FooServer) error
```

其中，`<ServerName>_FooServer`起双重作用：

+ 读取客户端至服务器的流式消息
+ 发送单个服务器响应

对于`<ServerName>_FooServer`而言，其结构如下：

```golang
type <ServerName>_FooServer interface {
    SendAndClose(*MsgA) error
    Recv() (*MsgB, error)
    grpc.ServerStream
}
```

服务器侧的handler通过重复调用参数的`Recv()`方法获取流式请求中的全部客户端请求，当流式请求中的全部请求读取完成后`Recv()`返回`(nil, io.EOF)`。

服务器侧的handler通过调用参数的`SendAndClose`方法发送服务器的单个响应。

参数的`SendAndClose`方法仅允许调用1次

### Bidi-streaming methods

对于双向流式方法而言，gRPC将生成如下的接口

```golang
Foo(<ServiceName>_FooServer) error
```

此时的`<ServiceName>_FooServer`的结构如下：

```golang
type <ServiceName>_FooServer interface {
	Send(*MsgA) error
	Recv() (*MsgB, error)
	grpc.ServerStream
}
```

+ 服务器侧的handler重复调用`Recv()`读取客户端发送至服务器侧的流式消息；当读取完成时，`Recv()`返回`(nil, io.EOF)`
+ 服务器侧的handler重复调用`Send`发送服务器至客户端的响应消息，当服务器侧的handler return时，表明服务器发送的服务器至客户端的流消息结束

## Methods on generated client interfaces

对于客户端侧而言，每个`.proto`文件中定义的service将生成一个生成函数，返回一个客户端的concrete type，该concreate type实现了客户端接口，如在`.proto`文件中定义了`RouteGuide service`，gRPC将生成如下生成函数

```golang
func NewRouteGuideClient(cc grpc.ClientConnInterface) RouteGuideClient {
	return &routeGuideClient{cc}
}
```

该函数返回一个类型为`routeGuiderClient`对象，该类型实现了`RouteGuideClient interface`

### Unary Methods

对于一元方法而言，生成遵循如下的function signature的方法

```golang
Foo(ctx context.Context, in *MsgA, opts ...grpc.CallOption) (*MsgB, error)
```

其中，`MsgA`是客户端发送至服务器的单个请求，`MsgB`则是服务器发送至客户端的消息

### Server-Streaming methods

对于服务器侧流式方法而言，生成遵循如下function signature的方法

```golang
Foo(ctx context.Context, in *MsgA, opts ...grpc.CallOption) (<ServiceName>_FooClient, error)
```

其中`<ServiceName>_FooClient`结构如下

```golang
type <ServiceName>_FooClient interface {
	Recv() (*MsgB, error)
	grpc.ClientStream
}
```

客户端侧重复调用**返回值**的`Recv()`方法读取服务器发送至客户端的流式响应，当所有流式响应读取完成，`Recv()`返回`(nil,io.EOF)`

### Client-Streaming methods

对于客户端侧流式方法而言，生成遵循如下function signature的方法

```golang
Foo(ctx context.Context, opts ...grpc.CallOption) (<ServiceName>_FooClient, error)
```

其中`<ServiceName>_FooClient`结构如下：

```golang
type <ServiceName>_FooClient interface {
	Send(*MsgA) error
	CloseAndRecv() (*MsgB, error)
	grpc.ClientStream
}
```

+ 客户端侧重复调用**返回值**的`Send()`方法发送客户端至服务器的流式消息
+ **返回值**的`CloseAndRecv`方法最多调用1次，用于接受服务器发送的单个响应并关闭客户端至服务器端的消息流

### Bidi-Streaming methods

对于双向流式方法而言，生成遵循如下function signature的方法

```golang
Foo(ctx context.Context, opts ...grpc.CallOption) (<ServiceName>_FooClient, error)
```

此时`<ServiceName>_FooClient`结构如下：

```golang
type <ServiceName>_FooClient interface {
	Send(*MsgA) error
	Recv() (*MsgB, error)
	grpc.ClientStream
}
```

+ 客户端侧通过调用Foo方法的返回值的`Send`方法重复发送客户端至服务器的流式消息
+ 客户端侧通过调用Foo方法的返回值的`Recv`方法重复接收服务器至客户端的流式消息
+ 对于客户端至服务器的流式消息而言，其流终止的信号由客户端调用ClientStream的`CloseSend`进行确认
+ 对于服务器至客户端的流式消息而言，其流终止的信号由`Recv`方法的`(nil, io.EOF)`进行确认

## Packages and Namespaces

When the protoc compiler is invoked with --go_out=plugins=grpc:, the proto package to Go package translation works the same as when the protoc-gen-go plugin is used without the grpc plugin.

So, for example, if foo.proto declares itself to be in package foo, then the generated foo.pb.go file will also be in the Go package foo.
