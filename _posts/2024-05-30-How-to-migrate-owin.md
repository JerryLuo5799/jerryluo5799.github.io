---
layout: post
title:  ".NET Framework升级.NET8之五: 如何处理OWIN"
date:   2024-05-30 21:42:55 +0800--
categories: [.NET]
tags: [NET Framework升级.NET8]  
---

### 前言

OWIN 是 [Open Web Interface for .NET](http://owin.org/)，旨在为 ASP.NET 提供 .NET Web 服务器和 Web 应用程序之间的标准接口。它提供了将中间件组合成管道以及注册模块的能力。

[Katana 库](https://github.com/aspnet/AspNetKatana/?wt.mc_id=MVP_324329) 为基于 OWIN 的 Web 应用程序提供了一套灵活的流行组件。这些组件通过以 `Microsoft.Owin` 开头的包提供。

中间件和模块注册功能现在是 ASP.NET Core 的核心功能。微软提供了 [ASP.NET OWIN 接口](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/owin?view=aspnetcore-8.0&wt.mc_id=MVP_324329) 的适配器，可用于逐步迁移自定义 OWIN 组件。相比之下，ASP.NET Core 为 Katana 组件提供了原生端口。

### CORS 功能

在 OWIN 中，使用 [UseCors(...)](https://learn.microsoft.com/en-us/previous-versions/aspnet/mt181143(v=vs.113)?wt.mc_id=MVP_324329) 扩展方法启用 CORS 功能。对于 ASP.NET Core，它由 [`Microsoft.AspNet.Cors`](https://www.nuget.org/packages/Microsoft.AspNet.Cors) 包中的 [UseCors(...)](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.builder.corsmiddlewareextensions.usecors?view=aspnetcore-8.0#microsoft-aspnetcore-builder-corsmiddlewareextensions-usecors(microsoft-aspnetcore-builder-iapplicationbuilder)) 扩展方法提供。

```cs
public void Configuration(Owin.IAppBuilder app) {
    // ... 其他逻辑 ...
    app.UseCors(getCorsOption());
    // ... 其他逻辑 ...
}

private static CorsOptions BuildCorsOptions() {
    var corsPolicy = new CorsPolicy
    {
        AllowAnyMethod = true,
        AllowAnyHeader = true,
        SupportsCredentials = true
    };
    corsPolicy.Origins.Add("https://staging.northwind.com");
    return new CorsOptions 
    {
        PolicyProvider = new CorsPolicyProvider
        {
            PolicyResolver = context => Task.FromResult(corsPolicy)
        }
    };
}
```

```cs
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.UseCors(corsPolicyBuilder => {
    corsPolicyBuilder.AllowAnyMethod()
                     .AllowAnyHeader()
                     .AllowCredentials()
                     .WithOrigins(new string[] { "https://staging.northwind.com" });
})
```

### 第三方身份验证

OWIN 的一个常见用途是提供对 [第三方身份验证源](/choosing-authentication/) 的访问。[AspNet.Security.OAuth.Providers](https://github.com/aspnet-contrib/AspNet.Security.OAuth.Providers?wt.mc_id=MVP_324329) 是一组安全中间件，原生支持 ASP.NET Core，用于支持 GitHub 和 Azure DevOps 等身份验证源。