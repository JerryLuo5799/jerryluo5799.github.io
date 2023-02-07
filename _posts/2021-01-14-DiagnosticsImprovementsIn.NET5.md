---
layout: post
title:  ".NET 5 中的诊断工具(Diagnostics)改进"
date:   2021-01-14 21:42:55 +0800--
categories: [.NET]
tags: [.NET Core, .NET 5, Diagnostics]  
---

### 无需.NET SDK即可使用诊断工具
一直以来, .NET 诊断工具套件仅作为 .NET SDK 全局工具提供。虽然这提供了一种获取和更新工具的便捷方式，但这意味着在没有完整 SDK 的环境中很难获取它们。我们现在提供一种单文件分发机制，只需要在目标机器上提供运行时 (3.1+)。

最新版本的工具始终可通过遵循以下架构的链接获得：
```
https://aka.ms/<tool-name>/<platform-runtime-identifier>
```
例如，如果您在 x64 Ubuntu 上运行 .NET Core，则地址为:
```
https://aka.ms/dotnet-trace/linux-x64
```
支持的平台列表及其下载链接可以在每个工具的文档中找到，例如[dotnet-counters](https://docs.microsoft.com/zh-cn/dotnet/core/diagnostics/dotnet-counters)文档。所有可用工具和支持的平台运行时标识符的列表可以在[诊断存储库](https://github.com/dotnet/diagnostics/blob/main/documentation/single-file-tools.md)中找到。

### 在Windows中分析Linux的dump
调试托管代码需要托管对象和构造的特殊知识。数据访问组件 (DAC) 是运行时执行引擎的一个子集，它了解这些构造并且无需运行时即可访问这些托管对象。在 .NET Core 3.1.8+ 和 .NET 5+ 中，我们已经开始针对 Windows 编译 Linux DAC。现在可以使用 WinDBG、dotnet dump analyze、Visual Studio 2019 16.8在 Windows 上分析在 Linux 上收集的 .NET Core 进程转储。

有关如何收集 .NET 内存转储以及如何分析它们的更多信息，请参见Visual Studio 博客。

### 启动跟踪
.NET 诊断工具套件的工作方式是连接到运行时创建的诊断端口，然后使用诊断 IPC 协议通过该通道请求运行时输出信息。在 .NET Core 3.1 中，无法执行启动跟踪（通过 EventPipe；ETW 仍然可能），因为在工具连接到运行时之前发出的事件将丢失。在 .NET 5 中，现在可以将运行时配置为在启动期间自行挂起，直到工具连接（或让运行时连接到工具）。

Net5.0中的dotnet-counters和dotnet-trace现在可以启动从过程开始DOTNET流程，并收集诊断信息。例如，以下命令将启动mydotnetapp.exe并开始监视计数器。

```CSharp
dotnet counters monitor -- mydotnetapp.exe
```