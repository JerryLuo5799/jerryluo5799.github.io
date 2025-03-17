---
layout: post
title:  ".NET Framework升级.NET8之一: 概述"
date:   2024-04-07 21:42:55 +0800--
categories: [.NET]
tags: [NET Framework升级.NET8]  
---

<iframe src="//player.bilibili.com/player.html?isOutside=true&aid=964927859&bvid=BV1uW4y1F7e4&cid=1372858369&p=1" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true" class="bilibili"></iframe>

链接地址: [https://www.bilibili.com/video/BV1uW4y1F7e4/](https://www.bilibili.com/video/BV1uW4y1F7e4/)
  
### 前言  
我们面临的是一个具有**十年历史**的旧项目，历经多次升级迭代，当前基于 **.NET Framework 4.7.2**。项目包含40+子模块，是一个单机应用，且依赖**大量第三方组件**。由于技术债务累积，升级到现代框架迫在眉睫。  

### 目标  
1. 在升级过程中保持现有功能稳定运行。  
2. 支持正常的功能迭代（不能因升级暂停开发）。  
3. 最终完全移除所有.NET Framework代码，过渡到**.NET 8**。  

## 二、升级前的准备工作  
### 1. 系统架构评估与技术债务梳理  
• **问题定位**：识别不再支持的组件（如`System.Web`）、过时的依赖项（如WCF服务）和第三方库兼容性问题。  
• **工具推荐**：使用[.NET Upgrade Assistant](https://github.com/dotnet/upgrade-assistant)扫描项目，生成兼容性报告。  

### 2. 多目标框架（Multi-Targeting）  
**核心思想**：让项目同时支持.NET Framework和.NET 8，逐步迁移代码。  

#### 操作步骤：  
1. **转换项目文件格式**：  
   • 使用官方工具[try-convert](https://github.com/dotnet/try-convert)将旧式`.csproj`转换为SDK风格。  
   ```bash  
   dotnet tool install -g try-convert  
   try-convert --keep-current-tfms  
   ```  
2. **修改目标框架**：  
   在项目文件中添加`<TargetFrameworks>net472;net8.0</TargetFrameworks>`，允许同时编译两个版本。  
3. **条件编译**：  
   使用`#if NET48`或`#if NET8_0`预处理指令隔离不兼容代码。  

```xml  
<!-- 示例：项目文件配置 -->  
<Project Sdk="Microsoft.NET.Sdk">  
  <PropertyGroup>  
    <TargetFrameworks>net472;net8.0</TargetFrameworks>  
  </PropertyGroup>  
</Project>  
```  

### 3. 并行增量项目（Side-by-Side）  
**适用场景**：Web应用（如ASP.NET WebForms迁移到ASP.NET Core）。  

#### 实现方案：  
1. **创建新ASP.NET Core项目**：  
   • 使用.NET Upgrade Assistant生成增量项目。  
2. **反向代理请求分发**：  
   • 通过[YARP](https://github.com/microsoft/reverse-proxy)将部分请求路由到新项目，旧项目继续处理其他请求。  
3. **逐步迁移功能模块**：  
   • 例如，先迁移用户认证模块，再迁移订单管理模块。  

## 三、升级中的常见问题与解决方案  
### 1. 依赖项兼容性问题  
• **场景1：依赖项有官方适配包**  
  例如，`System.Web`迁移到`Microsoft.AspNetCore.SystemWebAdapters`。  
• **场景2：API不兼容**  
  例如，`System.Configuration`替换为`Microsoft.Extensions.Configuration`，需重写配置读取逻辑。  
• **场景3：无替代组件**  
  评估是否更换第三方库（如用Dapper替代旧ORM）。  

### 2. Entity Framework迁移（EDMX到EF Core）  
**关键问题**：旧EDMX模型无法直接用于EF Core。  

#### 迁移步骤：  
1. **抽象接口**：  
   将`ObjectContext`和实体类抽象为接口（如`IDatabaseContext`）。  
2. **逆向工程生成EF Core模型**：  
   使用[EF Core Power Tools](https://github.com/ErikEJ/EFCorePowerTools)生成DbContext和实体类。  
3. **实现接口并替换**：  
   • 更新`DbContext.OnConfiguring`配置连接字符串。  
   • 用`DbSet<T>`替换`ObjectSet<T>`。  
4. **解决查询差异**：  
   • 禁用延迟加载（Lazy Loading），改用显式加载（`.Include()`）。  
   • 复杂查询可能需要重写为原生SQL或LINQ表达式。  

```csharp  
// 示例：EF Core查询调整  
var orders = context.Orders  
    .Include(o => o.Customer)  
    .Where(o => o.Date > DateTime.Now.AddDays(-7))  
    .ToList();  
```  

### 3. System.Web的替代方案  
• **问题**：ASP.NET Core中移除了`HttpContext.Current`。  
• **解决方案**：  
  1. 在.NET 8中使用`IHttpContextAccessor`注入当前上下文。  
  2. 在旧框架中封装静态访问器。  

```csharp  
// 示例：兼容性封装  
public interface IHttpContextProvider  
{  
    HttpContext Current { get; }  
}  

// .NET Framework实现  
public class LegacyHttpContextProvider : IHttpContextProvider  
{  
    public HttpContext Current => HttpContext.Current;  
}  

// .NET 8实现  
public class CoreHttpContextProvider : IHttpContextProvider  
{  
    private readonly IHttpContextAccessor _accessor;  
    public CoreHttpContextProvider(IHttpContextAccessor accessor) => _accessor = accessor;  
    public HttpContext Current => _accessor.HttpContext;  
}  
```  

## 四、工具与资源推荐  
1. **.NET Upgrade Assistant**：自动化升级脚手架生成。  
2. **YARP**：实现请求路由分发。  
3. **EF Core Power Tools**：逆向工程生成DbContext。  
4. **Microsoft.AspNetCore.SystemWebAdapters**：兼容旧ASP.NET API。  

## 五、总结  
升级大型旧项目需要**渐进式迭代**，而非一次性重写。关键点包括：  
• 通过多目标框架逐步替换代码。  
• 优先解决阻塞性依赖项（如EF和System.Web）。  
• 充分测试确保每一步的稳定性。  

**资源推荐**：  
• [EF Core迁移指南](https://learn.microsoft.com/en-us/ef/efcore-and-ef6/porting/port-edmx?wt.mc_id=MVP_324329)  
• [.NET官方升级文档](https://learn.microsoft.com/en-us/dotnet/core/porting/?wt.mc_id=MVP_324329)  