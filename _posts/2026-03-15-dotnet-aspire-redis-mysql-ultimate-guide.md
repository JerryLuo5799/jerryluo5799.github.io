---
layout: post
title:  ".NET Aspire实战指南"
date:   2026-03-15 09:30:00 +0800--
categories: [.NET]
tags: [.NET Aspire]
---

## 1. 前言

你是否经历过这种痛苦：为了在本地跑起一个简单的微服务，你需要手动启动 Redis 容器、配置复杂的 `appsettings.json` 连接字符串、担心服务 A 找不到服务 B，最后还要对着一堆乱码日志抓耳挠腮？

“在我的电脑上能跑通”已经成了程序员最大的谎言。为了解决分布式系统的开发痛点，微软推出了 **.NET Aspire**。它不是一个新框架，而是一个**云原生开发栈**，旨在让你像写单体工程一样轻松地开发、调试和部署分布式应用。

## 2. 核心概念：什么是 .NET Aspire？

.NET Aspire 并不是取消了环境配置，而是把“环境”变成了“代码”。它主要由三个部分组成：

1.  **AppHost (项目大脑)**：一个特殊的 .NET 项目，用来定义应用需要哪些资源（如 Redis、MySQL、API 服务）。
2.  **Components (组件)**：经过优化的 NuGet 包，能自动处理重试机制、健康检查和可观测性。
3.  **Dashboard (可观测性中心)**：启动项目时自动生成的网页，能实时看到所有服务的日志、链路追踪（Tracing）和性能指标。

![.NET Aspire Architecture](/assets/imgs/dotnet_aspire.png)

## 3. 实战演示：构建 Redis + MySQL 双数据库应用

我们将创建一个名为 `ShopApp` 的系统：一个 **Minimal API** 同时使用 **MySQL** 存储数据，并使用 **Redis** 缓存热点数据。

### 第一步：准备运行环境

Aspire 的运行依赖非常明确，你的电脑必须具备：
* **.NET SDK (8.0/9.0+)**。
* **Aspire Workload**：执行 `dotnet workload install aspire`。
* **Docker Desktop 或 Podman**：Aspire 需要它来拉起本地的 Redis 和 MySQL 容器。

### 第二步：在 AppHost 中定义资源
打开 `ShopApp.AppHost` 项目的 `Program.cs`，通过 C# 代码描述你的基础设施：

```csharp
var builder = DistributedApplication.CreateBuilder(args);

// 1. 定义 Redis 资源
var cache = builder.AddRedis("cache");

// 2. 定义 MySQL 资源并创建数据库
var mysql = builder.AddMySql("mysql-server")
                   .AddDatabase("shopdb");

// 3. 将资源“转交给” API 项目
builder.AddProject<Projects.ShopApp_ApiService>("api-service")
       .WithReference(cache)  // 自动处理服务发现和连接注入
       .WithReference(mysql); // 自动注入 MySQL 连接信息

builder.Build().Run();
````

### 第三步：在业务项目中接入组件

在 API 项目中，安装 `Aspire.StackExchange.Redis` 和 `Aspire.Pomelo.EntityFrameworkCore.MySql` 包，并在 `Program.cs` 中注册：

```csharp
var builder = WebApplication.CreateBuilder(args);

// 【重要】注册组件，名称必须与 AppHost 中的定义一致
builder.AddRedisClient("cache");
builder.AddMySqlDbContext<AppDbContext>("shopdb");

// 注入 Aspire 默认的健康检查与日志配置（包含 OpenTelemetry）
builder.AddServiceDefaults(); 

var app = builder.Build();

// 即使在生产环境，健康检查接口（/health）也是必须的
app.MapDefaultEndpoints(); 

app.MapGet("/orders/{id}", async (int id, AppDbContext db, IConnectionMultiplexer redis) => 
{
    var cacheDb = redis.GetDatabase();
    
    // 逻辑：先查缓存
    var cached = await cacheDb.StringGetAsync($"order:{id}");
    if (cached.HasValue) return Results.Ok(cached.ToString());

    // 缓存没有，查数据库
    var order = await db.Orders.FindAsync(id);
    if (order is not null) 
    {
        await cacheDb.StringSetAsync($"order:{id}", order.Name, TimeSpan.FromMinutes(10));
        return Results.Ok(order);
    }

    return Results.NotFound();
});

app.Run();
```

## 4. 揭秘：当你按下 F5 时，Aspire 在后台做了什么？

这是最关键的环节。当你启动 `AppHost` 项目时，它会瞬间完成以下四件事，把“魔法”变成现实：

1.  **资源调度 (Provisioning)**：自动检查 Docker 环境，执行 `docker pull` 并启动对应的 Redis 和 MySQL 镜像。
2.  **动态端口映射 (Dynamic Port Mapping)**：为了防止端口冲突，Aspire 会分配随机的高位端口（例如 MySQL 容器内的 3306 可能会被映射到宿主机的 54321）。
3.  **环境变量注入 (Environment Injection)**：在启动 `ApiService` 进程时，自动注入环境变量：`ConnectionStrings__shopdb="server=localhost;port=54321;..."`。这使得业务代码能自动找到数据库。
4.  **仪表盘启动 (Dashboard)**：启动一个网页服务器，通过 gRPC 实时抓取所有服务的日志和链路追踪数据。

## 5. 生产环境部署：从“魔法”回归“现实”

发布到正式环境时，情况会有所不同。`AppHost` 项目不再参与运行，我们需要根据场景选择方案。

### 方案 A：云原生一键发布（Azure / Kubernetes）

这是最省心的方式。通过 **Azure Developer CLI (azd)**，你可以运行 `azd up`。

  * **Manifest 清单**：Aspire 会生成一个描述资源的 JSON 文件。
  * **自动转换**：云平台会根据清单，自动为你创建**全托管**的云数据库（如 Azure Cache for Redis），而不是跑 Docker 容器，安全性更高。

### 方案 B：传统手工发布（VPS / 物理机）

如果你只是想把编译好的 DLL 拷贝到自己的服务器上：

1.  **物理隔离**：**只发布业务项目**（`ShopApp.ApiService`）。绝对不要发布 `AppHost` 项目，它只属于开发机。
2.  **手动补全配置**：由于没有了自动注入，你需要在服务器的环境变量或 `appsettings.Production.json` 中提供配置。
    ```bash
    # 在服务器设置环境变量，补丁 AppHost 留下的空缺
    export ConnectionStrings__cache="192.168.1.10:6379,password=***"
    ```

## 6. 深度思考：那个“对象”到底是怎么来的？

作为初学者，一定要记住这个优先级公式。当你调用 `AddRedisClient("cache")` 时，它是按顺序找的：
**环境变量 \> 配置文件 (appsettings.json) \> 默认值。**

通过这个机制，我们可以实现在开发环境用 Docker 魔法，而在生产环境通过环境变量接入真实的物理数据库，且**业务代码一行都不用改**。

## 7. 总结

.NET Aspire 标志着 .NET 进入了“环境即代码”的时代。它消除了繁琐的 YAML 和配置，让我们可以把精力集中在业务逻辑上。
  * **AppHost** 是你的架构师，负责在开发阶段“搬运资源”和“连线”。
  * **运行时**：通过动态端口和环境变量注入，消除了硬编码连接字符串。
  * **一致性**：无论本地、云端还是手动发布，组件包都会自动适配环境。

**官方资源引用**

  * [.NET Aspire 官方文档首页](https://learn.microsoft.com/dotnet/aspire/get-started/aspire-overview?wt.mc_id=MVP_324329)
  * [了解 .NET Aspire 运行时原理](https://learn.microsoft.com/dotnet/aspire/fundamentals/app-host-overview?wt.mc_id=MVP_324329)
  * [.NET 配置系统优先级详解](https://learn.microsoft.com/dotnet/core/extensions/configuration?wt.mc_id=MVP_324329)
