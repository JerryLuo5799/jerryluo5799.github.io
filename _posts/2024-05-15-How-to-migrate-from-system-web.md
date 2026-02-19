---
layout: post
title:  ".NET Framework升级.NET8之四: 如何处理System.Web"
date:   2024-05-15 21:42:55 +0800--
categories: [.NET]
tags: [NET Framework升级.NET8]  
---

### 前言

当将 Web 应用从 .NET Framework 升级到 .NET Standard 或 .NET 时，您可能需要处理 System.Web。在升级过程中，System.Web 的引用要么被移除，要么会导致编译时错误。

根据场景和迁移阶段，有几种选择可以解决此问题。

### 1. 用 IHttpContextAccessor 替换（仅限 .NET）

在迁移到 .NET 时，您会发现 `HttpContext.Current` 不再存在。您可以在构造函数中使用 `IHttpContextAccessor` 并通过 `httpContextAccessor.HttpContext` 访问它。

```csharp
public class SomeService
{
    public void DoSomething()
    {
        var httpContext = HttpContext.Current;
        // 其余代码...
    }
}
```

```text
❌坏例子 - 一个 .NET Framework 代码片段，演示在方法中使用 HttpContext.Current。
```

```csharp
public class SomeService
{
    private readonly IHttpContextAccessor _httpContextAccessor;

    public SomeService(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }

    public void DoSomething()
    {
        var httpContext = _httpContextAccessor.HttpContext;
        // 其余代码...
    }
}
```

```text
✅好例子 - 代码片段，显示在 .NET 服务类中用 IHttpContextAccessor 替换 HttpContext.Current。
```

### 2. 抽象功能

您还可以抽象出从 HttpContext 需要的内容，这在您希望在非 Web 环境（如控制台应用）中运行代码或希望保持对 .NET Framework 的依赖时非常有用。

如果您同时针对 .NET 和 .NET Framework，我们需要为 .NET Framework 添加回 System.Web 引用。我们通过更新 csproj 文件来实现这一点。

```xml
<!-- .NET Framework 对 HttpContext 的引用 -->
<ItemGroup Condition="'$(TargetFramework)' == 'net472'">
    <Reference Include="System.Web" />
</ItemGroup>

<!-- .NET 对 IHttpContextAccessor 的引用，如果项目是 WebApp，则已包含 -->
<ItemGroup Condition="'$(TargetFramework)' == 'net8.0'">
    <FrameworkReference Include="Microsoft.AspNetCore.App" />
</ItemGroup>
```

```text
在 .NET Framework 4.7.2 项目中条件包含 "System.Web" 引用。
```

接下来，定义一个接口。在这个例子中，我们只公开当前经过身份验证的用户。我们称之为 IApplicationContext，因为它包含当前请求的上下文，无论是来自 HttpContext 还是其他地方，比如后台作业或控制台应用。

```csharp
public interface IRequestContext
{
    string GetCurrentUsername();
}
```

```text
IRequestContext 接口，用于检索当前用户的用户名。
```

下面，您可以看到多个 `IRequestContext` 的实现示例。您可能需要在相应平台的 Web 应用项目中进行实现。

```csharp
#if NETFRAMEWORK

// 用于 .NET Framework
public sealed class LegacyHttpRequestContext : IRequestContext
{
    public string GetCurrentUsername()
        => HttpContext.Current?.User?.Identity?.Name;
}

#else // 或者 #if NET

// 用于 .NET Core 和 .NET
public sealed class HttpRequestContext : IRequestContext
{
    private readonly IHttpContextAccessor _httpContextAccessor;

    public HttpRequestContext(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }

    public string GetCurrentUsername()
        => _httpContextAccessor.HttpContext.User.Identity?.Name;
}

#endif

// 用于后台作业、控制台应用、MAUI 等。
public sealed class BackgroundJobRequestContext : IRequestContext
{
    private readonly string _username;

    public BackgroundJobRequestContext(string username)
    {
        _username = username;
    }

    public string GetCurrentUsername() => _username;
}
```

```text
针对不同环境（.NET Framework、.NET Core 和非 Web 上下文）的 `IRequestContext` 接口的多种实现。
```

```text
注意：如果上述代码需要位于一个多目标项目中，可以使用 `#if NET472_OR_GREATER` 编译指示符来专门针对 .NET Framework 代码。
```

### 3. 抽象 HttpContext 访问器（仅限 .NET Framework）

对于严重依赖 **`HttpContext`** 且抽象 **`HttpContext`** 不切实际的项目，可以在 .NET Framework 中模仿 **`IHttpContextAccessor`** 的行为。虽然 **`IHttpContextAccessor`** 对于 .NET 应用程序可用，但对于 .NET Framework 应用程序并非如此。

```csharp
namespace Microsoft.AspNetCore.Http
{
    // 确保仅在 .NET Framework 中使用，.NET 已有其自身的实现。
#if NET472_OR_GREATER
    // 与 .NET 的 IHttpContextAccessor 接口相同的接口。
    public interface IHttpContextAccessor
    {
        HttpContext HttpContext { get; }
    }

    // .NET Framework 的 IHttpContextAccessor 接口的简单实现
    public sealed class LegacyHttpContextAccessor : IHttpContextAccessor
    {
        public HttpContext HttpContext => HttpContext.Current;
    }
#endif
}
```

```text
特定于 .NET Framework 的 IHttpContextAccessor 接口实现，仅在目标框架为 .NET 4.7.2 或更高版本时可用。
```

**注意：确保您的 .csproj 配置正确。不要使用**

```xml
<!-- .NET Framework 对 HttpContext 的引用 -->
<ItemGroup Condition="'$(TargetFramework)' == 'net472'">
    <Reference Include="System.Web" />
</ItemGroup>

<!-- .NET 对 IHttpContextAccessor 的引用，如果项目是 WebApp，则已包含 -->
<ItemGroup Condition="'$(TargetFramework)' == 'net8.0'">
    <FrameworkReference Include="Microsoft.AspNetCore.App" />
</ItemGroup>
```

### 4. ⚠️ 最后手段：System.Web 适配器

作为最后的手段，可以使用 **`System.Web`** 适配器。这些适配器允许您在不需要移植代码的情况下访问 **`HttpContext.Current`**。请注意，虽然这看起来像是一个简单的解决方案，但并非所有项目都能在不出现问题的情况下采用此方法。因此，只有在所有其他选项都不可行时才应使用此方法。

有关详细信息，请参阅 [Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/migration/inc/adapters?view=aspnetcore-8.0?wt.mc_id=MVP_324329) 上的官方 Microsoft 文档，了解 [System.Web 适配器](https://learn.microsoft.com/en-us/aspnet/core/migration/inc/adapters?view=aspnetcore-8.0?wt.mc_id=MVP_324329)。

```text
注意：上述策略并非互斥，可以根据您的具体需求和约束条件结合使用。目标是使代码更具适应性，为迁移到 .NET 或 .NET Standard 做好准备。
```