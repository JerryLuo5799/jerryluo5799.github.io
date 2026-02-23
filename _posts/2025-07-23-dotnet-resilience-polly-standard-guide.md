---
layout: post
title:  "告别Polly, 拥抱.NET标准化弹性策略"
date:   2025-07-23 12:30:00 +0800--
categories: [.NET]
tags: [Resilience]
---

## 前言

在分布式系统和云原生开发中，**“弹性（Resilience）”** 是一个绕不开的话题。当我们的程序调用第三方 API 或数据库时，网络抖动、服务瞬时宕机或触发限流是家常便饭。

长期以来，.NET 社区一直依赖 [Polly](https://github.com/App-vNext/Polly?wt.mc_id=MVP_324329) 库来实现重试（Retry）、熔断（Circuit Breaker）等机制。但在 **.NET 8** 中，微软官方对这些策略进行了深度集成和标准化封装。今天，我们就来聊聊为什么你该放下手中的原始 Polly，转而使用官方推荐的新姿势。

## 核心概念：什么是“标准化弹性”？

以前，我们需要手动配置 Polly 的 `Policy` 对象，代码往往与业务逻辑耦合在一起。而在 .NET 8 中，微软推出了 `Microsoft.Extensions.Resilience` 库，其核心逻辑包括：

* **策略管道（Resilience Pipeline）**：将多种策略（重试、超时、限流等）组合成一个有序的“管道”。
* **依赖注入集成**：策略不再是孤零零的工具类，而是可以通过 DI 容器管理、按名称领取的系统组件。
* **抖动（Jitter）技术**：这是高级重试中的灵魂。**Jitter** 通过在重试间隔中引入随机性，打散并发压力，防止“惊群效应”。

## 实战演示：从零构建弹性管道

### 第一步：安装 NuGet 包

```bash
dotnet add package Microsoft.Extensions.Http.Resilience

```

### 第二步：配置并注入策略

在 `Program.cs` 中，我们可以定义一个名为 `"default"` 的弹性策略。

```csharp
builder.Services.AddResiliencePipeline("default", x =>
{
    x.AddRetry(new RetryStrategyOptions
    {
        // 定义什么情况下触发重试
        ShouldHandle = new PredicateBuilder().Handle<Exception>(),
        // 重试次数
        MaxRetryAttempts = 2,
        // 退避策略：指数级增加等待时间（2s, 4s...）
        BackoffType = DelayBackoffType.Exponential,
        // 开启 Jitter 随机化延迟
        UseJitter = true,
        Delay = TimeSpan.FromSeconds(2)
    })
    .AddTimeout(TimeSpan.FromSeconds(30)); 
});

```

### 第三步：在服务中使用策略

通过注入 `ResiliencePipelineProvider` 获取并执行代码：

```csharp
public class WeatherService(ResiliencePipelineProvider<string> pipelineProvider, HttpClient httpClient)
{
    public async Task<string> GetWeatherAsync(string city, CancellationToken ct)
    {
        var pipeline = pipelineProvider.GetPipeline("default");

        return await pipeline.ExecuteAsync(async token =>
        {
            var response = await httpClient.GetAsync($"https://api.weather.com/{city}", token);
            response.EnsureSuccessStatusCode();
            return await response.Content.ReadAsStringAsync(token);
        }, ct);
    }
}

```

## 深度探秘：底层实现原理

作为开发者，不仅要会写代码，更要明白一行代码背后的“内功”。`Microsoft.Extensions.Http.Resilience` 的实现主要基于以下两个支柱：

### 1. 基于 DelegatingHandler 的链式架构

`HttpClient` 的核心是一个类似 ASP.NET Core 中间件的链式结构。每一个请求都会经过一系列的 **处理器（Handlers）**。该扩展包的本质是向链条中插入了一个 `ResilienceHandler`。

当你发送请求时，它会进入此处理器，处理器根据你定义的管道包裹内部的 `SendAsync` 调用。这意味着重试、熔断等逻辑对你的业务代码是完全透明的。

### 2. 标准策略栈的“五层防护”

当你调用 `AddStandardResilienceHandler()` 时，微软实际上为你配置了一个经过工业验证的策略栈。它们的执行顺序（从外到内）非常有讲究：

| 策略层级 | 名称 | 核心作用 |
| --- | --- | --- |
| **第 1 层** | **Total Request Timeout** | 顶层超时。管你重试几次，总时间超过阈值（默认 30s）即断开。 |
| **第 2 层** | **Standard Retry** | 核心重试。处理 5xx 错误或特定异常，带 Jitter 的指数退避。 |
| **第 3 层** | **Standard Circuit Breaker** | 熔断器。发现下游持续挂掉时，直接“跳闸”保护系统。 |
| **第 4 层** | **Attempt Timeout** | 单次尝试超时。防止单次请求卡死太久。 |
| **第 5 层** | **Rate Limiter** | 限流。控制向目标服务发送请求的最高频率。 |

> **架构思考**：**Total Timeout** 必须在最外面，因为它要控制整个生命周期；而 **Attempt Timeout** 在最里面，它只负责管好当下的那一次连接。

## 性能与避坑

### 1. 避免频繁构建管道

**Resilience Pipeline 的构建是一个相对“重”的操作**。

* **错误做法**：在每个 Request 进来时现场 `new` 一个 Builder。
* **正确做法**：通过 DI 注册，由框架单例管理；或者使用 `AddStandardResilienceHandler()` 直接绑定到 `HttpClient`。

### 2. 幂等性陷阱

默认的 `StandardResilienceHandler` 会重试 `5xx` 错误。如果你的接口在 `500` 时仍会产生副作用（如非幂等扣费），请务必手动调整 `ShouldHandle` 逻辑，否则可能会导致严重的数据问题。

## 总结

.NET 8 的标准化弹性方案不仅简化了代码，更通过高性能的 **Polly V8** 引擎提升了系统稳定性。它完美契合了 .NET 的设计哲学：**高内聚（策略定义）、低耦合（通过 Handler 注入）**。

### 相关链接

* [Microsoft Learn: .NET 弹性策略官方文档](https://learn.microsoft.com/zh-cn/dotnet/core/resilience/?wt.mc_id=MVP_324329)
* [Polly 官方 GitHub 仓库](https://github.com/App-vNext/Polly?wt.mc_id=MVP_324329)
