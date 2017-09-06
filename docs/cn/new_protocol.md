# server端多协议

baidu-rpc server在同端口支持所有的协议，大部分时候这对部署和运维更加方便。由于不同协议的格式大相径庭，严格地来说，同端口很难无二义地支持所有协议。出于解耦和可扩展性的考虑，也不太可能集中式地构建一个针对所有协议的分类器。我们的做法就是把协议归三类后逐个尝试：

- 第一类协议：标记或特殊字符在最前面，比如[标准协议](http://gollum.baidu.com/ProtobufRPC)，[hulu协议](http://wiki.babel.baidu.com/twiki/bin/view/Com/Main/Hulu_rpc_protocols)的前4个字符分别分别是PRPC和HULU，解析代码只需要检查前4个字节就可以知道协议是否匹配，最先尝试这类协议。这些协议在同一个连接上也可以共存。
- 第二类协议：有较为复杂的语法，没有固定的协议标记或特殊字符，可能在解析一段输入后才能判断是否匹配，目前此类协议只有http。
- 第三类协议：协议标记或特殊字符在中间，比如[nshead](http://wiki.babel.baidu.com/twiki/bin/view/Com/Main/LibNshead)的magic_num在第25-28字节。由于之前的字段均为二进制，难以判断正确性，在没有读取完28字节前，我们无法判定消息是不是nshead格式的，所以处理起来很麻烦，若其解析排在http之前，那么<=28字节的http消息便可能无法被解析，因为程序以为是“还未完整的nshead消息”。

考虑到大多数链接上只会有一种协议，我们会记录前一次的协议选择结果，下次首先尝试。对于长连接，这几乎把甄别协议的开销降到了0；虽然短连接每次都得运行这段逻辑，但由于短连接的瓶颈也往往不在于此，这套方法仍旧是足够快的。在未来如果有大量的协议加入，我们可能得考虑一些更复杂的启发式的区分方法。

# client端多协议

不像server端必须根据连接上的数据动态地判定协议，client端作为发起端，自然清楚自己的协议格式，只要某种协议只通过连接池或短链接发送，即独占那个连接，那么它可以是任意复杂（或糟糕的）格式。因为client端发出时会记录所用的协议，等到response回来时直接调用对应的解析函数，没有任何甄别代价。像memcache，redis这类协议中基本没有magic number，在server端较难和其他协议区分开，但让client端支持却没什么问题。

# 支持新协议

baidu-rpc就是设计为可随时扩展新协议的，步骤如下：

> 以nshead开头的协议有统一支持，看[这里](nshead_service.md)。

## 增加ProtocolType

在[options.proto](http://icode.baidu.com/repo/baidu/opensource/baidu-rpc/files/master/blob/src/brpc/options.proto)的ProtocolType中增加新协议类型，如果你需要的话可以联系我们增加，以确保不会和其他人的需求重合。

目前的ProtocolType（16年底）:
```c++
enum ProtocolType {
    PROTOCOL_UNKNOWN = 0;
    PROTOCOL_BAIDU_STD = 1;
    PROTOCOL_STREAMING_RPC = 2;
    PROTOCOL_HULU_PBRPC = 3;
    PROTOCOL_SOFA_PBRPC = 4;
    PROTOCOL_RTMP = 5;
    PROTOCOL_HTTP = 6;
    PROTOCOL_PUBLIC_PBRPC = 7;
    PROTOCOL_NOVA_PBRPC = 8;
    PROTOCOL_NSHEAD_CLIENT = 9;        // implemented in baidu-rpc-ub
    PROTOCOL_NSHEAD = 10;
    PROTOCOL_HADOOP_RPC = 11;
    PROTOCOL_HADOOP_SERVER_RPC = 12;
    PROTOCOL_MONGO = 13;               // server side only
    PROTOCOL_UBRPC_COMPACK = 14;
    PROTOCOL_DIDX_CLIENT = 15;         // Client side only
    PROTOCOL_REDIS = 16;               // Client side only
    PROTOCOL_MEMCACHE = 17;            // Client side only
    PROTOCOL_ITP = 18;
    PROTOCOL_NSHEAD_MCPACK = 19;
    PROTOCOL_DISP_IDL = 20;            // Client side only
    PROTOCOL_ERSDA_CLIENT = 21;        // Client side only
    PROTOCOL_UBRPC_MCPACK2 = 22;       // Client side only
}
```
## 实现回调

均定义在struct Protocol中，该结构定义在[protocol.h](http://icode.baidu.com/repo/baidu/opensource/baidu-rpc/files/master/blob/src/brpc/protocol.h)。其中的parse必须实现，除此之外server端至少要实现process_request，client端至少要实现serialize_request，pack_request，process_response;

实现协议回调还是比较困难的，这块的代码不会像供普通用户使用的那样，有较好的提示和保护，你得先靠自己搞清楚其他协议中的类似代码，然后再动手，最后发给我们做code review。

### parse

```c++
typedef ParseResult (*Parse)(base::IOBuf* source, Socket *socket, bool read_eof, const void *arg);
```
用于把消息从source上切割下来，client端和server端使用同一个parse函数。返回的消息会被递给process_request(server端)或process_response(client端)。

参数：source是读取到的二进制内容，socket是对应的连接，read_eof为true表示连接已被对端关闭，arg在server端是对应server的指针，在client端是NULL。

ParseResult可能是错误，也可能包含一个切割下来的message，可能的值有：

- PARSE_ERROR_TRY_OTHERS ：不是这个协议，框架会尝试下一个协议。source不能被消费。
- PARSE_ERROR_NOT_ENOUGH_DATA : 到目前为止数据内容不违反协议，但不构成完整的消息。等到连接上有新数据时，新数据会被append入source并重新调用parse。如果不确定数据是否一定属于这个协议，source不应被消费，如果确定数据属于这个协议，也可以把source的内容转移到内部的状态中去。比如http协议解析中即使source不包含一个完整的http消息，它也会被http parser消费掉，以避免下一次重复解析。
- PARSE_ERROR_TOO_BIG_DATA : 消息太大，拒绝掉以保护server，连接会被关闭。
- PARSE_ERROR_NO_RESOURCE  : 内部错误，比如资源分配失败。连接会被关闭。
- PARSE_ERROR_ABSOLUTELY_WRONG  : 应该是这个协议（比如magic number匹配了），但是格式不符合预期。连接会被关闭。

### serialize_request
```c++
typedef bool (*SerializeRequest)(base::IOBuf* request_buf,
                                 Controller* cntl,
                                 const google::protobuf::Message* request);
```
把request序列化进request_buf，client端必须实现。发生在pack_request之前，一次RPC中只会调用一次。cntl包含某些协议（比如http）需要的信息。成功返回true，否则false。

### pack_request
```c++
typedef int (*PackRequest)(base::IOBuf* msg, 
                           uint64_t correlation_id,
                           const google::protobuf::MethodDescriptor* method,
                           Controller* controller,
                           const base::IOBuf& request_buf,
                           const Authenticator* auth);
```
把request_buf打包入msg，每次向server发送消息前（包括重试）都会调用。当auth不为空时，需要打包认证信息。成功返回0，否则-1。

### process_request
```c++
typedef void (*ProcessRequest)(InputMessageBase* msg_base);
```
处理server端parse返回的消息，server端必须实现。可能会在和parse()不同的线程中运行。多个process_request可能同时运行。

在r34386后必须在处理结束时调用msg_base->Destroy()，为了防止漏调，考虑使用DestroyingPtr<>。

### process_response
```c++
typedef void (*ProcessResponse)(InputMessageBase* msg);
```
处理client端parse返回的消息，client端必须实现。可能会在和parse()不同的线程中运行。多个process_response可能同时运行。

在r34386后必须在处理结束时调用msg_base->Destroy()，为了防止漏调，考虑使用DestroyingPtr<>。

### verify
```c++
typedef bool (*Verify)(const InputMessageBase* msg);
```
处理连接的认证，只会对连接上的第一个消息调用，需要支持认证的server端必须实现，不需要认证或仅支持client端的协议可填NULL。成功返回true，否则false。

### parse_server_address
```c++
typedef bool (*ParseServerAddress)(base::EndPoint* out, const char* server_addr_and_port);
```
把server_addr_and_port(Channel.Init的一个参数)转化为base::EndPoint，可选。一些协议对server地址的表达和理解可能是不同的。

### get_method_name
```c++
typedef const std::string& (*GetMethodName)(const google::protobuf::MethodDescriptor* method,
                                            const Controller*);
```
定制method name，可选。

### supported_connection_type

标记支持的连接方式。如果支持所有连接方式，设为CONNECTION_TYPE_ALL。如果只支持连接池和短连接，设为CONNECTION_TYPE_POOLED_AND_SHORT。

### name

协议的名称，会出现在各种配置和显示中，越简短越好，必须是字符串常量。

## 注册到全局

实现好的协议要调用RegisterProtocol[注册到全局](http://icode.baidu.com/repo/baidu/opensource/baidu-rpc/files/master/blob/src/brpc/global.cpp)，以便baidu-rpc发现。就像这样：
```c++
Protocol http_protocol = { ParseHttpMessage,
                           SerializeHttpRequest, PackHttpRequest,
                           ProcessHttpRequest, ProcessHttpResponse,
                           VerifyHttpRequest, ParseHttpServerAddress,
                           GetHttpMethodName,
                           CONNECTION_TYPE_POOLED_AND_SHORT,
                           "http" };
if (RegisterProtocol(PROTOCOL_HTTP, http_protocol) != 0) {
    exit(1);
}
```

## r34386引入的不兼容

为了进一步简化protocol的实现逻辑，r34386是一个不兼容改动，主要集中在下面几点：

- ProcessXXX必须在处理结束时调用msg_base->Destroy()。在之前的版本中，这是由框架完成的。这个改动帮助我们隐藏处理EOF的代码（很晦涩），还可以在未来支持更异步的处理（退出ProcessXXX不意味着处理结束）。为了确保所有的退出分支都会调用msg_base->Destroy()，可以使用定义在[destroying_ptr.h](http://icode.baidu.com/repo/baidu/opensource/baidu-rpc/files/master/blob/src/brpc/destroying_ptr.h)中的DestroyingPtr<>，可能像这样：
```c++
void ProcessXXXRequest(InputMessageBase* msg_base) {
    DestroyingPtr<MostCommonMessage> msg(static_cast<MostCommonMessage*>(msg_base));
    ...
}
```

- 具体请参考[其他协议](http://icode.baidu.com/repo/baidu/opensource/baidu-rpc/files/master/blob/src/brpc/policy/baidu_rpc_protocol.cpp)的实现。
- InputMessageBase::socket_id()被移除，而通过socket()可以直接访问到对应Socket的指针。ProcessXXX函数中Address Socket的代码可以移除。
  ProcessXXXRequest开头的修改一般是这样：
```c++

void ProcessXXXRequest(InputMessageBase* msg_base) {
     const int64_t start_parse_us = base::cpuwide_time_us();
-    MostCommonMessage* msg = static_cast<MostCommonMessage*>(msg_base);
+    DestroyingPtr<MostCommonMessage> msg(static_cast<MostCommonMessage*>(msg_base));
+    SocketUniquePtr socket(msg->ReleaseSocket());
     const Server* server = static_cast<const Server*>(msg_base->arg());
     ScopedNonServiceError non_service_error(server);
-    const SocketId sock = msg_base->socket_id();
-    SocketUniquePtr socket;
-    if (Socket::Address(sock, &socket) != 0) {
-        RPC_VLOG << "Fail to address client=" << sock;
-        return;
-    }
-    if (socket->CheckEOF()) {
-        // Received an EOF event
-        return;
-    }
```
ProcessXXXResponse开头的修改一般是这样：
```c++
void ProcessRpcResponse(InputMessageBase* msg_base) {
     const int64_t start_parse_us = base::cpuwide_time_us();
-    MostCommonMessage* msg = static_cast<MostCommonMessage*>(msg_base);
-    CheckEOFGuard eof_guard(msg->socket_id());
+    DestroyingPtr<MostCommonMessage> msg(static_cast<MostCommonMessage*>(msg_base));
 
     ...
 
-    // After a successful fight, EOF will no longer interrupt the
-    // following code. As a result, we can release `eof_guard'
-    eof_guard.check();
```

check_eof_guard.h被移除，所以对这个文件的include也得移除：

```c++
-    #include "brpc/details/check_eof_guard.h"
```

- AddClientSideHandler被移除，用如下方法代替：
```
-   if (AddClientSideHandler(handler) != 0) {
+   if (get_or_new_client_side_messenger()->AddHandler(handler) != 0) {
```