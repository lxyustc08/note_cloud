- [Basic Ops](#basic-ops)
  - [Defining A Message Type](#defining-a-message-type)
    - [Specifying Field Types](#specifying-field-types)
    - [Assigning Field Numbers](#assigning-field-numbers)
    - [Specifying Field Rules](#specifying-field-rules)
    - [Adding More Message Types](#adding-more-message-types)
    - [Adding Comments](#adding-comments)
    - [Reserved Fields](#reserved-fields)
    - [What's Generated From Your .proto?](#whats-generated-from-your-proto)
  - [Scalar Value Types](#scalar-value-types)
  - [Optional Fields And Default Values](#optional-fields-and-default-values)
  - [Enumerations](#enumerations)
    - [Reserved Values](#reserved-values)
  - [Using Other Message Types](#using-other-message-types)
    - [Importing Definitions](#importing-definitions)
    - [Using proto3 Message Types](#using-proto3-message-types)
  - [Nested Types](#nested-types)
  - [Updating A Message Type](#updating-a-message-type)
  - [Extensions](#extensions)

# Basic Ops

本文档介绍Google protocol buffers的基本使用流程以及相关概念

## Defining A Message Type

下列`.proto`文件定义了一个`SearchRequest`，包括搜索请求的字符串`query`，搜索结果的页数`page_number`，以及每页的结果数量`result_per_page`

```protobuf
message SearchRequest {
  required string query = 1;
  optional int32 page_number = 2;
  optional int32 result_per_page = 3;
} 
```

在protocol buffers中，每个键值对被称为fields，每个fields代表用户想在messages中包含的数据，每个field有`name`与`type`的概念

### Specifying Field Types

在上述的例子中，所有的fields均为`scalar types`，除`scalar types`外，也可以定义组合类型`composite types`，包括`enumerations`以及其他的`messages types`。

### Assigning Field Numbers

如例子中所示，每个fields被赋予一个独一无二的数字，这些数字被用于在messages的二进制形式中标识fields，当message处于使用状态中时，该数字不应该被更改。

对于fields的标识数字而言，1~15采用1字节进行编码，16~2047采用2字节进行编码，因此在实践中，1~15数字通常被保留作为被访问频次较高的fields的标识数字。

对于fields的标识数字而言，其范围为1~2<sup>29</sup>-1(536,870,911)，其中19000至19999为`protocol buffers`的保留标识数字，用户若使用相关保留标识数字，protocol buffers的编译器将报错。

### Specifying Field Rules

对于`protocol buffers`的fields而言，其有三种规则：

+ `required`: a well-formed message must have exactly one of this field.
+ `optional`: a well-formed message can have zero or one of this field (but not more than one).
+ `repeated`: this field can be repeated any number of times (including zero) in a well-formed message. The order of the repeated values will be preserved.

由于历史原因，对于`repeated` fields规则而言，若其fields类型为scalar numeric，此时fields不会以最有效的方式进行编码，需要手动指定[packer=true]以获得更加效率的编码。

```protobuf
repeated int32 samples = 4 [packed=true];
```

对于required规则而言，**required is forever**，因此在指定某一个fields为required的规则前，需要仔细进行思考。若将一个required规则的fields转换为optional规则的fields将导致，旧的应用程序接受到新消息时，认为该消息不合法。

基于**required is forever**的原因，工程实践中，部分工程师认为required的规则有害，不使用required的规则，仅使用optional与repeated的规则。

### Adding More Message Types

同一个`.proto`的文件中可以定义多个相关的messages类型。如下面一个例子所示：

```protobuf
message SearchRequest {
  required string query = 1;
  optional int32 page_number = 2;
  optional int32 result_per_page = 3;
}

message SearchResponse {
 ...
}
```

上述`.proto`文件中定义了两个message类型，`SearchRequest`以及`SearchResponse`

尽管同一个`.proto`文件中可以定义多个messages类型，但同一个`.proto`定义过多的存在不同依赖的messages类型时，容易引起依赖关系的膨胀。从工程实践上来说，单个`.proto`包含尽可能少的messages类型。

### Adding Comments

protocol buffers使用C/C++风格的注释，如下所示

```protobuf
/* SearchRequest represents a search query, with pagination options to
 * indicate which results to include in the response. */

message SearchRequest {
  required string query = 1;
  optional int32 page_number = 2;  // Which page number do we want?
  optional int32 result_per_page = 3;  // Number of results to return per page.
}
```

### Reserved Fields

考虑如下场景，用户A完全移除了message A中的fields A，但后续用户B重新将fields A添加回message A中，此时用户A开发的应用接收到用户B开发的应用发来的message A的请求时，将引起数据方面的错误。

通过将已删除的fields标记为reserved，后续用户继续使用这些fields时，protocol buffer compiler将报错。

> 引入了静态检查特性

```protobuf
message Foo {
  reserved 2, 15, 9 to 11; //通过标记标识数字为reserved标识fields无效
  reserved "foo", "bar"; //通过标记fields name为reserved标识fields无效
}
```

> 对于同一个reserved的语句，不能即使用标识数字又使用fields name

### What's Generated From Your .proto?

运行protocol buffers compiler后，编译器根据指定的语言不同，将`.proto`文件转换为处理messages的代码，代码一般包括获取/设置fields value，序列化messages，分析输入的messages等。不同语言生成的代码如下：

+ For **C++**, the compiler generates a .h and .cc file from each .proto, with a class for each message type described in your file.
+ For **Java**, the compiler generates a .java file with a class for each message type, as well as special Builder classes for creating message class instances.
+ **Python** is a little different – the Python compiler generates a module with a static descriptor of each message type in your .proto, which is then used with a metaclass to create the necessary Python data access class at runtime.
+ For **Go**, the compiler generates a .pb.go file with a type for each message type in your file.

## Scalar Value Types

scalar类型的fields有如下类型，其`.proto`文件中的类型，与编译器最终生成的文件类型之间的对应表如下

<table width="100%" border="2">

<tbody>

<tr>

<th>.proto Type</th>

<th>Notes</th>

<th>C++ Type</th>

<th>Java Type</th>

<th>Python Type<sup>[2]</sup></th>

<th>Go Type</th>

</tr>

<tr>

<td>double</td>

<td></td>

<td>double</td>

<td>double</td>

<td>float</td>

<td>*float64</td>

</tr>

<tr>

<td>float</td>

<td></td>

<td>float</td>

<td>float</td>

<td>float</td>

<td>*float32</td>

</tr>

<tr>

<td>int32</td>

<td>Uses variable-length encoding. Inefficient for encoding negative numbers – if your field is likely to have negative values, use sint32 instead.</td>

<td>int32</td>

<td>int</td>

<td>int</td>

<td>*int32</td>

</tr>

<tr>

<td>int64</td>

<td>Uses variable-length encoding. Inefficient for encoding negative numbers – if your field is likely to have negative values, use sint64 instead.</td>

<td>int64</td>

<td>long</td>

<td>int/long<sup>[3]</sup></td>

<td>*int64</td>

</tr>

<tr>

<td>uint32</td>

<td>Uses variable-length encoding.</td>

<td>uint32</td>

<td>int<sup>[1]</sup></td>

<td>int/long<sup>[3]</sup></td>

<td>*uint32</td>

</tr>

<tr>

<td>uint64</td>

<td>Uses variable-length encoding.</td>

<td>uint64</td>

<td>long<sup>[1]</sup></td>

<td>int/long<sup>[3]</sup></td>

<td>*uint64</td>

</tr>

<tr>

<td>sint32</td>

<td>Uses variable-length encoding. Signed int value. These more efficiently encode negative numbers than regular int32s.</td>

<td>int32</td>

<td>int</td>

<td>int</td>

<td>*int32</td>

</tr>

<tr>

<td>sint64</td>

<td>Uses variable-length encoding. Signed int value. These more efficiently encode negative numbers than regular int64s.</td>

<td>int64</td>

<td>long</td>

<td>int/long<sup>[3]</sup></td>

<td>*int64</td>

</tr>

<tr>

<td>fixed32</td>

<td>Always four bytes. More efficient than uint32 if values are often greater than 2<sup>28</sup>.</td>

<td>uint32</td>

<td>int<sup>[1]</sup></td>

<td>int/long<sup>[3]</sup></td>

<td>*uint32</td>

</tr>

<tr>

<td>fixed64</td>

<td>Always eight bytes. More efficient than uint64 if values are often greater than 2<sup>56</sup>.</td>

<td>uint64</td>

<td>long<sup>[1]</sup></td>

<td>int/long<sup>[3]</sup></td>

<td>*uint64</td>

</tr>

<tr>

<td>sfixed32</td>

<td>Always four bytes.</td>

<td>int32</td>

<td>int</td>

<td>int</td>

<td>*int32</td>

</tr>

<tr>

<td>sfixed64</td>

<td>Always eight bytes.</td>

<td>int64</td>

<td>long</td>

<td>int/long<sup>[3]</sup></td>

<td>*int64</td>

</tr>

<tr>

<td>bool</td>

<td></td>

<td>bool</td>

<td>boolean</td>

<td>bool</td>

<td>*bool</td>

</tr>

<tr>

<td>string</td>

<td>A string must always contain UTF-8 encoded or 7-bit ASCII text.</td>

<td>string</td>

<td>String</td>

<td>unicode (Python 2) or str (Python 3)</td>

<td>*string</td>

</tr>

<tr>

<td>bytes</td>

<td>May contain any arbitrary sequence of bytes.</td>

<td>string</td>

<td>ByteString</td>

<td>bytes</td>

<td>[]byte</td>

</tr>

</tbody>

</table>

+ <sup>[1]</sup> In Java, unsigned 32-bit and 64-bit integers are represented using their signed counterparts, with the top bit simply being stored in the sign bit.
+ <sup>[2]</sup> In all cases, setting values to a field will perform type checking to make sure it is valid.
+ <sup>[3]</sup> 64-bit or unsigned 32-bit integers are always represented as long when decoded, but can be an int if an int is given when setting the field. In all cases, the value must fit in the type represented when set. See [2].

## Optional Fields And Default Values

对于Optional fields而言，可配置其默认值，也即当某一message抵达时，若该message中的optional fields未被设置，该optional fields将被赋予默认值。

以[例子](#defining-a-message-type)为例，若要配置result_per_page的默认值为10，按照如下方式进行

```protobuf
optional int32 result_per_page = 3 [default = 10]; //3为标识数字，10为值
```

对于未通过`default`配置默认值的optional fields，根据fields的类型默认值进行默认值配置。

> If the default value is not specified for an optional element, a type-specific default value is used instead: for strings, the default value is the empty string. For bytes, the default value is the empty byte string. For bools, the default value is false. For numeric types, the default value is zero. For enums, the default value is the first value listed in the enum's type definition. This means care must be taken when adding a value to the beginning of an enum value list.

## Enumerations

protocol buffers支持`enume` fields，例子如下所示：

```protobuf
message SearchRequest {
  required string query = 1;
  optional int32 page_number = 2;
  optional int32 result_per_page = 3 [default = 10];
  enum Corpus {
    UNIVERSAL = 0; // UNIVERSAL为enum constants，0为enum constants的值
    WEB = 1;
    IMAGES = 2;
    LOCAL = 3;
    NEWS = 4;
    PRODUCTS = 5;
    VIDEO = 6;
  }
  optional Corpus corpus = 4 [default = UNIVERSAL];
}
```

`protocol buffers`中的enum为了解决某一fields的值来源于某一特定的值列表，比如上述例子中定义的message中的corpus域，该field的类型为Corpus类型，Corpus类型为enum，该field的值有7个，分别为`UNIVERSAL`, `WEB`, `IMAGES`, `LOCAL`, `NEWS`, `PRODUCTS` 以及 `VIDEO`，默认值为`UNIVERSAL`

`protocol buffers`中的enum支持别名，只需在定义是将同一个值赋给不同的enum常量(`enum constants`)，要利用该特性，需要在enum定义中将`allow_alias`选项设置为true，如下所示：

```protobuf
enum EnumAllowingAlias {
  option allow_alias = true;
  UNKNOWN = 0;
  STARTED = 1;
  RUNNING = 1;
}
enum EnumNotAllowingAlias {
  UNKNOWN = 0;
  STARTED = 1;
  // RUNNING = 1;  // Uncommenting this line will cause a compile error inside Google and a warning message outside.
}
```

上述例子中的STARTED与RUNNNING enum constants拥有同一个值1，因此，他们互为别名

> Enumerator constants must be in the range of a **32-bit integer**. Since enum values use varint encoding on the wire, n**egative values are inefficient and thus not recommended**. You can define enums within a message definition, as in the above example, or outside – these enums can be reused in any message definition in your .proto file. You can also use an enum type declared in one message as the type of a field in a different message, using the syntax `_MessageType_._EnumType_`.

> When you run the protocol buffer compiler on a .proto that uses an enum, the generated code will have a corresponding enum for Java or C++, or a special EnumDescriptor class for Python that's used to create a set of symbolic constants with integer values in the runtime-generated class.

### Reserved Values

通过reserved关键字将enum中的constants标记为废弃，后续用户若在使用该constants，则protocol buffer编译器直接报错

```protobuf
enum Foo {
  reserved 2, 15, 9 to 11, 40 to max;
  reserved "FOO", "BAR";
}
```

> 同样，对于同一行reserved语句而言，不可以既用数字形式又用constants name形式

## Using Other Message Types

protocol buffer支持用户自定义的message type作为message 中的fields类型，如例子所示

```protobuf
message SearchResponse {
  repeated Result result = 1;
}

message Result {
  required string url = 1;
  optional string title = 2;
  repeated string snippets = 3;
}
```

### Importing Definitions

protocol buffer同样支持从其他的`.proto`文件中导入已预先定义好的message类型，如下例子所示

```protobuf
import "myproject/other_protos.proto";
```

**通过`import public`可实现`.proto`文件的传递导入**，如下例子所示

```protobuf
// new.proto
// All definitions are moved here
```

```protobuf
// old.proto
// This is the proto that all clients are importing.
import public "new.proto";
import "other.proto";
```

```protobuf
// client.proto
import "old.proto";
// You use definitions from old.proto and new.proto, but not other.proto
```

> 默认情况下，protocol buffer的编译器仅在命令执行的文件夹下搜索导入的`.proto`文件，若需要导入其他路径下的`.proto`文件，需要在调用protocol buffer编译器时使用`-I/--proto_path`标志

> 通常`-I/--proto_path`标志参数被设置为项目的根路径，导入的`.proto`文件需要使用`fully qualified names`

### Using proto3 Message Types

可在proto2中导入proto3 message类型并使用，但在proto3中不可使用proto2的enum

## Nested Types

protocol buffers支持嵌套message类型，如下例子所示

```protobuf
message SearchResponse {
  message Result {
    required string url = 1;
    optional string title = 2;
    repeated string snippets = 3;
  }
  repeated Result result = 1;
}
```

上述例子中定义了名称为`SearchResponse` message类型，在`SearchResponse`中嵌套定义了名称为`Result`的Message类型，在`SearchRespone` 外部使用`Result`类型可通过下列方式进行，也即protocol buffers支持通过`_Parent_._Type_`方式引用message类型中嵌套定义的子message类型

```protobuf
message SomeOtherMessage {
  optional SearchResponse.Result result = 1;
}
```

protocol buffer支持任意深度的message类型嵌套，如下例子所示

```protobuf
message Outer {       // Level 0
  message MiddleAA {  // Level 1
    message Inner {   // Level 2
      required int64 ival = 1;
      optional bool  booly = 2;
    }
  }
  message MiddleBB {  // Level 1
    message Inner {   // Level 2
      required int32 ival = 1;
      optional bool  booly = 2;
    }
  }
}
```

## Updating A Message Type

更新message类型时遵循如下原则：

+ Don't change the field numbers for any existing fields.
Any new fields that you add should be optional or repeated. This means that any messages serialized by code using your "old" message format can be parsed by your new generated code, as they won't be missing any required elements. You should set up sensible default values for these elements so that new code can properly interact with messages generated by old code. Similarly, messages created by your new code can be parsed by your old code: old binaries simply ignore the new field when parsing. However, the unknown fields are not discarded, and if the message is later serialized, the unknown fields are serialized along with it – so if the message is passed on to new code, the new fields are still available.
+ Non-required fields can be removed, as long as the field number is not used again in your updated message type. You may want to rename the field instead, perhaps adding the prefix "OBSOLETE_", or make the field number reserved, so that future users of your .proto can't accidentally reuse the number.
+ A non-required field can be converted to an extension and vice versa, as long as the type and number stay the same.
+ int32, uint32, int64, uint64, and bool are all compatible – this means you can change a field from one of these types to another without breaking forwards- or backwards-compatibility. If a number is parsed from the wire which doesn't fit in the corresponding type, you will get the same effect as if you had cast the number to that type in C++ (e.g. if a 64-bit number is read as an int32, it will be truncated to 32 bits).
+ 0sint32 and sint64 are compatible with each other but are not compatible with the + other integer types.
+ string and bytes are compatible as long as the bytes are valid UTF-8.
+ Embedded messages are compatible with bytes if the bytes contain an encoded version of the message.
+ fixed32 is compatible with sfixed32, and fixed64 with sfixed64.
+ For string, bytes, and message fields, optional is compatible with repeated. Given serialized data of a repeated field as input, clients that expect this field to be optional will take the last input value if it's a primitive type field or merge all input elements if it's a message type field. Note that this is not generally safe for numeric types, including bools and enums. Repeated fields of numeric types can be serialized in the packed format, which will not be parsed correctly when an optional field is expected.
+ Changing a default value is generally OK, as long as you remember that default values are never sent over the wire. Thus, if a program receives a message in which a particular field isn't set, the program will see the default value as it was defined in that program's version of the protocol. It will NOT see the default value that was defined in the sender's code.
+ enum is compatible with int32, uint32, int64, and uint64 in terms of wire format (note that values will be truncated if they don't fit), but be aware that client code may treat them differently when the message is deserialized. Notably, unrecognized enum values are discarded when the message is deserialized, which makes the field's has.. accessor return false and its getter return the first value listed in the enum definition, or the default value if one is specified. In the case of repeated enum fields, any unrecognized values are stripped out of the list. However, an integer field will always preserve its value. Because of this, you need to be very careful when upgrading an integer to an enum in terms of receiving out of bounds enum values on the wire.
+ In the current Java and C++ implementations, when unrecognized enum values are stripped out, they are stored along with other unknown fields. Note that this can result in strange behavior if this data is serialized and then reparsed by a client that recognizes these values. In the case of optional fields, even if a new value was written after the original message was deserialized, the old value will be still read by clients that recognize it. In the case of repeated fields, the old values will appear after any recognized and newly-added values, which means that order will not be preserved.
+ Changing a single optional value into a member of a new oneof is safe and binary compatible. Moving multiple optional fields into a new oneof may be safe if you are sure that no code sets more than one at a time. Moving any fields into an existing oneof is not safe.
+ Changing a field between a map<K, V> and the corresponding repeated message field is binary compatible (see Maps, below, for the message layout and other restrictions). However, the safety of the change is application-dependent: when deserializing and reserializing a message, clients using the repeated field definition will produce a semantically identical result; however, clients using the map field definition may reorder entries and drop entries with duplicate keys.

## Extensions


