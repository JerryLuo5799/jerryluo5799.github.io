---
layout: post
title:  ".NET Framework升级.NET8之六: 如何处理global.asax文件"
date:   2024-05-30 21:42:55 +0800--
categories: [.NET]
tags: [NET Framework升级.NET8]  
---

### 前言

`Global.asax` 是一个可选文件，用于定义 ASP.NET 应用程序如何处理应用程序、会话和请求事件。这些事件的处理代码通常写在 `Global.asax.cs` 文件中。在将应用程序迁移到 ASP.NET Core 时，需要对这些代码进行重构。

### 应用程序事件

以下方法在运行时会自动链接到 `HttpApplication` 类的事件处理程序。

`Application_Start()` 或 `Application_OnStart()` 方法在服务器接收到第一个请求时被调用，通常用于初始化静态值。此起始方法的逻辑应包含在 ASP.NET Core 项目 `Program.cs` 文件的开头。

`Application_Init()` 方法在添加所有事件处理程序模块后被调用。可以通过将逻辑注册到 `WebApplication.Lifetime` 属性的 `ApplicationStarted` 来迁移此逻辑。

`Application_End()` 和 `Application_Disposed()` 方法在应用程序终止时被触发。可以通过将逻辑注册到 `WebApplication.Lifetime` 属性的 `ApplicationStopping` 和 `ApplicationStopped` 来迁移这些方法。

因此，以下 `Global.asax.cs` 代码片段将按照以下图示进行迁移。

```cs
public class MvcApplication : HttpApplication
{
    protected void Application_Start() {
        Console.WriteLine("Start");
    }

    protected void Application_Init() {
        Console.WriteLine("Init");
    }

    protected void Application_Stopping() {
        Console.WriteLine("Stopping");
    }

    protected void Application_Stopped() {
        Console.WriteLine("Stopped");
    }
}
```

```cs
Console.WriteLine("Start");

var builder = WebApplication.CreateBuilder(args);
// ...
var app = builder.Build();
app.Lifetime.ApplicationStarted.Register(() => Console.WriteLine("Init"));
app.Lifetime.ApplicationStopping.Register(() => Console.WriteLine("Stopping"));
app.Lifetime.ApplicationStopped.Register(() => Console.WriteLine("Stopped"));
```

### 会话事件

当检测到新用户会话时，将调用 `Session_Start()` 方法。可以使用中间件来确定是否已预先设置会话变量，以替换 `Session_Start()` 方法。有关替换 `Session_Start()` 的其他方法，请参阅 [此 StackOverflow 线程](https://stackoverflow.com/questions/52533831/is-there-a-session-start-equivalent-in-net-core-mvc-2-1)。

当用户会话因超时而结束时，将调用 `Session_End()` 方法。ASP.NET Core 中没有等效的功能，因此需要重构任何会话管理逻辑以解决此问题。

### 请求生命周期方法

在请求期间引发的事件在 [HttpApplication API 文档](https://learn.microsoft.com/en-us/dotnet/api/system.web.httpapplication?view=netframework-4.8.1#remarks) 中有所记录。应在请求前后执行的逻辑应使用 [中间件](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/middleware/?view=aspnetcore-8.0?wt.mc_id=MVP_324329) 实现。

```cs
public class MvcApplication : HttpApplication
{
    protected void Application_BeginRequest() {
        Console.WriteLine("Begin request");
    }

    protected void Application_EndRequest() {
        Console.WriteLine("End request");
    }
}
```

```cs
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.Use(async (context, next) =>
{
    Console.WriteLine("Begin request");
    await next.Invoke();
    Console.WriteLine("End request");
})
```

### 错误处理

`Application_Error()` 方法中的全局错误处理逻辑应迁移到使用 `UseExceptionHandler()` 方法注册的中间件。

```cs
public class MvcApplication : HttpApplication
{
    protected void Application_Error(object sender, EventArgs e) {
        var error = Server.GetLastError();
        Console.WriteLine("Error was: " + error.ToString());
    }
}
```

```cs
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.UseExceptionHandler(exceptionHandlerApp =>
{
    exceptionHandlerApp.Run(async context =>
    {
        var handlerPathFeature = context.Features.Get<IExceptionHandlerPathFeature>();
        var error = handlerPathFeature.Error;
        Console.WriteLine("Error was: " + error.ToString());

        // 注意：context.Response 允许你设置返回的状态码
        // 和响应内容。
    })
});
```

有关在 ASP.NET Core 中处理错误的更多选项，请参阅 [此处](https://learn.microsoft.com/en-us/aspnet/core/web-api/handle-errors?view=aspnetcore-8.0&wt.mc_id=MVP_324329)