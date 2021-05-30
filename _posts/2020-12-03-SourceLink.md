---
layout: post
title:  "使用Source Link提高调试效率"
date:   2020-12-03 21:42:55 +0800--
categories: [.NET Core]
tags: [.NET Core, Source Link, 调试]  
---

### 前言
您有多少次在调试器中跟踪错误、单步执行代码、查看哪些局部变量值发生了变化、当您碰壁时 — 值不是您所期望的，并且您无法进入该方法产生它是因为它来自库或 .NET 框架本身？或者您设置了一个条件断点，等待检查某个值是如何设置的，然后注意到一个调用堆栈大部分是灰色的，而不是让您看到调用堆栈中先前发生的事情？如果您可以轻松进入、设置断点并在 NuGet 依赖项或框架本身上使用调试器的所有功能，那不是很好吗？

2020 年的 .NET 开发实践与十年前相比有很大不同，而且在很多方面都更好。最大的变化是.NET平台是开源的，并在GitHub上维护。我们日常使用的许多 NuGet 库也在 GitHub 上进行维护。这意味着我真正想在我的调试器中看到的源只是一个 HTTPS GET。我们可以拥有这个非常高效的生态系统，在那里我们可以一直使用源代码调试所有依赖项。那样就好了！事实上，由 Cameron Taggart 发起的 Source Link 项目意识到了这一点，并构建了一种体验。让我告诉你吧。

使用启用了源链接的库，调试器可以在您介入时下载底层源文件，并且您可以像设置任何其他源一样设置断点/跟踪点。启用源链接的调试可以更轻松地了解从代码到运行时的完整代码流。Source Link与语言无关，因此您可以从任何.NET语言和某些本机库中受益。

### 调试框架
让我们看一个例子。有时您想进入框架以查看正在发生的事情，特别是如果发生了您没有预料到的事情。使用 Source Link，您可以像处理自己的代码一样进入框架方法，检查所有变量并设置断点。
如果您在没有 Source Link 的情况下尝试过，这就是您在按 F11 进入之前和之后会看到的内容。
![图片](https://devblogs.microsoft.com/dotnet/wp-content/uploads/sites/10/2020/11/debug-framework-before.png)

![图片](https://devblogs.microsoft.com/dotnet/wp-content/uploads/sites/10/2020/11/debug-framework-nosourcelink-after.png)

调试器不会进入，因为它没有符号或源。一旦我们配置了Source Link，当我们介入时，我们得到了不同的结果：**Console.WriteLine**

![图片](https://devblogs.microsoft.com/dotnet/wp-content/uploads/sites/10/2020/11/debug-framework-sourcelink-after.png)

可以看到 Visual Studio 已经下载了匹配的源代码并步入了该方法。如果您查看该Autos窗口，它会显示传入的局部变量。您可以按自己的意愿进入、跳过和退出框架代码。

### 调试依赖
通常，您尝试解决的问题是依赖关系。如果您也可以进入依赖项的源，那不是很好吗？如果依赖在构建过程中添加了 Source Link 信息，则可以！这是一个带有. 因为是用 Source Link 信息构建的，你可以进入它的代码：**Newtonsoft.JsonNewtonsoft.Json**

![图片](https://devblogs.microsoft.com/dotnet/wp-content/uploads/sites/10/2020/11/newtonsoft-json-before.png)

![图片](https://devblogs.microsoft.com/dotnet/wp-content/uploads/sites/10/2020/11/newtonsoft-json-after.png)

当我介入时，调试器跳过了几个标记为DebuggerStepThrough并在CreateDefault方法中的下一条语句处停止的方法。由于源来自互联网（在本例中为 GitHub），系统会提示您允许它，无论是单个文件还是所有文件。

### 异常
Source Link 帮助您处理来自框架或依赖项的异常。您看到此消息多少次，而您真正想要的是检查变量？

![图片](https://devblogs.microsoft.com/dotnet/wp-content/uploads/sites/10/2020/11/unhandled-exception-before.png)

![图片](https://devblogs.microsoft.com/dotnet/wp-content/uploads/sites/10/2020/11/unhandled-exception-after-thrown.png)

![图片](https://devblogs.microsoft.com/dotnet/wp-content/uploads/sites/10/2020/11/unhandled-exception-after.png)

使用Source Link，调试器将带您到引发异常的地方，然后您可以浏览调用堆栈并进行调查。

### 如何启用
由于 Source Link 从 Internet 下载源文件，因此默认情况下不启用。启用方法如下：
1. 转到“工具” >“选项” >“调试” >“符号”并确保选中“NuGet.org 符号服务器”选项。为符号缓存指定一个目录是避免再次下载相同符号的好主意。
   
![图片](https://devblogs.microsoft.com/dotnet/wp-content/uploads/sites/10/2020/11/visual-studio-step-1.png)

如果您想进入 .NET 框架代码，您还需要选中“Microsoft Symbol Servers”选项。

2. 在**工具>选项>调试>常规**中禁用，因为我们希望调试器尝试为解决方案之外的代码定位符号。 验证是否已选中（默认情况下）。如果您想进入 .NET Framework 代码，您还需要检查. 这对于 .NET Core 不是必需的。
   
![图片](https://devblogs.microsoft.com/dotnet/wp-content/uploads/sites/10/2020/11/visual-studio-step-2.png)

### Visual Studio Code中启用
Visual Studio Code 在以下文件中为每个项目配置了调试器设置：**launch.json**

```Json
"justMyCode": false,
"symbolOptions": {
    "searchMicrosoftSymbolServer": true,
    "searchNuGetOrgSymbolServer": true
},
"suppressJITOptimizations": true,
"env": {
    "COMPlus_ZapDisable": "1",
    "COMPlus_ReadyToRun": "0"
}
```