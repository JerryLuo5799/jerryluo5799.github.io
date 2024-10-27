---
layout: post
title:  "如何在C#代码中执行JavaScript代码"
date:   2024-08-02 21:42:55 +0800--
categories: [.NET]
tags: [C#, JavaScript]  
---

### 1. 前言

选择在C#中执行JavaScript代码的库取决于你的具体需求、性能考虑、以及与Node.js生态系统的集成程度。主要有Jint、Edge.js、ClearScript三个选项。

### 2. Jint

Jint 是一个用于 .NET 的 JavaScript 解释器。它允许你在 .NET 应用程序中嵌入和执行 JavaScript 代码。Jint 是基于 ECMAScript 5.1 标准实现的，并且支持大部分现代 JavaScript 特性。

#### 2.1 优点

- 纯C#实现，不需要额外的本地依赖，部署和使用相对简单。
- 适合执行简单的JavaScript代码片段。

#### 2.2 缺点

- 性能相对较低，不适合执行复杂或性能敏感的JavaScript代码。
- 不支持最新的ECMAScript特性。

#### 2.3 适用场景

- 需要在.NET环境中快速轻松地执行少量JavaScript代码。
- 不需要与Node.js生态系统集成。

#### 2.4 示例

安装Jint库：

```csharp
Install-Package Jint
```

创建一个C#方法，该方法使用Jint执行JavaScript代码，并将C#变量作为参数传递给JavaScript函数：

```csharp
using Jint;

public class JavaScriptInterop
{
    public void ExecuteJavaScriptFunctionWithCSharpParameter()
    {
        // 定义JavaScript函数，这里的函数只是一个示例，它接受一个参数并返回一个字符串
        string jsFunction = @"
            function sayHello(name) {
                return 'Hello, ' + name;
            }
        ";

        // C#中的参数
        string name = "World";

        // 创建Jint引擎
        var engine = new Engine()
                        .Execute(jsFunction) // 执行定义的JavaScript函数
                        .SetValue("name", name); // 将C#变量设置为JavaScript环境中的变量

        // 调用JavaScript函数，并传递C#变量
        var result = engine.Execute("sayHello(name)").GetCompletionValue().AsString();

        // 输出结果
        Console.WriteLine(result); // 应该输出: Hello, World
    }
}
```

### 3. Edge.js

如果你的环境中已经安装了Node.js，你可以使用Edge.js来在C#中执行JavaScript代码。Edge.js允许你在.NET应用程序中运行Node.js代码，并实现两者之间的互操作。

#### 3.1 优点

- 允许.NET和Node.js代码之间的直接互操作。
- 支持使用Node.js的全部功能，包括异步操作和访问Node.js生态系统。

#### 3.2 缺点

- 需要Node.js环境，增加了部署的复杂性。
- 性能开销较大，特别是在.NET和Node.js之间频繁交互时。

#### 3.3 适用场景

- 需要在.NET应用程序中利用Node.js的强大功能，如访问大量的Node.js模块。
- 应用程序已经在使用Node.js，或者对Node.js有特定的依赖。

#### 3.4 示例

首先需要安装Edge.js：

```csharp
Install-Package Edge.js
```

然后，你可以这样使用它：

```csharp
using EdgeJs;

public async Task RunJavaScript()
{
    var func = Edge.Func(@"
        return function (data, callback) {
            callback(null, 'Hello, ' + data);
        }
    ");

    Console.WriteLine(await func(".NET"));
}
```

### 4. ClearScript

ClearScript是另一个库，它允许.NET应用程序与JavaScript引擎（如V8）进行互操作。使用ClearScript，你可以在C#中执行JavaScript代码，并与JavaScript对象进行交互。

#### 4.1 优点

- 直接绑定到V8（Chrome的JavaScript引擎），提供高性能的JavaScript执行能力。
- 支持最新的JavaScript特性。
- 可以访问和操作.NET对象，实现.NET和JavaScript之间的深度集成。

#### 4.2 缺点

- 需要本地依赖（如V8），可能会增加部署和配置的复杂性。
- 在Windows之外的平台上可能需要额外的工作来编译或配置V8引擎。

#### 4.3 适用场景

- 需要执行复杂的JavaScript代码，或者对性能有较高要求。
- 需要在.NET和JavaScript之间进行深度集成和互操作。

#### 4.4 示例

首先需要安装ClearScript：

```csharp
Install-Package Microsoft.ClearScript
```

然后，你可以这样使用它：

```csharp
using Microsoft.ClearScript.V8;

public void RunJavaScript()
{
    using (var engine = new V8ScriptEngine())
    {
        engine.Execute("function add(a, b) { return a + b; }");
        var result = engine.Script.add(1, 2);
        Console.WriteLine(result); // 输出: 3
    }
}

```

### 5. 总结

- 如果你需要简单地执行一些JavaScript代码，而且不想引入复杂的依赖，Jint可能是最好的选择。
- 如果你的项目已经依赖于Node.js，或者你需要访问Node.js的特定功能，Edge.js可能更适合。
- 如果你需要高性能的JavaScript执行，或者需要使用最新的JavaScript特性，并且不介意处理本地依赖，ClearScript可能是最佳选择。
最终的选择应基于你的具体需求、项目的技术栈、以及你愿意接受的复杂度水平。