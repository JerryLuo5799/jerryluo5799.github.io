---
layout: post
title:  "ASP.NET Core应用基于Basic认证实现Swagger需要登录才能访问"
date:   2024-07-03 21:42:55 +0800--
categories: [.NET]
tags: [Swagger]  
---

### 1. 背景

Swagger是一个广泛使用的API文档和交互工具。在开发过程中，我们通常希望Swagger文档能够被公开访问，以便于开发者理解和使用API。然而，在生产环境中，出于安全和隐私的考虑，我们可能需要限制对Swagger文档的访问。Basic认证是一种简单而有效的方法，可以帮助我们实现这一目标。

### 2. 实现步骤

在ASP.NET Core应用中实现Swagger Basic认证的步骤如下：

#### 步骤1：添加Basic认证中间件

在Startup.cs的Configure方法中，添加一个中间件来检查访问Swagger的请求是否包含有效的Basic认证信息。

```csharp
public void Configure(IApplicationBuilder app, IWebHostEnvironment env, ILoggerFactory loggerFactory)
{
    // 其他配置...

    //非开发环境才需要认证后才能看到Swagger文档
    if (!env.IsDevelopment())
    {
        app.Use(async (context, next) =>
        {
            // 检查是否是访问Swagger的请求
            if (context.Request.Path.StartsWithSegments("/swagger"))
            {
                // 获取Authorization头部
                var authHeader = context.Request.Headers["Authorization"].FirstOrDefault();
                if (authHeader != null && authHeader.StartsWith("Basic "))
                {
                    // 解析用户名和密码
                    var encodedUsernamePassword = authHeader.Split(' ', 2, StringSplitOptions.RemoveEmptyEntries)[1]?.Trim();
                    var decodedUsernamePassword = Encoding.UTF8.GetString(Convert.FromBase64String(encodedUsernamePassword));
                    var username = decodedUsernamePassword.Split(':', 2)[0];
                    var password = decodedUsernamePassword.Split(':', 2)[1];
                    
                    //验证用户名和密码
                    bool isValid = ValidateCredentials(username, password);
                    if (isValid)
                    {
                        // 认证成功，继续处理请求
                        await next();
                        return;
                    }
                }
                // 认证失败，返回401未授权状态码
                context.Response.StatusCode = 401;
                context.Response.Headers.Add("WWW-Authenticate", "Basic realm=\"Swagger\"");
                return;
            }
            await next();
        });
    }
    // 其他配置...
}
```

#### 步骤2：确保Swagger已添加和配置

在Startup.cs的ConfigureServices方法中，确保Swagger已经被添加和配置。通常通过调用services.AddSwaggerGen()来完成。

### 3. 注意事项

- 上述代码中的用户名和密码是硬编码的，这仅用于演示目的。在生产环境中，你应该使用更安全的方式来存储和验证凭据，例如使用环境变量或安全的凭据存储。
- Basic认证是一种简单的认证方式，它不适合所有场景，特别是在不使用HTTPS的情况下，因为它会以明文形式发送凭据。在生产环境中，建议使用更安全的认证方式，如OAuth或JWT。

### 4. 总结

通过上述步骤，你可以在ASP.NET Core应用中实现Swagger Basic认证，以保护你的Swagger文档不被未授权用户访问。这不仅提高了应用的安全性，也确保了API的稳定性和可靠性。

### 5. 参考资料

- ASP.NET Core官方文档：[https://docs.microsoft.com/en-us/aspnet/core/security/authentication/basic?wt.mc_id=MVP_324329](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/basic?wt.mc_id=MVP_324329)
- Swagger官方文档：[https://swagger.io/docs/specification/authentication/basic-authentication/](https://swagger.io/docs/specification/authentication/basic-authentication/)
