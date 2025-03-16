---
layout: post
title:  ".NET9详解系列之五: .NET Aspire 9.0的新特性 "
date:   2025-01-03 21:42:55 +0800--
categories: [.NET]
tags: [.NET9]  
---

## 前言

随着.NET 9的发布，.NET Aspire迎来了重大更新，为开发人员带来了诸多便利和强大的功能。本文将深入探讨.NET Aspire 9.0的新特性，并通过示例代码帮助初级开发人员更好地理解和应用这些新特性。

## 一、安装流程简化

在.NET Aspire 9.0中，安装流程得到了极大简化，不再需要执行`dotnet workload install aspire`命令，因为Aspire已经内置于.NET 9中。如果你之前安装过旧版本，建议先执行以下命令卸载：

```bash
dotnet workload uninstall aspire
```

要获取Aspire的项目模板，只需执行以下命令：

```bash
dotnet new install Aspire.ProjectTemplates
```

这将安装所有必要的模板，包括App Host、Aspire demo、Service default starter和测试模板等。

## 二、仪表板增强功能

### 资源生命周期管理

.NET Aspire仪表板现在支持管理资源的生命周期，你可以方便地停止、启动和重启资源。这对于项目、容器和可执行文件都适用。例如，当调试器附加到项目资源时，重启后调试器会重新附加。

### 移动和响应式支持

仪表板现在是移动友好的，能够自适应各种屏幕尺寸，使你可以在移动设备上随时随地管理部署的.NET Aspire应用程序。此外，还进行了一些可访问性改进，例如在移动设备上显示设置和内容溢出。

### 敏感属性、卷和健康检查

在资源详细信息中，可以将属性标记为敏感，自动在仪表板UI中屏蔽它们，避免在屏幕共享时意外泄露密钥或密码。此外，配置的容器卷也会在资源详细信息中列出。.NET Aspire 9还添加了对健康检查的支持，可以在资源详细信息窗格中查看这些检查的详细信息，了解资源为何可能被标记为不健康或降级。

### 彩色控制台日志

现在仪表板的控制台日志页面支持更丰富的ANSI转义码格式，可以同时渲染多种ANSI转义码，例如同时显示红色和粗体文本。此外，还隐藏了与显示文本无关的转义码，使日志更加清晰。

## 三、依赖管理革新

### WaitFor功能

新增的`WaitFor`功能允许服务等待依赖项就绪。例如：

```csharp
var builder = DistributedApplication.CreateBuilder(args);
var cache = builder.AddRedis("cache");
var pgdb = builder.AddPostgres("pgdb");
var apiService = builder.AddProject<Projects.ApiService>("apiservice")
    .WithReference(pgdb)
    .WithReference(cache)
    .WaitFor(pgdb)
    .WaitFor(cache);
builder.Build().Run();
```

在这个示例中，`apiService`将等待`pgdb`和`cache`处于运行状态才会启动，配合自动重试机制，可以可靠地连接数据库。

### 健康检查集成

支持多种健康检查方式，包括HTTP健康检查、自定义状态码支持和可配置健康检查参数等。

## 四、持久化容器

支持容器在Aspire停止后继续运行，通过`WithLifetime`配置容器生命周期。例如：

```csharp
builder.AddRedis("redis")
    .WithLifetime(ServiceLifetime.Persistent);
```

这可以减少重复启动时间，保持数据持久性，简化开发工作流程。

## 五、集成增强

### Redis Insight支持

在Redis资源上支持Redis Insight：

```csharp
var builder = DistributedApplication.CreateBuilder(args);
builder.AddRedis("redis")
    .WithRedisInsight(); // 启动一个Redis Insight容器镜像
```

`WithRedisInsight`扩展方法可以应用于多个Redis资源，它们都会显示在Redis Insight仪表板上。

### OpenAI集成（预览）

从.NET Aspire 9开始，提供了一个额外的OpenAI集成，允许直接使用最新的官方OpenAI dotnet库。客户端集成在服务集合中注册`OpenAIClient`作为单例服务。可以使用`AddOpenAIClientFromConfiguration`方法从配置中添加`OpenAIClient`。

### MongoDB支持

添加了在使用`AddMongoDB`扩展方法时指定MongoDB用户名和密码的支持。如果未指定，将生成随机用户名和密码，但也可以使用参数资源手动指定。

### Azure改进

#### Azure SQL、PostgreSQL和Redis更新

在.NET Aspire 9中，对Azure SQL、PostgreSQL和Redis资源的API模式进行了更新，使其更加合理和易于使用。例如：

```csharp
var builder = DistributedApplication.CreateBuilder(args);
var sql = builder.AddAzureSqlServer("sql")
    .RunAsContainer();
var pgsql = builder.AddAzurePostgresFlexibleServer("pgsql")
    .RunAsContainer();
var cache = builder.AddAzureRedis("cache")
    .RunAsContainer();
```

#### 默认使用Microsoft Entra ID

为了提高安全性，Azure Database for PostgreSQL和Azure Cache for Redis资源更新为默认使用Microsoft Entra ID。需要连接到这些资源的应用程序需要进行相应配置。

#### Azure Functions支持（预览）

.NET Aspire现在支持Azure Functions（预览版），配置了一个模拟的Azure存储资源用于主机的簿记，可以在本地启动函数主机，并通过`azd` CLI部署应用程序到Azure Container Apps。

## 六、网络改进

所有资源默认添加到公共网络，简化了服务间通信。

## 七、可视化与管理工具

支持自动部署Redis Commander、PgAdmin、PhpMyAdmin等工具，集成Redis Insight，提供直观的数据查看界面。

## 八、其他更新

### 版本支持策略

Aspire采用独立的版本周期，新版本发布后，旧版本立即停止支持，这可能影响企业采用决策。

### 资源关系

仪表板现在反映了“父”和“子”资源关系的概念，例如，如果你创建了一个Postgres实例并带有多个数据库，它们现在将在“资源”页面中嵌套在同一实例中。

### 语言覆盖

仪表板默认使用浏览器设置的语言，此版本引入了覆盖此设置的能力，可以独立于浏览器语言更改仪表板语言。

### 过滤功能

现在可以在“资源”页面中通过“资源类型”、“状态”和“健康状态”来过滤你看到的内容。

### 更多资源详细信息

当你在仪表板中选择一个资源时，详细信息窗格中现在提供了更多的数据点，包括“引用”、“反向引用”和“卷”及其挂载类型。

### 自定义本地域的CORS支持

现在可以设置`DOTNET_DASHBOARD_CORS_ALLOWED_ORIGINS`环境变量，以允许仪表板从其他浏览器应用（例如在自定义本地主机域上运行的资源）接收遥测数据。

### 控制台日志的灵活性

控制台日志页面有两个新选项：现在可以下载日志以便在自己的诊断工具中查看，还可以打开或关闭时间戳以减少视觉杂乱。

### 各种UX改进

在.NET Aspire 9.1中添加的几个新功能增强了并简化了热门任务：

- 资源命令（如“开始”和“停止”按钮）现在在“控制台日志”页面上可用。
- 单选选择即可打开文本可视化程序。
- 日志中的URL现在自动可点击，端点中的逗号已删除。
- 此外，切换不同资源时，滚动位置现在会重置。

## 总结

.NET Aspire 9.0的发布为开发人员带来了众多实用的新特性和改进，从简化的安装流程到增强的仪表板功能，再到强大的依赖管理和集成能力，这些更新使得.NET Aspire更加成熟和易于使用。无论是初级开发人员还是有经验的开发者，都可以从中受益，提高开发效率和应用程序的质量。希望本文的介绍和示例能够帮助你更好地理解和应用这些新特性。

更多信息: [.NET Aspire 9.0](https://learn.microsoft.com/en-us/dotnet/aspire/whats-new/dotnet-aspire-9?wt.mc_id=MVP_324329)