---
layout: post
title:  ".NET Framework升级.NET8之三: 将Web应用迁移到.NET8"
date:   2024-05-05 21:42:55 +0800--
categories: [.NET]
tags: [NET Framework升级.NET8]  
---

### 1. 前言

ASP.NET Web应用与ASP.NET CoreWeb应用之间存在巨大差异。整个请求管道经历了重大变化，有时甚至无法进行就地迁移。那么，如何才能正确应对这些挑战呢？

### 2. 迁移模式: 大爆炸(big bang) vs 扼杀者(Strangler Fig)

- 大爆炸（Big Bang）模式, 是说我们必须一次性将所有代码迁移到.NET8, 然后将它们都部署到生产环境中。这是很危险的, 因为这意味着我们需要漫长的开发时间，同时，你必须冻结新功能的开发工作。对于大多数公司来说，冻结应用程序开发工作可能存在风险，因为他们必须根据业务环境的变化随时调整软件。
- 扼杀者(Strangler Fig) 模式允许在旧系统上进行持续开发，采用增量方法用新服务替换特定功能, 最终完成整个系统的迁移，在这个过程中，我们也可以根据需要进行新功能的开发。

两个模式之前存在一条分界线，这条分界线是处理Web应用迁移的推荐方式。

- 你估计迁移工作需要多少个冲刺？
- 在迁移过程中，功能开发是否会继续进行？
- 你在上述两个方面都有很大的灵活性吗？

如果你的[迁移计划](/_posts/2024-04-12-PreparetoUpgradeToNET8.md)很扎实，你应该对迁移Web应用所需的工作量有一个相当清晰的认识。如果你有信心可以在合理的时间内完成迁移，并且可以在那段时间内实施功能冻结，那么选择大爆炸(big bang)模式可能是一个合理的选择。

然而，如果你知道迁移将需要很长时间，或者有其他开发人员/团队将从事其他非迁移工作（例如，功能开发），那么采用带有YARP的扼杀者(Strangler Fig)模式通常是一个更好的选择。

### 3. 创建并行Web应用项目

第一步是创建一个新的ASP.NET Core Web应用项目，你将逐步将页面/端点迁移到其中。完成此操作的最佳方法是通过[.NET升级助手](https://dotnet.microsoft.com/en-us/platform/upgrade-assistant)。这将创建一个新的.NET 8项目，并包含YARP。对于尚未迁移的功能，YARP将把它们重定向到.NET Framework Web应用。

### 4. 配置YARP

接下来要做的就是配置YARP（Yet Another Reverse Proxy）。这段代码将决定请求是发送到新的ASP.NET Core Web应用（对于已迁移的路由）还是旧的.NET Framework Web应用（对于尚未迁移的路由）。

以下是一个YARP路由配置的示例：

```csharp
var webRoutes = new List<RouteConfig>
{
    // 令牌路由
    new()
    {
        RouteId = "tokenServePath",
        ClusterId = tokenClusterId,
        Match = new RouteMatch
        {
            Path = "/token/{**catch-all}",
        },
    },

    // WebUI应用路由
    new RouteConfig
    {
        RouteId = "webUIServePath",
        ClusterId = webUiClusterId,
        Match = new RouteMatch
        {
            Path = "/api/v2/{**catch-all}",
        },
    },

    // Web应用路由
    new RouteConfig
    {
        RouteId = "webAppServePath",
        ClusterId = webAppClusterId,
        Match = new RouteMatch
        {
            Path = "/api/{**catch-all}",
        },
    },

    // Angular路由
    new RouteConfig
    {
        RouteId = "angularUIServePath",
        ClusterId = angularClusterId,
        Match = new RouteMatch
        {
            Path = "{**catch-all}",
        },
    }
};
```

### 5. 使用升级助手升级组件

创建并行项目后，选择需要迁移的项目并右键单击 | 升级。

![项目迁移上下文菜单](https://github.com/SSWConsulting/SSW.Rules.Content/assets/3699937/3303daaf-0dea-4b34-9f59-53fd55acf2ef?wt.mc_id=MVP_324329)

升级助手将显示摘要视图，并检测项目已链接到Yarp代理。您还可以以饼图形式查看从.NET Framework到.NET的端点迁移进度。

![升级助手摘要页](https://github.com/SSWConsulting/SSW.Rules.Content/assets/3699937/8564c9a7-b3a7-4b40-b002-be9c6fabcb16)

在这里，您可以通过“端点资源管理器”浏览端点，它还会指示哪些端点已经迁移，哪些尚未迁移。链图标表示该端点已迁移，并在旧项目和Yarp代理项目之间的控制器之间建立了链接。

![链图标](https://github.com/SSWConsulting/SSW.Rules.Content/assets/3699937/d89d7150-e0b3-4947-abeb-0e1f865ab6f8)

![端点资源管理器显示旧.NET Framework项目和新.NET Core项目之间的端点](https://github.com/SSWConsulting/SSW.Rules.Content/assets/3699937/15377711-45a9-41dd-88b5-c555b64e6a87)

使用“升级”功能应用自动代码转换并加快迁移过程。在最佳情况下，控制器已完全移植，不需要任何手动工作。在大多数情况下，您需要查看控制器并更新升级助手无法自动转换的任何自定义代码。

![升级助手升级控制器进度](https://github.com/SSWConsulting/SSW.Rules.Content/assets/3699937/51fab5b1-eed3-48b9-8bd3-5a611e568b20)