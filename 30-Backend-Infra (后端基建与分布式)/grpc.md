---
tags:
  - grpc
  - network
  - infra
status: evergreen
related_project: "[[50-Projects (项目实战 - 最有价值的数据)/IM系统/业务需求]]"
---
# 什么是RPC
RPC 是 **远程过程调用（Remote Procedure Call）**

## 特性
1. RPC框架用长链接，不需要每次通信都3次握手，减少网络开销
2. RPC框架一般都有注册中心，有丰富的监控管理。发布、下线接口、动态扩展等，对调用方来说是无感知、统一化的操作协议私密，安全性较高
3. RPC协议更简单内容更小，效率更高，服务化架构、服务化治理，RPC框架是一个强力支撑

## 如何实现
如何找到目标服务：通过ip和port确定服务目标

调度目标：调用的是服务的方法：如useSever中的GetUser方法

```go
type Request struct {
    Method string `json:"method"`
    Params interface{} `json:"params"`
    Id string `json:"id"`
}

type Response struct {
    Id string `json:"id"`
    Result string `json:"result"`
    Error error `json:"error"`
}
```

数据可以用 json/xml/binary

rpc服务客户端基于 tcp/udp 及 http/websocket 等网络协议实现

### 调用流程
<!-- 这是一个文本绘图，源码为：sequenceDiagram
    participant 客户端
    participant client_stub as client stub
    participant server_stub as server stub
    participant 服务端

    客户端->>client_stub: 1.客户端调用
    activate client_stub
    client_stub->>client_stub: 2.序列化
    client_stub->>server_stub: 3.发送消息
    activate server_stub
    server_stub->>server_stub: 4.反序列化
    server_stub->>服务端: 5.服务端处理
    activate 服务端
    服务端->>server_stub: 6.返回处理结果
    deactivate 服务端
    server_stub->>server_stub: 7.结果序列化
    server_stub->>client_stub: 8.返回消息
    deactivate server_stub
    client_stub->>client_stub: 9.反序列化结果
    client_stub->>客户端: 10.返回结果
    deactivate client_stub -->
![](https://cdn.nlark.com/yuque/__mermaid_v3/560af155c5d397277e024f68f64036a7.svg)

### 核心功能实现
1. **客户端（服务消费端）**：远程调用方法的一端
2. **client stub（桩）：**代理类。把调用方法、类、方法参数等信息传递到服务端。
3. **网络传输：**
4. **服务端Stub（桩）：**这个桩不是代理类。接收到客户端执行方法的请求后，去执行对应的方法然后返回结果给客户端
5. **服务端：**提供远程方法的一端

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/61947663/1773103083314-da95366d-9f47-47ee-8c8d-0179feb5aab6.png)

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/61947663/1774232473855-abe0cee9-703c-4e27-990e-37b9bb089dd0.png)

**具体步骤：**

1. <font style="color:rgb(25, 27, 31);">服务消费者（client客户端）通过本地调用的方式调用服务。</font>
2. <font style="color:rgb(25, 27, 31);">客户端存根（client stub）接收到请求后负责将方法、入参等信息序列化（组装）成能够进行网络传输的消息  
</font><font style="color:rgb(25, 27, 31);">体。</font>
3. <font style="color:rgb(25, 27, 31);">客户端存根（client stub）找到远程的服务地址，并且将消息通过网络发送给服务端。</font>
4. <font style="color:rgb(25, 27, 31);">服务端存根（server stub）收到消息后进行解码（反序列化操作）。</font>
5. <font style="color:rgb(25, 27, 31);">服务端存根（server stub）根据解码结果调用本地的服务进行相关处理。</font>
6. <font style="color:rgb(25, 27, 31);">本地服务执行具体业务逻辑并将处理结果返回给服务端存根（server stub）。</font>
7. <font style="color:rgb(25, 27, 31);">服务端存根（server stub）将返回结果重新打包成消息（序列化）并通过网络发送至消费方。</font>
8. <font style="color:rgb(25, 27, 31);">客户端存根（client stub）接收到消息，并进行解码（反序列化）。</font>
9. <font style="color:rgb(25, 27, 31);">服务消费方得到最终结果。</font>

## 深入解析rpc
### 序列化技术
（1）作用：网络传输中，数据采用二进制形式，所以调用过程中需要对如蚕对象和返回值对象进行序列化和反序列化

（2）如何？

自定义二进制协议实现序列化

## 存在的问题
+ 缺乏跨语言的支持
+ 缺乏错误处理机制
+ 缺乏服务发现机制
+ 没有清晰的接口语义
+ 没有可热拔插的中间件
+ ……



# gRPC
> **<font style="color:rgb(34, 34, 34);">与许多 RPC 系统一样，gRPC 基于定义服务、指定可远程调用的方法及其参数和返回类型的理念。在服务器端，服务器实现此接口并运行 gRPC 服务器来处理客户端调用。在客户端，客户端有一个存根（在某些语言中简称为客户端），它提供与服务器相同的方法</font>**<font style="color:rgb(34, 34, 34);">。可以用java创建 gRPC服务器，用Go、Python或Ruby创建客户端</font>
>

**安装**

```go
go get github.com/golang/protobuf/proto
go getgoogle.golang.org/grpc（无法使用，用如下命令代替）
	git clone https://github.com/grpc/grpc-go.git $GopATH/src/google.golang.org/grpc
	git clone https://github.com/golang/net.git $GoPATH/src/golang.org/x/net
	git clone https://github.com/golang/text.git $GOPATH/src/golang.org/x/text
	go get -u github.com/golang/protobuf/{proto,protoc-gen-go}
	git clone https://github.com/google/go-genproto.git $GOPATH/src/google.golang.org/genproto
cd $GOPATH/src/
	go install google.golang.org/grpc
go get github.com/golang/protobuf/protoc-gen-go
上面安装好后，会在GOPATH/bin下生成protoc-gen-go.exe
但还需要一个protoc.exe，windows平台编译受限，很难自己手动编译，直接去网站下载一个，地址：
https://github.com/protocolbuffers/protobuf/releases/tag/v3.21.8，
同样放在GOPATH/bin下
```

## 序列化机制
默认情况下，gRPC使用 Protocol Buffers，成熟的开源结构化数据序列化机制

### 使用
使用 Protocol Buffers 的第一步是在proto 文件中定义要序列化的数据结构：这是一个带有 .proto 扩展名的普通文本文件。Protocol Buffer 数据以消息的形式结构化，其中每个消息都是一个包含一系列名称-值对（称为字段）的小型逻辑信息记录。

## 核心概念
### gRPC允许的四种服务方法：
+ **一元 RPC**
+ **服务器流式 RPC**

客户端向服务器发送请求获得一个流，用于读取一系列返回的消息。客户端从返回的流中读取，直到没有更多消息。保证单个 RPC 调用中的消息顺序

`rpc LotsOfReplies(HelloRequest) returns (stream HelloResponse);`

+ **客户端流式 RPC**

客户端写入一系列消息并发送到服务器，使用提供的流。客户端完成消息写入后会等待服务器读取消息并返回响应。同样保证单个 RPC 调用中的消息顺序。

`rpc LotsOfGreetings(stream HelloRequest) returns (HelloResponse);`

+ **双向流式 RPC**

双方使用读写流发送一系列消息。两个流独立运行，客户端和服务器可以以任何顺序读写。

`rpc BidiHello(stream HelloRequest) returns (stream HelloResponse);`

### 使用API
+ 服务器端：<font style="color:rgb(34, 34, 34);">服务器实现服务声明的方法，并运行 gRPC 服务器来处理客户端调用。gRPC 基础设施解码传入的请求，执行服务方法，并编码服务响应。</font>
+ <font style="color:rgb(34, 34, 34);">客户端：客户端有一个称为 </font>_<font style="color:rgb(34, 34, 34);">存根</font>_<font style="color:rgb(34, 34, 34);">（对于某些语言，首选术语是 </font>_<font style="color:rgb(34, 34, 34);">客户端</font>_<font style="color:rgb(34, 34, 34);">）的本地对象，它实现与服务相同的方法。客户端可以直接在本地对象上调用这些方法，这些方法会将调用参数封装成相应的 Protocol Buffers 消息类型，发送请求到服务器，并返回服务器的 Protocol Buffers 响应。</font>

### 同步与异步
同步 RPC 调用会阻塞知道服务器响应到达。网络本质上是异步的，在大部分场景下，在不阻塞当前线程的情况下启动 RPC 很有用

### RPC 生命周期
#### 一元RPC 
<font style="color:rgb(34, 34, 34);">客户端发送单个请求并获得单个响应。</font>

1. <font style="color:rgb(34, 34, 34);">一旦客户端调用存根方法，服务器会收到通知，表明 RPC 已被调用，并带有客户端的此调用</font><font style="color:rgb(34, 34, 34);"> </font>[<font style="color:rgb(55, 156, 156);">元数据</font>](https://grpc.org.cn/docs/what-is-grpc/core-concepts/#metadata)<font style="color:rgb(34, 34, 34);">、方法名称以及适用的指定</font><font style="color:rgb(34, 34, 34);"> </font>[<font style="color:rgb(55, 156, 156);">截止时间</font>](https://grpc.org.cn/docs/what-is-grpc/core-concepts/#deadlines)<font style="color:rgb(34, 34, 34);">。</font>
2. <font style="color:rgb(34, 34, 34);">服务器随后可以立即发送回自己的初始元数据（必须在任何响应之前发送），或者等待客户端的请求消息。哪个先发生，取决于应用程序。</font>
3. <font style="color:rgb(34, 34, 34);">一旦服务器收到客户端的请求消息，它会执行必要的工作来创建并填充响应。然后，响应（如果成功）与状态详情（状态码和可选状态消息）以及可选的尾随元数据一起返回给客户端。</font>
4. <font style="color:rgb(34, 34, 34);">如果响应状态为 OK，则客户端收到响应，从而在客户端完成调用。</font>

#### 服务器流式 RPC 
与一元 RPC 类似，不同之处在于服务器响应客户端请求时返回的是消息流。<font style="color:rgb(34, 34, 34);">发送完所有消息后，服务器的状态详情（状态码和可选状态消息）和可选的尾随元数据会发送给客户端。这表示服务器端处理完成。</font>**<font style="color:rgb(34, 34, 34);">客户端在收到所有服务器消息后完成。</font>**

#### 客户端流式 RPC
与一元 RPC 类似，不同之处在于客户端向服务器发送的是消息流而不是单个消息。<font style="color:rgb(34, 34, 34);">服务器会返回</font>**<font style="color:rgb(34, 34, 34);">单个消息</font>**<font style="color:rgb(34, 34, 34);">（连同其状态详情和可选的尾随元数据），</font>**<font style="color:rgb(34, 34, 34);">通常（但不一定）</font>**<font style="color:rgb(34, 34, 34);">在收到所有客户端消息之后。</font>

### <font style="color:rgb(34, 34, 34);">元数据</font>
元数据是关于特定 RPC 调用的信息，形式为键值对列表，键是字符串，值通常是字符串，也可以是二进制数据

<font style="color:rgb(34, 34, 34);">键不区分大小写，由 ASCII 字母、数字和特殊字符 </font>`<font style="background-color:rgba(0, 0, 0, 0.05);">-</font>`<font style="color:rgb(34, 34, 34);">、</font>`<font style="background-color:rgba(0, 0, 0, 0.05);">_</font>`<font style="color:rgb(34, 34, 34);">、</font>`<font style="background-color:rgba(0, 0, 0, 0.05);">.</font>`<font style="color:rgb(34, 34, 34);"> 组成，且不得以 </font>`<font style="background-color:rgba(0, 0, 0, 0.05);">grpc-</font>`<font style="color:rgb(34, 34, 34);"> 开头（此部分保留给 gRPC 自身使用）。二进制值键以 </font>`<font style="background-color:rgba(0, 0, 0, 0.05);">-bin</font>`<font style="color:rgb(34, 34, 34);"> 结尾，而 ASCII 值键则不以 </font>`<font style="background-color:rgba(0, 0, 0, 0.05);">-bin</font>`<font style="color:rgb(34, 34, 34);"> 结尾。</font>

<font style="color:rgb(34, 34, 34);">用户定义的元数据不被 gRPC 使用，它允许客户端向服务器提供与调用相关的信息，反之亦然。</font>

### <font style="color:rgb(34, 34, 34);">通道</font>
<font style="color:rgb(34, 34, 34);">gRPC 通道提供与指定主机和端口上的 gRPC 服务器的连接。它在创建客户端存根时使用。客户端可以指定通道参数来修改 gRPC 的默认行为，例如打开或关闭消息压缩。通道具有状态，包括</font><font style="color:rgb(34, 34, 34);"> </font>`<font style="background-color:rgba(0, 0, 0, 0.05);">connected</font>`<font style="color:rgb(34, 34, 34);">（已连接）和</font><font style="color:rgb(34, 34, 34);"> </font>`<font style="background-color:rgba(0, 0, 0, 0.05);">idle</font>`<font style="color:rgb(34, 34, 34);">（空闲）。</font>

<font style="color:rgb(34, 34, 34);">gRPC 处理关闭通道的方式是语言相关的。一些语言也允许查询通道状态</font>

### 
## <font style="color:rgb(34, 34, 34);">Grpc应用</font>
### 原理实现
#### 服务端
##### 拦截器
`chainUnaryInterceptors` 多个拦截器按序执行

递归闭包

假设 `interceptors` 长度为 3：`[Int0, Int1, Int2]`，业务逻辑为 `Final`

**step1** `chainUnaryInterceptors` 返回一个闭包。当你真正发起 RPC 调用时，它执行： `interceptors[0](ctx, req, info, getChainUnaryHandler(..., 0, ...))`

**此时状态**：`Int0` 开始执行。它手里拿到的 `handler` 是 `getChainUnaryHandler` 返回的一个待命函数。 

**step2 **`Int0`执行的`handler`是`getChainUnaryHandler`闭包，此时`curr`为0，`len(interceptors)-1`为2，执行`interceptors[1](ctx, req, info, getChainUnaryHandler(..., 1, ...))`

**此时状态**：执行流进入 `Int1`。

**step3 **跟上述一致，递归执行`Int2`

**step4 **执行`Int2`时，此时 `curr = 2`，满足了 `if curr == len(interceptors)-1`。 这一次，`getChainUnaryHandler` 不再返回调用 `interceptors[3]` 的闭包，而是直接返回`**finalHandler**`

**step5 **`Int2` 调用 `handler`，实际上就是直接运行了业务代码 `Final`

```go
func chainUnaryInterceptors(interceptors []UnaryServerInterceptor) UnaryServerInterceptor {
	return func(ctx context.Context, req any, info *UnaryServerInfo, handler UnaryHandler) (any, error) {
		return interceptors[0](ctx, req, info, getChainUnaryHandler(interceptors, 0, info, handler))
	}
}

func getChainUnaryHandler(interceptors []UnaryServerInterceptor, curr int, info *UnaryServerInfo, finalHandler UnaryHandler) UnaryHandler {
	if curr == len(interceptors)-1 {
		return finalHandler
	}
	return func(ctx context.Context, req any) (any, error) {
		return interceptors[curr+1](ctx, req, info, getChainUnaryHandler(interceptors, curr+1, info, finalHandler))
	}
}
```

##### 请求监听处理
整体作用：不断从监听器 `lis` 调用 `Accept()` 获取新连接，并为每个连接开启一个 goroutine 去处理（调用 `handleRawConn`），直到遇到致命错误或服务器关闭。



1. **临时错误重试：**

通过类型断言判断错误是否实现了 `Temporary() bool`（通常是 net.Error 的临时错误）。

若为临时错误，使用指数退避：初始 5ms，每次翻倍，最大限为 1s（`tempDelay = min(tempDelay, 1*time.Second)`）。

日志记录重试信息（加锁 `s.mu.Lock() -> s.printf() -> s.mu.Unlock()`）。

使用 `time.NewTimer(tempDelay)` 等待退避期，同时在 `select` 中监听 `s.quit.Done()`，如果收到退出信号则停止定时器并返回 `nil`，终止 `Serve`。



2. **非临时错误处理：**

记录 `done serving; Accept = %v` 日志（也在 s.mu 保护下）。

若 `s.quit.HasFired()`（表示是正常的停止/优雅停机触发），返回 `nil`（正常退出），否则返回该错误（将向调用者上报）。

成功接收连接时：



3. **将 tempDelay 重置为 0（清除任何退避状态）。**

在开启处理连接的 goroutine 之前调用 `s.serveWG.Add(1)`：这是为了使 `GracefulStop`/停服流程能正确等待或不在 `race` 条件下清理 `s.conns`（注释说明：确保在并发下不会在连接还没被添加到记录中就被置空）。

启动匿名 `goroutine`：调用 `s.handleRawConn(lis.Addr().String(), rawConn)` 处理该原始连接，结束时调用 `s.serveWG.Done()` 以配对前面的 `Add(1)`。



4. **并发与资源管理要点：**

使用 `serveWG` 跟踪为每个连接启动的 goroutine，确保在优雅停机时能等待它们结束或正确同步。

日志调用被 `s.mu` 保护，避免竞态。

在临时错误等待期间监听 `s.quit.Done()`，保证在服务关闭时不会无谓等待完成退避。

```go
	var tempDelay time.Duration // how long to sleep on accept failure
	for {
		rawConn, err := lis.Accept()
		if err != nil {
			if ne, ok := err.(interface {
				Temporary() bool
			}); ok && ne.Temporary() {
				if tempDelay == 0 {
					tempDelay = 5 * time.Millisecond
				} else {
					tempDelay *= 2
					tempDelay = min(tempDelay, 1*time.Second) // 最大为1秒
				}
				s.mu.Lock()
				s.printf("Accept error: %v; retrying in %v", err, tempDelay)
				s.mu.Unlock()
				timer := time.NewTimer(tempDelay)
				select {
				case <-timer.C:
				case <-s.quit.Done(): //管理员按下停止，退出
					timer.Stop()
					return nil
				}
				continue
			}
			s.mu.Lock()
			s.printf("done serving; Accept = %v", err)
			s.mu.Unlock()

			if s.quit.HasFired() {
				return nil
			}
			return err
		}
		tempDelay = 0
        // Start a new goroutine to deal with rawConn so we don't stall this Accept
		// loop goroutine.
		//
		// Make sure we account for the goroutine so GracefulStop doesn't nil out
		// s.conns before this conn can be added.
		s.serveWG.Add(1)
		go func() {
			s.handleRawConn(lis.Addr().String(), rawConn)
			s.serveWG.Done()
		}()
    }
```

<!-- 这是一个文本绘图，源码为：flowchart TD
    %% 定义节点与样式
    Start([Accept])
    CheckErr{err != nil}
    
    %% 主流程控制
    Start --> CheckErr
    
    %% 虚线框内部逻辑
    subgraph 处理循环体内
        direction TB
        
        %% 出错分支 (True)
        CheckErr -- "true" --> SetDelay[set tempDelay]
        SetDelay --> Sleep["select 睡眠"]
        
        %% 成功分支 (False)
        CheckErr -- "false" --> ClearDelay[clear tempDelay]
        ClearDelay --> HandleConn["s.handleRawConn 处理"]
    end
    
    %% 返回 Accept 的循环线 (对应 for 循环)
    Sleep -. "继续监听" .-> Start
    HandleConn -. "继续监听" .-> Start -->
![](https://cdn.nlark.com/yuque/__mermaid_v3/654779cdd8a988feca3784d2f84b9fe9.svg)

#### 客户端
##### Grpc注册
通过类库文件`RegistreService`完成注册，调用grpc server去注册

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/61947663/1774095407102-4fdea024-e60c-48d5-9847-c96567e204c5.png)

编译`.proto`文件，生成了`user_grpc.pb.go`的类库文件，`client`端调用类库文件通过grpc调到请求对象。



初始化`clientConn` -> 初始化重试机制和拦截器 -> 初始化`channelz` -> 验证连接协议方式并初始化 -> 设置请求超时 -> 初始化解析器 -> 与服务连接

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/61947663/1774233890018-a68b6e8b-7c3d-494b-bde7-bf0386b14cf0.png)

### 如何
加载服务 `map[服务名]*serverInfo(map[方法名]方法)`

