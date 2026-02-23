---
layout: post
title:  "搞懂.NET依赖注入生命周期：Scoped 不仅仅是为了Web请求"
date:   2025-04-23 12:30:00 +0800--
categories: [.NET]
tags: [DI]
---

## 前言

在 .NET 开发中，依赖注入（Dependency Injection）是构建解耦系统的基石。大部分开发者都能秒答 **Transient（瞬时）** 和 **Singleton（单例）** 的区别，但面对 **Scoped（作用域）** 时，往往存在一个思维定势：“它就是为一次 HTTP 请求准备的。”

如果你的代码离开了 Web 环境，比如在后台任务（Background Service）或消息消费端中，Scoped 服务该如何使用？如果不小心在 Scoped 作用域里开了一个新线程，会发生什么灾难？今天我们就来拆解 Scoped 命周期的“内功心法”。

## 核心概念：Scoped 的本质是“气泡”

在 DI 容器中，`Scoped` 对象的生命周期绑定到了 `IServiceScope` 这个对象上。

* **自动Scope（Web）**：在 ASP.NET Core 中，框架为每个 HTTP 请求自动创建一个 Scope。请求结束，服务销毁。
* **手动Scope（非 Web）**：在后台服务等场景，没有自动创建的 Scope。如果你试图在单例中直接注入 Scoped 服务，系统会抛出异常，防止“生命周期提升”导致的内存泄漏。

为了更好地理解这个逻辑，我们可以观察下面的生命周期拓扑图：

![DI Scoped 生命周期](/assets/imgs/DI-Scoped-Lifecycle.jpg)

> **核心逻辑**：同一个“气泡”（Scope）内，无论你通过多少层依赖调用，拿到的都是同一个服务实例；不同Scope之间完全隔离。

## 实战演示：在单例后台服务中使用 Scoped

假设我们有一个单例的后台服务 `Worker`，它需要每隔 3 秒处理一次业务，且每次处理都要确保 `DbContext` 或 `Service` 是全新的且在本次处理中共享。

### 1. 注册服务

我们将 `IOrderProcessor` 注册为 Scoped。

```csharp
builder.Services.AddScoped<IOrderProcessor, OrderProcessor>();
builder.Services.AddHostedService<Worker>();

```

### 2. 通过 IServiceScopeFactory 手动划定边界

由于 `Worker` 是单例，我们不能直接注入 `IOrderProcessor`，而是注入它的“工厂”。

```csharp
public class Worker : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;

    public Worker(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            // 关键：手动创建一个作用域
            using (IServiceScope scope = _scopeFactory.CreateScope())
            {
                // 从当前Scope中解析服务
                var processor = scope.ServiceProvider.GetRequiredService<IOrderProcessor>();
                await processor.ProcessAsync();
                
            } // 离开 using 块，scope 被销毁，processor 及其占用的资源（如数据库连接）被释放

            await Task.Delay(3000, stoppingToken);
        }
    }
}

```

## 深度思考：跨线程的“生命周期车祸”

这是一个非常经典的问题：**如果我在一个 Scoped 作用域内开启了一个新的线程（`Task.Run`），并在新线程里访问原作用域的服务，会发生什么？**

### 现象：ObjectDisposedException（对象已释放）

这是最常见的后果。由于 `Task.Run` 是异步执行的，主线程通常不会等待它完成就会继续向下走。

* **主线程**：执行完 `using` 块，触发 `scope.Dispose()`。
* **DI 容器**：收到销毁信号，立即释放该作用域内所有实现了 `IDisposable` 的服务（如 `DbContext`、`HttpClient`）。
* **新线程（Task.Run）**：此时可能刚开始运行，尝试访问刚才的服务。
* **结果**：访问到一个已经被销毁的对象，直接抛出 `ObjectDisposedException`。

### 潜在隐患：闭包与生存期错位

如果你注入的是一个没有实现 `IDisposable` 的普通 Scoped 服务（比如一个简单的 `UserSession` 类），虽然可能不会立刻报错，但会产生**逻辑错误**：

1. **线程不安全**：Scoped 服务通常设计为单线程访问（在一次请求内）。多个线程同时操作同一个 Scoped 实例，可能导致内部状态错乱。
2. **根级容器逃逸**：如果主作用域已经释放，而你的子线程还在持有该实例，这个对象就成了一个“孤儿”，它可能持有着本该被回收的资源，导致内存占用持续上升。

### 架构建议：如何正确处理异步任务？

如果你确实需要在后台线程执行任务，且需要用到 Scoped 服务，绝对不要直接透传原有的 Scope。

* **方案 A：在子线程中创建独立的作用域（推荐）**

这是最安全、最符合解耦原则的做法。给每个后台任务一个属于它自己的“小天地”。

```csharp
public async Task HandleRequest(IServiceScopeFactory scopeFactory)
{
    // 启动后台任务
    _ = Task.Run(async () => 
    {
        // 关键：在子线程内部开启全新的 Scope
        using var scope = scopeFactory.CreateAsyncScope();
        var scopedService = scope.ServiceProvider.GetRequiredService<IMyScopedService>();
        
        await scopedService.DoWorkAsync();
    });
}
```

* **方案 B：数据透传**

如果后台任务只需要原 Scoped 服务中的某些数据（例如当前用户的 ID），那么你应该在开启 Task.Run 之前，把数据提取出来，以参数的形式传进去。

```csharp
var userId = _currentUserService.UserId; // 先提取值类型或简单 DTO

Task.Run(() => 
{
    // 此时后台任务不再依赖外部的 Scoped 服务实例
    ProcessDataForUser(userId); 
});
```

## 总结与避坑指南

理解 Scoped 不应局限于 Web 请求。它的核心价值在于**“在一段逻辑路径中共享状态，且能被自动回收”**。

* **避坑 1**：绝对不要在 Singleton 服务中通过构造函数注入 Scoped 服务。
* **避坑 2**：在异步编程中，时刻警惕“父作用域”提前消亡导致子线程崩溃。
* **避坑 3**：务必使用 `using`，否则你的 Scoped 对象会一直常驻内存直到进程结束（内存泄漏）。

**参考资料：**

* [.NET 依赖注入详解 - Microsoft.com](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection%3Fwt.mc_id=MVP_324329)