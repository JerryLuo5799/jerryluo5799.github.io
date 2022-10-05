---
layout: post
title:  "使用winget安装.NET环境"
date:   2022-09-13 21:42:55 +0800--
categories: [Other]
tags: [.NET, .NET Core, winget]  
---

### 1. 前言
目前在Windows上可以直接使用 **winget** 命令来安装.NET的SDK和运行时了。

### 2. 可安装版本列表
| .NET版本 | package-id |
|---|---|
| 3.1 | dotnet-sdk-3_1,dotnet-runtime-3_1,dotnet-desktop-3_1,aspnetcore-3_1 |
| 5.0 | dotnet-sdk-5,dotnet-runtime-5,dotnet-desktop-5,aspnetcore-5 |
| 6.0 | dotnet-sdk-6,dotnet-runtime-6,dotnet-desktop-6,aspnetcore-6 |
| 7.0 (Preview) | dotnet-sdk-preview,dotnet-runtime-preview, dotnet-desktop-preview,aspnetcore-preview |

### 3. 如何使用

#### 3.1 搜索.NET可用版本
命令行:
```
winget search Microsoft.DotNet
```
![.NET Search Result Screenshot](https://devblogs.microsoft.com/dotnet/wp-content/uploads/sites/10/2022/09/Search-Dotnet-New.png)

#### 3.2 安装.NET
命令行:
```
winget install [package-id]
```

例如, 如果我们想要安装.NET 6的SDK, 则可以执行以下命令:
```
winget install Microsoft.DotNet.SDK.6
```
![.NET Install Screenshot](https://devblogs.microsoft.com/dotnet/wp-content/uploads/sites/10/2022/09/Install-Dotnet-New.png)

如需要安装指定体系结构（x86、x64 和 Arm64），可以使用以下命令：
```
winget install --architecture x64 Microsoft.DotNet.SDK.6
```
#### 3.3 卸载.NET
命令行:
```
winget uninstall Microsoft.DotNet.SDK.6
```

### 4. 更多
更多关于如何安装和使用winget, 请查阅 [如何安装和使用 winget 工具。](https://docs.microsoft.com/windows/package-manager/winget/)
