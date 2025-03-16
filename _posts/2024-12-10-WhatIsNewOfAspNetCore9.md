---
layout: post
title:  ".NET9详解系列之四: ASP.NET Core 9.0的新特性 "
date:   2024-11-29 21:42:55 +0800--
categories: [.NET]
tags: [.NET9]  
---

## 前言

伴随着.NET9的发布，ASP.NET Core 也在持续更新，为开发者带来更强大、更高效的开发体验。今天，我们就来深入探讨一下 ASP.NET Core 9.0 的新特性，这些特性将帮助开发者构建更安全、更高效的 Web 应用程序。

## 1. 静态资源交付优化

在现代 Web 应用中，静态资源（如 JavaScript、CSS 文件）的加载速度对用户体验至关重要。ASP.NET Core 9.0 引入了新的静态资源交付优化功能，通过在构建和发布时对静态资源进行处理，减少了资源的大小和传输时间。

### 工作原理

该功能通过以下方式优化静态资源的交付：

1. **构建时压缩**：在开发期间使用 gzip 压缩，在发布时使用 gzip 和 brotli 压缩，以最小化资源大小。
2. **基于内容的 ETags**：为每个资源生成基于其内容的 ETag，确保浏览器只在资源内容更改时才重新下载。
3. **缓存控制**：设置适当的缓存头，以优化浏览器缓存行为。

### 实现步骤

1. 在 `Program.cs` 文件中，使用 `MapStaticAssets` 方法替代传统的 `UseStaticFiles` 方法：

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddRazorPages();

var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Error");
    app.UseHsts();
}

app.UseHttpsRedirection();

app.UseRouting();

app.UseAuthorization();

app.MapStaticAssets(); // 使用新的静态资源交付优化
// app.UseStaticFiles(); // 传统方法

app.MapRazorPages();

app.Run();
```

2. 创建一个简单的 HTML 页面，包含一些静态资源（如 CSS 和 JavaScript 文件），以观察优化效果。

```html
<!DOCTYPE html>
<html>
<head>
    <title>静态资源测试</title>
    <link rel="stylesheet" href="/css/site.css" />
</head>
<body>
    <h1>欢迎使用 ASP.NET Core 9.0</h1>
    <script src="/js/site.js"></script>
</body>
</html>
```

3. 在 `wwwroot` 文件夹下创建 `css` 和 `js` 文件夹，并分别添加 `site.css` 和 `site.js` 文件，添加一些简单的样式和脚本。

4. 运行应用程序，观察网络请求，查看静态资源的压缩情况和缓存控制头的设置。

## 2\. SignalR 的新特性

SignalR 是 ASP.NET Core 中用于构建实时 Web 应用的库。在 9.0 版本中，SignalR 引入了一些新特性，如对多态类型的支持和对原生 AOT 的支持。

### SignalR Hubs 中的多态类型支持

现在，SignalR Hubs 的方法可以接受基类类型参数，从而实现多态场景。这使得在处理不同派生类型的消息时更加灵活。

### 实现步骤

1. 定义基类和派生类：

```csharp
[JsonPolymorphic]
[JsonDerivedType(typeof(JsonPersonExtended), nameof(JsonPersonExtended))]
[JsonDerivedType(typeof(JsonPersonExtended2), nameof(JsonPersonExtended2))]
public class JsonPerson
{
    public string Name { get; set; }
    public Person Child { get; set; }
    public Person Parent { get; set; }
}

public class JsonPersonExtended : JsonPerson
{
    public int Age { get; set; }
}

public class JsonPersonExtended2 : JsonPerson
{
    public string Location { get; set; }
}
```

2. 在 Hub 中定义一个接受基类类型参数的方法：

```csharp
public class MyHub : Hub
{
    public void SendMessage(JsonPerson person)
    {
        if (person is JsonPersonExtended extendedPerson)
        {
            // 处理扩展人员类型
        }
        else if (person is JsonPersonExtended2 extendedPerson2)
        {
            // 处理另一种扩展类型
        }
    }
}
```

3. 在客户端，根据需要发送不同类型的对象：

```javascript
const connection = new signalR.HubConnectionBuilder()
    .withUrl("/myHub")
    .build();

// 发送 JsonPersonExtended 类型的消息
connection.invoke("SendMessage", {
    name: "张三",
    age: 30
});

// 发送 JsonPersonExtended2 类型的消息
connection.invoke("SendMessage", {
    name: "李四",
    location: "北京"
});
```

### SignalR 对原生 AOT 的支持

SignalR 现在支持原生 AOT 编译，这可以提高应用程序的性能，特别是在使用原生 AOT 构建的应用中。

### 实现步骤

1. 安装最新的 .NET 9 SDK。

2. 使用 `webapiaot` 模板创建一个新的解决方案：

```bash
dotnet new webapiaot -o SignalRChatAOTExample
```

3. 替换 `Program.cs` 文件的内容为 SignalR 相关代码，如下所示：

```csharp
using Microsoft.AspNetCore.SignalR;
using System.Text.Json.Serialization;

var builder = WebApplication.CreateSlimBuilder(args);

builder.Services.AddSignalR();
builder.Services.Configure<JsonHubProtocolOptions>(o =>
{
    o.PayloadSerializerOptions.TypeInfoResolverChain.Insert(0, AppJsonSerializerContext.Default);
});

var app = builder.Build();

app.MapHub<ChatHub>("/chatHub");
app.MapGet("/", () => Results.Content("""
<!DOCTYPE html>
<html>
<head>
    <title>SignalR Chat</title>
</head>
<body>
    <input id="userInput" placeholder="Enter your name" />
    <input id="messageInput" placeholder="Type a message" />
    <button onclick="sendMessage()">Send</button>
    <ul id="messages"></ul>

    <script src="https://cdnjs.cloudflare.com/ajax/libs/microsoft-signalr/8.0.7/signalr.min.js"></script>
    <script>
        const connection = new signalR.HubConnectionBuilder()
            .withUrl("/chatHub")
            .build();

        connection.on("ReceiveMessage", (user, message) => {
            const li = document.createElement("li");
            li.textContent = `${user}: ${message}`;
            document.getElementById("messages").appendChild(li);
        });

        async function sendMessage() {
            const user = document.getElementById("userInput").value;
            const message = document.getElementById("messageInput").value;
            await connection.invoke("SendMessage", user, message);
        }

        connection.start().catch(err => console.error(err));
    </script>
</body>
</html>
""", "text/html"));

app.Run();

[JsonSerializable(typeof(string))]
internal partial class AppJsonSerializerContext : JsonSerializerContext { }

public class ChatHub : Hub
{
    public async Task SendMessage(string user, string message)
    {
        await Clients.All.SendAsync("ReceiveMessage", user, message);
    }
}
```

4. 构建并运行应用程序，观察原生 AOT 编译后的 SignalR 应用的性能表现。

## 3\. 身份验证和授权的新特性

在 ASP.NET Core 9.0 中，身份验证和授权方面也有一些新的改进，特别是对 OpenID Connect 的支持得到了增强。

### OpenIdConnectHandler 对 Pushed Authorization Requests (PAR) 的支持

PAR 是一种新的 OAuth 标准，通过将授权参数从前端通道移动到后端通道，提高了 OAuth 和 OIDC 流程的安全性。

### 实现步骤

1. 在 `Program.cs` 文件中配置身份验证服务：

```csharp
builder.Services
    .AddAuthentication(options =>
    {
        options.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
        options.DefaultChallengeScheme = OpenIdConnectDefaults.AuthenticationScheme;
    })
    .AddCookie()
    .AddOpenIdConnect("oidc", oidcOptions =>
    {
        // 其他提供程序特定的配置

        // 如果需要禁用 PAR，可以设置如下
        oidcOptions.PushedAuthorizationBehavior = PushedAuthorizationBehavior.Disable;
    });
```

2. 如果需要确保仅在使用 PAR 时身份验证才成功，可以将 `PushedAuthorizationBehavior` 设置为 `Require`。

3. 处理 PAR 的自定义逻辑，可以通过 `OnPushAuthorization` 事件来实现：

```csharp
oidcOptions.Events = new OpenIdConnectEvents
{
    OnPushAuthorization = async context =>
    {
        // 自定义 PAR 请求的处理逻辑
        await Task.CompletedTask;
    }
};
```

## 4\. Blazor 的新特性

Blazor 在 ASP.NET Core 9.0 中也有一些重要的更新，特别是在渲染模式和性能方面。

### .NET MAUI Blazor Hybrid 和 Web App 解决方案模板

现在有一个新的解决方案模板，可以更轻松地创建共享相同 UI 的 .NET MAUI 本机和 Blazor Web 客户端应用。此模板展示了如何创建最大化代码重用并针对 Android、iOS、Mac、Windows 和 Web 的客户端应用。

### 实现步骤

1. 安装 .NET 9 SDK 和包含模板的 .NET MAUI 工作负载：

```bash
dotnet workload install maui
```

2. 使用以下命令从项目模板创建解决方案：

```bash
dotnet new maui-blazor-web
```

该模板在 Visual Studio 中也可用。

### 在运行时检测渲染位置、交互性和分配的渲染模式

引入了一个新的 API，旨在简化在运行时查询组件状态的过程。此 API 提供了以下功能：

1. **确定组件的当前执行位置**：这对于调试和优化组件性能非常有用。
2. **检查组件是否在交互环境中运行**：这对于具有不同交互环境行为的组件非常有帮助。
3. **检索组件的分配渲染模式**：了解渲染模式有助于优化渲染过程并提高组件的整体性能。

## 5\. Minimal APIs 的新特性

Minimal APIs 在 ASP.NET Core 9.0 中也有一些增强，特别是在错误处理和 OpenAPI 支持方面。

### 添加了 InternalServerError 和 InternalServerError 到 TypedResults

TypedResults 类现在包括工厂方法和类型，用于从端点返回“500 Internal Server Error”响应。以下是一个返回 500 响应的示例：

```csharp
var app = WebApplication.Create();

app.MapGet("/", () => TypedResults.InternalServerError("Something went wrong!"));

app.Run();
```

### 在路由组上调用 ProducesProblem 和 ProducesValidationProblem

`ProducesProblem` 和 `ProducesValidationProblem` 扩展方法已更新，以支持在路由组上使用。这些方法表示路由组中的所有端点都可以返回 `ProblemDetails` 或 `ValidationProblemDetails` 响应，用于 OpenAPI 元数据。

```csharp
var app = WebApplication.Create();

var todos = app.MapGroup("/todos")
    .ProducesProblem();

todos.MapGet("/", () => new Todo(1, "Create sample app", false));
todos.MapPost("/", (Todo todo) => Results.Ok(todo));

app.Run();

record Todo(int Id, string Title, boolean IsCompleted);
```

## 6\. OpenAPI 的新特性

ASP.NET Core 9.0 提供了内置支持，用于生成表示基于控制器或 Minimal APIs 的 OpenAPI 文档。

### 内置支持生成 OpenAPI 文档

以下代码调用：

- `AddOpenApi` 将所需的依赖项注册到应用的 DI 容器中。
- `MapOpenApi` 将所需的 OpenAPI 端点注册到应用的路由中。

```csharp
var builder = WebApplication.CreateBuilder();

builder.Services.AddOpenApi();

var app = builder.Build();

app.MapOpenApi();

app.MapGet("/hello/{name}", (string name) => $"Hello {name}"!);

app.Run();
```

安装 `Microsoft.AspNetCore.OpenApi` 包：

```bash
dotnet add package Microsoft.AspNetCore.OpenApi
```

运行应用并导航到 `openapi/v1.json` 查看生成的 OpenAPI 文档。

## 总结

ASP.NET Core 9.0 的新特性为开发者带来了更多的便利和强大的功能。无论是静态资源的优化交付，还是 SignalR 和身份验证方面的改进，都旨在提高 Web 应用的性能和安全性。作为开发者，及时了解和掌握这些新特性，将有助于我们构建更高效、更安全的现代 Web 应用程序。

更多信息: [ASP.NET Core 9.0](https://learn.microsoft.com/en-us/aspnet/core/release-notes/aspnetcore-9.0?wt.mc_id=MVP_324329)