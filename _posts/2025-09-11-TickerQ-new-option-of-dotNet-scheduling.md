---
layout: post
title:  "TickerQ - .NET任务调度新选择"
date:   2025-09-11 12:30:00 +0800--
categories: [.NET]
tags: [TickerQ]
---

### 前言

在 .NET 生态中，提到后台任务和定时调度，开发者第一时间想到的往往是 **Quartz.NET**（配置繁琐）或者 **Hangfire**（免费版功能受限且依赖反射）。但现在，一个名为 **TickerQ** 的新开源库正在异军突起。它不仅解决了老牌框架的痛点，还带来了极速的性能和原生的开发者体验。

### 核心概念：为什么 TickerQ 值得关注？

TickerQ 是一个现代化的 .NET 调度库，它从底层架构上就做了很多优化：

* **零反射 (Zero Reflection)**：利用 C# 的 **Source Generators（源码生成）** 技术，在编译期就完成了任务发现。这意味着更快的启动速度和更低的内存开销。
* **原生异步 (Truly Async)**：从头到尾的异步支持，不再有阻塞线程的困扰。
* **内置仪表盘 (Dashboard)**：无需复杂配置，即可拥有一个精美的可视化管理界面。
* **强力持久化**：深度集成 **Entity Framework Core**，任务状态、重试记录、参数数据都能轻松持久化到数据库（如 PostgreSQL）。

### 实战演示：从零构建调度系统

#### 1. 安装必要的 NuGet 包

我们需要安装以下三个核心组件：

* `TickerQ`
* `TickerQ.Dashboard`
* `TickerQ.EntityFrameworkCore`

#### 2. 配置数据库与服务注册

首先，在 `DbContext` 中应用 TickerQ 的表结构，并在 `Program.cs` 中注册。

```csharp
// 1. 配置 DbContext
public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
        // 关键：自动应用 TickerQ 所需的表结构配置
        modelBuilder.ApplyTickerQConfigurations();
    }
}

// 2. 在 Program.cs 中注册服务
builder.Services.AddTickerQ(options =>
{
    // 配置持久化存储
    options.AddOperationalStore<AppDbContext>(storeOptions =>
    {
        storeOptions.UsePostgres(); 
    });
});

// 添加带有基础认证的仪表盘
builder.Services.AddTickerQDashboard(options => 
{
    options.UseBasicAuth("admin", "admin123");
});

var app = builder.Build();

app.UseTickerQ();
app.UseTickerQDashboard();

```

#### 3. 定义 Job 任务

TickerQ 支持 **Time Ticker**（定时/延时）和 **Cron Ticker**（周期性）。

```csharp
public class MyJobs(ILogger<MyJobs> logger)
{
    // 周期性任务：每分钟执行一次
    [TickerFunction("CleanUpLogs")]
    [CronTicker("0 * * * * *")] 
    public Task CleanLogsAsync(CancellationToken ct)
    {
        logger.LogInformation("[{Time}] 正在清理系统日志...", DateTime.Now);
        return Task.CompletedTask;
    }

    // 带参数的动态任务
    [TickerFunction("ProcessOrder")]
    public Task ProcessOrderAsync(TickerFunctionContext<OrderInfo> context)
    {
        var order = context.Request;
        logger.LogInformation("正在处理订单：{OrderId}", order.Id);
        return Task.CompletedTask;
    }
}

public record OrderInfo(string Id);

```

#### 4. 编程式动态调度

你可以在业务逻辑中随时触发一个未来的任务：

```csharp
app.MapPost("/schedule-order", async (ITimeTickerManager manager, string orderId) =>
{
    // 调度一个 30 秒后执行的单次任务
    await manager.CreateAsync(new TimeTickerRequest
    {
        FunctionName = "ProcessOrder",
        Request = new OrderInfo(orderId),
        ExecutionTime = DateTimeOffset.Now.AddSeconds(30),
        RetryCount = 3,
        RetryIntervals = [TimeSpan.FromSeconds(10), TimeSpan.FromMinutes(1)] 
    });

    return Results.Ok("订单处理任务已挂载");
});

```
https://www.youtube.com/watch?v=DpyjAKmNwpI
### 技术内功：分布式集群下的“防重”机制

在集群部署（如 Kubernetes 或多实例物理机）场景下，多个服务实例会同时连接同一个数据库。TickerQ 采用了一种**基于数据库状态控制的分布式锁机制**，确保同一个任务在同一时间内只会被一个实例获取并执行。

1. **原子抢占**：每个实例在扫描待执行任务时，会尝试通过数据库的原子操作（如 `UPDATE ... WHERE Status = Pending`）来“标记”任务。
2. **实例绑定**：一旦某个实例成功更新了任务状态，它会将自己的 `InstanceId` 和当前执行的 `Heartbeat`（心跳时间）写入数据库，锁住该任务。
3. **心跳续约**：执行任务的实例会定期更新数据库中的心跳。如果实例意外宕机，心跳停止。
4. **容错接管**：其他健康实例发现某个任务处于“执行中”但“心跳已过期”时，会自动回收该任务并重新调度，保证了系统的高可用。

### 总结

TickerQ 凭借其**编译期安全**、**原生异步**以及**稳健的分布式锁**，成为了 .NET 后台任务领域的一匹黑马。它既有 Hangfire 的易用性，又在性能上更贴合现代云原生应用的需求。

1. **重试间隔策略**：TickerQ 允许自定义 `RetryIntervals`。对于第三方接口调用，**强烈建议使用递增的重试间隔**（如 10s -> 1min -> 5min），避免在对方服务宕机时频繁冲击导致其无法恢复。
2. **数据库迁移注意点**：由于 TickerQ 会创建独立的 Schema（默认为 `ticker`），在进行 EF Core 迁移时，请确保数据库账号拥有创建 Schema 的权限。
3. **零反射的红利**：在高并发场景下，TickerQ 几乎不产生额外的 GC 压力。如果你正在开发微服务且对容器启动时间有严格要求，TickerQ 的 Source Generator 特性会让你受益匪浅。

### 参考资料

* [TickerQ GitHub 源码仓库](https://github.com/Arcenox-co/TickerQ?wt.mc_id=MVP_324329)
