---
layout: post
title:  ".NET MAUI简介"
date:   2022-03-15 21:42:55 +0800--
categories: [.NET]
tags: [.NET, .NET 6, MAUI]  
---

### 1. 前言
说起跨平台开发, 大家第一时间想到的可能是Flutter, Ionic, Ionic, React native等, 伴随着.NET 6的发布, 微软也加入了这个战场, 推出了.NET MAUI。

### 2. 什么是.NET MAUI

.NET MAUI（.NET Multi-platform App UI）是⼀个跨平台框架，⽤于使⽤ C# 和 XAML 创建本机移动和桌⾯应⽤。.NET MAUI是开源的，是 Xamarin.Forms 的演变，从移动⽅案扩展到桌⾯⽅案，UI 控件从基础重新⽣成，以提⾼性能和扩展性。使⽤.NET MAUI，可以使⽤单个项⽬创建多平台应⽤，但如有必要，可以添加特定于平台的源代码和资源。 [开源地址](https://github.com/dotnet/maui)
![.NET MAUI](https://docs.microsoft.com/zh-cn/dotnet/maui/media/what-is-maui/maui.png)

### 3.⼯作原理

NET MAUI将 Android、iOS、macOS 和 Windows API 统⼀到单个 API 中，该 API 允许随处运⾏⼀次开发⼈员体验，同时提供对每个本机平台的各个⽅⾯的深⼊访问。.NET 6 提供了⼀系列特定于平台的框架⽤于创建应⽤：.NET for Android、.NET for iOS、.NET for macOS 和Windows UI 3 (WinUI 3) 库。 这些框架都有权访问 BCL 中相同的 .NET 6 基类 (库) 。 此库从代码中抽象出基础平台的详细信息。 BCL 依赖于 .NET 运⾏时为代码提供执⾏环境。 对于 Android、iOS 和 macOS，环境由Mono（.NET 运⾏时的实现）实现。 在Windows，Win32 提供执⾏环境。
![.NET MAUI](https://docs.microsoft.com/zh-cn/dotnet/maui/media/what-is-maui/architecture.png)

.NET MAUI PC 或 Mac 上编写应⽤，并编译为本机应⽤包：
- 使⽤ .NET MAUI 构建的 Android 应⽤从 C# 编译为中间语⾔ (IL) ，然后在应⽤启动时 (JIT) 实时编译为本机程序集。
- 使⽤ .NET MAUI 构建的 iOS 应⽤完全提前 (AOT) C# 编译为本机 ARM 程序集代码。
- 使⽤ .NET MAUI 构建的 macOS 应⽤使⽤ Apple 提供的解决⽅案，该解决⽅案将使⽤ UIKit 构建的 iOS应⽤引⼊桌⾯，并按需要通过其他 AppKit 和平台 API 来增强它。
- Windows 使⽤ .NET MAUI 构建的 Windows UI 3 (WinUI 3) 库创建⾯向 Windows 桌⾯的本机应⽤。

### 项目示例
.NET MAUI应用通常由一个项目组成，该项目可以面向 Android、iOS、macOS 和 Windows。 这具有以下优势：

- 一个面向多个平台和设备的项目。
- 用于管理字体和图像等资源的一个位置。
- 多目标用于组织特定于平台的代码。

![.NET MAUI](https://docs.microsoft.com/zh-cn/dotnet/maui/media/what-is-maui/single-project.png)


### Roadmap
现阶段的 MAUI 最新版本是 Preview 14，2022 年 4 ⽉会有 RC，在 2022年 Q2 会有正式版本。
具体可查看 [.NET MAUI Roadmap](https://github.com/dotnet/maui/wiki/Roadmap)