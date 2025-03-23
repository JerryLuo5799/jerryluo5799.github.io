---
layout: post
title:  ".NET Framework升级.NET8之四: 如何处理Web.config"
date:   2024-05-25 21:42:55 +0800--
categories: [.NET]
tags: [NET Framework升级.NET8]  
---

### 前言

[`Web.Config`](https://learn.microsoft.com/en-us/troubleshoot/developer/webapps/aspnet/health-diagnostic-performance/create-web-config?wt.mc_id=MVP_324329) 文件在 ASP.NET 中用于控制各个 ASP.NET 应用程序的行为并配置 IIS。默认情况下，现代 ASP.NET Core 应用程序使用 Kestrel Web 服务器，该服务器在代码中配置。除非您使用 IIS 部署应用程序，否则需要将 `Web.Config` 文件迁移到 ASP.NET Core。

`Web.Config` 文件包含有关包包含、模块包含和配置值的数据。

### 1. 包包含

在 ASP.NET Core 中，项目包含在项目的 CSPROJ 文件中列出。需要检查应用程序的依赖项是否仍然需要，并使用 NuGet 包管理器按需添加。

### 2. 服务器配置

服务器的配置需要转移到 `Program.cs` 文件中的代码里。

#### 2.1 自定义错误页面

`<system.web>` 中的 `<customErrors>` 元素指定了服务器在生成具有 HTTP 错误代码的响应时要使用的重定向。如何是 'RemoteOnly'，这意味着重定向仅在从单独的主机访问时使用。`<customErrors>` 元素将提供默认重定向，并且可以包含 `<error>` 元素，为特定的错误代码提供更具体的重定向。

最简单的转换此配置的方法是使用 [`UseStatusCodePagesWithRedirects`](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/error-handling?view=aspnetcore-7.0#usestatuscodepageswithredirects?wt.mc_id=MVP_324329)。

```xml
<customErrors mode="RemoteOnly" defaultRedirect="~/Error">
    <error statusCode="403" redirect="~/Error?code=403" />
    <error statusCode="404" redirect="~/Error?code=404" />
</customErrors>
```

**自定义错误重定向的典型示例**

```cs
var app = builder.Build();
app.UseStatusCodePagesWithRedirects("/Error?code={0}");
```

**迁移到 ASP.NET Core 的配置**

#### 2.2 命名空间

`<pages>/<namespaces>` 元素定义了在程序集预编译期间使用的导入指令。可以使用现代 C# 的[隐式导入功能](https://learn.microsoft.com/en-us/dotnet/core/project-sdk/overview#implicit-using-directives?wt.mc_id=MVP_324329)实现相同的效果。

### 3. HTTP 处理程序路由

`<httpHandlers>` 元素将路由链接到 [`IHttpHandler`](https://learn.microsoft.com/en-us/dotnet/api/system.web.ihttphandler?view=netframework-4.8.1?wt.mc_id=MVP_324329) 实现。有关替换选项，请参阅 [ASP.NET Core 基础知识文章关于路由](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/routing?view=aspnetcore-8.0?wt.mc_id=MVP_324329)，包括使用 [`MapGet`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.builder.endpointroutebuilderextensions.mapget?view=aspnetcore-8.0#microsoft-aspnetcore-builder-endpointroutebuilderextensions-mapget(microsoft-aspnetcore-routing-iendpointroutebuilder-system-string-system-delegate)) 和 [`MapPost`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.builder.endpointroutebuilderextensions.mappost?view=aspnetcore-8.0#microsoft-aspnetcore-builder-endpointroutebuilderextensions-mappost(microsoft-aspnetcore-routing-iendpointroutebuilder-system-string-system-delegate))。

#### 3.1 HTTP 模块

`<httpModules>` 元素配置了注册到 [`HttpApplication`](https://learn.microsoft.com/en-us/dotnet/api/system.web.httpapplication?view=netframework-4.8.1?wt.mc_id=MVP_324329) 的模块。有关各个模块的现代等效项如何与 ASP.NET Core 应用程序一起使用，请参阅各个模块的文档。

### 4. 自定义配置值

应用程序的自定义配置值存储在 `<appSettings>` 元素中。配置值的迁移位置取决于它们是否应该是机密。

对于非机密值，可以将它们迁移到 `appsettings.json` 文件中。

```xml
<appSettings>
    <add key="DefaultVisibility" value="public" />
    <add key="DefaultClientCount" value="30" />
</appSettings>
```

**Web.config 中应用程序设置的典型示例**

```json
{
  "DefaultVisibility": "public",
  "DefaultClientCount": 30
}
```

**迁移到 `appsettings.json` 的应用程序设置示例**

如果程序正在使用 [System.Configuration.ConfigurationManager](https://learn.microsoft.com/en-us/dotnet/api/system.configuration.configurationmanager?wt.mc_id=MVP_324329) 访问配置值，则需要更改该类的使用方式，因为该类在 ASP.NET Core 下不可用。相反，应使用来自 `Microsoft.Extensions.Configuration` 包的依赖注入 [`IConfiguration`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.configuration.iconfiguration?view=dotnet-plat-ext-7.0?wt.mc_id=MVP_324329) 实现。

```cs
String visibility = ConfigurationManager.AppSettings["DefaultVisibility"];
int clientCountStr = int.Parse(ConfigurationManager.AppSettings["DefaultClientCount"]);
// 使用配置值执行操作。
```

**使用 ConfigurationManager 获取设置的典型示例**

```cs
public class TestService
{
    private readonly IConfiguration Configuration;

    public TestService(IConfiguration configuration)
    {
        Configuration = configuration;
    }

    public void Act() {
        var visibility = Configuration.GetValue<string>("DefaultVisibility");
        var clientCountStr = Configuration.GetValue<int>("DefaultClientCount");
        // 使用配置值执行操作。
    }
}
```

**迁移到 ASP.NET Core 的示例代码**

### 5. 连接字符串

连接字符串存储在 `<connectionStrings>` 元素中，只要不包含任何机密，就可以直接迁移到 `appsettings.json` 文件中。

```xml
<connectionStrings>
    <add name="DefaultConnection"
         providerName="System.Data.SqlClient"
         connectionString="Server=localhost,1200" />
</connectionStrings>
```

**Web.config 中典型的连接字符串示例**

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost,1200"
  }
}
```

**迁移到 ASP.NET Core 的连接字符串示例**

如上所述，`ConfigurationManager` 类不再可用，其用法需要替换为使用 `IConfiguration` 的调用。

```cs
var connStr = ConfigurationManager.ConnectionsStrings["DefaultConnection"]
                                  .ConnectionString;
```

**从 Web.config 获取连接字符串的典型示例**

```cs
var build = WebApplication.CreateBuilder(args);
var app = builder.Build();
var connStr = app.Configuration.GetConnectionString("DefaultConnection");
```

**迁移到在 `Program.cs` 中获取连接字符串的示例**
