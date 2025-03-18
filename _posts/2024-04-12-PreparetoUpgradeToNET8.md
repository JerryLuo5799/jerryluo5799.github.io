---
layout: post
title:  ".NET Framework升级.NET8之二: 前期准备工作"
date:   2024-04-12 21:42:55 +0800--
categories: [.NET]
tags: [NET Framework升级.NET8]  
---

### 一、全面审查应用程序架构
**迁移不仅仅是框架升级，更是清理技术债务的绝佳机会！**  
如果应用程序存在架构混乱或技术债务（如未修复的代码缺陷、临时解决方案），迁移难度会指数级增长。以下是关键准备工作：

1. **架构审计**  
   • 检查分层架构（如典型的N层架构）是否清晰。  
   • 确保各层之间没有**非法依赖**（例如`System.Web`出现在业务逻辑层或数据层）。  
   • 示例问题：Web层直接操作数据库，导致耦合度过高。

2. **创建技术债务清单**  
   • 将发现的问题记录为产品待办项（PBIs），优先修复后再迁移。

### 二、依赖项分析：手动检查第三方库
**依赖项兼容性是迁移成功的关键！**  

1. **内部依赖检查**  
   • 使用工具（如Visual Studio的依赖关系图）分析项目引用。  
   • 移除跨层的不合理依赖（如业务层直接依赖UI组件）。

2. **第三方依赖处理**  
   • 验证所有第三方库（如财务系统、报表工具）是否支持.NET 8。  
   • 若官方未提供兼容版本，需评估替代方案或联系供应商支持。  
   • 示例：某日志库仅支持.NET Framework，需替换为`Serilog`或`NLog`。

### 三、基础设施验证
**环境准备是基础中的基础！**  
1. **服务器环境检查**  
   • 若应用程序部署在本地服务器，需确认已安装.NET 8运行时。  
   • 使用命令`dotnet --list-runtimes`查看已安装的运行时版本。

2. **容器化部署（可选）**  
   • 考虑将应用迁移到Docker容器，简化环境依赖管理。

### 四、制定迁移策略：自下而上逐步推进
**错误的迁移顺序会导致重复劳动！**  
1. **分层迁移优先级**  
   • 对于N层架构：**从底层开始（如数据层→业务层→UI层）**。  
   • 对于洋葱架构：**从核心层向外围层迁移**。  
   • *错误示例*：先迁移UI层会导致需要反复修改底层代码。

2. **处理破坏性变更（Breaking Changes）**  
   • 使用.NET Portability Analyzer工具扫描不兼容API。  
   • 常见问题：`HttpContext.Current`在.NET 8中已废弃，需替换为依赖注入方式。

### 五、升级项目文件（csproj）
**SDK风格项目文件是现代化开发的基础！**  
1. **使用工具自动转换**  
   • 安装转换工具：  
     ```bash
     dotnet tool install -g try-convert
     ```
   • 执行转换命令（保留现有目标框架）：  
     ```bash
     try-convert --keep-current-tfms
     ```

2. **新旧项目文件对比**  
   • 旧版csproj：手动维护复杂XML结构，包含大量`<Reference>`标签。  
   • 新版SDK风格：自动引用依赖，文件简洁。  
     ```xml
     <Project Sdk="Microsoft.NET.Sdk">
       <PropertyGroup>
         <TargetFramework>net8.0</TargetFramework>
       </PropertyGroup>
     </Project>
     ```

### 六、多目标框架（TFM）支持
**逐步迁移的利器：同时兼容新旧框架！**  
1. **配置多目标框架**  
   • 修改csproj文件以支持多个框架：  
     ```xml
     <TargetFrameworks>net48;net8.0</TargetFrameworks>
     ```
   • 允许在迁移过程中逐步替换代码，无需一次性重写。

2. **条件编译策略**  
   • 使用`#if NET48`和`#if NET8_0`指令处理框架差异代码。  
   • 示例：  
     ```csharp
     #if NET48
     var connection = new SqlConnection("LegacyConnectionString");
     #else
     var connection = new SqlConnection(config.GetConnectionString("Default"));
     #endif
     ```

### 七、特别提示：Web应用程序迁移
**Web项目需单独处理！**  
1. 对于ASP.NET Web Forms或MVC项目：  
   • 先迁移类库项目，Web项目留到最后阶段。  
   • 使用`.NET Upgrade Assistant`工具自动化部分迁移步骤。

2. 前端资源处理：  
   • 旧版Web项目可能依赖`Bundling`或`WebGrease`，需替换为现代前端工具链（如Webpack）。