---
layout: post
title:  ".NET 开发者必读的 MCP 协议实战指南"
date:   2025-06-13 12:30:00 +0800--
categories: [AI]
tags: [MCP]
---

## 前言

在构建 AI 应用时，我们常面临一个核心瓶颈：**LLM（大语言模型）无法直接访问业务数据。** 无论是公司的私有数据库、复杂的业务 API，还是本地的文件系统，AI 默认是“看不见、摸不着”的。

**Model Context Protocol (MCP)** 的出现彻底改变了游戏规则。它定义了一套标准，让 AI 能够以统一的方式接入外部工具和数据。

## 什么是 MCP？

MCP 是由 Anthropic 提出的开放协议，它在 AI 客户端（Host）与数据源（Server）之间架起了一座标准化的桥梁。

![MCP架构](/assets/imgs/MCP-Architecture.png)

### 核心架构

* **Host（宿主）**：如 IDE (VS Code, JetBrains)、Claude Desktop。
* **Client（客户端）**：负责发起协议请求的模块。
* **Server（服务端）**：由我们开发的 .NET 程序，负责执行具体的业务逻辑。

### 三大支柱概念

1. **Tools（工具）**：AI 可以**执行**的操作（如：查询订单、发送邮件）。
2. **Resources（资源）**：AI 可以**读取**的数据（如：日志文件、配置说明）。
3. **Prompts（提示词）**：预设的对话**模板**（如：代码审查指令）。

## 实战：使用 .NET 构建播客查询助手

我们将创建一个 .NET 服务，让 AI 能够通过标签查询播客剧集。

### 1. 环境准备

安装最新的 MCP .NET SDK（预览版）：

```bash
dotnet new console -n McpPodcastServer
dotnet add package Microsoft.ModelContextProtocol.Sdk --prerelease

```

### 2. 定义业务工具

在 MCP 中，工具是通过 C# 特性（Attributes）来定义的。**注意：`Description` 是给 AI 看的，直接决定了模型能否正确调用。**

```csharp
using Microsoft.ModelContextProtocol.Sdk;

public class PodcastTools
{
    [McpTool(Name = "search_episodes", Description = "根据技术标签（如 .NET, Docker）搜索播客剧集")]
    public async Task<List<EpisodeDto>> SearchAsync(
        [McpParameter(Description = "技术标签")] string tag)
    {
        // 模拟数据库查询逻辑
        var episodes = await DataStore.GetByTagAsync(tag);
        return episodes;
    }
}

public record EpisodeDto(string Title, string Date, string[] Tags);

```

### 3. 启动协议传输层

MCP 最常用的通信方式是 **Standard IO (Stdio)**，即 Host 启动 Server 进程后，通过输入输出流交换 JSON 数据。

```csharp
using Microsoft.Extensions.DependencyInjection;
using Microsoft.ModelContextProtocol.Sdk.Server;

var builder = McpServer.CreateBuilder();

// 注册 MCP 服务并配置 Stdio 传输
builder.Services.AddMcpServer()
       .WithStdioTransport() 
       .WithToolsFromAssembly(); // 自动扫描 [McpTool]

var app = builder.Build();
await app.RunAsync();

```

## 深度思考：如何写出“高质量”的 MCP 工具？

### 1. 描述性重于逻辑

在传统 Web API 中，接口文档是给人看的；而在 MCP 中，接口描述是给 AI 路由逻辑看的。

* **反例**：`Description = "获取数据"`
* **正例**：`Description = "从播客库中检索包含特定技术栈标签的剧集标题和发布日期"`

### 2. 采样（Sampling）：AI 调用的闭环

MCP 支持**反向调用**。你的 .NET 代码在执行工具时，可以请求 Host 端的 LLM 帮你处理数据。例如：获取到原始剧集描述后，请求 AI 总结为一个搞笑的段子再返回给用户。

```csharp
var aiResponse = await context.CreateSampleAsync(new McpSamplingRequest {
    Prompt = "请将以下信息总结成一段幽默的推文...",
    MaxTokens = 100
});

```

### 3. 调试技巧：Debugger.Launch

由于 Stdio 模式下 Server 是由 IDE 启动的，普通 F5 调试无法介入。你可以在代码开头加入：

```csharp
#if DEBUG
System.Diagnostics.Debugger.Launch();
#endif

```

当 AI 触发调用时，系统会提示你附加到 Visual Studio 或 Rider 进行断点调试。

## 安全与避坑指南

> [!IMPORTANT]
> **不要在代码中使用 `Console.WriteLine`！**
> 在 Stdio 模式下，标准输出流用于传输协议 JSON 数据。如果你在代码中随意打印日志到控制台，会导致 JSON 解析异常，从而中断 AI 的通信。请使用文件日志或标准错误流（stderr）。

* **参数校验**：LLM 可能会生成错误的参数，永远不要直接将 AI 传入的字符串拼接到 SQL 中。
* **原子化设计**：遵循“单一职责原则”。将复杂的业务拆分为多个小工具，让 AI 根据需要自行编排调用顺序。

## 总结

MCP 协议为 .NET 开发者打开了一扇通往 AI 原生应用的大门。它将我们的业务逻辑转化为 AI 的“技能点”，让 AI 真正从“对话者”变成了“执行者”。

**参考链接：**

* [Model Context Protocol 官网](https://modelcontextprotocol.io)
* [Microsoft MCP SDK for .NET (GitHub)](https://github.com/modelcontextprotocol/csharp-sdk?wt.mc_id=MVP_324329)
