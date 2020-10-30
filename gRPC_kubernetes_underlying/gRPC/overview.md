- [Introduction to gRPC](#introduction-to-grpc)
  - [Overview of gRPC](#overview-of-grpc)
  - [Working with Protocol Buffers](#working-with-protocol-buffers)
    - [Protocol Buffers简介](#protocol-buffers简介)
      - [Step1 定义数据结构](#step1-定义数据结构)
      - [Step2 使用buffer compiler](#step2-使用buffer-compiler)
    - [gRPC对于protocol buffers的使用](#grpc对于protocol-buffers的使用)

# Introduction to gRPC

## Overview of gRPC

gRPC can use protocol buffers as both its Interface Definition Language (IDL) and as its underlying message interchange format.

In gRPC, a client application can directly call a method on a server application on a different machine as if it were a local object, making it easier for you to create distributed applications and services. As in many RPC systems, gRPC is based around the idea of **defining a service**, specifying **the methods that can be called remotely with their parameters and return types**. On the server side, the server implements this interface and runs a gRPC server to handle client calls. On the client side, **the client has a stub** (referred to as just a client in some languages) that provides the same methods as the server.

![Alt Text](gRPC_pictures/archtecture%20of%20gRPC.svg)

gRPC clients and servers can run and talk to each other in a variety of environments - from servers inside Google to your own desktop - and can be written in any of gRPC’s supported languages. So, for example, you can easily create a gRPC server in Java with clients in Go, Python, or Ruby. In addition, the latest Google APIs will have gRPC versions of their interfaces, letting you easily build Google functionality into your applications.

## Working with Protocol Buffers

默认情况下，gRPC使用google的Protocol Buffers作为数据序列化/反序列化工具，接口定义语言(`Interface Definition Language`)，以及底层消息交换格式。

### Protocol Buffers简介

对于Protocol Buffers，使用步骤如下：

#### Step1 定义数据结构

在后缀名为`.proto`proto文本文件中定义用于序列化数据的数据结构，Protocol Buffer data被组织成一个个`message`，每个`message`可视为存储一系列被称为`fields`的键值对的小的逻辑记录(a small logical record)
   
```protobuf
message Person {
    string name = 1;
    int32 id = 2;
    bool has_ponycopter = 3;
}
```

#### Step2 使用buffer compiler

在定义完数据结构后，使用`protocol buffers`的编译器`protoc`，按照proto definition中指定的编程语言生成数据访问类(`data access classes`)。比如上述例子中的Person，其name fields将生成简单的accessors，比如`name()`与`set_name()`。此外编译器`protoc`还将生成负责序列化数据的方法。

对于[Step1](#step1-定义数据结构)中的例子来说，若指定C++语言，`protoc`将生成一个Person 类，通过使用这个类在应用中填充、序列化以及找回`Person` protocol buffer messages

### gRPC对于protocol buffers的使用

+ define gRPC services in ordinary proto files, with RPC method parameters and return types specified as protocol buffer messages:
  
  ```protobuf
  // The greeter service definition.
  service Greeter {
    // Sends a greeting
    rpc SayHello (HelloRequest) returns (HelloReply) {}
  }

  // The request message containing the user's name.
  message HelloRequest {
    string name = 1;
  }

  // The response message containing the greetings
  message HelloReply {
    string message = 1;
  }
  ```

+ gRPC uses `protoc` with **a special gRPC plugin** to generate code from your proto file: you get generated gRPC client and server code, as well as the regular protocol buffer code for populating, serializing, and retrieving your message types.
