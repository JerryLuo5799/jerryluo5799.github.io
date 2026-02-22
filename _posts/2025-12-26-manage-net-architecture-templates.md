---
layout: post
title:  "如何开发和管理.NET架构模板 - .NET Conf China 2025"
date:   2025-12-26 12:30:00 +0800--
categories: [Activities]
tags: [Template]
---

<iframe src="//player.bilibili.com/player.html?isOutside=true&aid=115784936464243&bvid=BV1PUBeBAE5q&cid=34991443944&p=1" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true" class="bilibili"></iframe>

链接地址: [https://www.bilibili.com/video/BV1PUBeBAE5q/](https://www.bilibili.com/video/BV1PUBeBAE5q/)

### 前言

每当启动新项目时，你是否还在通过 **Ctrl+C** 和 **Ctrl+V** 复制旧项目，然后花大半天时间去删除无关代码和修改命名空间？这种原始做法不仅低效，还会导致团队项目结构各异，极大增加了后期的维护成本 。今天，我们将深入探讨如何通过创建标准化的 **.NET 软件架构模板** 以及 **自定义 CLI 工具**，将重复性劳动转化为自动化的生产力闭环 。

### 一、 核心概念：什么是“好的”架构模板？

一个成熟的架构模板不仅仅是文件夹的堆砌，它是团队技术规范的物理载体 。

#### 1. 架构选型：没有最好，只有最适合

模板应内置分层架构规范 ，常见的包括：

* **清洁架构 (Clean Architecture)**：关注业务逻辑核心，实现依赖倒置 。
* **三层架构 (Three-Tier)**：传统且稳健的逻辑划分 。
* **领域驱动 (DDD) 架构**：应对复杂业务建模的利器 。
* **垂直切片架构 (Vertical Slice)**：按功能维度切分，减少层间耦合 。

#### 2. 最佳实践集成

一个生产级模板必须集成以下通用能力：

* **基础设施**：内置认证授权、日志记录、异常处理、健康检查等模块 。
* **内置 Demo**：包含标准示例代码，让新成员快速上手并理解规范 。
* **开箱即用**：最小化配置即可运行，新成员无需排查环境问题 。

### 二、 实战演示：三步打造你的 .NET 模板引擎

#### 第一步：打磨“模范项目”

首先，精心编写一个标准项目（例如基于 Clean Architecture）。以 `SSW.CleanArchitecture` 为例，其核心结构应清晰划分 `Application`、`Domain`、`Infrastructure` 等层级 。

#### 第二步：配置 `template.json`

在项目 `.template.config` 目录下创建 `template.json`。这是定义生成逻辑的核心文件 ：

| 字段名称 | 类型 | 必填 | 作用说明 |
| --- | --- | --- | --- |
| `shortName` | string | 是 | 使用 `dotnet new [shortName]` 调用模板的简短命令 。|
| `sourceName` | string | 否 | <br>**关键占位符**：生成项目时，会自动将此占位字符串替换为用户指定的项目名 。|
| `identity` | string | 是 | 模板的唯一标识符，用于内部区分不同模板 。|
| `sources.exclude` | array | 否 | 排除不需要打包的文件，如 `bin/obj`、`.git` 等 。|

#### 第三步：打包与发布

1. **创建模板项目文件 (.csproj)**：指定项目类型为 `Template` 并设置包标识 。
2. **打包发布**：执行 `dotnet pack` 将其封装为 NuGet 包，并推送至 NuGet 源 。
3. **安装测试**：
```bash
# 从本地路径安装测试
dotnet new install <PATH_TO_NUPKG_FILE>
# 基于模板创建新项目
dotnet new ssw-ca --name NETConf2025.Demo

```

### 三、 提效进阶：.NET CLI vs 自定义 CLI

虽然 `dotnet new` 解决了起点的标准化，但在开发过程中频繁创建业务模块（如 CRUD 接口）依然耗时。这时就需要 **自定义 CLI** 。

| 维度 | .NET CLI (`dotnet` 命令) | 自定义 CLI (如 `my-cli`) |
| --- | --- | --- |
| **定位** | 平台级工具，.NET 开发的基础标准 。| 业务级工具，针对特定架构封装 。|
| **主要目的** | 通用操作（构建、运行）和项目模板管理 。| 实现团队规范、自动化特定业务流程 。|
| **典型场景** | <br>`dotnet new`, `dotnet build` 。| <br>`my-cli add-module -n Product` 。|

#### 开发自定义 CLI（C# 示例）

推荐使用 `System.CommandLine` 库实现 。它可以将一系列底层操作封装成一个有业务语义的命令。

```csharp
using System.CommandLine;

var rootCommand = new RootCommand("团队提效 CLI 工具");
var nameArgument = new Argument<string>("name", "项目名称");
var newCommand = new Command("new", "创建并自动配置新项目") { nameArgument };

newCommand.SetHandler(async (name) => {
    Console.WriteLine($"🚀 正在创建项目: {name}...");
    // 逻辑：1. 调用 dotnet new 生成基础代码
    // 2. 自动配置团队内部 NuGet 依赖
    // 3. 执行 dotnet build 验证环境成功
}, nameArgument);

rootCommand.AddCommand(newCommand);
await rootCommand.InvokeAsync(args);

```

### 四、 深度思考：像管理产品一样管理模板

如果模板不进行持续演进，它很快就会过时 。你需要引入工程化的管理机制：

* **语义化版本控制 (SemVer)**：主版本变更代表不兼容架构调整，次版本代表新增功能，修订号代表 Bug 修复 。
* **发布管理与 CI/CD**：使用 GitHub Actions 等流水线自动化打包，并配合 Git Tags 标记历史版本 。
* **分支策略**：建议使用 `main` 分支保持稳定发布，`dev` 分支进行日常演进 。
* **持续迭代**：建立反馈闭环，基于团队实际使用中的痛点不断完善模板 。

### 总结

架构模板的价值在于**秒级创建项目、保证质量统一、降低上手门槛以及固化团队智慧** 。在 AI 辅助开发的趋势下，标准化的开发范式更是大规模提效的基石。

**官方参考链接：**

* [Custom templates for dotnet new ](https://learn.microsoft.com/dotnet/core/tools/custom-templates?wt.mc_id=MVP_324329)
* [SSW.CleanArchitecture Open Source](https://github.com/SSWConsulting/SSW.CleanArchitecture?wt.mc_id=MVP_324329)
