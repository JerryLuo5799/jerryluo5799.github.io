---
layout: post
title:  ".NET 6 中 gRPC 的新增功能"
date:   2022-01-15 21:42:55 +0800--
categories: [.NET Core]
tags: [.NET Core, .NET 6, gRPC]  
---

### 1. 前言
gRPC是一个现代的、跨平台的、高性能的 RPC 框架, 基于ASP.NET Core构建。
.NET 6 进一步提高了 gRPC 的性能，并添加了一系列新功能，使 gRPC 在云原生应用程序中比以往更好。

### 2. 客户端负载均衡

客户端负载平衡允许 gRPC 客户端在可用服务器之间优化分配负载。客户端负载平衡可以消除对负载平衡代理的需要。这有几个好处：

- **性能改进**。没有代理意味着消除额外的网络跃点并减少延迟，因为 RPC 直接发送到 gRPC 服务器。
- **更有效利用服务器资源**。负载平衡代理必须解析然后重新发送通过它发送的每个 HTTP 请求。删除代理可以节省 CPU 和内存资源。
- **更简单的应用程序架构**。代理服务器的配置和维护都需要一定的时间, 没有代理服务器意味着更简单的网络拓扑结构。
  

创建通道时配置客户端负载平衡。使用负载平衡时要考虑的两个组件：

解析器，用于解析通道的地址。解析器支持从外部来源获取地址。这也称为服务发现。
负载均衡器，它创建连接并选择 gRPC 调用将使用的地址。
以下代码示例将通道配置为使用具有循环负载平衡的 DNS 服务发现：

```CSharp
var channel = GrpcChannel.ForAddress(
    "dns:///my-example-host",
    new GrpcChannelOptions
    {
        Credentials = ChannelCredentials.Insecure,
        ServiceConfig = new ServiceConfig { LoadBalancingConfigs = { new RoundRobinConfig() } }
    });
var client = new Greet.GreeterClient(channel);

var response = await client.SayHelloAsync(new HelloRequest { Name = "world" });
```
![原理图](https://devblogs.microsoft.com/dotnet/wp-content/uploads/sites/10/2021/12/loadbalancing.gif)

更多信息, 可以查阅 [gRPC 客户端负载平衡](https://docs.microsoft.com/zh-cn/aspnet/core/grpc/loadbalancing?view=aspnetcore-6.0)

### 3. 故障处理
gRPC 调用可能会被瞬间的故障中断。瞬间故障包括：
- 瞬间失去网络连接。
- 服务暂时不可用。
- 由于服务器负载超时。

当 gRPC 调用中断时，客户端会抛出RpcException有关错误的详细信息。客户端应用程序必须捕获异常并选择如何处理错误。

```CSharp
var client = new Greeter.GreeterClient(channel);
try
{
    var response = await client.SayHelloAsync(
        new HelloRequest { Name = ".NET" });

    Console.WriteLine("From server: " + response.Message);
}
catch (RpcException ex)
{
    // Write logic to inspect the error and retry
    // if the error is from a transient fault.
}
```
自己实现整个重试逻辑是麻烦且容易出错的。所以, .NET gRPC 客户端现在内置了对自动重试的支持。重试在通道上集中配置，并且有许多重试策略.

```CSharp
var defaultMethodConfig = new MethodConfig
{
    Names = { MethodName.Default },
    RetryPolicy = new RetryPolicy
    {
        MaxAttempts = 5,
        InitialBackoff = TimeSpan.FromSeconds(1),
        MaxBackoff = TimeSpan.FromSeconds(5),
        BackoffMultiplier = 1.5,
        RetryableStatusCodes = { StatusCode.Unavailable }
    }
};

// Clients created with this channel will automatically retry failed calls.
var channel = GrpcChannel.ForAddress("https://localhost:5001", new GrpcChannelOptions
{
    ServiceConfig = new ServiceConfig { MethodConfigs = { defaultMethodConfig } }
});
```
更多信息可以查阅 [使用 gRPC 重试的瞬态故障处理](https://docs.microsoft.com/aspnet/core/grpc/retries)

### 4. Protobuf 性能提升
.NET 的 gRPC 使用 Google.Protobuf 包作为消息的默认序列化程序。Protobuf 是一种高效的二进制序列化格式。在 .NET 5 中，微软与 Protobuf 团队合作，为序列化程序添加了对Span<T>, ReadOnlySequence<T> 和 IBufferWriter<T>的支持。
.NET 6 中添加了矢量化字符串序列化 ([protocolbuffers/protobuf#8147](https://github.com/protocolbuffers/protobuf/pull/8147))。SIMD 指令允许并行处理多个字符，从而在序列化某些字符串值时显着提高性能。

### 5. 下载速度优化
用户发现有时gRPC的下载速度很慢。原因是当客户端和服务器之间存在延迟时，HTTP/2 流控制会限制下载。服务器在客户端耗尽之前填充接收缓冲区窗口，导致服务器暂停发送数据。gRPC 消息以启动/停止突发的形式下载。
这已经在[dotnet/runtime#54755](https://github.com/dotnet/runtime/pull/54755)中修复。HttpClient现在动态缩放接收缓冲区窗口。当建立 HTTP/2 连接时，客户端将向服务器发送 ping 以测量延迟。如果存在高延迟，客户端会自动增加接收缓冲区窗口，从而实现快速、连续的下载。

### 6. HTTP/3的支持
.NET 上的 gRPC 现在支持 HTTP/3。gRPC 构建在添加到 ASP.NET Core 和 .NET 6 中的 HTTP/3 支持之上HttpClient。有关更多信息，请参阅[.NET 6 中的 HTTP/3 支持](https://devblogs.microsoft.com/dotnet/http-3-support-in-dotnet-6/)。

### 7. 如何使用
如果希望使用 gRPC 的新功能，可以查阅 [ASP.NET Core 中创建 gRPC 客户端和服务器教程](https://docs.microsoft.com/zh-cn/aspnet/core/tutorials/grpc/grpc-start?view=aspnetcore-6.0&tabs=visual-studio)。



