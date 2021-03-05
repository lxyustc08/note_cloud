- [Core concepts, architecture and lifecycle](#core-concepts-architecture-and-lifecycle)
  - [Overview](#overview)
    - [Service definition](#service-definition)
    - [Using the API](#using-the-api)
    - [Synchronous vs. asynchronous](#synchronous-vs-asynchronous)
  - [RPC life cycle](#rpc-life-cycle)
    - [Unary RPC](#unary-rpc)
    - [Server streaming RPC](#server-streaming-rpc)
    - [Client streaming RPC](#client-streaming-rpc)
    - [Bidirectional streaming RPC](#bidirectional-streaming-rpc)
    - [Deadlines/ Timeouts](#deadlines-timeouts)
    - [RPC termination](#rpc-termination)
    - [Cancelling an RPC](#cancelling-an-rpc)
    - [Metadata](#metadata)
    - [Channels](#channels)

# Core concepts, architecture and lifecycle

## Overview

### Service definition

同大部分RPC系统类似，`gRPC`基于服务定义的思想，`gRPC`指定可以远程调用的`methods`以及该methods的参数`parameters`以及`返回类型`。

默认状态下，gRPC使用protocol buffers作为接口定义语言Interface Definition Language（IDL），通过protocol buffers，gRPC描述服务接口以及消息载体

一个protocol buffers的例子

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
gRPC支持的**4**种服务方法

+ 一元RPC(`unary RPCs`)，该类型下，client发送单个request至服务器server，获得单个response，类似于正常的`function call`

```protobuf
rpc SayHello(HelloRequest) returns (HelloResponse);
```

+ 服务流RPC(`Server streaming RPCs`)，该类型下客户端发送单个request到服务器，从服务器得到一个流，客户端从得到的流中顺序读取一系列消息。**客户端从返回的流中读取消息直到没有更多的消息。对于该类RPC，gRPC保证消息顺序**。

```protobuf
rpc LotsofReplies(HelloRequest) returns (stream HelloResponse);
```

+ 客户端流RPC(`Client streaming RPCs`)，该类型下客户端写入一系列消息并将该系列消息发送至服务器，消息发送使用gRPC提供的流进行。当客户端结束消息写入过程后，客户端等待服务器读取消息完成后返回的响应。**同样的，gRPC在同一个RPC调用期间保证消息的顺序**。

```protobuf
rpc LotsofGreetings(stream HelloRequest) returns (HelloResponse)
```

+ 双向流RPC(`Bidirectional streaming RPCs`)，该类型下rpc双方各使用一个独立的`read-write`流；客户端、服务器可以以其自定义的顺序使用各自的流；如，服务器可以等待所有的client messages接收完成后再发送其响应，也可以交替进行消息读取与消息写入操作，亦可以边读取边写入。每个流中的消息顺序是固定的。

```protobuf
rpc BidiHello(stream HelloRequest) returns (stream HelloResponse);
```

### Using the API

定义好`.proto`文件后，gRPC提供protocol buffer compiler插件，生成客户侧与服务侧代码。通常情况下gRPC用户**在客户端侧调用API，在服务器侧对API进行实现**。

+ 在服务器侧，服务器实现services声明的methods，并运行一个gRPC服务器以处理客户端调用。gRPC基础设施解码requests，执行service方法并编码服务响应
+ 在客户端侧，服务器拥有一个被称为stub的本地对象，该本地对象实现了service的方法(On the client side, the client has a local object known as stub that implements the same methods as the service)，**客户端通过该stub对象调用service方法**，将方法调用参数打包为合适的protocol buffer消息类型。gRPC在将请求发送到服务器并返回服务器响应之后进行查找。

### Synchronous vs. asynchronous

同步RPC调用处于阻塞状态，直到从服务器的响应到达客户端，同步RPC是最接近理想RPC调用过程的抽象。

异步RPC调用更符合网络异步场景的要求，在许多场景下是十分有用的。

## RPC life cycle

gRPC life cycle: gRPC client calls a gRPC server method until the correspond response returns

### Unary RPC

1. Once the client calls a stub method, the server is notified that the RPC has been invoked with the client's metadata for this call, the method name, and the specified dealine if applicable.
2. The server can then either send back its own inital metadata (**which must be sent before any response**) straight away, or wait for the client's request message. Which happens first, is application-specific.
3. Once the server has the client's request message, **it does whatever work is nessary to create and populate a response**. The response is then returned (if successful) to the client together with status details (status code and optional status message) and optional trailing metadata.
4. **If the response status is OK, then the client gets the response**, which completes the call on the client side.

> 1. 当客户端调用stub 对象的方法时，服务器被通知该对象方法对应的RPC被调用，通知附带本次调用的客户端元数据，方法名称以及可以被容忍的deadline

### Server streaming RPC

A server-streaming RPC is similar to a unary RPC, except that the server **returns a stream of messages** in response to a client’s request. **After sending all its messages, the server’s status details (status code and optional status message) and optional trailing metadata are sent to the client**. This completes processing on the server side. **The client completes once it has all the server’s messages**.

### Client streaming RPC

A client-streaming RPC is similar to a unary RPC, except **that the client sends a stream of messages to the server instead of a single message**. **The server responds with a single message (along with its status details and optional trailing metadata)**, typically but not necessarily after it has received all the client’s messages

### Bidirectional streaming RPC

In a bidirectional streaming RPC, the call is initiated by the client invoking the method and the server receiving the client metadata, method name, and deadline. **The server can choose to send back its initial metadata or wait for the client to start streaming messages**. 

> 与unary rpc类似

**Client- and server-side stream processing is application specific**. Since the two streams are independent, the client and server can read and write messages in any order. For example, a server can wait until it has received all of a client’s messages before writing its messages, or the server and client can play “ping-pong” – the server gets a request, then sends back a response, then the client sends another request based on the response, and so on

> 1. 客户端、服务器端的流处理过程由应用决定（双向流式RPC，两个流独立）
> 2. 服务器流与客户端流之间的协同可以拥有一定的自由处理顺序

### Deadlines/ Timeouts

gRPC允许客户端指定以错误`DEADLINE_EXCEEDED`终止RPC之前，客户端愿意等待的时长。在服务端服务器可以查询一个特定的RPC是否超时，以及可以完成RPC的剩余时长。

Specifying a deadline or timeout is language specific: some language APIs work in terms of timeouts (durations of time), and some language APIs work in terms of a deadline (a fixed point in time) and may or maynot have a default deadline.

> 指定deadline或者timeout的方式由语言决定

### RPC termination

In gRPC, both the client and server make independent and local determinations of the success of the call, and their conclusions may not match. This means that, for example, you **could have an RPC that finishes successfully on the server side (“I have sent all my responses!") but fails on the client side (“The responses arrived after my deadline!")**. It’s also possible for a server to decide to complete before a client has sent all its requests

> 1. 对于RPC而言，客户端侧，服务器侧双方对于调用成功的判断独立进行，之间无关联

### Cancelling an RPC

客户端侧或服务器侧均可在任意时间退出RPC，退出立即终止RPC，因此该RPC不再有任何工作进行

> 退出前的更改操作不会保留

### Metadata

Metadata is information about a particular RPC call (such as authentication details) in the form of a list of key-value pairs, where the keys are strings and the values are typically strings, but can be binary data. Metadata is opaque to gRPC itself - it lets the client provide information associated with the call to the server and vice versa.

> 1. 元数据对于gRPC而言是不透明的
> 2. 元数据的访问依赖于各语言具体实现

### Channels

A gRPC channel provides a connection to a gRPC server on a specified host and port. It is used when creating a client stub. Clients can specify channel arguments to modify gRPC’s default behavior, such as switching message compression on or off. A channel has state, including connected and idle.

> 1. channel在client stub创建时即被使用
> 2. channel具有状态`connected`以及`idle`
> 3. gRPC对于channel的关闭操作处理取决于语言的具体实现
